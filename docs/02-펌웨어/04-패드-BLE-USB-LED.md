# 패드 펌웨어: BLE·USB·LED·워치독

이 문서로 답하는 것: 패드는 어떤 큐브에 어떻게 연결하고 SYNC를 보내나, USB로 언제 무엇을 주고받나, LED를 어떻게 바꿔 내보내고 전원을 켜고 끄나, 멈추면 어떻게 되나.

관련: [03-패드-프레임-광학](03-패드-프레임-광학.md) · [01-구조-빌드](01-구조-빌드.md) · [BLE 규격](../03-인터페이스/01-BLE.md) · [USB 규격](../03-인터페이스/02-USB.md) · [패드 하드웨어](../01-하드웨어/02-패드.md)

## 1. BLE 중앙 (`bleCtl`, `pad_link`)

### 1.1 연결 정책

- 패드 = 중앙(광고 안 함). 켜진 등록 큐브 전부, 최대 6 연결. 제어기 활동 ≤ 7(링크 6 + 스캔 1).
- 등록 표(슬롯 0~5의 MAC, NVS)에 있는 공개 MAC에만 연결한다. 본딩·암호화 없음 [권장안 U-74].
- 연결 값(CI·지연·supervision·PHY·MTU)은 [BLE 1](../03-인터페이스/01-BLE.md). `ble_gap_connect`의 `conn_params`로 처음부터 준다.
- 송신 출력은 기본 +9 dBm(설정 호출 없음). F05-3 미달이면 `esp_ble_tx_power_set(ESP_BLE_PWR_TYPE_DEFAULT, ESP_PWR_LVL_P15)`.

### 1.2 스캔

| 모드 | 값 |
|---|---|
| 평상시 | `ble_gap_disc(BLE_OWN_ADDR_PUBLIC, BLE_HS_FOREVER, {itvl 0x61(60.625 ms), window 0x18(15 ms), filter_policy 1(허용 목록), passive 1, filter_duplicates 0})`. 허용 목록 = 등록 MAC. 등록 표가 바뀌면 스캔 멈춤 → `ble_gap_wl_set()` → 다시 켬 |
| 등록 모드 | PC REG_SCAN on. 같은 값에 `filter_policy 0`, 30 s 뒤 자동 끔. 서비스 UUID를 광고하는 미등록 주소만 CAND로(주소마다 2000 ms에 1번) |
| 연결 개시 | 한 번에 1개. 스캔 멈춤 → `ble_gap_connect(…, 2000 ms, {scan_itvl 16, scan_window 16(10 ms), …})`. 30 s 기본값 금지 |

### 1.3 설정 단계 (전체에서 한 슬롯만, 제한 3 s)

1. MTU 교환(`ble_gattc_exchange_mtu`)
2. 서비스 탐색(`ble_gattc_disc_svc_by_uuid`) → 특성 전부(`ble_gattc_disc_all_chrs`) → EVT 설명자(`ble_gattc_disc_all_dscs`)
3. CCCD 쓰기 0x0001
4. CFG 쓰기(`ble_gattc_write_flat`, 응답)
5. LINK `up=1` → 스캔 재개 → SYNC 타이머 시작(예열 간격)

- 같은 MAC의 설정이 연속 3번 실패하면(연결 실패, 설정 중 끊김, 3 s 초과, 원격 GATT 오류) 10 s 무시한다. 실패마다 LOG 수준 2(슬롯·단계·NimBLE 오류 코드).
- 로컬 자원 부족 `BLE_HS_ENOMEM`은 실패로 세지 않고 50 ms 뒤 같은 단계를 다시 한다.

### 1.4 링크 상태기계 (슬롯마다)

