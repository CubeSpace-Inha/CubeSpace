# 패드 ↔ PC USB 규격

이 문서로 답하는 것: 패드와 PC가 USB CDC로 주고받는 프레임의 바이트·주기·검사·복구 규칙은?

> 표시: `[권장안 U-nn]` 확정 전 값, 이 값으로 구현. `[미정 …]` 값 없음(번호는 [03-미정-결정](../00-공통/03-미정-결정.md)), 잠정이 있으면 함께 적음. `[실측 필요]` 시작값, 시험으로 고정.
> 기준일 2026-10-11. 묶기 재설계(10/8 확정)를 반영했다. 큐브·패드가 보내는 값과 바이트 형식은 규격 v1로 확정(10/11, 판 규칙은 7절).
> 공통: 모든 정수는 리틀 엔디언, 구조체에 빈 바이트 없음. 값은 한 곳에만 둔다: 펌웨어 `proto.h`(BLE는 `x1_proto.h`), PC `proto.ts`.
> 같은 묶음: [01-BLE](01-BLE.md) · [02-USB](02-USB.md) · [03-PC-내부](03-PC-내부.md) · [04-시험-벡터](04-시험-벡터.md)

## 1. 선로·머리·CRC

```
선로 : C0 | SLIP(머리 ‖ 본문 ‖ CRC) | C0     (안의 C0 → DB DC, DB → DB DD)
머리 : type@0 u8 | boot@1 u32 | seq@5 u32 | t_us@9 u64     (17 B)
CRC  : u32, 머리‖본문의 CRC-32(zlib과 같은 값)
```

| 항목 | 패드 | PC |
|---|---|---|
| CRC 함수 | `cs_crc32`(proto.cpp, 호스트 시험과 같은 코드) | `node:zlib`의 `crc32(p)` |
| SLIP 부호화 | 모든 프레임 앞뒤에 `C0` | `gateway`의 `slipEncode`·`SlipDecoder`(순수 코드, 앞뒤 `C0`. 순수 모듈은 `@serialport/*`를 못 쓴다) |
| `boot` | `pad_boot_id`: BLE 초기화 뒤 `esp_random()`, 0이면 다시 | `pc_boot`: 게이트웨이 시작마다 u32 난수 |
| `seq` | `pad_seq`: 1부터, 만든 프레임마다 +1(못 보내고 버린 것 포함). 부호화 전에 올림 | `pc_seq`: 같은 규칙 |
| `t_us` | 프레임을 만드는 순간 `esp_timer_get_time()` | `process.hrtime.bigint() / 1000n`(진단용) |

받는 쪽 처리:
- 복원 길이 21~1200 B 밖 → 버림. CRC 틀림 → 버림. 종류별 본문 길이(2절)가 틀림 → 버림, ACK 대상이면 ACK `code 1`.
- 모르는 `type` → 답 없이 버림. `C0 C0`(빈 프레임) → 무시. ROM 부팅 문구·패닉 출력은 한 덩어리로 떨어져 CRC에서 버려진다.
- 패드: 위 실패는 `rx_bad`++. PC: 버린 프레임은 기록 `bad`.
- 최대 복원 프레임: 패드→PC LOG 222 B, PC→패드 LED 1173 B(SLIP 최악 2348 B).
- USB에는 JSON·텍스트 줄을 싣지 않는다.

## 2. 메시지 목록

오프셋은 본문 시작 기준. 복원 B = 17 + 본문 + 4.

