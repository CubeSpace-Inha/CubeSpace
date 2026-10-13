# 규격 v1 공용 코덱 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 패드↔PC USB 프레임과 큐브↔패드 BLE 구조체를 PC(TypeScript)와 펌웨어(C)에 각각 구현하고, 독립 기준 구현(Python)이 만든 벡터 파일 하나로 두 구현이 같은 바이트를 만드는지 확인한다.

**Architecture:** CubeSpace 저장소의 `spec/vectors/make_vectors.py`(표준 라이브러리만)가 규격 표를 그대로 옮겨 `spec/vectors/proto_vectors.hex`를 만든다. PC(CubeSpace-Client) `src/core/gateway/proto.ts`와 펌웨어(CubeSpace-Firmware) `libraries/CubeSpaceProto/src/proto.cpp`는 서로의 코드를 보지 않고 같은 파일(둘 다 그대로 복사한 것)만 통과하면 된다. 셋 다 입출력·시각이 없는 순수 코드라 하드웨어 없이 시험한다.

**Tech Stack:** Python 3(표준 라이브러리), TypeScript 6.0 + Vitest 5(CubeSpace-Client, pnpm, Node 24, `node:zlib`의 `crc32`), C++17 + make(Arduino 스케치와 같은 C++. 로컬 Apple clang++, CI g++, Windows는 MSYS2 UCRT64 g++ 또는 WSL2).

**Spec:** [`docs/03-인터페이스/02-USB.md`](../../03-인터페이스/02-USB.md) 1·2·3·5·7절, [`01-BLE.md`](../../03-인터페이스/01-BLE.md) 2·3절, [`04-시험-벡터.md`](../../03-인터페이스/04-시험-벡터.md), [`02-펌웨어/05-선개발-시험.md`](../../02-펌웨어/05-선개발-시험.md) 1.1·4절, [`04-소프트웨어/01-구성.md`](../../04-소프트웨어/01-구성.md) 6절.

**실행 위치:** 이 계획은 저장소 셋에서 나눠 실행한다. Task 1(생성기)은 CubeSpace(이 문서 저장소), Task 2(PC 코덱)는 CubeSpace-Client, Task 3(펌웨어 코덱)은 CubeSpace-Firmware. 브랜치·커밋·파일 경로는 그 Task가 적은 저장소 기준이다.

**시작점:** 각 저장소의 기본 브랜치에서 새 브랜치 `feat/proto-v1`을 만들어 진행한다.

**검증 기록:** 이 문서의 코드 블록은 2026-10-11에 임시 폴더에서 그대로 돌려 본 것이다. 생성기 69벡터, PC `Tests 65 passed`·`tsc` 오류 0, 펌웨어 `proto: crc 1, frame 53, slip 2, ble 3, pad 4 vectors OK`(clang++ `-Wpedantic`까지 경고 0). 일부러 필드 순서·CRC 다항식을 틀리게 하면 PC·펌웨어 시험이 모두 실패하는 것도 확인했다. g++(CI·MSYS2)와 Windows 실기로는 아직 돌려 보지 않았다(Task 3 Step 8). 이 결과는 저장소를 나누기 전 폴더 배치에서 돌린 것이라, CubeSpace `spec/vectors/`·CubeSpace-Client(경로, tsconfig include, ESLint를 포함한 `pnpm check`)·CubeSpace-Firmware에서 다시 돌려야 한다 [실측 필요].

## 이 계획의 위치

| 순서 | 계획(`docs/superpowers/plans/`) | 범위 |
|---|---|---|
| **1** | **proto-v1(이 문서)** | 펌웨어 [05-선개발](../../02-펌웨어/05-선개발-시험.md) 4절 2단계, PC [01-구성](../../04-소프트웨어/01-구성.md) 6절 1·2단계의 코덱 부분 |
| 2 | pc-gateway-estimate | 게이트웨이 세션 G1~G8, `estimate`([시험 벡터 부록](../../03-인터페이스/04-시험-벡터.md)의 `estimate()` 이식), 시뮬레이터 입력 |
| 3 | fw-cube-logic | `est`·`frame`·`tpad`, `cube_logic`(버튼 4 s 포함) |
| 4 | fw-pad-logic | `pad_link`, `optics`, `pad_pump`·스냅샷·`led_map`, 가상 큐브·패드 → L0 파일 |
| 5 | pc-engine-events | `engine`, `cube-events` |
| 6 | pc-shell-adapter | 서비스 셸, C# 창 어댑터(CubeSpace-Program-Win32), G2 대본 |

- 중간 시연 G2(10/19~23, 시뮬레이터 큐브 4개로 실제 창 이동)는 PC 1 → 2 → 5 → 6이 임계 경로다. 펌웨어 3·4는 나란히 간다.
- 2번부터는 앞 계획이 끝난 뒤 그때의 코드를 보고 쓴다.

## Global Constraints

- 모든 정수는 리틀 엔디언, 구조체에 빈 바이트 없음. USB 머리 17 B, CRC는 u32 CRC-32(zlib과 같은 값).
- 복원 프레임 21~1200 B, LOG text ≤ 200 B. SLIP은 앞뒤 `C0`, 안의 `C0 → DB DC`, `DB → DB DD`.
- `PROTO = 1`. 메시지·EVT type 추가, 필드 길이·뜻 변경은 `PROTO` + 1(펌웨어·게이트웨이 함께).
- PC 순수 모듈(CubeSpace-Client `src/core/gateway/`): `fs`·`net`·`http`·`serialport`·`@serialport/*`·Electron import 금지, `Date.now()`·`Math.random()`·`new Date()` 금지(`test/rules.test.ts`. CubeSpace-Client에 아직 없으면 리뷰로 확인한다). `node:zlib`는 허용.
- PC 판은 CubeSpace-Client에 고정된 판을 따른다. 새 의존성을 넣지 않는다.
- 펌웨어 호스트 시험은 시험 틀 없이 모듈마다 assert 실행 파일 하나, CubeSpace-Firmware에서 `make -C host_test`. 플래그 `-std=c++17 -Wall -Wextra -Werror`(Arduino 스케치가 C++이라 코덱도 C++).
- 같은 값은 한 곳: 펌웨어 `proto.h`(BLE는 `x1_proto.h`), PC `proto.ts`.
- 벡터 파일은 CubeSpace `spec/vectors/`에서 생성기로만 만든다. 손으로 고치지 않는다. CubeSpace-Client는 `test/vectors/proto_vectors.hex`, CubeSpace-Firmware는 `host_test/vectors/proto_vectors.hex`에 그대로 복사해 두고, 생성기가 다시 만들면 다시 복사한다. 복사 커밋 메시지에 원본 CubeSpace 커밋을 적는다.
- `[실측 필요]` 값(큐브 전압 검사 2500~4500 mV)은 상수 한 곳에 둔다.

## Windows에서 실행

팀 개발 PC는 Windows다. 명령은 아래처럼 바꿔 친다.

| 명령 | Windows |
|---|---|
| `python3 …` | `py …`(Python 3.10 이상). 생성기는 인코딩을 UTF-8·LF로 고정해 Windows 기본(cp949·CRLF)에 흔들리지 않는다 |
| `pnpm`·`git` | PowerShell·cmd 어디서나. `pnpm install`은 Windows PowerShell에서 한다(WSL에서 설치한 `node_modules`는 Windows에서 못 쓴다). Windows PowerShell 5.1은 `&&`를 모르니 한 줄씩 친다 |
| `make -C host_test`(CubeSpace-Firmware) | MSYS2 UCRT64 셸(`pacman -S make mingw-w64-ucrt-x86_64-gcc`) 또는 WSL2 Ubuntu(`sudo apt install build-essential`). 저장소는 `C:\dev\CubeSpace-Firmware`처럼 짧은 ASCII 경로에 둔다 |
| `gh run watch` | GitHub CLI 설치 필요. 없으면 웹에서 Actions 탭을 본다 |

## Review Focus

1. **USB 바이트가 1바이트씩, 또는 SLIP 이스케이프(`DB`) 바로 뒤에서 끊겨 온다** → 이어 붙여 같은 프레임이 된다. Task 2 "조각 경계", Task 3 1바이트씩 `cs_slip_push`.
2. **`C0` 없이 끝없이 오는 쓰레기(ROM 부팅 문구·패닉 출력)** → 버퍼가 1201 B에서 더 쌓지 않고, 해독에서 `len`으로 버린다. Task 2·3 쓰레기 시험.
3. **LOG text 200 B / 201 B 경계** → 200은 통과, 201은 `drop_body`. Task 1 벡터 `log_200`·`log_201`.
4. **출력 자리가 모자라거나 본문 바이트열 길이가 틀린 부호화** → 아무것도 쓰지 않는다(C는 0을 돌려줌, TS는 `RangeError`). Task 2·3.
5. **부호 있는 필드(CAND `rssi` −70, CFG `bias_us` −250)** → 부호가 그대로 온다. Task 1 벡터, Task 3 `sign` 비교.

## 파일 구조

| 파일 | 책임 |
|---|---|
| 새 `spec/vectors/make_vectors.py`(CubeSpace) | 규격 표를 옮긴 독립 기준 구현. 벡터 생성과 `--check` |
| 새 `spec/vectors/proto_vectors.hex`(CubeSpace) | 생성된 공용 벡터 69개 |
| 고침 `.github/workflows/ci.yml`·`.gitignore`(CubeSpace) | 벡터 재생성 검사를 CI에 |
| 새 `src/core/gateway/proto.ts`(CubeSpace-Client) | PC 코덱: 해독·부호화·SLIP·시각 복원·PC 값 검사 |
| 새 `src/core/gateway/index.ts`(CubeSpace-Client) | `proto` 내보내기 |
| 복사 `test/vectors/proto_vectors.hex`(CubeSpace-Client) | CubeSpace 벡터 파일을 그대로 복사 |
| 새 `test/proto.test.ts`(CubeSpace-Client) | 벡터로 PC 코덱 시험 |
| 새 `libraries/CubeSpaceProto/src/x1_proto.h`(CubeSpace-Firmware) | BLE UUID, CFG·EVT 구조체 |
| 새 `libraries/CubeSpaceProto/src/proto.h`·`proto.cpp`(CubeSpace-Firmware) | 펌웨어 코덱: 구조체·CRC·해독·부호화·SLIP·패드 인자 검사 |
| 복사 `host_test/vectors/proto_vectors.hex`(CubeSpace-Firmware) | CubeSpace 벡터 파일을 그대로 복사 |
| 새 `host_test/Makefile`·`test_proto.cpp`(CubeSpace-Firmware) | 벡터로 펌웨어 코덱 시험 |
| 고침 `.github/workflows/`·`.gitignore`(CubeSpace-Firmware) | 호스트 시험을 CI에 |
| 고침 문서(CubeSpace) | 04-시험-벡터(벡터 파일 형식) |

