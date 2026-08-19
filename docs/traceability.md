# Requirements Traceability

`requirements.md`의 각 요구사항이 **어떤 종류의 증거로** 검증되는지 매핑합니다.

## Evidence 종류

| 코드 | 종류 | 의미 |
|---|---|---|
| `UT` | Unit test | 호스트에서 실행되는 하드웨어 독립 단위테스트 (`Tests/*.c`) |
| `IT` | Integration test | 여러 모듈 또는 노드가 함께 동작하는 상태에서의 자동/반자동 테스트 |
| `HIL` | Board / HIL verification | 실제 보드에서 수행하는 회귀 체크리스트 (`verification.md`) |
| `AN` | Analysis / inspection | 코드 인스펙션, 계산, 설계 리뷰. 실행 가능한 테스트가 아님 |

`AN`은 **최소한으로만** 사용하고, 가능한 항목은 `UT`/`IT`/`HIL`로 승격시킵니다.

## 상태 표기

- ✅ 증거 존재
- 🟡 부분적 — 보강 필요
- ❌ 증거 없음

---

## Input and sensor (`IN`, `SEN`)

| ID | 요구사항 요약 | UT | IT | HIL | AN | 상태 |
|---|---|---|---|---|---|---|
| `IN-001` | ADC1_IN1 10 ms 이상 간격 sample | `input_service_test.c` | — | 보드 관찰 | — | ✅ |
| `IN-002` | 12-bit `0..4095` → `0..1000` permille | `input_service_test.c` | — | 가변저항 경계 시험 | — | ✅ |
| `IN-003` | ADC 실패 시 `adc_valid=0` + error counter | `input_service_test.c` | — | — | — | ✅ |
| `IN-004` | Zone switch 20 ms 디바운스 | `input_service_test.c` | — | 버튼 시험 | — | ✅ |
| `SEN-001` | I2C readiness와 API init 결과 분리 기록 | — | — | 보드 관찰 | — | 🟡 UT 없음 |
| `SEN-002` | 거리 측정이 superloop를 장시간 block하지 않음 | — | — | — | 코드 리뷰 | ❌ **Phase 0B에서 실측 필요** |
| `SEN-003` | 측정 100 ms 미완료 시 timeout counter + invalid | — | — | — | 구현 인스펙션 | 🟡 명시적 fault test 미작성 |

> `SEN-002`는 현재 `AN`만 있고, 코드 인스펙션 결과 **개별 I2C 호출이 blocking(최대 100 ms)** 임이 확인됐습니다. Phase 0B에서 `HIL` 측정으로 승격하고 [ADR-0001](adr/0001-scheduling-model-for-edge-nodes.md)에 반영합니다.

## Communication (`COM`)

| ID | 요구사항 요약 | UT | IT | HIL | AN | 상태 |
|---|---|---|---|---|---|---|
| `COM-001` | Front Status: ID `0x180`, 16-byte CAN FD | `can_protocol_test.c` | — | loopback | — | ✅ |
| `COM-002` | Front Status 주기 100 ms | — | — | 디버거 관찰 | — | 🟡 자동 검증 없음 |
| `COM-003` | Central Command: ID `0x200`, 8-byte | `can_protocol_test.c` | — | loopback | — | ✅ |
| `COM-004` | Central Simulator 200 ms 주기 | `central_sim_test.c` | — | loopback | — | ✅ (Phase 1에서 제거 예정) |
| `COM-005` | cyclic frame에 4-bit alive counter | `can_protocol_test.c` | — | — | — | ✅ |
| `COM-006` | length·range·alive counter 연속성 진단 | `can_protocol_test.c` | — | 디버거 카운터 | — | ✅ |
| `COM-007` | FDCAN RX를 FIFO0 인터럽트로 처리 | — | — | call stack, RX counter | — | 🟡 |
| `COM-008` | Front Status RX 300 ms 초과 시 timeout 기록 | — | — | fault 관찰 및 복구 시험 | — | 🟡 |

## Application and safety (`APP`, `SAFE`)

| ID | 요구사항 요약 | UT | IT | HIL | AN | 상태 |
|---|---|---|---|---|---|---|
| `APP-001` | ECU state는 `INIT`/`NORMAL`/`DEGRADED`/`SAFE` 중 하나 | `front_zonal_app_test.c`, `can_protocol_test.c` | — | — | — | ✅ |
| `APP-002` | 200 mm ON / 250 mm OFF 히스테리시스 | `central_sim_test.c` | — | LED 관찰 | — | ✅ (Phase 3에서 중앙 앱으로 이전) |
| `SAFE-001` | Command 500 ms 미수신 시 LED 출력 차단 | `front_zonal_app_test.c` | — | — | — | ✅ |
| `SAFE-002` | command timeout 동안 `SAFE` 상태 | `front_zonal_app_test.c` | — | — | — | ✅ |
| `SAFE-003` | 거리센서 미초기화 시 `INIT` | `front_zonal_app_test.c` | — | 부팅 관찰 | — | ✅ |
| `SAFE-004` | 센서 준비됐으나 측정 invalid → `DEGRADED` | `front_zonal_app_test.c` | — | — | — | ✅ |
| `SAFE-005` | fault injection 시 distance invalid, range 255, not-ready 보고 | `front_zonal_app_test.c` | — | — | — | ✅ |
| `SAFE-006` | status 300 ms 초과/invalid 시 warning off | `central_sim_test.c` | — | — | — | ✅ |

## Diagnostics (`DIAG`)

| ID | 요구사항 요약 | UT | IT | HIL | AN | 상태 |
|---|---|---|---|---|---|---|
| `DIAG-001` | encode/HAL TX/payload/sequence/timeout/unknown-ID 독립 counter | — | — | 디버거 Expressions | — | 🟡 UT 없음 |
| `DIAG-002` | 최근 CAN service 오류 원인을 enum으로 보존 | — | — | 디버거 Expressions | — | 🟡 UT 없음 |
| `DIAG-003` | 인터럽트 갱신 순서·tick rollover에서 false timeout 없음 | `time_utils_test.c` | — | 보드 회귀 | — | ✅ |

---

## 현재 공백 요약

| 우선순위 | 항목 | 조치 |
|---|---|---|
| 높음 | `SEN-002` — blocking 상한이 측정되지 않음 | **Phase 0B** |
| 중간 | `COM-002`, `COM-007`, `COM-008` — 자동 검증 없이 디버거 관찰에 의존 | Phase 1에서 `IT`로 승격 |
| 중간 | `DIAG-001`, `DIAG-002` — 디버거 관찰만 존재 | 카운터 로직의 UT 추가 검토 |
| 낮음 | `SEN-001` | 하드웨어 의존이라 `HIL` 유지 가능 |

## 유지 규칙

- 새 요구사항 추가 시 이 표에 행을 함께 추가합니다.
- 요구사항을 `AN`만으로 두지 말고, 가능한 한 `UT`/`IT`/`HIL`로 승격시킵니다.
- Phase 1에서 `central_sim`이 제거되면 `COM-004`, `APP-002`, `SAFE-006`의 검증 위치를 Pi 서비스 테스트로 이관하고 이 표를 갱신합니다.