| type | 이름 | 방향 | 본문 필드@오프셋 | 본문 B | 복원 B | SLIP 최악 B | 보내는 때 |
|---|---|---|---|---:|---:|---:|---|
| 0x01 | HELLO | 패드→PC | `proto@0 u8(=1), reset_reason@1 u8, fw@2 u32, frame_us@6 u32, bad_map@10 u8[10]` | 20 | 41 | 84 | 부팅 직후 1회, PC HELLO를 받을 때마다 |
| 0x02 | STATUS | 패드→PC | `flags@0 u8, led_scale@1 u8, led_seq@2 u32, tx_drop@6 u16, rx_bad@8 u16, overrun@10 u16` | 12 | 33 | 68 | 500 ms마다, `flags`가 바뀌면 즉시, 스냅샷 끝 |
| 0x10 | OBS | 패드→PC | `frame_no@0 u32, led_seq@4 u32, slot@8 u8, st@9 u8, r0@10 u8, c0@11 u8, h@12 u8, w@13 u8, v@14 i16[h·w]` | 14 + 2hw(≤ 56) | 35~77 | 156 | 링크 있는 슬롯마다, 그 슬롯 읽기 직후, 광학 프레임마다 |
| 0x11 | EVT | 패드→PC | `slot@0 u8, cube_boot_id@1 u32, seq@5 u16, type@7 u8, v0@8 u8, v1@9 u16, t_pad32@11 u32`(BLE EVT 14 B 그대로) | 15 | 36 | 74 | 큐브 알림을 받은 뒤 첫 펌프 |
| 0x12 | LINK | 패드→PC | `slot@0 u8, up@1 u8, reason@2 u8, mac@3 u8[6]` | 9 | 30 | 62 | CFG 응답이 끝나면 `up=1`, 끊기면 `up=0`(BYE면 바로), 스냅샷 |
| 0x13 | CAND | 패드→PC | `mac@0 u8[6], rssi@6 i8` | 7 | 28 | 58 | 등록 스캔 중 서비스 UUID를 광고하는 미등록 주소. 주소마다 2000 ms에 1번 이하 |
| 0x7E | ACK | 패드→PC | `acked_type@0 u8, acked_seq@1 u32(pc_seq), code@5 u8` | 6 | 27 | 56 | REG·REG_SCAN·ICON 처리 뒤 |
| 0x7F | LOG | 패드→PC | `level@0 u8(1 오류·2 경고·3 정보), text@1 UTF-8 ≤ 200 B` | 1~201 | 22~222 | 446 | 자가진단·오류, 초당 10개 이하 |
| 0x81 | HELLO | PC→패드 | `proto@0 u8(=1)` | 1 | 22 | 46 | 포트를 연 뒤 패드 HELLO가 올 때까지 500 ms마다 |
| 0x82 | HB | PC→패드 | 없음 | 0 | 21 | 44 | 다른 PC 프레임 없이 1000 ms가 지나면 |
| 0x90 | LED | PC→패드 | `rgb@0 u8[1152]`(4절) | 1152 | 1173 | 2348 | 그림이 바뀔 때만, `frame_us`에 1번 이하. 답 없음 |
| 0x91 | LED_OFF | PC→패드 | 없음 | 0 | 21 | 44 | 서비스 정상 종료. 패드는 검정 → `LED_EN` LOW, 다음 LED에서 다시 켬 |
| 0xA0 | REG | PC→패드 | `mac[k]@6k u8[6]`, k = 슬롯 0~5(모두 0 = 빈칸) | 36 | 57 | 116 | HELLO 교환 직후, 등록 변경. 답 ACK |
| 0xA1 | REG_SCAN | PC→패드 | `on@0 u8` | 1 | 22 | 46 | 등록 시작·끝. 답 ACK. 패드는 30 s 뒤 스스로 끔 |
| 0xA2 | ICON | PC→패드 | `slot@0 u8, rgb@1 u8[192]`(밝기 반영 최종값) | 193 | 214 | 430 | 그 슬롯 LINK `up=1`마다, 아이콘이 바뀔 때. 답 ACK(BLE 쓰기 결과) |

- 큐브는 USB에서 슬롯 번호(0~5)로만 부른다. MAC은 REG·LINK·CAND에만 싣는다.
- MAC 바이트 순서: `AA:BB:CC:DD:EE:FF`의 `AA`가 `mac[0]`. NimBLE `ble_addr_t.val`은 반대라 패드 `proto.cpp` 한 곳에서 뒤집는다 [실측 필요].
- 없는 것: OTA 중계, 큐브 밝기 명령(밝기는 ICON 값).

## 3. 필드 세부