---

### Task 1: 벡터 파일 형식과 생성기(CubeSpace `spec/vectors/`)

**Files:**(CubeSpace 저장소)
- Create: `spec/vectors/make_vectors.py`
- Create(생성): `spec/vectors/proto_vectors.hex`
- Modify: `docs/03-인터페이스/04-시험-벡터.md`(벡터 파일 형식 문단), `.github/workflows/ci.yml`, `.gitignore`

**Interfaces:**
- Consumes: 없음
- Produces: `proto_vectors.hex` 줄 형식. 한 줄 = `<묶음> <이름> <기대> <16진|-> [키=값 ...]`(공백 하나로 나눔, `#` 줄은 주석).
  - 묶음: `crc`, `frame`(복원 프레임), `slip`(+ `wire=h:…`), `ble`(이름이 `cfg`로 시작하면 CFG, 아니면 EVT), `pad`, `time`
  - 기대: `ok`, `bad_field`, `drop_len`, `drop_crc`, `drop_type`, `drop_body`, `ack2`
  - 값: 정수는 10진, 바이트열은 `h:16진`, OBS `v`는 쉼표 목록(빈 창은 빈 값)
  - 키: 머리 `type`·`boot`·`seq`·`t_us`, 본문은 USB 2절 이름. 겹치는 EVT 본문은 `evt_seq`·`evt_type`, REG 본문 36 B는 `macs`

- [ ] **Step 1: 깨끗한 worktree를 만든다**

작업 트리를 깨끗하게 두려고 새 폴더에 따로 연다. 이후 모든 명령은 새 폴더에서 실행한다.

```bash
git worktree add ../CubeSpace-proto-v1 -b feat/proto-v1
cd ../CubeSpace-proto-v1
```

- [ ] **Step 2: 생성기를 쓴다**

`spec/vectors/make_vectors.py`:

```python
"""규격 v1 공용 시험 벡터 생성기. 표준 라이브러리만 쓰는 독립 기준 구현이다.

docs/03-인터페이스/02-USB.md 2·3·5·7절, 01-BLE.md 2절, 04-시험-벡터.md를 그대로 옮겼다.
  python3 make_vectors.py          # proto_vectors.hex를 다시 만든다(Windows는 py make_vectors.py)
  python3 make_vectors.py --check  # 다시 만든 결과가 저장된 파일과 같은지 본다
줄 형식은 04-시험-벡터.md "벡터 파일 형식".
"""
import struct
import sys
import zlib
from pathlib import Path

OUT = Path(__file__).with_name('proto_vectors.hex')
I16_MIN = -32768

# USB 2절 본문 고정 부분 [(이름, struct 형)]. OBS 뒤에 v(i16 × h·w), LOG 뒤에 text가 붙는다.
# EVT 본문의 seq·type은 머리와 이름이 겹쳐 evt_seq·evt_type으로 적는다.
LAYOUT = {
    0x01: [('proto', 'B'), ('reset_reason', 'B'), ('fw', 'I'), ('frame_us', 'I'), ('bad_map', '10s')],
    0x02: [('flags', 'B'), ('led_scale', 'B'), ('led_seq', 'I'), ('tx_drop', 'H'), ('rx_bad', 'H'), ('overrun', 'H')],
    0x10: [('frame_no', 'I'), ('led_seq', 'I'), ('slot', 'B'), ('st', 'B'), ('r0', 'B'), ('c0', 'B'), ('h', 'B'), ('w', 'B')],
    0x11: [('slot', 'B'), ('cube_boot_id', 'I'), ('evt_seq', 'H'), ('evt_type', 'B'), ('v0', 'B'), ('v1', 'H'), ('t_pad32', 'I')],
    0x12: [('slot', 'B'), ('up', 'B'), ('reason', 'B'), ('mac', '6s')],
    0x13: [('mac', '6s'), ('rssi', 'b')],
    0x7E: [('acked_type', 'B'), ('acked_seq', 'I'), ('code', 'B')],
    0x7F: [('level', 'B')],
    0x81: [('proto', 'B')],
    0x82: [],
    0x90: [('rgb', '1152s')],
    0x91: [],
    0xA0: [('macs', '36s')],
    0xA1: [('on', 'B')],
    0xA2: [('slot', 'B'), ('rgb', '192s')],
}
CFG = [('ver', 'B'), ('slot', 'B'), ('pad_boot_id', 'I'), ('frame_us', 'I'), ('on_offset_us', 'I'), ('on_len_us', 'I'), ('bias_us', 'h')]
BEVT = [('cube_boot_id', 'I'), ('seq', 'H'), ('type', 'B'), ('v0', 'B'), ('v1', 'H'), ('t_pad32', 'I')]

lines = []


def fmt(v):
    if isinstance(v, bytes):
        return 'h:' + v.hex()
    if isinstance(v, list):
        return ','.join(map(str, v))
    return str(v)


def add(group, name, expect, data, **kv):
    fields = ''.join(f' {k}={fmt(v)}' for k, v in kv.items())
    lines.append(f'{group} {name} {expect} {data.hex() if data else "-"}{fields}')


def pack(layout, f):
    return struct.pack('<' + ''.join(t for _, t in layout), *(f[n] for n, _ in layout))


def frame(t, boot, seq, t_us, body=b''):
    p = struct.pack('<BIIQ', t, boot, seq, t_us) + body
    return p + struct.pack('<I', zlib.crc32(p))


def slip(b):
    return b'\xc0' + b.replace(b'\xdb', b'\xdb\xdd').replace(b'\xc0', b'\xdb\xdc') + b'\xc0'


def msg(name, t, expect='ok', group='frame', boot=0x11223344, seq=1, t_us=1000, **f):
    body = pack(LAYOUT[t], f)
    if t == 0x10:
        body += struct.pack(f'<{len(f["v"])}h', *f['v'])
    if t == 0x7F:
        body += f['text']
    raw = frame(t, boot, seq, t_us, body)
    add(group, name, expect, raw, type=t, boot=boot, seq=seq, t_us=t_us, **f)
    return raw


def spaced(b):
    return b.hex(' ').upper()


# CRC
assert zlib.crc32(b'123456789') == 0xCBF43926
add('crc', 'check', 'ok', b'123456789', crc=0xCBF43926)

# 04-시험-벡터.md 계산 예 3개
assert spaced(msg('doc_hb', 0x82, boot=0x11223344, seq=1, t_us=0)) == \
    '82 44 33 22 11 01 00 00 00 00 00 00 00 00 00 00 00 02 54 F4 5E'
assert spaced(msg('doc_reg_scan', 0xA1, boot=0x11223344, seq=2, t_us=0, on=1)) == \
    'A1 44 33 22 11 02 00 00 00 00 00 00 00 00 00 00 00 01 FF 43 5F 41'
doc_evt = msg('doc_evt', 0x11, boot=0x0A0B0C0D, seq=5, t_us=1000500,
              slot=0, cube_boot_id=0xDBC0A5A5, evt_seq=7, evt_type=0x01, v0=1, v1=3, t_pad32=1000000)
assert spaced(doc_evt) == ('11 0D 0C 0B 0A 05 00 00 00 34 44 0F 00 00 00 00 00 00 A5 A5 C0 DB 07 00 '
                           '01 01 03 00 40 42 0F 00 8C 95 85 74')
add('slip', 'doc_evt', 'ok', doc_evt, wire=slip(doc_evt))

# 메시지 15종 정상값. bad_map: 비트 0·2(행 0 열 0·2)와 76(행 6 열 10) → 하위 비트부터
msg('hello', 0x01, proto=1, reset_reason=1, fw=0x1A2B3C4D, frame_us=50000,
    bad_map=bytes([0b101, 0, 0, 0, 0, 0, 0, 0, 0, 0b10000]))
msg('status', 0x02, flags=0b111, led_scale=255, led_seq=42, tx_drop=1, rx_bad=2, overrun=3)
TRACK = [5, 40, 38, 6, 20, 610, 580, 22, 4, 35, 30, 3]
msg('obs_track', 0x10, frame_no=1234, led_seq=42, slot=2, st=2 | 4 | 8, r0=2, c0=4, h=3, w=4, v=TRACK)
disc = lambda c: [x for r in range(7) for x in (10 * r + c, I16_MIN, 10 * r + c + 1)]
msg('obs_disc', 0x10, frame_no=1235, led_seq=42, slot=2, st=2 | 4 | 8, r0=0, c0=4, h=7, w=3, v=disc(4))
msg('obs_disc_c0_0', 0x10, frame_no=1236, led_seq=42, slot=2, st=2 | 4 | 8, r0=0, c0=0, h=7, w=3, v=disc(0))
msg('obs_disc_c0_8', 0x10, frame_no=1237, led_seq=42, slot=2, st=2 | 4 | 8, r0=0, c0=8, h=7, w=3, v=disc(8))
msg('obs_disc_9', 0x10, frame_no=1238, led_seq=42, slot=2, st=2 | 4 | 8, r0=0, c0=9, h=7, w=1, v=[1, 2, 3, 4, 5, 6, 7])
msg('obs_not_read', 0x10, frame_no=1239, led_seq=0, slot=3, st=64, r0=0, c0=0, h=0, w=0, v=[])
msg('evt_btn_short', 0x11, slot=1, cube_boot_id=0x01020304, evt_seq=8, evt_type=0x06, v0=1, v1=999, t_pad32=2000000)
msg('evt_btn_long', 0x11, slot=1, cube_boot_id=0x01020304, evt_seq=9, evt_type=0x06, v0=2, v1=1000, t_pad32=2000001)
msg('evt_batt', 0x11, slot=1, cube_boot_id=0x01020304, evt_seq=10, evt_type=0x03, v0=0, v1=3900, t_pad32=0xFFFFFFFF)
msg('link', 0x12, slot=1, up=1, reason=0, mac=bytes.fromhex('AABBCCDDEEFF'))
msg('link_down', 0x12, slot=1, up=0, reason=0x11, mac=bytes.fromhex('AABBCCDDEEFF'))
msg('cand', 0x13, mac=bytes.fromhex('112233445566'), rssi=-70)
msg('ack', 0x7E, acked_type=0xA2, acked_seq=9, code=5)
msg('log', 0x7F, level=3, text='패드 부팅'.encode())
msg('log_bad_utf8', 0x7F, level=2, text=b'\xff\xfe')
msg('log_200', 0x7F, level=3, text=b'a' * 200)
msg('pc_hello', 0x81, proto=1)
msg('led_off', 0x91)
led = msg('led', 0x90, rgb=bytes((i * 7) % 256 for i in range(1152)))
assert b'\xc0' in led and b'\xdb' in led
add('slip', 'led', 'ok', led, wire=slip(led))
REG = bytes.fromhex('AABBCCDDEEFF' + '00' * 24 + '010203040506')
msg('reg', 0xA0, macs=REG)
msg('icon', 0xA2, slot=3, rgb=bytes(range(192)))

# 버림(USB 1절): 길이·CRC·모르는 type·본문 길이
hb = frame(0x82, 0x11223344, 3, 0)
add('frame', 'bad_crc', 'drop_crc', hb[:-1] + bytes([hb[-1] ^ 1]))
add('frame', 'short', 'drop_len', hb[:-1])
add('frame', 'unknown_type', 'drop_type', frame(0x55, 0x11223344, 4, 0))
add('frame', 'status_body_11', 'drop_body', frame(0x02, 0x11223344, 5, 0, bytes(11)))
add('frame', 'log_201', 'drop_body', frame(0x7F, 0x11223344, 7, 0, b'\x03' + b'a' * 201))
add('frame', 'obs_body_short', 'drop_body', frame(0x10, 0x11223344, 6, 0, struct.pack('<IIBBBBBB', 1, 0, 0, 2, 0, 0, 3, 4)))

# PC 값 검사 경계(USB 7절, 04-시험-벡터 표)
B = 'bad_field'
msg('obs_slot6', 0x10, B, frame_no=1, led_seq=0, slot=6, st=2, r0=2, c0=4, h=3, w=4, v=TRACK)
msg('obs_st_b7', 0x10, B, frame_no=1, led_seq=0, slot=2, st=0x82, r0=2, c0=4, h=3, w=4, v=TRACK)
msg('obs_r0_h', 0x10, B, frame_no=1, led_seq=0, slot=2, st=2, r0=5, c0=4, h=3, w=4, v=TRACK)
msg('obs_abs_v', 0x10, B, frame_no=1, led_seq=0, slot=2, st=2, r0=2, c0=4, h=3, w=4, v=TRACK[:11] + [3301])
msg('obs_i16min_track', 0x10, B, frame_no=1, led_seq=0, slot=2, st=2, r0=2, c0=4, h=3, w=4, v=TRACK[:11] + [I16_MIN])
msg('obs_slot_ff', 0x10, B, frame_no=1, led_seq=0, slot=0xFF, st=2, r0=2, c0=4, h=3, w=4, v=TRACK)
msg('obs_window_off_pad', 0x10, B, frame_no=1, led_seq=0, slot=2, st=1, r0=2, c0=4, h=3, w=4, v=TRACK)
msg('evt_type7', 0x11, B, slot=0, cube_boot_id=1, evt_seq=1, evt_type=0x07, v0=0, v1=0, t_pad32=0)
msg('evt_hello_v0_2', 0x11, B, slot=0, cube_boot_id=1, evt_seq=1, evt_type=0x05, v0=2, v1=0x0100, t_pad32=0)
msg('evt_btn_long_999', 0x11, B, slot=0, cube_boot_id=1, evt_seq=1, evt_type=0x06, v0=2, v1=999, t_pad32=0)
msg('evt_batt_2499', 0x11, B, slot=0, cube_boot_id=1, evt_seq=1, evt_type=0x03, v0=0, v1=2499, t_pad32=0)
msg('hello_proto2', 0x01, B, proto=2, reset_reason=1, fw=1, frame_us=50000, bad_map=bytes(10))
msg('hello_frame_us', 0x01, B, proto=1, reset_reason=1, fw=1, frame_us=19999, bad_map=bytes(10))
msg('hello_bad_map_77', 0x01, B, proto=1, reset_reason=1, fw=1, frame_us=50000, bad_map=bytes(9) + b'\x20')
msg('status_flags_b5', 0x02, B, flags=0x20, led_scale=255, led_seq=0, tx_drop=0, rx_bad=0, overrun=0)
msg('link_up_reason', 0x12, B, slot=1, up=1, reason=0x10, mac=bytes(6))
msg('link_reason_5', 0x12, B, slot=1, up=0, reason=5, mac=bytes(6))
msg('cand_rssi_21', 0x13, B, mac=bytes(6), rssi=21)
msg('ack_type', 0x7E, B, acked_type=0x90, acked_seq=1, code=0)
msg('log_level0', 0x7F, B, level=0, text=b'x')
msg('boot0', 0x82, B, boot=0)

# 패드 값 검사(USB 7절 패드 표): ACK code 2
msg('reg_ok', 0xA0, 'ok', 'pad', macs=REG)
msg('reg_dup', 0xA0, 'ack2', 'pad', macs=bytes.fromhex('AABBCCDDEEFF' * 2 + '00' * 24))
msg('reg_scan_2', 0xA1, 'ack2', 'pad', on=2)
msg('icon_slot6', 0xA2, 'ack2', 'pad', slot=6, rgb=bytes(192))

# BLE 구조체(01-BLE 2절)
for name, f in (('cfg_slot2', dict(ver=1, slot=2, pad_boot_id=0xCAFEF00D, frame_us=50000, on_offset_us=14460 + 5460 * 2, on_len_us=5460, bias_us=1000)),
                ('cfg_bias_neg', dict(ver=1, slot=5, pad_boot_id=1, frame_us=91667, on_offset_us=14460 + 5460 * 5, on_len_us=5460, bias_us=-250))):
    add('ble', name, 'ok', pack(CFG, f), **f)
f = dict(cube_boot_id=0xDBC0A5A5, seq=7, type=0x01, v0=1, v1=3, t_pad32=1000000)
add('ble', 'evt_contact', 'ok', pack(BEVT, f), **f)
assert struct.calcsize('<' + ''.join(t for _, t in CFG)) == 20 and struct.calcsize('<' + ''.join(t for _, t in BEVT)) == 14


# 시각 복원(USB 5절)
def restore(t_us, t32):
    if t32 == 0xFFFFFFFF:
        return t_us, 0
    d = ((t_us & 0xFFFFFFFF) - t32 + 2 ** 31) % 2 ** 32 - 2 ** 31
    return t_us - d, 1


for name, t_us, age in (('age0', 5000000000, 0), ('age1000', 5000000000, 1000), ('age_m1', 5000000000, -1),
                        ('age_m500', 5000000000, -500), ('near_2_31', 6442450954, 20)):
    t32 = (t_us - age) & 0xFFFFFFFF
    assert restore(t_us, t32) == (t_us - age, 1)
    add('time', name, 'ok', b'', t_us=t_us, t_pad32=t32, t_pad_us=t_us - age, t_valid=1)
add('time', 'invalid', 'ok', b'', t_us=5000000000, t_pad32=0xFFFFFFFF, t_pad_us=5000000000, t_valid=0)
assert restore(5000000000, 0x2A05EE18) == (4999999000, 1)                 # 04-시험-벡터 표

text = '# 규격 v1 공용 시험 벡터. make_vectors.py가 만든다(손으로 고치지 않는다).\n' + '\n'.join(lines) + '\n'
if '--check' in sys.argv:
    assert OUT.read_text(encoding='utf-8') == text, 'proto_vectors.hex가 make_vectors.py 결과와 다르다'
    print(f'OK {len(lines)} vectors')
else:
    OUT.write_text(text, encoding='utf-8', newline='\n')  # Windows 기본 cp949·CRLF를 피한다
    print(f'wrote {OUT.name}: {len(lines)} vectors')
```