| 상태 | 사건 → 다음(할 일) |
|---|---|
| NONE | REG에 MAC 생김 → DOWN(허용 목록 갱신) |
| DOWN | 광고 보고 ∧ 다른 슬롯이 CONNECTING·SETUP 아님 ∧ 무시 중 아님 → CONNECTING(스캔 멈춤, connect 2 s) |
| CONNECTING | CONNECT status 0 → SETUP(conn_handle 저장, 3 s 시작, MTU) / status ≠ 0·2 s 초과 → DOWN(실패 +1, LOG, 스캔 재개) |
| SETUP | 단계 완료 → 다음 단계 / CFG 성공 → UP(LINK `up=1`, 실패 0, 스캔 재개, SYNC 예열) / DISCONNECT(0x3E 포함) → DOWN(실패 +1, LOG, 스캔 재개, PC 알림 없음) / 원격 오류·3 s 초과 → CLOSING(terminate, 실패 +1, LOG, PC 알림 없음) / ENOMEM → 50 ms 뒤 같은 단계 |
| UP | EVT BYE → CLOSING(LINK `up=0`, BYE 사유를 먼저 `evtQ`에) / 같은 MAC 광고 → 사유 0x11만 예약(끊지 않음) / REG에서 빠짐 → CLOSING(0x12) / DISCONNECT → DOWN(LINK `up=0`, 사유 매핑, `gen + 1`) / EVT SYNC 유효 → SYNC 간격 유효로 / `CFG_REWRITE` → 쓰기 대기열에 CFG, SYNC 간격 예열로 / ICON → 쓰기 대기열 |
| CLOSING | DISCONNECT → DOWN(아직 안 알렸으면 예약 사유로 LINK `up=0`) / terminate가 `BLE_HS_ENOTCONN` → 바로 DOWN. REG에서 빠졌으면 NONE |

- 모든 GAP·GATT 사건은 conn_handle이 그 슬롯의 현재 값과 같을 때만 처리한다. 다르면 버린다.
- 끊김 사유 매핑은 [BLE 5](../03-인터페이스/01-BLE.md). LINK `up=1`을 보낸 적 없는 연결은 알리지 않는다.
- 옛 링크: 링크가 있는 MAC의 광고를 보면 끊지 않는다. supervision 만료 DISCONNECT에 예약한 0x11을 붙여 알리고 평소처럼 다시 연결한다.
- 재빌드에 `CONN_REATTEMPT=n`이 없을 때만 대체 규칙: SETUP은 `BLE_GAP_EVENT_LINK_ESTAB`(status 0) 뒤 시작, `REATTEMPT_COUNT`를 받으면 CONNECTING으로 돌아가 스캔 재개 없이 새 CONNECT를 기다린다.

### 1.5 GATT 쓰기 대기열

- 응답형 절차는 동시에 2개 이하: 설정 1 + ICON·CFG_REWRITE 1(NimBLE 절차 풀 4개를 모든 연결이 나눔).
- ICON·CFG_REWRITE는 슬롯별 미결 표에 둔다. CFG_REWRITE 먼저, 그다음 슬롯 번호 순으로 하나씩 쓴다. 끝나면(응답·오류) 다음 것.
- ICON 성공 → ACK `code 0`을 `evtQ`에. 원격 실패 → ACK 실패 코드. CFG 원격 실패 → 끊고 다시 연결(실패 1회).

### 1.6 SYNC 송신 (esp_timer, 링크마다 one-shot 1개)

```
콜백: gen0 = 링크.gen, h = 링크.conn_handle
  if (h == NONE || gen0 != 타이머를 건 때의 gen) return;     // 다시 걸지 않음
  pad_us = esp_timer_get_time();
  rc = ble_gattc_write_no_rsp_flat(h, h_sync, &pad_us, 8);
  if (rc == 0) sync_ok_ms[slot] = now_ms;  // ENOMEM이면 건너뛰고 셈만
  다음 간격 = esp_random() 균등: 예열 5000~15000 µs, 유효 뒤 20000~40000 µs → 다시 걺
끊김: bleCtl이 conn_handle = NONE, gen + 1을 먼저 쓰고 그다음 esp_timer_stop
```