| 필드 | 값 |
|---|---|
| `HELLO.reset_reason` | `esp_reset_reason()`. 11 = USB 리셋 |
| `HELLO.fw` | git 커밋 해시 앞 8자리를 16진수로 읽은 u32(`1a2b3c4d…` → `0x1A2B3C4D`). 진단 표시만 |
| `HELLO.bad_map` | 부팅 자가진단 불량 센서. 비트 번호 n = 행 × 11 + 열(77개). 비트 n은 `bad_map[n / 8]`의 `1 << (n % 8)`(하위 비트부터) |
| `STATUS.flags` | b0 `adapter_present`(GPIO38 ADAPTER_N LOW), b1 `LED_EN`, b2 PC 살아 있음, b3 대기 패턴 중, b4 등록 스캔 중, b5~b7 0 |
| `STATUS.led_scale` | 전류 상한 감광 비율 0~255(255 = 감광 없음) |
| `STATUS.led_seq` | 마지막으로 켠 LED 프레임의 `pc_seq`, 없으면 0 |
| `tx_drop`·`rx_bad`·`overrun` | 부팅 뒤 누적, 넘치면 0으로 돎. `tx_drop` = 자리가 없어 버린 프레임 + 이벤트 큐 넘침, `overrun` = 프레임 루프가 `frame_us`를 넘긴 횟수 |
| `OBS.frame_no` | 광학 프레임 번호 k. 프레임 시작 패드 시각 = `frame_no × frame_us` |
| `OBS.led_seq` | 이 광학 프레임 동안 켜져 있던 LED의 `pc_seq`(대기 패턴·꺼짐이면 0) |
| `OBS.st` | b0~1 광학(0 NA, 1 SEARCHING, 2 ON_PAD, 3 OFF_PAD), b2 동기 유효, b3 접촉 놓임(마지막 CONTACT = 1), b4 SAT(창 안 원시값 ≥ `SAT_MV` 1550 mV [실측 필요], 빈 칸 제외), b5 이번 프레임 `sync_fault`, b6 이번 프레임 읽지 않음(광학 NA, `h = w = 0`, PC는 광학 상태를 이어 둠), b7 0 |
| `OBS` 창 | `r0` 센서 행(먹스 0~6), `c0` 센서 열(0~10), `h × w`, 값 = 행 우선 (슬롯 읽기 − 같은 센서 배경) mV. ON_PAD가 아니면 `h = w = 0`. 격자 끝에서 자르지 않고 안쪽으로 밀어 크기 유지 |
| 추적 창 | 3행 × 4열(12값) |
| 발견 OBS | `r0 = 0, h = 7, c0 =` 청크 앞 열, `w = 3`(21값). 읽지 않은 가운데 열 7칸은 `INT16_MIN`(−32768). 청크 {9}는 `w = 1`(7값, `INT16_MIN` 없음). 추적 창은 늘 3행이라 PC는 `h = 7`로 발견 OBS를 알아본다 |
| `LINK.reason` | [01-BLE.md](01-BLE.md) 5절. `up=1`이면 0. 부팅 뒤 연결된 적 없는 슬롯의 스냅샷 `up=0`도 0 |
| `ACK.code` | 0 OK, 1 길이, 2 인자(값 오류·판 다름), 3 링크 없음, 4 BLE 쓰기 실패, 5 바쁨(그 슬롯에 처리 중 ICON) |

- 좌표·신뢰도는 PC가 계산한다. OBS는 광학 원시 창만 싣는다 [권장안 U-32].
- `slot 0xFF`(전체 격자 7×11)는 v1에 없다(홀 대안 빌드에서 `PROTO` + 1과 함께 추가).
- 큐브 연결 정보 `link_up{slot, mac, cube_boot_id, fw}`는 LINK `up=1` + 바로 뒤 EVT HELLO(`cube_boot_id`, `fw = v1`)로 채운다.

## 4. LED 프레임·ICON

LED 본문 1152 B = 384 × (R, G, B), 감광 전 값. 논리 순서: 줄 1 → 16, 줄 안에서 열 1 → 24([하드웨어 02](../01-하드웨어/02-패드.md) 2절의 LED 줄·열).