- [ ] **Step 3: 실행해 벡터를 만든다**

생성기 안의 assert가 문서 계산 예 3개(HB·REG_SCAN·EVT 바이트), CRC 검사값, BLE 구조체 크기, 시각 복원 표를 대조한다. 하나라도 어긋나면 `AssertionError`로 멈춘다.

Run: `python3 spec/vectors/make_vectors.py`
Expected: `wrote proto_vectors.hex: 69 vectors`

Run: `python3 spec/vectors/make_vectors.py --check`
Expected: `OK 69 vectors`

- [ ] **Step 4: 문서에 줄 형식을 적는다**

벡터 위치(F4)는 이미 문서에 반영돼 있다(`CubeSpace/spec/vectors/`, CubeSpace-Firmware·CubeSpace-Client는 복사본). 다시 고치지 말고 맞는지만 본다:
- `docs/03-인터페이스/04-시험-벡터.md`의 위치 문장이 `spec/vectors/proto_vectors.hex`를 원본으로, Firmware `host_test/vectors/`·Client `test/vectors/`를 복사본으로 적고 있다
- `grep -rn "F4" docs/0*/*.md` 결과에 미정 F4 행이 없다(하드웨어 문서의 `F05-…` 같은 시험 이름은 상관없음)

어긋나면 이 Task에서 고치지 말고 팀에 알린다.

`docs/03-인터페이스/04-시험-벡터.md`의 위치 문장 바로 뒤에 아래 문단을 넣는다(이미 같은 문단이 있으면 건너뛴다):

```text
벡터 파일 형식(10/11 확정):

- 한 줄 한 벡터: `<묶음> <이름> <기대> <16진 바이트 또는 -> [키=값 ...]`. 칸은 공백 하나로 나누고, `#`로 시작하는 줄은 주석이다.
- 묶음: `crc`(CRC 입력), `frame`(복원 프레임, SLIP 전), `slip`(복원 프레임 + `wire=` 선로 바이트), `ble`(CFG 20 B·EVT 14 B), `pad`(패드 인자 검사용 PC→패드 프레임), `time`(시각 복원, 바이트 칸은 `-`).
- 기대: `ok`, `bad_field`(해독은 되고 PC 값 검사에 걸림), `drop_len`·`drop_crc`·`drop_type`·`drop_body`(USB 1절 버림), `ack2`(패드 인자 검사 ACK code 2).
- 값: 정수는 10진(음수 포함), 바이트열은 `h:16진`, OBS `v`는 쉼표로 나눈 10진(빈 창은 빈 값). 키는 [USB 2](02-USB.md) 필드 이름 그대로, 머리는 `type`·`boot`·`seq`·`t_us`. EVT 본문의 `seq`·`type`은 머리와 겹쳐 `evt_seq`·`evt_type`, REG 본문 36 B는 `macs`.
```

- [ ] **Step 5: CI와 무시 목록**

`.github/workflows/ci.yml`의 `jobs:` 아래, 기존 작업 뒤에 붙인다:

```yaml
  vectors:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: python3 spec/vectors/make_vectors.py --check