### 1.7 명령(`cmdQ`)과 ACK

| 명령 | 출처 | 처리 |
|---|---|---|
| `REG_APPLY` | PC REG | 바뀐 때만 NVS 쓰기 → `ble_gap_wl_set` → ACK `code 0`. 빠진 슬롯 링크는 끊음(0x12) |
| `REG_SCAN(on/off)` | PC REG_SCAN | 1.2 등록 모드 |
| `ICON(slot)` | PC ICON | 192 B는 루프가 슬롯별 버퍼에 쓰고, 미결 플래그는 `bleCtl`이 쓰기 뒤 내림 |
| `CFG_REWRITE(slot)` | sync_fault | 1.5 |
| `DROP(slot)` | 등록 해제 | CLOSING |

- `xQueueSend(…, 0)`. 넣지 못하면 ACK를 보내지 않는다. ACK 코드 뜻과 PC 재시도는 [USB 3·6](../03-인터페이스/02-USB.md).
- NVS 쓰기 동안 두 코어가 멈춰 프레임 1~2개가 넘칠 수 있다. 쓰기 뒤 첫 LED 창에서 LED 프레임을 다시 보낸다.

### 1.8 PC 부재 [권장안 U-98]

- 유효 PC 프레임(HB 포함)이 10분 없으면 모든 링크를 끊고(0x12) 스캔을 멈춘다. 큐브는 무연결 5분으로 꺼진다. 유효 PC 프레임이 다시 오면 스캔을 재개한다.
- 기각이면 BLE·스캔·IR 슬롯을 그대로 둔다.

## 2. USB (`usb_link`, `pad_pump`)

- 메시지·머리·CRC·SLIP·필드는 [USB 1~3](../03-인터페이스/02-USB.md), 세션·스냅샷·재시도는 [USB 6](../03-인터페이스/02-USB.md), 값 검사는 [USB 7](../03-인터페이스/02-USB.md).
- HWCDC: `begin()` 전 RX 8192·TX 4096, 뒤 `setTxTimeoutMs(0)`. `usbInit`은 코어 0 태스크.
- USB 읽기·쓰기는 프레임 루프 한 곳에서만 한다.

### 2.1 펌프 (LED 창 끝·배경 끝·슬롯마다·여유 구간)

1. RX 비우기: `n = available()`만큼 `read(buf, n)` → SLIP 복호 → 유효 프레임 처리(길이·CRC 오류면 `rx_bad` +1).
2. `evtQ`의 EVT·LINK·CAND·ACK·LOG 보내기.
3. 남은 스냅샷 프레임 보내기(PC HELLO를 받으면 HELLO → LINK들 → 보관 EVT → STATUS).
4. 방금 읽은 슬롯의 OBS(슬롯 펌프만).
5. 500 ms가 지났거나 플래그가 바뀌었으면 STATUS.

- 보내기: `seq`를 올리고 부호화. `need = 길이 + (OBS·LOG면 512)`. `availableForWrite() < need`면 버리고 `tx_drop` +1. 아니면 `write(buf, len)` 한 번.
- 펌프마다 ADAPTER_N도 읽는다(4절).
- OBS는 링크가 있는 슬롯마다 프레임마다 늘 보낸다(읽지 않았으면 NA).
- LOG는 초당 10개 이하.

### 2.2 PC 명령 처리

| 받은 것 | 처리 |
|---|---|
| HELLO | 스냅샷 예약 |
| HB | PC 생존 시각 갱신 |
| LED | 본문을 `led_next`에 복사, `next_seq = pc_seq`, `has_next = 1`(덮어쓰기) |
| LED_OFF | 검정 → L0 |
| REG / REG_SCAN / ICON | `cmdQ`(1.7) |

## 3. LED 출력 (`led_out`, `led_map`)