논리 → 물리(패드 한 곳): 줄 k(0~15), 열 i(0~23) →

| 값 | 식 |
|---|---|
| 데이터선 | `k / 4`(정수 나눗셈, 0~3) |
| 체인 번호 | `(k % 4) × 24 + (k % 2 == 0 ? i : 23 − i)` |

패드 처리:
- LED 프레임 수신: `led_next`에 덮어쓰고 `next_seq = pc_seq`, `has_next = 1`.
- `has_next`·감광 비율 변경·대기 패턴 전환 중 하나면 그 뒤 처음 오는 여유 구간(마지막 슬롯 뒤 ~ 프레임 끝, P 50 ms면 47.22~50 ms)에서 논리→물리 → 감광 → GRB로 부호화하고, 다음 광학 프레임의 LED 출력 창(0~3.2 ms)에서 `led_cur ← led_next`, `cur_seq ← next_seq`, `has_next = 0` → 4체인 송신 → 데이터선 LOW ≥ 300 µs. 여유 구간이 시작된 뒤 받은 프레임은 한 프레임 늦게 켜진다. 세부는 [펌웨어 04 3.1](../02-펌웨어/04-패드-BLE-USB-LED.md).
- 그 프레임의 모든 OBS `led_seq = cur_seq`. 대기 패턴·LED_OFF 검정도 같은 경로.
- 갱신 상한 = 광학 프레임당 1번(P 50 ms면 20 fps).

PC 송신기:
- 렌더러가 만든 프레임은 대기 칸 1개를 덮어쓴다(쌓지 않음).
- `LED_TICK` = HELLO의 `frame_us` 틱마다, 대기 칸이 바뀌었고 `port.writableLength < 4096`일 때만 보낸다. 아니면 다음 틱.

ICON 경로: USB ICON 193 B → 패드가 그 슬롯 BLE ICON 192 B Write(응답) → 쓰기가 끝나면 ACK. 슬롯 > 5면 `code 2`, 링크 없음 `code 3`, 처리 중 `code 5`, BLE 실패 `code 4`.
- `?` 깜빡임은 PC가 ICON 두 장을 0.5 s마다 번갈아 보낸다(10/11 확정, 큐브마다 초당 2번 쓰기). 깜빡임 표지는 규격에 없다.

## 5. 순번·시각 복원

| 값 | 만드는 쪽 | 싣는 곳 | 받는 쪽 쓰임 |
|---|---|---|---|
| `pad_boot_id` | 패드 | 패드 프레임 머리 `boot` | 패드 재부팅 감지 |
| `pad_seq` | 패드 | 머리 `seq` | 중복·역순 버림, 잃은 수 = 틈 − `tx_drop` 증가분 |
| `t_us`(패드) | 패드 | 머리 | 송신 시각. OBS는 표본 시각으로 씀(오차 ≤ 1 ms [실측 필요]) |
| `frame_no` | 패드 | OBS | 같은 프레임 슬롯끼리 묶음 |
| `cube_boot_id`, `seq` | 큐브 | EVT | 큐브 재부팅 감지, 중복 제거 |
| `t_pad32` | 큐브 | EVT | 사건 시각 |
| `pc_boot`, `pc_seq` | PC | PC 프레임 머리 | ACK 짝짓기, 켜진 LED 확인 |

`t_pad32` → 64비트 패드 시각(대칭식, 그 EVT 프레임 머리 `t_us` 기준):

```
d = ((lo32(t_us) − t_pad32 + 2^31) mod 2^32) − 2^31
t_pad_us = t_us − d,  t_valid = true
t_pad32 == 0xFFFFFFFF 이면  t_pad_us = t_us,  t_valid = false
```

|사건 나이| < 2³¹ µs(35.8분)에서 맞고, 음수 나이(사건이 머리보다 늦게 찍힘)도 맞다.

패드 시각 → PC 시각 `t_pc = t_pad + off`(패드 프레임을 받을 때마다, `sample = t_pc_rx − t_us`):
- 처음, 또는 새 `pad_boot_id`: `off = sample`.
- 그 뒤: `off = min(off + (t_pc_rx − t_prev) × 100e-6, sample)`(누설 100 ppm).

