# 0001. Edge ECU scheduling model

- Status: **Proposed** — 측정 대기 중 (Phase 0B)
- Date: 2026-08-20
- Phase: 0B

## Context

Front Edge ECU와 Rear Legacy ECU(STM32G474RE)의 실행 모델을 superloop으로 유지할지, RTOS를 도입할지 결정해야 합니다.

기존 문서에는 "FreeRTOS task architecture 도입"이 다음 단계로 기재돼 있었으나, 이는 측정 근거 없이 작성된 계획이었습니다. 반대로 "superloop으로 충분하다"는 결론도 아직 데이터가 없습니다.

**이 ADR은 Phase 0B 측정이 끝나기 전까지 결론을 확정하지 않습니다.**

## 알려진 제약 (측정 전 코드 인스펙션 결과)

현재 구현은 완전한 non-blocking superloop가 아닙니다.

| 위치 | 내용 | 최악 blocking |
|---|---|---|
| `firmware/front-zonal-ecu/App/Src/input_service_stm32.c:56` | `HAL_ADC_PollForConversion(&hadc1, 10U)` | 10 ms |
| `firmware/front-zonal-ecu/Drivers/VL53L0X/Src/vl53l0x_i2c_platform.c:26` | `VL53L0X_I2C_TIMEOUT_MS = 100U`, `HAL_I2C_Mem_Read/Write` | 호출당 100 ms |
| `firmware/front-zonal-ecu/App/Src/distance_sensor_service.c` | 상태머신은 단계형이나 개별 I2C 호출은 blocking | 위와 동일 |

관련 요구사항 주기:

| 항목 | 주기 / 데드라인 |
|---|---|
| Front Status 송신 | 100 ms (`COM-002`) |
| ADC 샘플링 | 10 ms 이상 간격 (`IN-001`) |
| Front Status RX timeout | 300 ms (`COM-008`) |
| Central Command timeout | 500 ms (`SAFE-001`) |

**I2C fault 한 번(100 ms)이 100 ms 상태 송신 주기와 같은 크기입니다.** 따라서 "마진이 충분하다"는 주장은 정상 조건에서만 성립할 가능성이 있으며, 고장 조건 측정이 필수입니다.

## Measurement

> **작성 대기.** Phase 0B에서 채웁니다.

### 측정 계획

| 항목 | 방법 |
|---|---|
| ADC 호출 시간 | GPIO toggle + 로직 애널라이저, 또는 DWT cycle counter |
| I2C 호출 시간 (정상) | 동일 |
| I2C 호출 시간 (고장) | 동일 + fault injection |
| Superloop 1회전 시간 분포 | 루프 진입/이탈 GPIO toggle |
| Front Status 송신 주기 jitter | 송신 시점 GPIO toggle, 100 ms 기준 편차 |
| Deadline miss 횟수 | 주기 초과 카운트 |

### Fault injection 방법

**Phase 0B 착수 전에 아래 중 하나를 선택하고 재현 절차를 확정합니다.**

- [ ] VL53L0X 전원 차단 (센서 미응답)
- [ ] SDA/SCL 개방 (버스 단선)
- [ ] 미응답 I2C 주소로 전환 (소프트웨어 주입)

### 보고 형식

§5 공통 불변조건 #6에 따라 **평균만 보고하지 않습니다.** sample 수와 함께 p50 / p95 / p99 / p99.9 / max, deadline miss count를 기록하고, 원시 데이터(CSV)와 생성 방법을 함께 보존합니다.

## Analysis

> 작성 대기.

## Options

측정 결과에 따라 다음 중에서 선택합니다. 사전에 하나를 배제하지 않습니다.

| # | 대안 | 예상 trade-off |
|---|---|---|
| A | superloop 유지 + I2C timeout 단축 | 가장 적은 변경. timeout을 줄이면 느린 센서 응답을 실패로 오판할 위험 |
| B | superloop 유지 + I2C를 IT/DMA 비동기 전환 | 근본 해결(블로킹 제거). VL53L0X 벤더 API가 동기 호출 전제라 platform 계층 재작성 필요 |
| C | superloop 유지 + 센서 접근을 별도 시간 슬롯으로 격리 | 중간 비용. 최악 지연은 남지만 예측 가능해짐 |
| D | FreeRTOS 도입 | 블로킹을 낮은 우선순위 태스크에 격리. 공유 상태 동기화·스택·우선순위 역전이라는 새 문제 유입 |

**주의**: D를 선택하더라도 blocking 자체는 사라지지 않고 *격리*될 뿐입니다. 반대로 B는 RTOS 없이 blocking을 제거합니다.

## Decision

> **미확정.** Phase 0B 측정 완료 후 작성합니다.

## Consequences

> 작성 대기.

## References

- [docs/project-brief.md](../project-brief.md) §1.3, §3.2, §5
- [docs/requirements.md](../requirements.md) — `IN-001`, `COM-002`, `COM-008`, `SAFE-001`, `DIAG-003`
- [docs/development-log.md](../development-log.md) Milestone 6 — 인터럽트 타임스탬프 race 진단 사례