### 3.1 이중 버퍼와 교체 시점

| 시점 | 하는 일 |
|---|---|
| 여유 구간(47.22~50 ms) | `has_next`이거나 감광 비율·대기 패턴이 바뀌었으면 `led_next`를 대기 심볼 버퍼에 부호화: 논리 → 물리 순서 → 감광 → RGB → GRB 심볼. `led_cur`·`cur_seq`는 아직 안 바꿈 |
| LED 창(0~3.2 ms) | 대기 버퍼가 있으면 `led_cur ← led_next`, `cur_seq ← next_seq`, 4체인 송신, 완료 시각까지 대기 |
| 배경·슬롯 구간 | RMT를 건드리지 않는다 |

- 그 프레임의 OBS `led_seq` = `cur_seq`(대기 패턴·꺼짐이면 0). STATUS도 지금 켜진 프레임의 값을 보고한다.
- 논리 → 물리: 4체인 0~3(줄 1~4, 5~8, 9~12, 13~16), 묶음 안 지그재그. 식과 바이트 순서는 [USB 4](../03-인터페이스/02-USB.md). `led_map`은 384행 대응표와 대조해 시험한다.

### 3.2 RMT 구동 (T9로 고름)

| 방식 | 구현 | 고르는 조건 |
|---|---|---|
| A(기본) | `rmtInit(pin, RMT_TX_MODE, RMT_MEM_NUM_BLOCKS_1, 10000000)` × 4(GPIO15~18), 체인마다 `rmt_data_t` 2304개 정적 버퍼, `rmtWriteAsync` × 4 연달아 | T9 A 통과 |
| B | 같은 코드로 체인마다 보내고 기다림(약 11.5 ms, P 58.33) | A 미달, B 통과 |
| C | 배선 변경(체인 이어 붙임) 뒤 IDF `rmt_new_tx_channel`(`with_dma = 1`, `mem_block_symbols` ≥ 1024)·`rmt_new_bytes_encoder`·`rmt_transmit`. 환경 변경 없음 | 둘 다 미달 |

- 틱 0.1 µs, 비트 1.2 µs(0 = 4/8틱, 1 = 8/4틱). 칩 실물 타이밍으로 확인 [실측 필요: T9].
- 완료는 이벤트 비트가 아니라 계산한 시각으로 기다린다: `LED_TX_WAIT_US` = 실제 심볼 틱 합 + 50 µs(2304 × 1.2 + 50 ≈ 2815 µs). 그 뒤 데이터선은 휴지 LOW라 3.50 ms까지 LOW ≥ 300 µs가 지켜진다.
- `rmtTransmitCompleted`는 그 뒤 한 번만 읽어 진단에 쓴다(거짓이면 `led_late` +1, 10 s마다 LOG).

### 3.3 감광

```
I_est = N_on × Iq + (Σr·Ir + Σg·Ig + Σb·Ib) / 255     // N_on = LED 전원 켜짐이면 384, 아니면 0
I_dyn = I_est − N_on × Iq
I_est > I_cap 이면 동적 항 전체에 s = (I_cap − N_on·Iq) / I_dyn, 하한 0.05
STATUS led_scale = round(s × 255)
```

| 상수 | 값 |
|---|---|
| `IQ_MA` | 2.5 [실측 필요: T10a·b] |
| `IR_MA`·`IG_MA`·`IB_MA` | 20 [실측 필요: T10a·b] |
| `I_CAP_A` | 4.0 [실측 필요: T10d에서 V_in ≥ 4.50 못 지키면 낮춤] |

- 프레임을 받을 때마다 송신 전에 적용한다. PC가 이미 감광한 프레임도 다시 검사한다. 패드 지도 상시 점등도 이 감광이 4.0 A를 지킨다 [미정 C5].

### 3.4 대기 패턴