```

`.gitignore` 끝에 붙인다:

```text
__pycache__/
```

- [ ] **Step 6: 커밋**

```bash
git add spec/vectors docs/03-인터페이스/04-시험-벡터.md .github/workflows/ci.yml .gitignore
git commit -m "feat(proto): 공용 시험 벡터 생성기와 벡터 파일 형식"
```

---

### Task 2: PC 코덱 `proto.ts`

**Files:**(CubeSpace-Client 저장소)
- Create: `src/core/gateway/proto.ts`, `src/core/gateway/index.ts`
- Copy: `test/vectors/proto_vectors.hex`(CubeSpace Task 1 결과 그대로)
- Test: `test/proto.test.ts`

**Interfaces:**
- Consumes: Task 1의 `proto_vectors.hex`(형식은 Task 1 Interfaces)
- Produces(`src/core/gateway/index.ts`에서 내보냄):
  - 상수 `PROTO`, `HEADER_LEN`, `MIN_FRAME`, `MAX_FRAME`, `INT16_MIN`, `BATT_MV_MIN`, `BATT_MV_MAX`, `T`(메시지 type 15개)
  - `type Value = number | number[] | Uint8Array`
  - `interface Frame { type: number; boot: number; seq: number; t_us: bigint; body: Record<string, Value> }`
  - `type Drop = 'len' | 'crc' | 'type' | 'body'`, `type Decoded = { ok: true; frame: Frame } | { ok: false; drop: Drop }`
  - `decodeFrame(buf: Uint8Array): Decoded`, `encodeFrame(f: Frame): Uint8Array`
  - `slipEncode(frame: Uint8Array): Uint8Array`, `class SlipDecoder { push(chunk: Uint8Array): Uint8Array[] }`
  - `restorePadTime(t_us: bigint, t_pad32: number): { t_pad_us: bigint; t_valid: boolean }`
  - `checkFields(f: Frame): string | null`(null = 통과, 아니면 어긴 항목 이름)

- [ ] **Step 1: 브랜치를 만들고 벡터 파일을 복사한다**

CubeSpace-Client 폴더에서 한 줄씩(`<CubeSpace>`는 Task 1의 worktree `CubeSpace-proto-v1` 경로):

```text
git switch -c feat/proto-v1
mkdir -p test/vectors
cp <CubeSpace>/spec/vectors/proto_vectors.hex test/vectors/
git -C <CubeSpace> rev-parse --short HEAD
pnpm install
```

Windows PowerShell은 `mkdir -p` 대신 `New-Item -ItemType Directory -Force test/vectors`, `cp` 대신 `Copy-Item`. 마지막 `rev-parse` 출력(원본 CubeSpace 커밋)은 Step 7 커밋 메시지에 쓴다. 복사한 파일은 손으로 고치지 않는다.

- [ ] **Step 2: 실패하는 시험을 쓴다**

`test/proto.test.ts`:

```ts
import { readFileSync } from 'node:fs';
import { describe, expect, it } from 'vitest';
import { SlipDecoder, checkFields, decodeFrame, encodeFrame, restorePadTime, slipEncode, type Value } from '../src/core/gateway/proto';

// 펌웨어 host_test와 같은 파일(줄 형식: docs/03-인터페이스/04-시험-벡터.md "벡터 파일 형식")
const FILE = new URL('./vectors/proto_vectors.hex', import.meta.url);
const hex = (s: string) => Uint8Array.from(s.match(/../g) ?? [], (h) => parseInt(h, 16));
const vectors = readFileSync(FILE, 'utf8').split(/\r?\n/).filter((l) => l && !l.startsWith('#')).map((l) => {
  const [group, name, expect, data, ...kv] = l.split(' ');
  return { group, name, expect, data: data === '-' ? new Uint8Array() : hex(data), kv: Object.fromEntries(kv.map((p) => p.split('='))) as Record<string, string> };
});
const of = (g: string) => vectors.filter((v) => v.group === g);

// 벡터 문자열을 해독 결과와 같은 형으로
function want(s: string, got: Value | bigint | undefined): Value | bigint {
  if (s.startsWith('h:')) return hex(s.slice(2));
  if (Array.isArray(got)) return s === '' ? [] : s.split(',').map(Number);
  return typeof got === 'bigint' ? BigInt(s) : Number(s);
}

it('벡터 파일을 읽었다', () => {
  expect(of('frame').length).toBeGreaterThanOrEqual(50);
});

describe('USB 프레임 벡터', () => {
  for (const v of of('frame')) {
    it(`${v.name} → ${v.expect}`, () => {
      const r = decodeFrame(v.data);
      if (v.expect.startsWith('drop_')) return expect(r).toEqual({ ok: false, drop: v.expect.slice(5) });
      if (!r.ok) throw new Error(`버려짐: ${r.drop}`);
      const { body, ...head } = r.frame;
      for (const [k, s] of Object.entries(v.kv)) {
        const got = k in head ? head[k as keyof typeof head] : body[k];
        expect(got, k).toEqual(want(s, got));
      }
      expect(encodeFrame(r.frame)).toEqual(v.data);
      expect(checkFields(r.frame) === null ? 'ok' : 'bad_field').toBe(v.expect);
    });
  }
});

describe('SLIP 벡터', () => {
  for (const v of of('slip')) {
    it(v.name, () => {
      const wire = hex(v.kv.wire.slice(2));
      expect(slipEncode(v.data)).toEqual(wire);
      const d = new SlipDecoder();
      expect([...wire].flatMap((x) => d.push(Uint8Array.of(x)))).toEqual([v.data]); // 1바이트씩 넣어도 같다
    });
  }

  it('C0 C0는 건너뛰고, 1200 B를 넘는 쓰레기는 쌓지 않고 len으로 버려진다', () => {
    const d = new SlipDecoder();
    const out = d.push(Uint8Array.of(0xc0, 0xc0, ...new Uint8Array(5000).fill(1), 0xc0));
    expect(out).toHaveLength(1);
    expect(out[0].length).toBe(1201);
    expect(decodeFrame(out[0])).toEqual({ ok: false, drop: 'len' });
  });

  it('조각 경계에 걸린 DB 이스케이프도 이어 붙는다', () => {
    const d = new SlipDecoder();
    expect(d.push(Uint8Array.of(0xc0, 0x01, 0xdb))).toEqual([]);
    expect(d.push(Uint8Array.of(0xdc, 0xc0))).toEqual([Uint8Array.of(0x01, 0xc0)]);
  });
});

describe('시각 복원 벡터', () => {
  for (const v of of('time')) {
    it(v.name, () => {
      expect(restorePadTime(BigInt(v.kv.t_us), Number(v.kv.t_pad32))).toEqual({ t_pad_us: BigInt(v.kv.t_pad_us), t_valid: v.kv.t_valid === '1' });
    });
  }
});

it('본문 바이트열 길이가 틀리면 부호화하지 않는다', () => {
  expect(() => encodeFrame({ type: 0xa2, boot: 1, seq: 1, t_us: 0n, body: { slot: 0, rgb: new Uint8Array(191) } })).toThrow(RangeError);
});
```

- [ ] **Step 3: 실패를 확인한다**

Run(CubeSpace-Client 폴더): `pnpm exec vitest run test/proto.test.ts`
Expected: FAIL. `../src/core/gateway/proto`를 찾지 못한다는 오류

- [ ] **Step 4: 코덱을 쓴다**

`src/core/gateway/proto.ts`:

```ts
// USB 규격 v1 부호화·해독(docs/03-인터페이스/02-USB.md 1·2·3·5·7절). 순수 코드: 입출력·시각·난수 없음.
import { crc32 } from 'node:zlib';

export const PROTO = 1;
export const HEADER_LEN = 17;
export const MIN_FRAME = 21;
export const MAX_FRAME = 1200;
export const INT16_MIN = -32768;
// [실측 필요] 큐브 전압 검사 범위(USB 7절). 바뀌면 여기만 고친다.
export const BATT_MV_MIN = 2500;
export const BATT_MV_MAX = 4500;

export const T = {
  HELLO: 0x01, STATUS: 0x02, OBS: 0x10, EVT: 0x11, LINK: 0x12, CAND: 0x13, ACK: 0x7e, LOG: 0x7f,
  PC_HELLO: 0x81, HB: 0x82, LED: 0x90, LED_OFF: 0x91, REG: 0xa0, REG_SCAN: 0xa1, ICON: 0xa2,
} as const;

type Kind = 'u8' | 'i8' | 'u16' | 'u32' | number; // number = 그 길이의 바이트열
// 본문 고정 부분 [이름, 형]. OBS 뒤에 v(i16 × h·w), LOG 뒤에 text(나머지)가 붙는다.
// EVT의 seq·type은 머리와 이름이 겹쳐 evt_seq·evt_type으로 부른다.
const LAYOUT: Record<number, [string, Kind][]> = {
  [T.HELLO]: [['proto', 'u8'], ['reset_reason', 'u8'], ['fw', 'u32'], ['frame_us', 'u32'], ['bad_map', 10]],
  [T.STATUS]: [['flags', 'u8'], ['led_scale', 'u8'], ['led_seq', 'u32'], ['tx_drop', 'u16'], ['rx_bad', 'u16'], ['overrun', 'u16']],
  [T.OBS]: [['frame_no', 'u32'], ['led_seq', 'u32'], ['slot', 'u8'], ['st', 'u8'], ['r0', 'u8'], ['c0', 'u8'], ['h', 'u8'], ['w', 'u8']],
  [T.EVT]: [['slot', 'u8'], ['cube_boot_id', 'u32'], ['evt_seq', 'u16'], ['evt_type', 'u8'], ['v0', 'u8'], ['v1', 'u16'], ['t_pad32', 'u32']],
  [T.LINK]: [['slot', 'u8'], ['up', 'u8'], ['reason', 'u8'], ['mac', 6]],
  [T.CAND]: [['mac', 6], ['rssi', 'i8']],
  [T.ACK]: [['acked_type', 'u8'], ['acked_seq', 'u32'], ['code', 'u8']],
  [T.LOG]: [['level', 'u8']],
  [T.PC_HELLO]: [['proto', 'u8']],
  [T.HB]: [],
  [T.LED]: [['rgb', 1152]],
  [T.LED_OFF]: [],
  [T.REG]: [['macs', 36]],
  [T.REG_SCAN]: [['on', 'u8']],
  [T.ICON]: [['slot', 'u8'], ['rgb', 192]],
};

export type Value = number | number[] | Uint8Array;
export interface Frame { type: number; boot: number; seq: number; t_us: bigint; body: Record<string, Value> }
export type Drop = 'len' | 'crc' | 'type' | 'body';
export type Decoded = { ok: true; frame: Frame } | { ok: false; drop: Drop };

