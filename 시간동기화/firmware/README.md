# firmware

ESP32-S3 스케치 세 개입니다. Arduino IDE로 굽습니다 (`USB CDC On Boot: Enabled`, 네이티브 `USB` 포트에 연결).

| 폴더 | 용도 | PPS 파형 | 부팅 배너 |
|---|---|---|---|
| `method2_gprmc_line1_handshake` | **원본 — 현재 사용 중** | 평소 LOW, 1초마다 20 ms HIGH | `BUILD` 줄 없음 |
| `method2_gprmc_pps_inverted` | 반전 회로(로우사이드 TR 등)용 사본 | 평소 HIGH, 1초마다 20 ms LOW | `# BUILD: PPS-INVERTED` |
| `method2_pps_level_diag` | 멀티미터 측정용 진단 사본 | 1 Hz 50% 사각파, `P1`/`P0`로 고정 | `# BUILD: PPS-50PCT-DIAG` |

사본은 원본에서 PPS 관련 부분만 바꿨고, 바뀐 곳마다 `[PPS 반전]` / `[진단]` 주석이 달려 있습니다. FSYNC·GPRMC·Line1·GO/STOP은 원본과 같습니다. **진단 사본은 PPS 규격이 달라 라이다 동기에 쓰면 안 됩니다.**

## 원본 펌웨어 요약

| 항목 | 값 |
|---|---|
| 타이머 | 10 Hz (`FREQ_HZ`), PPS는 10틱마다 (`PPS_DIVIDER`) |
| FSYNC | GPIO4, 1 ms HIGH, `GO` 이후에만 |
| PPS | GPIO5, 20 ms HIGH, 부팅 즉시 |
| GPRMC | GPIO17 (UART1), 9600 8N1, 부팅 즉시, `$…*XX\r\n` |
| 시리얼 | USB CDC 921600 — `READY`, `T`, `PPS`, `NMEA`, `EXP`, `CNT` |
| 명령 | `GO`, `STOP`, `PING`, `L`(Line1 진단), `R`(통계 리셋) |
| GPRMC 시각 | 부팅할 때마다 **12:00:00, 날짜 140826**부터 셈 (실제 세계 시각 아님) |

## 알려진 문제 — USB 출력이 막히면 PPS가 멈춤

PPS를 타이머 인터럽트에서 올리고 `loop()`의 `servicePulses()`에서 내립니다. PC가 시리얼을 읽지 않으면 `Serial.print`가 막혀 `loop()`가 멈추고, PPS가 HIGH에 머물러 라이다가 `Absent`가 됩니다. GPRMC 송신도 멈춥니다.

해결책(아직 적용 안 함): `setup()`에 한 줄.

```cpp
Serial.setTxTimeoutMs(0);   // USB 출력이 막혀도 기다리지 않고 버린다
```

더 근본적으로는 PPS·FSYNC의 하강도 `loop()`가 아니라 타이머(두 번째 알람 등)에서 처리하는 것이 맞습니다.

## 부팅 로그의 PSRAM 오류

UART 포트로 보면 `E (183) quad_psram: PSRAM chip is not connected, or wrong PSRAM line mode`가 찍힙니다. Arduino IDE의 PSRAM 설정이 보드와 안 맞아서 나오는 메시지로, 이 펌웨어는 PSRAM을 쓰지 않아 동작에는 영향이 없습니다. `Tools → PSRAM`을 보드에 맞추면(N16R8이면 `OPI PSRAM`, 없으면 `Disabled`) 사라집니다.