- 유효 PC 프레임이 5 s 없으면 다음 LED 창에서 대기 패턴, STATUS 플래그 "PC 살아 있음" 0·"대기 패턴" 1. 10분이면 검정 → L0.
- 모양 [미정 F6]. 잠정: `led_map`의 고정 그림 하나. 동적 전류 ≤ 0.3 A를 단위 시험으로 지킨다.

## 4. LED 전원 상태기계 (프레임 루프 소유)

| 상태 | GPIO40 | 들어가는 조건 | 나가는 조건 |
|---|---|---|---|
| L0 꺼짐 | LOW | 부팅, ADAPTER_N HIGH(펌프 2번 연속, ≤ 23 ms), LED_OFF(검정 보낸 뒤), PC 무프레임 10분(검정 보낸 뒤) | ADAPTER_N LOW 200 ms ∧ (부팅 뒤 처음이거나 L0에 든 뒤 LED 프레임을 받음) → L1. ADAPTER_N으로 든 L0은 어댑터가 돌아오면 바로 L1 |
| L1 켜는 중 | HIGH | L0에서. HIGH는 펌프 ②(약 13~14.5 ms)에서 씀 | 다음 LED 창에서 검정 송신 → L2(HIGH → ≥ 20 ms → 50 ms 안 검정 규칙 안) |
| L2 정상 | HIGH | L1, L3에서 LED 프레임 받음 | PC 무프레임 5 s → L3. ADAPTER_N HIGH → L0 |
| L3 대기 패턴 | HIGH | PC 무프레임 5 s | LED 프레임 → L2. 10분 → 검정 → L0. ADAPTER_N HIGH → L0 |

- ADAPTER_N HIGH면 검정을 보내지 않고 곧바로 GPIO40을 내린다(LED 전원이 이미 없다).
- LED_EN은 데이터선이 LOW일 때만 바꾼다. STATUS 플래그(어댑터·LED_EN 등)는 [USB 3](../03-인터페이스/02-USB.md).

## 5. 워치독

| 대상 | 방법 | 멈추면 |
|---|---|---|
| 프레임 루프 | `enableLoopWDT()`(5 s). `loop()` 반환마다 먹음 | 작업 워치독 패닉 → 코어 덤프 → CPU 리셋 → `initVariant()`에서 GPIO40 LOW. PC는 새 `pad_boot_id`로 재부팅을 안다 |
| `bleCtl` | `esp_task_wdt_add(NULL)`, `xQueueReceive(…, 500 ms)`, 돌 때마다 `esp_task_wdt_reset()` | 패닉 리셋. 등록 안 하면 `bleCtl`만 멈출 때 링크·SYNC는 살아 있어 PC가 알 길이 없다 |

- 작업 워치독은 기본으로 IDLE0만 본다. 코어 1 바쁜 대기는 IDLE1을 굶겨도 괜찮다.
- `bleCtl`의 긴 블로킹 호출은 NVS 쓰기(수십 ms)뿐이다.
- 시험: `CS_HANG_S=<s>`면 그 시각에 `bleCtl`이 무한 루프에 든다(F05-6).

## 6. 조건부: Zero 2개 역할 분리

- 발동: ① BLE 대리 부하를 켠 빌드에서만 X2-U2 읽기 창 편차 > 200 µs, 또는 ② T4(T4a·T4b)에서 BLE 부하가 잡음을 늘리고 부하를 다른 Zero로 옮기면 기준 안. 먼저 펌프 위치·루프 우선순위 → e·P 확대를 한 뒤 마지막에 쓴다.
- Zero-A = 프레임 루프·USB·LED·ADC, 패드 시각의 주인. Zero-B = BLE 중앙·SYNC. UART1 GPIO43(TX)·44(RX), SLIP + CRC-32로 `evtQ`·`cmdQ` 항목 그대로. 프레임 펄스 GPIO39(A → B), B는 `est.h`로 A 시각을 따라간다. 세부 규격은 발동 때 쓴다.