const size = (k: Kind) => (typeof k === 'number' ? k : k === 'u32' ? 4 : k === 'u16' ? 2 : 1);
const fixedLen = (type: number) => LAYOUT[type].reduce((n, [, k]) => n + size(k), 0);

function bodyLenOk(type: number, b: Uint8Array): boolean {
  const n = fixedLen(type);
  if (type === T.OBS) return b.length >= n && b.length === n + 2 * b[12] * b[13];
  if (type === T.LOG) return b.length >= n && b.length <= n + 200;
  return b.length === n;
}

function readBody(type: number, b: Uint8Array): Record<string, Value> {
  const dv = new DataView(b.buffer, b.byteOffset, b.length);
  const out: Record<string, Value> = {};
  let o = 0;
  for (const [name, k] of LAYOUT[type]) {
    out[name] = k === 'u8' ? dv.getUint8(o) : k === 'i8' ? dv.getInt8(o) : k === 'u16' ? dv.getUint16(o, true)
      : k === 'u32' ? dv.getUint32(o, true) : b.slice(o, o + k);
    o += size(k);
  }
  if (type === T.OBS) out.v = Array.from({ length: (b.length - o) / 2 }, (_, i) => dv.getInt16(o + 2 * i, true));
  if (type === T.LOG) out.text = b.slice(o);
  return out;
}

// 복원 프레임(머리 ‖ 본문 ‖ CRC, SLIP 전) 하나를 해독한다. 버리는 이유는 USB 1절.
export function decodeFrame(buf: Uint8Array): Decoded {
  if (buf.length < MIN_FRAME || buf.length > MAX_FRAME) return { ok: false, drop: 'len' };
  const dv = new DataView(buf.buffer, buf.byteOffset, buf.length);
  const end = buf.length - 4;
  if (crc32(buf.subarray(0, end)) !== dv.getUint32(end, true)) return { ok: false, drop: 'crc' };
  const type = buf[0];
  if (!(type in LAYOUT)) return { ok: false, drop: 'type' };
  const b = buf.subarray(HEADER_LEN, end);
  if (!bodyLenOk(type, b)) return { ok: false, drop: 'body' };
  const frame = { type, boot: dv.getUint32(1, true), seq: dv.getUint32(5, true), t_us: dv.getBigUint64(9, true), body: readBody(type, b) };
  return { ok: true, frame };
}

export function encodeFrame(f: Frame): Uint8Array {
  const v = f.type === T.OBS ? (f.body.v as number[]) : [];
  const text = f.type === T.LOG ? (f.body.text as Uint8Array) : new Uint8Array();
  const buf = new Uint8Array(HEADER_LEN + fixedLen(f.type) + 2 * v.length + text.length + 4);
  const dv = new DataView(buf.buffer);
  dv.setUint8(0, f.type);
  dv.setUint32(1, f.boot, true);
  dv.setUint32(5, f.seq, true);
  dv.setBigUint64(9, f.t_us, true);
  let o = HEADER_LEN;
  for (const [name, k] of LAYOUT[f.type]) {
    const x = f.body[name];
    if (k === 'u8') dv.setUint8(o, x as number);
    else if (k === 'i8') dv.setInt8(o, x as number);
    else if (k === 'u16') dv.setUint16(o, x as number, true);
    else if (k === 'u32') dv.setUint32(o, x as number, true);
    else if ((x as Uint8Array).length !== k) throw new RangeError(`${name}: ${k} B여야 한다`);
    else buf.set(x as Uint8Array, o);
    o += size(k);
  }
  for (const x of v) { dv.setInt16(o, x, true); o += 2; }
  buf.set(text, o);
  o += text.length;
  dv.setUint32(o, crc32(buf.subarray(0, o)), true);
  return buf;
}

export function slipEncode(frame: Uint8Array): Uint8Array {
  const out = [0xc0];
  for (const x of frame) {
    if (x === 0xc0) out.push(0xdb, 0xdc);
    else if (x === 0xdb) out.push(0xdb, 0xdd);
    else out.push(x);
  }
  out.push(0xc0);
  return Uint8Array.from(out);
}

// 선로 바이트를 조각으로 받아 끝난 복원 프레임들을 돌려준다. C0 C0(빈 프레임)은 건너뛴다.
// MAX_FRAME + 1 B에서 더 쌓지 않아, 끝 없는 쓰레기도 메모리를 늘리지 않고 decodeFrame에서 'len'으로 버려진다.
export class SlipDecoder {
  private buf: number[] = [];
  private esc = false;

  push(chunk: Uint8Array): Uint8Array[] {
    const frames: Uint8Array[] = [];
    for (let x of chunk) {
      if (x === 0xc0) {
        if (this.buf.length) frames.push(Uint8Array.from(this.buf));
        this.buf = [];
        this.esc = false;
        continue;
      }
      if (this.esc) {
        x = x === 0xdc ? 0xc0 : x === 0xdd ? 0xdb : x;
        this.esc = false;
      } else if (x === 0xdb) {
        this.esc = true;
        continue;
      }
      if (this.buf.length <= MAX_FRAME) this.buf.push(x);
    }
    return frames;
  }
}

// EVT t_pad32 → 64비트 패드 시각(USB 5절 대칭식, 그 EVT 프레임 머리 t_us 기준).
export function restorePadTime(t_us: bigint, t_pad32: number): { t_pad_us: bigint; t_valid: boolean } {
  if (t_pad32 === 0xffffffff) return { t_pad_us: t_us, t_valid: false };
  return { t_pad_us: t_us - BigInt.asIntN(32, BigInt.asUintN(32, t_us) - BigInt(t_pad32)), t_valid: true };
}

// USB 7절 PC 값 검사. 통과면 null, 아니면 어긴 항목 이름. 같은 boot에서 t_us 역행은 세션 상태라 gateway 코어가 본다.
export function checkFields(f: Frame): string | null {
  if (f.boot === 0) return 'boot';
  const b = f.body as Record<string, number>;
  switch (f.type) {
    case T.HELLO:
      if (b.proto !== PROTO) return 'proto';
      if (b.frame_us < 20000 || b.frame_us > 200000) return 'frame_us';
      return (f.body.bad_map as Uint8Array)[9] & 0xe0 ? 'bad_map' : null;
    case T.STATUS:
      return b.flags & 0xe0 ? 'flags' : null;
    case T.OBS:
      return checkObs(b, f.body.v as number[]);
    case T.EVT:
      return b.slot > 5 ? 'slot' : checkEvt(b.evt_type, b.v0, b.v1);
    case T.LINK:
      if (b.slot > 5 || b.up > 1 || (b.up === 1 && b.reason !== 0)) return 'link';
      return [0, 1, 2, 3, 0x10, 0x11, 0x12].includes(b.reason) ? null : 'reason';
    case T.CAND:
      return b.rssi < -127 || b.rssi > 20 ? 'rssi' : null;
    case T.ACK:
      return [T.REG, T.REG_SCAN, T.ICON].includes(b.acked_type as 0xa0) && b.code <= 5 ? null : 'ack';
    case T.LOG:
      return b.level >= 1 && b.level <= 3 ? null : 'level'; // 잘못된 UTF-8은 표시할 때 U+FFFD로 바꾼다
    default:
      return null; // PC→패드 종류는 PC가 만든다
  }
}

function checkObs(b: Record<string, number>, v: number[]): string | null {
  if (b.slot > 5) return 'slot';
  if (b.st & 0x80) return 'st';
  const optical = b.st & 3;
  if (b.st & 0x40 && optical !== 0) return 'not_read';
  if (optical !== 2 && (b.h || b.w)) return 'window';
  if (b.h * b.w > 21 || b.r0 + b.h > 7 || b.c0 + b.w > 11) return 'window';
  const disc = b.h === 7 && b.w === 3; // 발견 7 × 3: 가운데 열 7칸만 INT16_MIN
  for (let i = 0; i < v.length; i++) {
    const mid = disc && i % 3 === 1;
    if (mid ? v[i] !== INT16_MIN : Math.abs(v[i]) > 3300) return 'v'; // INT16_MIN도 3300을 넘어 걸린다
  }
  return null;
}