| 규칙 | 값 |
|---|---|
| 관측 나이 | `age = t_pc_rx − (t_us + off)` |
| 오래된 OBS | `age > 100 ms` [실측 필요]면 상태기계로 보내지 않고 `obs_stale`++(기록은 남김). EVT·LINK·STATUS·ACK·CAND는 나이 무관, 받은 순서대로 |
| "지금" | 마지막으로 받은 패드 프레임 머리 `t_us`. 모든 판정 시각은 패드 시각, PC 시계는 표시·지연 측정만 |
| 프레임 공백 | 같은 boot에서 연속한 두 패드 프레임 `t_us` 차 > 1 s면 `lost` 다음 `up`으로 처리 |

## 6. 세션·스냅샷·재시도

게이트웨이 핵심은 `onFrame(buf, t_pc_us)`, `onBad(buf, t_pc_us)`, `onPort(event, t_pc_us)`, `onTick(t_pc_us)`만 받는 순수 모듈이다. 안에서 시계를 읽지 않는다(재생·장애 주입이 같은 모듈을 씀).

| 규칙 | 내용 |
|---|---|
| G1 열기 | 포트를 열면 `C0` 1바이트 먼저 → PC HELLO 500 ms마다 → 패드 HELLO 전 패드 프레임은 모두 버림 → 패드 HELLO~첫 STATUS 사이의 LINK·EVT를 모아 `pad.session.snapshot`으로 한 번에 |
| G2 재부팅 | HELLO `boot`가 직전 세션 값과 같으면 `up`, 다르면 `rebooted(reset_reason)`. 마지막 `pad_boot_id`는 저장소에 둔다. 세션 중 머리 `boot`가 바뀌면 `rebooted`로 보고 G1부터 |
| G3 순번 | 같은 boot에서 `seq ≤ 마지막`이면 버림. `seq > 마지막 + 1`은 틈으로 세고 받음 |
| G4 나이 | 5절 |
| G5 큐브 중복 | 슬롯마다 `(cube_boot_id, seq)`. boot가 다르면 받고 `cube.rebooted`. 같으면 `(seq − 마지막) mod 65536`이 1~32767일 때만 받음. 표는 패드 재부팅에도 지우지 않고 게이트웨이 재시작 때 비움. 스냅샷 안 EVT는 표만 고치고 `cube.evt`로 따로 올리지 않음 |
| G6 사유 | 1 `dock`, 2 `button`, 3 `low_batt`, `0x10` `timeout`, `0x11` `stale`, `0x12` `local` |
| G7 부재 | 포트 `close`·`error` 또는 유효 패드 프레임 1500 ms 없음 → `pad.session lost`, 포트 닫고 1000 ms마다 다시 찾음. `lost` 동안 상위는 판정 동결. 복귀 유예: 같은 boot 2 s, 다르거나 모르면 20 s [권장안 U-33] [실측 필요] |
| G8 다시 보냄 | 패드 HELLO → REG → 현재 LED. LINK `up=1`마다(스냅샷 포함) 그 슬롯 ICON |

스냅샷(패드가 PC HELLO를 받으면, 그리고 부팅 직후 스스로 보낸 HELLO 뒤에도):
1. HELLO
2. REG에 있는 슬롯마다 LINK(링크 있으면 `up=1`, 없으면 `up=0` + 마지막 사유)
3. 링크 있는 슬롯마다 보관해 둔 마지막 EVT HELLO·CONTACT·BATT 원본(같은 `seq`)
4. STATUS

자리가 모자라면 다음 펌프에서 이어 보낸다(OBS보다 먼저). 스냅샷 중에도 OBS는 평소대로 나간다.

패드 송신 순서(펌프마다): RX 비우기 → 큐의 EVT·LINK·CAND·ACK → 남은 스냅샷 → 방금 읽은 슬롯 OBS → STATUS(500 ms). 그래서 슬롯 i 읽기 끝 펌프 전에 받은 EVT는 그 OBS보다 먼저 나간다. 펌프 간격 최대 약 11.3 ms. 보낼 자리(`부호화 길이 + OBS·LOG면 512`)가 없으면 버리고 `tx_drop`++.