function checkEvt(type: number, v0: number, v1: number): string | null {
  switch (type) {
    case 0x01: case 0x02: case 0x10: return v0 <= 1 ? null : 'v0';
    case 0x03: return v0 === 0 && v1 >= BATT_MV_MIN && v1 <= BATT_MV_MAX ? null : 'batt';
    case 0x04: return v0 >= 1 && v0 <= 3 ? null : 'v0';
    case 0x05: return v0 === 1 ? null : 'hello'; // 큐브 판 다름: 알림·EVT 버림은 gateway 코어
    case 0x06: // 끄기 시간은 큐브 상수라 길게의 상한은 보지 않는다(BLE 3절)
      return (v0 === 0 && v1 === 0) || (v0 === 1 && v1 < 1000) || (v0 === 2 && v1 >= 1000) ? null : 'btn';
    default: return 'evt_type';
  }
}
```

`src/core/gateway/index.ts`(새 파일):

```ts
// gateway: USB 프레임 검증·순번·재부팅·세션·시각 변환. 순수 모듈(입출력·Electron·시각 없음).
export * from './proto';
```

- [ ] **Step 5: 통과를 확인한다**

tsconfig의 `include`에 `src/core/**`와 `test/**`가 들어 있는지 먼저 본다 [실측 필요].

Run(CubeSpace-Client 폴더): `pnpm exec vitest run test/proto.test.ts`
Expected: `Tests  65 passed (65)`

Run(CubeSpace-Client 폴더): `pnpm check`
Expected: lint·typecheck·test 모두 실패 0(`test/rules.test.ts`가 있으면 순수 모듈 규칙 포함). ESLint 지적이 나오면 동작을 바꾸지 않는 범위에서 고치고 벡터 시험을 다시 돌린다 [실측 필요].

- [ ] **Step 6: USB 문서를 확인한다(CubeSpace)**

`docs/03-인터페이스/02-USB.md` 1절 SLIP 줄의 PC 칸이 `gateway`의 `slipEncode`·`SlipDecoder`인지 본다(문서에는 이미 반영돼 있다). 어긋나면 이 Task에서 고치지 말고 팀에 알린다.

- [ ] **Step 7: 커밋(저장소마다 따로)**

CubeSpace-Client(`<커밋>`은 Step 1의 `rev-parse` 출력):
```bash
git add src/core/gateway test/proto.test.ts test/vectors/proto_vectors.hex
git commit -m "feat(gateway): USB 규격 v1 코덱과 공용 벡터 시험(벡터: CubeSpace <커밋>)"
```


---

### Task 3: 펌웨어 코덱 `CubeSpaceProto`와 CI

**Files:**(CubeSpace-Firmware 저장소)
- Create: `libraries/CubeSpaceProto/src/x1_proto.h`, `proto.h`, `proto.cpp`
- Copy: `host_test/vectors/proto_vectors.hex`(CubeSpace Task 1 결과 그대로)
- Test: `host_test/Makefile`, `host_test/test_proto.cpp`
- Modify: `.github/workflows/host-test.yml`(없으면 새로), `.gitignore`

**Interfaces:**
- Consumes: Task 1의 `proto_vectors.hex`(복사본)
- Produces(`proto.h`·`x1_proto.h`, 펌웨어 계획 3·4가 쓴다):
  - 구조체 `cs_hdr_t`, `cs_hello_t`, `cs_status_t`, `cs_obs_t`(14 B, 뒤에 `v` i16 × h·w), `cs_evt_t`(`slot` + `x1_evt_t e`), `cs_link_t`, `cs_cand_t`, `cs_ack_t`, `cs_log_t`, `cs_pc_hello_t`, `cs_led_t`, `cs_reg_t`, `cs_reg_scan_t`, `cs_icon_t`, `x1_cfg_t`, `x1_evt_t`
  - 상수 `CS_PROTO`, `CS_HDR_LEN`, `CS_MIN_FRAME`, `CS_MAX_FRAME`, `CS_T_*`, `CS_OK`·`CS_DROP_LEN`·`CS_DROP_CRC`·`CS_DROP_TYPE`·`CS_DROP_BODY`, `X1_*_UUID`, `X1_CFG_VER`, `X1_ICON_LEN`, `X1_EVT_*`
  - `uint32_t cs_crc32(const uint8_t *p, size_t n)`
  - `size_t cs_frame_encode(uint8_t type, uint32_t boot, uint32_t seq, uint64_t t_us, const void *body, size_t body_len, uint8_t *out, size_t cap)`(자리가 모자라면 0)
  - `int cs_frame_decode(const uint8_t *buf, size_t len, cs_hdr_t *hdr, const uint8_t **body, size_t *body_len)`
  - `size_t cs_slip_encode(const uint8_t *in, size_t n, uint8_t *out, size_t cap)`, `cs_slip_rx_t`, `size_t cs_slip_push(cs_slip_rx_t *rx, uint8_t x)`
  - `uint8_t cs_pad_check(uint8_t type, const uint8_t *body)`(0 또는 ACK code 2)

- [ ] **Step 0: 브랜치를 만들고 벡터 파일을 복사한다**

CubeSpace-Firmware 폴더에서 한 줄씩(`<CubeSpace>`는 Task 1의 worktree `CubeSpace-proto-v1` 경로):

```text
git switch -c feat/proto-v1
mkdir -p host_test/vectors
cp <CubeSpace>/spec/vectors/proto_vectors.hex host_test/vectors/
git -C <CubeSpace> rev-parse --short HEAD
```

Windows PowerShell은 `mkdir -p` 대신 `New-Item -ItemType Directory -Force host_test/vectors`, `cp` 대신 `Copy-Item`. 마지막 `rev-parse` 출력(원본 CubeSpace 커밋)은 Step 8 커밋 메시지에 쓴다. 복사한 파일은 손으로 고치지 않는다.

- [ ] **Step 1: 실패하는 시험을 쓴다**

`host_test/Makefile`(들여쓰기는 탭):

```make
# 순수 모듈 호스트 시험(펌웨어 05 1.1). 모듈마다 assert 실행 파일 하나. 실행: make -C host_test
# Windows는 MSYS2 UCRT64 셸(pacman -S make mingw-w64-ucrt-x86_64-gcc) 또는 WSL2에서.
LIB = ../libraries/CubeSpaceProto/src
VEC = vectors/proto_vectors.hex
CXXFLAGS = -std=c++17 -Wall -Wextra -Werror -I$(LIB)

test: test_proto
	./test_proto $(VEC)

test_proto: test_proto.cpp $(LIB)/proto.cpp $(LIB)/proto.h $(LIB)/x1_proto.h
	$(CXX) $(CXXFLAGS) -o $@ test_proto.cpp $(LIB)/proto.cpp

clean:
	rm -f test_proto test_proto.exe

.PHONY: test clean
```

`host_test/test_proto.cpp`:

```cpp
// 공용 벡터(proto_vectors.hex)로 proto.cpp·x1_proto.h를 맞춘다. 줄 형식: docs/03-인터페이스/04-시험-벡터.md
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "proto.h"

static const char *cur = "";  // 지금 보는 벡터 이름(실패 메시지용)
#define CHECK(c) do { if (!(c)) { fprintf(stderr, "%s:%d [%s] %s\n", __FILE__, __LINE__, cur, #c); exit(1); } } while (0)

typedef struct { const char *k; size_t off, size; int sign; } fld_t;  // 구조체 안 필드 위치
#define NF(name, T, m, s) {name, offsetof(T, m), sizeof(((T *)0)->m), s}
#define U(T, m) NF(#m, T, m, 0)

static const fld_t HDR_F[] = {U(cs_hdr_t, type), U(cs_hdr_t, boot), U(cs_hdr_t, seq), U(cs_hdr_t, t_us), {}};
static const fld_t HELLO_F[] = {U(cs_hello_t, proto), U(cs_hello_t, reset_reason), U(cs_hello_t, fw),
                                U(cs_hello_t, frame_us), U(cs_hello_t, bad_map), {}};
static const fld_t STATUS_F[] = {U(cs_status_t, flags), U(cs_status_t, led_scale), U(cs_status_t, led_seq),
                                 U(cs_status_t, tx_drop), U(cs_status_t, rx_bad), U(cs_status_t, overrun), {}};
static const fld_t OBS_F[] = {U(cs_obs_t, frame_no), U(cs_obs_t, led_seq), U(cs_obs_t, slot), U(cs_obs_t, st),
                              U(cs_obs_t, r0), U(cs_obs_t, c0), U(cs_obs_t, h), U(cs_obs_t, w), {}};
static const fld_t EVT_F[] = {U(cs_evt_t, slot), NF("cube_boot_id", cs_evt_t, e.cube_boot_id, 0),
                              NF("evt_seq", cs_evt_t, e.seq, 0), NF("evt_type", cs_evt_t, e.type, 0),
                              NF("v0", cs_evt_t, e.v0, 0), NF("v1", cs_evt_t, e.v1, 0),
                              NF("t_pad32", cs_evt_t, e.t_pad32, 0), {}};
static const fld_t LINK_F[] = {U(cs_link_t, slot), U(cs_link_t, up), U(cs_link_t, reason), U(cs_link_t, mac), {}};
static const fld_t CAND_F[] = {U(cs_cand_t, mac), NF("rssi", cs_cand_t, rssi, 1), {}};
static const fld_t ACK_F[] = {U(cs_ack_t, acked_type), U(cs_ack_t, acked_seq), U(cs_ack_t, code), {}};
static const fld_t LOG_F[] = {U(cs_log_t, level), {}};
static const fld_t PCH_F[] = {U(cs_pc_hello_t, proto), {}};
static const fld_t LED_F[] = {U(cs_led_t, rgb), {}};
static const fld_t REG_F[] = {NF("macs", cs_reg_t, mac, 0), {}};
static const fld_t SCAN_F[] = {U(cs_reg_scan_t, on), {}};
static const fld_t ICON_F[] = {U(cs_icon_t, slot), U(cs_icon_t, rgb), {}};
static const fld_t NONE_F[] = {{}};
static const fld_t CFG_F[] = {U(x1_cfg_t, ver), U(x1_cfg_t, slot), U(x1_cfg_t, pad_boot_id), U(x1_cfg_t, frame_us),
                              U(x1_cfg_t, on_offset_us), U(x1_cfg_t, on_len_us), NF("bias_us", x1_cfg_t, bias_us, 1), {}};
static const fld_t BEVT_F[] = {U(x1_evt_t, cube_boot_id), U(x1_evt_t, seq), U(x1_evt_t, type), U(x1_evt_t, v0),
                               U(x1_evt_t, v1), U(x1_evt_t, t_pad32), {}};

static const fld_t *body_fields(uint8_t type) {
  switch (type) {
    case CS_T_HELLO: return HELLO_F;
    case CS_T_STATUS: return STATUS_F;
    case CS_T_OBS: return OBS_F;
    case CS_T_EVT: return EVT_F;
    case CS_T_LINK: return LINK_F;
    case CS_T_CAND: return CAND_F;
    case CS_T_ACK: return ACK_F;
    case CS_T_LOG: return LOG_F;
    case CS_T_PC_HELLO: return PCH_F;
    case CS_T_LED: return LED_F;
    case CS_T_REG: return REG_F;
    case CS_T_REG_SCAN: return SCAN_F;
    case CS_T_ICON: return ICON_F;
    default: return NONE_F;
  }
}

static const fld_t *find(const fld_t *t, const char *k) {
  for (; t->k; t++) if (!strcmp(t->k, k)) return t;
  return NULL;
}

static size_t unhex(const char *s, uint8_t *out, size_t cap) {
  size_t n = strlen(s) / 2;
  CHECK(n <= cap);
  for (size_t i = 0; i < n; i++) { unsigned x; CHECK(sscanf(s + 2 * i, "%2x", &x) == 1); out[i] = (uint8_t)x; }
  return n;
}

// 벡터 값 s(10진 정수 또는 h:16진 바이트열)와 base + f->off의 필드를 비교한다.
static void check_field(const uint8_t *base, const fld_t *f, const char *s) {
  if (!strncmp(s, "h:", 2)) {
    static uint8_t b[2048];
    size_t n = unhex(s + 2, b, sizeof b);
    CHECK(n == f->size && !memcmp(base + f->off, b, n));
    return;
  }
  uint64_t raw = 0;
  memcpy(&raw, base + f->off, f->size);
  int64_t got = (int64_t)raw;
  if (f->sign && f->size < 8) { int sh = 64 - 8 * (int)f->size; got = (int64_t)(raw << sh) >> sh; }
  if (got != strtoll(s, NULL, 10)) { fprintf(stderr, "[%s] %s: %lld != %s\n", cur, f->k, (long long)got, s); exit(1); }
}

// OBS v: 쉼표로 나눈 i16 목록
static void check_v(const uint8_t *body, size_t body_len, const char *s) {
  size_t n = (body_len - sizeof(cs_obs_t)) / 2, i = 0;
  for (char *p = (char *)s; *p; i++) {
    int16_t got;
    memcpy(&got, body + sizeof(cs_obs_t) + 2 * i, 2);
    CHECK(i < n && got == strtol(p, &p, 10));
    if (*p == ',') p++;
  }
  CHECK(i == n);
}

static int frame_vec(const char *expect, const uint8_t *d, size_t n, char **kv, int nkv) {
  static const char *names[] = {"ok", "drop_len", "drop_crc", "drop_type", "drop_body"};
  cs_hdr_t h;
  const uint8_t *body;
  size_t bl;
  int r = cs_frame_decode(d, n, &h, &body, &bl);
  if (!strncmp(expect, "drop_", 5)) { CHECK(!strcmp(names[r], expect)); return 0; }
  CHECK(r == CS_OK);  // bad_field도 해독은 된다(값 검사는 PC 몫)
  for (int i = 0; i < nkv; i++) {
    char *k = kv[i], *s = strchr(k, '=');
    *s++ = 0;
    if (!strcmp(k, "v")) { check_v(body, bl, s); continue; }
    if (!strcmp(k, "text")) {
      static uint8_t t[256];
      size_t tn = unhex(s + 2, t, sizeof t);
      CHECK(tn == bl - 1 && !memcmp(body + 1, t, tn));
      continue;
    }
    const fld_t *f = find(HDR_F, k);
    const uint8_t *base = (const uint8_t *)&h;
    if (!f) { f = find(body_fields(h.type), k); base = body; }
    CHECK(f != NULL);
    check_field(base, f, s);
  }
  static uint8_t out[CS_MAX_FRAME];
  CHECK(cs_frame_encode(h.type, h.boot, h.seq, h.t_us, body, bl, out, sizeof out) == n && !memcmp(out, d, n));
  return r;
}

int main(int argc, char **argv) {
  CHECK(argc == 2);
  FILE *fp = fopen(argv[1], "r");
  CHECK(fp != NULL);
  static char line[16384];
  static uint8_t d[CS_MAX_FRAME + 8], wire[2 * CS_MAX_FRAME + 2];
  int count[6] = {0};  // crc frame slip ble pad time
  while (fgets(line, sizeof line, fp)) {
    if (line[0] == '#' || line[0] == '\n' || line[0] == '\r') continue;
    char *tok[64];
    int nt = 0;
    for (char *p = strtok(line, " \r\n"); p && nt < 64; p = strtok(NULL, " \r\n")) tok[nt++] = p;
    CHECK(nt >= 4);
    const char *group = tok[0], *expect = tok[2];
    cur = tok[1];
    size_t n = strcmp(tok[3], "-") ? unhex(tok[3], d, sizeof d) : 0;
    char **kv = tok + 4;
    int nkv = nt - 4;
    if (!strcmp(group, "crc")) {
      CHECK(cs_crc32(d, n) == strtoul(kv[0] + 4, NULL, 10));
      count[0]++;
    } else if (!strcmp(group, "frame")) {
      frame_vec(expect, d, n, kv, nkv);
      count[1]++;
    } else if (!strcmp(group, "slip")) {
      size_t wn = unhex(kv[0] + 7, wire, sizeof wire);  // "wire=h:"
      static uint8_t enc[sizeof wire];
      CHECK(cs_slip_encode(d, n, enc, sizeof enc) == wn && !memcmp(enc, wire, wn));
      static cs_slip_rx_t rx;
      size_t got = 0;
      for (size_t i = 0; i < wn; i++) { size_t r = cs_slip_push(&rx, wire[i]); if (r) got = r; }
      CHECK(got == n && !memcmp(rx.buf, d, n));
      CHECK(cs_slip_encode(d, n, enc, wn - 1) == 0);  // 한 바이트 모자라면 쓰지 않는다
      count[2]++;
    } else if (!strcmp(group, "ble")) {
      const fld_t *t = !strncmp(cur, "cfg", 3) ? CFG_F : BEVT_F;
      CHECK(n == (t == CFG_F ? sizeof(x1_cfg_t) : sizeof(x1_evt_t)));
      for (int i = 0; i < nkv; i++) {
        char *k = kv[i], *s = strchr(k, '=');
        *s++ = 0;
        const fld_t *f = find(t, k);
        CHECK(f != NULL);
        check_field(d, f, s);
      }
      count[3]++;
    } else if (!strcmp(group, "pad")) {
      cs_hdr_t h;
      const uint8_t *body;
      size_t bl;
      CHECK(cs_frame_decode(d, n, &h, &body, &bl) == CS_OK);
      CHECK(cs_pad_check(h.type, body) == (strcmp(expect, "ack2") ? 0 : 2));
      count[4]++;
    } else {
      CHECK(!strcmp(group, "time"));  // 시각 복원은 PC 몫(펌웨어 tpad.h는 따로)
      count[5]++;
    }
  }
  fclose(fp);
  cur = "count";
  for (int i = 0; i < 6; i++) CHECK(count[i] > 0);
  // 끝 없는 쓰레기: 1201 B에서 멈추고 해독은 len
  static cs_slip_rx_t rx;
  for (int i = 0; i < 5000; i++) cs_slip_push(&rx, 1);
  size_t junk = cs_slip_push(&rx, 0xC0);
  cs_hdr_t h;
  const uint8_t *body;
  size_t bl;
  cur = "junk";
  CHECK(junk == CS_MAX_FRAME + 1 && cs_frame_decode(rx.buf, junk, &h, &body, &bl) == CS_DROP_LEN);
  cur = "encode_cap";
  static uint8_t small[CS_MIN_FRAME];
  CHECK(cs_frame_encode(CS_T_HB, 1, 1, 0, NULL, 0, small, CS_MIN_FRAME - 1) == 0);  // 자리가 모자라면 쓰지 않는다
  CHECK(cs_frame_encode(CS_T_HB, 1, 1, 0, NULL, 0, small, CS_MIN_FRAME) == CS_MIN_FRAME);
  printf("proto: crc %d, frame %d, slip %d, ble %d, pad %d vectors OK\n", count[0], count[1], count[2], count[3], count[4]);
  return 0;
}
```

- [ ] **Step 2: 실패를 확인한다**

Run(CubeSpace-Firmware 폴더): `make -C host_test`
Expected: FAIL. `../libraries/CubeSpaceProto/src/proto.cpp`를 만들 규칙이 없다는 make 오류

- [ ] **Step 3: BLE 구조체를 쓴다**

`libraries/CubeSpaceProto/src/x1_proto.h`:

```c
// BLE 규격 v1(docs/03-인터페이스/01-BLE.md 2·3절). 큐브·패드 펌웨어가 같이 쓴다. 모든 정수는 리틀 엔디언.
#pragma once
#include <stdint.h>

#define X1_SVC_UUID  "92707afe-9ddd-4585-8e49-0b307dcd400e"
#define X1_SYNC_UUID "79532248-5f32-4b11-a3be-8af7e558bc3a"
#define X1_CFG_UUID  "40d2c3b6-2a89-4611-a088-412719dd8c08"
#define X1_EVT_UUID  "5df28f48-7cb1-46e7-9f86-85c031b2488c"
#define X1_ICON_UUID "a8243c3b-780f-46e0-8283-f8630da1e8b6"

#define X1_CFG_VER 1
#define X1_ICON_LEN 192

enum { X1_EVT_CONTACT = 0x01, X1_EVT_SYNC = 0x02, X1_EVT_BATT = 0x03, X1_EVT_BYE = 0x04,
       X1_EVT_HELLO = 0x05, X1_EVT_BTN = 0x06, X1_EVT_RAW_S1 = 0x10 };

typedef struct __attribute__((packed)) {
  uint8_t ver, slot;
  uint32_t pad_boot_id, frame_us, on_offset_us, on_len_us;
  int16_t bias_us;
} x1_cfg_t;

typedef struct __attribute__((packed)) {
  uint32_t cube_boot_id;
  uint16_t seq;
  uint8_t type, v0;
  uint16_t v1;
  uint32_t t_pad32;
} x1_evt_t;

static_assert(sizeof(x1_cfg_t) == 20 && sizeof(x1_evt_t) == 14, "BLE 2절 크기");
static_assert(__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__, "리틀 엔디언 기기·호스트만");
```

- [ ] **Step 4: USB 코덱을 쓴다**

`libraries/CubeSpaceProto/src/proto.h`:

```c
// USB 규격 v1(docs/03-인터페이스/02-USB.md 1·2·7절). 패드 펌웨어와 호스트 시험(g++)이 같이 쓴다.
#pragma once
#include <stddef.h>
#include <stdint.h>
#include "x1_proto.h"

#define CS_PROTO 1
#define CS_HDR_LEN 17
#define CS_MIN_FRAME 21
#define CS_MAX_FRAME 1200

enum {
  CS_T_HELLO = 0x01, CS_T_STATUS = 0x02, CS_T_OBS = 0x10, CS_T_EVT = 0x11, CS_T_LINK = 0x12,
  CS_T_CAND = 0x13, CS_T_ACK = 0x7E, CS_T_LOG = 0x7F,
  CS_T_PC_HELLO = 0x81, CS_T_HB = 0x82, CS_T_LED = 0x90, CS_T_LED_OFF = 0x91,
  CS_T_REG = 0xA0, CS_T_REG_SCAN = 0xA1, CS_T_ICON = 0xA2,
};
// cs_frame_decode 결과. 이유는 USB 1절 "받는 쪽 처리".
enum { CS_OK = 0, CS_DROP_LEN, CS_DROP_CRC, CS_DROP_TYPE, CS_DROP_BODY };

typedef struct __attribute__((packed)) { uint8_t type; uint32_t boot, seq; uint64_t t_us; } cs_hdr_t;
typedef struct __attribute__((packed)) { uint8_t proto, reset_reason; uint32_t fw, frame_us; uint8_t bad_map[10]; } cs_hello_t;
typedef struct __attribute__((packed)) { uint8_t flags, led_scale; uint32_t led_seq; uint16_t tx_drop, rx_bad, overrun; } cs_status_t;
typedef struct __attribute__((packed)) { uint32_t frame_no, led_seq; uint8_t slot, st, r0, c0, h, w; } cs_obs_t;  // 뒤에 v i16 × h·w
typedef struct __attribute__((packed)) { uint8_t slot; x1_evt_t e; } cs_evt_t;   // BLE EVT 14 B 그대로
typedef struct __attribute__((packed)) { uint8_t slot, up, reason, mac[6]; } cs_link_t;
typedef struct __attribute__((packed)) { uint8_t mac[6]; int8_t rssi; } cs_cand_t;
typedef struct __attribute__((packed)) { uint8_t acked_type; uint32_t acked_seq; uint8_t code; } cs_ack_t;
typedef struct __attribute__((packed)) { uint8_t level; } cs_log_t;  // 뒤에 text ≤ 200 B
typedef struct __attribute__((packed)) { uint8_t proto; } cs_pc_hello_t;
typedef struct __attribute__((packed)) { uint8_t rgb[1152]; } cs_led_t;
typedef struct __attribute__((packed)) { uint8_t mac[6][6]; } cs_reg_t;
typedef struct __attribute__((packed)) { uint8_t on; } cs_reg_scan_t;
typedef struct __attribute__((packed)) { uint8_t slot, rgb[X1_ICON_LEN]; } cs_icon_t;

static_assert(sizeof(cs_hdr_t) == 17 && sizeof(cs_hello_t) == 20 && sizeof(cs_status_t) == 12 &&
               sizeof(cs_obs_t) == 14 && sizeof(cs_evt_t) == 15 && sizeof(cs_link_t) == 9 &&
               sizeof(cs_cand_t) == 7 && sizeof(cs_ack_t) == 6 && sizeof(cs_reg_t) == 36 &&
               sizeof(cs_icon_t) == 193 && sizeof(cs_led_t) == 1152, "USB 2절 크기");

// CRC-32(zlib과 같은 값).
uint32_t cs_crc32(const uint8_t *p, size_t n);
// 복원 프레임(머리 ‖ 본문 ‖ CRC)을 out에 쓰고 길이를 돌려준다. cap·1200 B를 넘으면 0.
size_t cs_frame_encode(uint8_t type, uint32_t boot, uint32_t seq, uint64_t t_us,
                       const void *body, size_t body_len, uint8_t *out, size_t cap);
// 복원 프레임을 해독한다. CS_OK면 hdr·body(buf 안을 가리킴)·body_len을 채운다.
int cs_frame_decode(const uint8_t *buf, size_t len, cs_hdr_t *hdr, const uint8_t **body, size_t *body_len);
// SLIP 부호화(앞뒤 C0). cap이 모자라면 0.
size_t cs_slip_encode(const uint8_t *in, size_t n, uint8_t *out, size_t cap);

// SLIP 받기. 바이트를 하나씩 넣고, 프레임이 끝나면 그 길이(> 0)를 돌려준다(내용은 rx->buf, 다음 push 전까지 유효).
// 1201 B에서 더 쌓지 않아 끝 없는 쓰레기는 해독에서 CS_DROP_LEN이 된다.
typedef struct { uint8_t buf[CS_MAX_FRAME + 1]; size_t n; uint8_t esc; } cs_slip_rx_t;
size_t cs_slip_push(cs_slip_rx_t *rx, uint8_t x);

// USB 7절 패드 값 검사 중 인자 검사(REG 같은 MAC 두 슬롯, REG_SCAN on, ICON slot). 0 또는 ACK code 2.
// PC HELLO 전·링크 없음·처리 중은 세션 상태라 pad_pump가 본다.
uint8_t cs_pad_check(uint8_t type, const uint8_t *body);
```

`libraries/CubeSpaceProto/src/proto.cpp`:

```cpp
#include "proto.h"

#include <stddef.h>
#include <string.h>

// ponytail: 비트 단위 CRC, 240 MHz에서 LED 프레임 1173 B당 약 0.16 ms. X2-U2에서 펌프가 넘치면 표 방식으로.
uint32_t cs_crc32(const uint8_t *p, size_t n) {
  uint32_t c = 0xFFFFFFFFu;
  while (n--) {
    c ^= *p++;
    for (int k = 0; k < 8; k++) c = (c >> 1) ^ (0xEDB88320u & (0u - (c & 1u)));
  }
  return ~c;
}

// 종류별 본문 길이(USB 2절). 1 맞음, 0 틀림, -1 모르는 type.
static int body_len_ok(uint8_t type, const uint8_t *b, size_t n) {
  switch (type) {
    case CS_T_HELLO: return n == sizeof(cs_hello_t);
    case CS_T_STATUS: return n == sizeof(cs_status_t);
    case CS_T_OBS:
      return n >= sizeof(cs_obs_t) &&
             n == sizeof(cs_obs_t) + 2u * b[offsetof(cs_obs_t, h)] * b[offsetof(cs_obs_t, w)];
    case CS_T_EVT: return n == sizeof(cs_evt_t);
    case CS_T_LINK: return n == sizeof(cs_link_t);
    case CS_T_CAND: return n == sizeof(cs_cand_t);
    case CS_T_ACK: return n == sizeof(cs_ack_t);
    case CS_T_LOG: return n >= 1 && n <= 201;
    case CS_T_PC_HELLO: return n == sizeof(cs_pc_hello_t);
    case CS_T_HB: case CS_T_LED_OFF: return n == 0;
    case CS_T_LED: return n == sizeof(cs_led_t);
    case CS_T_REG: return n == sizeof(cs_reg_t);
    case CS_T_REG_SCAN: return n == sizeof(cs_reg_scan_t);
    case CS_T_ICON: return n == sizeof(cs_icon_t);
    default: return -1;
  }
}

size_t cs_frame_encode(uint8_t type, uint32_t boot, uint32_t seq, uint64_t t_us,
                       const void *body, size_t body_len, uint8_t *out, size_t cap) {
  size_t n = CS_HDR_LEN + body_len + 4;
  if (n > cap || n > CS_MAX_FRAME) return 0;
  cs_hdr_t h = {type, boot, seq, t_us};
  memcpy(out, &h, CS_HDR_LEN);
  if (body_len) memcpy(out + CS_HDR_LEN, body, body_len);
  uint32_t crc = cs_crc32(out, n - 4);
  memcpy(out + n - 4, &crc, 4);
  return n;
}

int cs_frame_decode(const uint8_t *buf, size_t len, cs_hdr_t *hdr, const uint8_t **body, size_t *body_len) {
  if (len < CS_MIN_FRAME || len > CS_MAX_FRAME) return CS_DROP_LEN;
  uint32_t crc;
  memcpy(&crc, buf + len - 4, 4);
  if (cs_crc32(buf, len - 4) != crc) return CS_DROP_CRC;
  memcpy(hdr, buf, CS_HDR_LEN);
  *body = buf + CS_HDR_LEN;
  *body_len = len - CS_HDR_LEN - 4;
  int ok = body_len_ok(hdr->type, *body, *body_len);
  return ok < 0 ? CS_DROP_TYPE : ok ? CS_OK : CS_DROP_BODY;
}

size_t cs_slip_encode(const uint8_t *in, size_t n, uint8_t *out, size_t cap) {
  size_t o = 0;
  if (cap < 2) return 0;
  out[o++] = 0xC0;
  for (size_t i = 0; i < n; i++) {
    uint8_t x = in[i];
    if (x == 0xC0 || x == 0xDB) {
      if (o + 3 > cap) return 0;
      out[o++] = 0xDB;
      out[o++] = x == 0xC0 ? 0xDC : 0xDD;
    } else {
      if (o + 2 > cap) return 0;
      out[o++] = x;
    }
  }
  out[o++] = 0xC0;
  return o;
}

size_t cs_slip_push(cs_slip_rx_t *rx, uint8_t x) {
  if (x == 0xC0) {
    size_t n = rx->n;
    rx->n = 0;
    rx->esc = 0;
    return n;
  }
  if (rx->esc) {
    x = x == 0xDC ? 0xC0 : x == 0xDD ? 0xDB : x;
    rx->esc = 0;
  } else if (x == 0xDB) {
    rx->esc = 1;
    return 0;
  }
  if (rx->n < sizeof rx->buf) rx->buf[rx->n++] = x;
  return 0;
}

uint8_t cs_pad_check(uint8_t type, const uint8_t *body) {
  static const uint8_t zero[6] = {};
  switch (type) {
    case CS_T_REG_SCAN: return body[0] <= 1 ? 0 : 2;
    case CS_T_ICON: return body[0] <= 5 ? 0 : 2;
    case CS_T_REG:
      for (int i = 0; i < 6; i++)
        for (int j = i + 1; j < 6; j++)
          if (memcmp(body + 6 * i, zero, 6) && !memcmp(body + 6 * i, body + 6 * j, 6)) return 2;
      return 0;
    default: return 0;
  }
}
```

- [ ] **Step 5: 통과를 확인한다**

Run(CubeSpace-Firmware 폴더): `make -C host_test`
Expected: `proto: crc 1, frame 53, slip 2, ble 3, pad 4 vectors OK`

- [ ] **Step 6: CI와 무시 목록**

CubeSpace-Firmware `.github/workflows/host-test.yml`(이미 CI 파일이 있으면 그 `jobs:` 아래에 `host-test` 작업만 붙인다):

```yaml
name: host-test

on:
  push:
  pull_request:

jobs:
  host-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: make -C host_test
```

벡터 재생성 검사(`--check`)는 CubeSpace CI(Task 1 Step 5)가 맡는다.

`.gitignore` 끝에 붙인다:

```text
host_test/test_proto
host_test/test_proto.exe
```

- [ ] **Step 7: 문서를 확인한다**

`docs/03-인터페이스/02-USB.md` 1절 CRC 줄의 패드 칸이 `cs_crc32`(proto.cpp)이고, `04-시험-벡터.md` CRC 줄이 "PC·펌웨어 같은 벡터"인지 본다(문서에는 이미 반영돼 있다).

- [ ] **Step 8: 커밋하고 CI(gcc)를 본다(푸시는 사용자 확인 뒤)**

CubeSpace-Firmware 폴더(커밋 메시지 끝에 Step 0의 원본 CubeSpace 커밋을 적는다):
```bash
git add libraries/CubeSpaceProto/src host_test .github/workflows/host-test.yml .gitignore
git commit -m "feat: CubeSpaceProto 코덱과 호스트 시험, CI (벡터: CubeSpace <커밋>)"
git push -u origin feat/proto-v1
gh run watch
```

Expected: `host-test` 작업 성공. 이 작업이 처음으로 g++에서 `-Werror` 빌드를 한다. 경고가 나오면 이 Task 안에서 고치고 다시 커밋한다.


---

## 끝난 기준

- CubeSpace: `python3 spec/vectors/make_vectors.py --check` → `OK 69 vectors`, CI `vectors` 성공
- CubeSpace-Client: `pnpm check` → 실패 0
- CubeSpace-Firmware: `make -C host_test` → `… vectors OK`, CI `host-test` 성공
- Client `test/vectors/proto_vectors.hex`와 Firmware `host_test/vectors/proto_vectors.hex`가 CubeSpace `spec/vectors/proto_vectors.hex`와 바이트까지 같다