PC 재시도:

| 명령 | 규칙 |
|---|---|
| REG, REG_SCAN | ACK가 올 때까지 1000 ms마다 같은 내용으로 다시(멱등) |
| ICON | 슬롯마다 미결 1개. 미결 중 같은 슬롯의 새 ICON은 대기 칸 1개를 덮어쓰고 ACK 뒤에 보낸다(재시도보다 최신이 먼저). 1000 ms 안 ACK 없음 또는 `code ∉ {0, 5}`이면 1번 다시. 또 실패면 `cube.icon_failed{slot}`. `code 5`(앞 쓰기 처리 중)는 실패로 세지 않고 1000 ms 더 기다린다. 그 ICON으로 보낸 어느 `pc_seq`의 `code 0`이든 성공이다. 여러 슬롯은 간격 없이 보내도 됨(패드 RX 8192 B) |
| LED | ACK 없음. `STATUS.led_seq ≠` 마지막 LED `pc_seq`이고 보낸 지 200 ms가 지났으면 최신 LED를 다시 |

패드 PC 생존 판단(유효 PC 프레임 도착 기준):

| 조건 | 패드 동작 |
|---|---|
| PC 프레임 5 s 없음 | 다음 LED 출력 창에서 대기 패턴(동적 ≤ 0.3 A), `flags` b2 = 0, b3 = 1 |
| 10분 없음 | 검정 → `LED_EN` LOW. 모든 링크를 끊고(LINK `up=0`, `reason 0x12`) 스캔 멈춤 [권장안 U-98]. 기각이면 BLE·스캔·IR 슬롯 유지 |
| 다음 LED 프레임 | 대기 패턴 끝 |

패드 명령 처리: REG는 NVS와 비교해 바뀐 때만 쓰고, 빠진 슬롯의 링크를 끊고(LINK `up=0, reason 0x12`) ACK. USB를 뽑으면 패드 제어부가 꺼지므로 PC는 늘 `lost` → `rebooted` 순서를 본다.

## 7. 판 규칙·값 검사

판 규칙:
- `PROTO = 1`. 선택 필드 없음. 메시지·EVT type 추가, 필드 길이·뜻 변경은 `PROTO` + 1(펌웨어·게이트웨이 함께). v1은 2026-10-11 확정.
- 게이트웨이: 패드 HELLO `proto ≠ 1` → 세션 열지 않음, `pad.proto_mismatch{pad, pc}`, PC HELLO 멈춤, 포트 닫기. 그 포트가 다시 연결될 때(USB 재연결, 게이트웨이 재시작)만 다시 조사.
- 패드: PC HELLO `proto`가 다르면 HELLO로만 답함. 같은 `proto`의 PC HELLO를 받기 전까지 REG·REG_SCAN·ICON은 ACK `code 2`, LED 무시, LED_OFF는 따름.

PC 값 검사(길이·CRC 통과 뒤). 어기면 버리고 `bad_field`++, 기록 `bad`:

| type | 검사 |
|---|---|
| 공통 | 머리 `boot ≠ 0`. 같은 boot에서 `t_us`가 줄지 않음 |
| HELLO | `proto` 같음. `20000 ≤ frame_us ≤ 200000`. `bad_map` 비트 77~79 = 0 |
| STATUS | `flags` b5~b7 = 0 |
| OBS | `slot ≤ 5`. `st` b7 = 0. b6 = 1이면 광학 NA·`h = w = 0`(→ `not_read: true`). `h·w ≤ 21`, `r0 + h ≤ 7`, `c0 + w ≤ 11`, 본문 = 14 + 2hw. ON_PAD 아니면 `h = w = 0`. `INT16_MIN`은 `h = 7, w = 3` 창의 가운데 열에서만, 그 열 7칸 모두. 그 밖 `abs(v) ≤ 3300`. `slot 0xFF`는 `bad_field` |
| EVT | `slot ≤ 5`. CONTACT·SYNC `v0 ∈ {0,1}`. BATT `v0 = 0`, `2500 ≤ v1 ≤ 4500` [실측 필요]. BYE `v0 ∈ {1,2,3}`. HELLO `v0 = 1`(다르면 알림 `cube_proto_mismatch{slot}`, 그 슬롯 EVT를 다음 `v0 = 1` HELLO까지 버림). BTN `v0 ∈ {0,1,2}`, `v0 = 0`이면 `v1 = 0`, `v0 = 1`이면 `v1 < 1000`, `v0 = 2`면 `v1 ≥ 1000`(끄기 시간은 큐브 상수라 검사하지 않음). RAW_S1은 `v0 ∈ {0,1}`이면 기록만(상태기계로 안 감). 그 밖 type은 `bad_field` |
| LINK | `slot ≤ 5`, `up ∈ {0,1}`, `up = 1`이면 `reason = 0`, `reason ∈ {0,1,2,3,0x10,0x11,0x12}` |
| CAND | `−127 ≤ rssi ≤ 20` |
| ACK | `acked_type ∈ {0xA0,0xA1,0xA2}`, `code ≤ 5` |
| LOG | `level ∈ {1,2,3}`. 잘못된 UTF-8은 U+FFFD로 바꿈(버리지 않고 `bad_field`도 안 올림) |

패드 값 검사(머리·길이·CRC 틀림은 `rx_bad`):

| type | 검사 | 실패 |
|---|---|---|
| REG·REG_SCAN·ICON | 같은 `proto`의 PC HELLO를 받은 뒤 | `code 2` |
| REG | 0이 아닌 같은 MAC이 두 슬롯에 없음 | `code 2`, 적용 안 함 |
| REG_SCAN | `on ∈ {0,1}` | `code 2` |
| ICON | `slot ≤ 5` / 링크 있음 / 처리 중 ICON 없음 | `code 2` / `3` / `5` |
| LED, LED_OFF, HB | 길이만 | `rx_bad` |

## 8. 포트 열기

| 항목 | 값 |
|---|---|
| 열기 | `new SerialPort({ path, baudRate: 115200, hupcl: false, rtscts: false })`. 선 상태 DTR = 0, RTS = 0(리셋·다운로드 모드 회피). 연 뒤 `set({rts})`·`set({dtr})`를 부르지 않음. `baudRate`는 무시됨 |
| 후보 | `SerialPort.list()`에서 `vendorId` 303A, `productId` 1001(대소문자 무시). 지난번 패드 `serialNumber`부터 |
| 식별 | G1대로 HELLO, 1500 ms 안 패드 HELLO 없으면 닫고 다음 후보. 처음 답한 패드를 씀(패드 1대). 큐브 A1(같은 ID)은 답하지 않아 떨어짐 |
| 다시 찾기 | `lost`면 1000 ms마다 |
| Windows 열기 | 기본 드라이버 usbser.sys(Windows 10 이상 자동). 열고 닫을 때 DTR·RTS가 잠깐 바뀌어 패드가 리셋되는지 확인한다: 열기·닫기 20회 동안 패드 HELLO `boot`가 그대로 [실측 필요] |
| 독점 | 시연 중에는 게이트웨이만 연다. 펌웨어를 올릴 때는 게이트웨이를 멈춤. 다른 프로그램이 먼저 열었으면 알림 `port_busy`만 |
| 입력원 | 시작 인자 `--source=pad\|sim\|csv:<포트>\|csvfile:<파일>`, 기본 `pad`. CSV는 한 줄 ≤ 512 B, 숫자 아닌 칸이 있는 줄은 버림, `#` 줄은 로그. 이진 규격과 같은 포트에 섞지 않음 |
| 패드 빌드 | USB Mode = Hardware CDC, CDC On Boot = Enabled, Core Debug Level = None(`proto.h` `#error` 검사). USB 초기화는 코어 0 태스크, RX 8192 B·TX 4096 B(`begin` 전), `setTxTimeoutMs(0)`, `esp_log_level_set("*", ESP_LOG_NONE)`. `flush()`·`print`류 금지, USB 쓰기는 프레임 루프 한 곳 |
