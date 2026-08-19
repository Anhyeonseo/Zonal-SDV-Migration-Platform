# Project Brief — Automotive Zonal SDV Migration Platform

이 문서는 프로젝트의 의도, 주장 가능한 범위, 목표 아키텍처, Release/Phase 로드맵, 검증 기준과 작업 규칙을 정의합니다. 코드·문서 변경은 현재 Phase의 Acceptance Criteria를 만족하고 증거를 남긴 뒤 멈춥니다. 임의로 다음 Phase까지 진행하거나 스코프를 확장하지 않습니다.

## 0. 포지셔닝과 주장 규율

이 프로젝트는 현대차그룹이 공개한 **CODA**(Computing & I/O Domain-based Architecture)의 **HPVC–Zone Controller 분리 원칙**을 상용 개발 보드로 축소해 검증하는 학부 수준의 연구·구현 프로젝트입니다.

정확한 포지셔닝은 다음과 같습니다.

> 기존 CAN ECU와 Zonal Ethernet 경로가 공존하는 전환 구조에서 CAN 신호를 서비스/VSS 데이터로 추상화하고, 중앙 정책 소프트웨어의 독립 배포와 실시간 I/O 경로의 장애 격리를 정량적으로 검증한다.

### 사용하지 않는 표현

- "현대·기아 양산 E/E 아키텍처를 재현했다."
- "현대차그룹이 KUKSA/VSS 또는 이 프로젝트의 프로토콜을 사용한다."
- "S32G/R-Car 또는 AUTOSAR를 재현했다."
- "컨테이너 교체만으로 양산급 OTA를 구현했다."
- "ASIL, lockstep 또는 기능안전을 구현했다."

### 사용하는 표현

- "현대차그룹의 공개 CODA 원칙에서 영감을 받은 축소형 프로토타입"
- "HPVC surrogate와 Zone Controller 사이의 역할 분리를 기능적으로 검증"
- "서비스 지향 통신의 핵심 의미를 구현한 경량 프로토콜"
- "정책 소프트웨어와 Edge ECU 펌웨어의 배포 독립성을 검증"
- "application plane과 deterministic I/O plane 분리의 기능적 축소 실험"

### 공개 근거 인용 규칙

공개 아키텍처와의 관계를 문서화할 때는 현대차그룹·42dot·계열사의 **공식 공개 자료만** 근거로 사용하고, 공개되지 않은 양산 내부 구현은 추정하지 않습니다. 인용 시 **출처 URL과 확인 날짜를 함께 기록**합니다.

현재까지 확인된 공개 근거:

| 근거 | 출처 | 확인일 |
|---|---|---|
| CODA = Computing & I/O 도메인 기반 아키텍처, HPVC + 조널 컨트롤러 구조 | 뉴시스 「비밀 병기 'CODA', 차를 플랫폼으로 바꾸다」 현대차 SDV 리포트 <https://www.newsis.com/view/NISX20250711_0003248221> | 2026-08-20 |
| E/E architecture 개요 | 현대케피코 Future Tech <https://www.hyundai-kefico.com/en/future-tech/modular-architecture/content.do> | 2026-08-20 |
| 도메인→존 전환 배경 | <https://kidd.co.kr/news/244684> | 2026-08-20 |

---

## 1. 프로젝트가 증명해야 하는 것

### 1.1 SDV 정책 분리

차량 정책 로직(경고 임계값과 히스테리시스 등)을 Edge ECU가 아니라 중앙 차량 앱에서 실행합니다. 정책 변경 시 G474 펌웨어를 다시 플래시하지 않고 중앙 앱 이미지만 교체할 수 있어야 합니다.

증거는 최소한 다음을 포함합니다.

- 변경 전·후 중앙 앱 image digest
- 변경 전·후 G474 firmware SHA-256
- 같은 입력에 대해 정책 결과가 달라지는 재현 가능한 테스트
- 실패한 배포를 감지하는 health check와 직전 이미지로의 rollback 절차

이는 **배포 독립성의 증명**이며, 양산급 차량 OTA 전체를 구현했다는 뜻이 아닙니다.

### 1.2 전송 경로 추상화

중앙 차량 앱은 Front 데이터가 Ethernet 경로로 오고 Rear 데이터가 직접 CAN 경로로 온다는 사실을 몰라도 동일한 VSS API로 값을 읽어야 합니다.

- Front: Zone Controller를 거치는 Zonal 경로
- Rear: 중앙 컴퓨터에 직접 연결된 Legacy CAN 공존 경로

이 비대칭은 임시 배선이 아니라 **분산형 CAN 구조에서 Zonal 구조로 이행하는 migration architecture**를 비교하기 위한 의도적 실험입니다.

### 1.3 결정론과 장애 격리

시간 제약이 있는 CAN/I/O 경로는 Linux userspace, Ethernet 서비스 또는 RPMsg 응답을 기다리며 block되어서는 안 됩니다.

reset 종류를 반드시 구분해 검증합니다.

| 구분 | 기대치 |
|---|---|
| Linux userspace 프로세스 crash/restart | M4 실시간 경로 **무중단이 필수** |
| A7/Linux reboot | MP157의 실제 reset tree를 먼저 확인한 뒤, M4 유지 가능 여부와 중단 시간을 **실측** |
| 전체 SoC reset / power cycle | M4 연속 동작을 **주장하지 않음**. 안전 출력 상태와 복구 시간을 검증 |

"Linux 재부팅 중 M4가 반드시 유지된다"는 문장을 사전 가정으로 사용하지 않습니다. reset 종류와 보드 구현에 따라 결과가 달라지므로 측정 결과로만 주장합니다.

### 1.4 학술적 질문

- 동일 하드웨어에서 superloop와 RTOS의 WCET, jitter, deadline miss는 어떻게 달라지는가?
- CAN 신호를 Ethernet 서비스로 변환할 때 추가되는 지연과 고장 전파 경로는 무엇인가?
- Linux와 M4 중 누가 CAN을 소유하는지가 지연·지터·복구성에 어떤 영향을 주는가?
- 중앙 정책 변경과 Edge 펌웨어 변경의 배포 범위를 어떻게 분리할 수 있는가?
- Legacy CAN 경로와 Zonal Ethernet 경로를 상위 앱에서 동일하게 추상화할 수 있는가?

---

## 2. 명칭과 현업 대응 관계

| 프로젝트 명칭 | 역할 | 현업 개념과의 관계 |
|---|---|---|
| HPVC Surrogate | Raspberry Pi 5 기반 중앙 차량 컴퓨터 | HPVC 역할을 실험하는 비차량용 대체 장치 |
| Zone Controller v1 | NUCLEO-H723ZG | CAN I/O와 Ethernet 서비스를 연결하는 Zone Controller 프로토타입 |
| Zone Controller v2 | ODYSSEY-STM32MP157C | Linux application plane과 M4 deterministic I/O plane을 분리하는 이종코어 프로토타입 |
| Front Edge ECU | NUCLEO-G474RE | Zone Controller 아래의 로컬 센서·액추에이터 ECU |
| Rear Legacy ECU | NUCLEO-G474RE | 중앙 CAN에 직접 연결된 기존 ECU 공존 경로 |
| Vehicle Data Abstraction Layer | KUKSA Databroker + VSS | 공개 표준 기반의 차량 데이터 추상화 실험 |
| Lightweight Service Transport | protobuf-over-UDP | SOME/IP 자체가 아닌 서비스 지향 의미의 축소 구현 |

`Zone Gateway` 명칭은 새 코드부터 `Zone Controller`로 사용합니다. Gateway는 Zone Controller가 수행하는 역할 중 하나로 설명합니다.

### 진행 중인 명칭 정리 (별도 rename PR로 처리)

기존 파일명과 API를 한 번에 파괴적으로 변경하지 않습니다. 아래는 후속 rename PR에서 참조와 문서를 함께 갱신합니다.

| 현재 | 목표 | 상태 |
|---|---|---|
| `firmware/front-zonal-ecu/` | `firmware/front-edge-ecu/` | 미착수 |
| `FrontZonalApp`, `front_zonal_app.*` | Front Edge ECU 용어와 정렬 | 미착수 |
| `FrontZonalStatus` (CAN 메시지명) | **변경하지 않음** — DBC/프로토콜 계약 안정성 우선 | 확정 |

---

## 3. 현재 저장소 상태 (as-is)

### 3.1 확인된 빌드·프로세스 문제

- STM32CubeIDE GUI 빌드만 있고 재현 가능한 CMake/CLI 빌드가 없음
- 호스트 테스트의 소스 구성이 수동이며 `App/Src/*.c` 일괄 컴파일 불가
  (`can_service.c`는 `fdcan.h` 의존, `time_utils`는 헤더 전용)
- CI 없음
- 프로젝트 자체 코드의 라이선스 정책 미확정 (third-party 고지는 `THIRD_PARTY_NOTICES.md`로 분리 완료)
- GitHub 웹 업로드 중심 이력이라 feature branch/PR 단위 변경 근거 부족

### 3.2 확인된 런타임 문제

현재 구현은 완전한 non-blocking superloop가 아닙니다.

| 위치 | 내용 |
|---|---|
| `firmware/front-zonal-ecu/App/Src/input_service_stm32.c:56` | `HAL_ADC_PollForConversion(..., 10U)` — 최대 10 ms blocking |
| `firmware/front-zonal-ecu/Drivers/VL53L0X/Src/vl53l0x_i2c_platform.c:26` | `VL53L0X_I2C_TIMEOUT_MS = 100U` — 호출당 최대 100 ms |
| `firmware/front-zonal-ecu/App/Src/distance_sensor_service.c` | 상태머신은 단계형이나 개별 HAL I2C 호출은 blocking |

따라서 "현재 superloop가 모든 deadline을 만족한다"거나 "RTOS가 반드시 필요하다"는 결론을 미리 내리지 않습니다. Phase 0B 측정 후 [ADR-0001](adr/0001-scheduling-model-for-edge-nodes.md)에서 판단합니다.

### 3.3 RTOS 관련 문서 일관성 정리 대상

RTOS 미도입은 아직 **확정된 결정이 아니라 측정 대기 상태**입니다. 다만 "도입 예정"으로 단정한 서술은 정리하고, "현재 범위 밖"이라는 정확한 서술은 유지합니다.

| 위치 | 처리 | 상태 |
|---|---|---|
| `README.md` Roadmap `FreeRTOS task architecture` 체크박스 | 제거 (Release 구조로 대체) | ✅ 완료 |
| `docs/architecture.md` §6 "FreeRTOS를 최소 task 구성으로 도입하고" | 개정 — 측정 기반 판단으로 | ✅ 완료 |
| `docs/development-log.md` "Next gate: introduce FreeRTOS" | 개정 | ✅ 완료 |
| `docs/verification.md` "FreeRTOS runtime/task supervision evidence" | 제거 | ✅ 완료 |
| `docs/requirements.md:7, :61` "범위에 포함하지 않음" | **유지** — 이미 현재 상태와 일치 | ✅ 유지 |
| `docs/architecture.md` §4 "FreeRTOS로 전환하더라도 핵심 로직을 다시 작성하지 않는 구조" | **유지** — 모듈화 설계 근거 설명 맥락 | ✅ 유지 |
| `docs/development-log.md` Milestone 5 "later FreeRTOS migration" | **유지** — 당시 판단 기록(history) | ✅ 유지 |

---

## 4. 목표 아키텍처

```text
                         [Raspberry Pi 5]
                           HPVC Surrogate
          VSS / KUKSA / Vehicle App / Deployment Manager
                      │                         │
           Ethernet   │                         │ CAN-B
                      ▼                         ▼
          ┌───────────────────────┐      [Rear Legacy ECU]
          │    Zone Controller    │       NUCLEO-G474RE
          │                       │       sensor / switch
          │ v1: NUCLEO-H723ZG     │
          │                       │
          │ v2: ODYSSEY-MP157C    │
          │  ├─ A7 Linux          │
          │  │  service/diagnosis │
          │  └─ M4 FreeRTOS       │
          │     deterministic I/O │
          └──────────┬────────────┘
                     │ CAN-A
                     ▼
              [Front Edge ECU]
               NUCLEO-G474RE
               ADC / switch / VL53L0X
```

### 4.1 물리 CAN 분리 — 필수

| 세그먼트 | 구간 | 종단저항 |
|---|---|---|
| CAN-A | Zone Controller ↔ Front Edge ECU | 양 끝 120 Ω |
| CAN-B | Pi 5(MCP2518FD) ↔ Rear Legacy ECU | 양 끝 120 Ω |

두 구간을 같은 버스에 연결하지 않습니다. 같은 버스에서는 Pi와 Zone Controller가 모든 프레임을 동시에 관찰하므로 "Front는 Ethernet service provider, Rear는 direct CAN provider"라는 전송 추상화 실험이 성립하지 않습니다.

모듈의 종단저항 내장 여부를 확인하고, 같은 세그먼트에 내장 종단과 외부 종단을 중복 적용하지 않습니다.

### 4.2 최종 순수 Zonal 구조와의 관계

이 프로젝트의 비대칭 구조는 전환 단계 실험입니다. 장기 목표 구조는 문서로만 제시하며 현재 스코프가 아닙니다.

```text
                         ┌─ Front Zone Controller ─ CAN/LIN ─ Local I/O
HPVC ─ Automotive Ethernet
                         └─ Rear Zone Controller  ─ CAN/LIN ─ Local I/O
```

### 4.3 생산 환경과 다른 점

다음은 의도적인 교육용 대체입니다.

| 이 프로젝트 | 양산 |
|---|---|
| 일반 Ethernet | Automotive Ethernet (100/1000BASE-T1) |
| Raspberry Pi 5 | 차량용 HPVC (자율주행·인포테인먼트 통합 연산) |
| protobuf-over-UDP | SOME/IP, DDS, OEM middleware |
| FreeRTOS / bare-metal | AUTOSAR Classic OS 및 양산 BSW |
| 컨테이너 재배포 | 서명·manifest·A/B·fleet orchestration을 포함한 OTA |

---

## 5. 공통 설계 불변조건

모든 Release에서 지킵니다.

1. **실시간 경로는 IPC를 기다리지 않는다.** CAN RX/TX와 로컬 안전동작은 RPMsg, Ethernet, Linux 응답에 block되지 않는다.
2. **주변장치 소유자는 한 명이다.** 같은 CAN/SPI/GPIO를 Linux와 M4가 동시에 제어하지 않는다. Device Tree와 firmware 설정에 소유권을 명시한다.
3. **상태와 정책을 분리한다.** Edge ECU는 센싱·최소 안전상태·명령 실행을 담당하고, 가변 정책은 중앙 앱에 둔다.
4. **전송과 의미를 분리한다.** CAN ID나 UDP endpoint가 차량 앱의 도메인 로직으로 누출되지 않도록 provider 계층에서 VSS로 변환한다.
5. **주장에는 증거가 필요하다.** 로그, trace, waveform, hash, CI artifact 또는 재현 스크립트가 없는 성능·안전·복구 주장은 문서에 쓰지 않는다.
6. **평균값만 보고하지 않는다.** latency/jitter는 sample 수와 함께 p50, p95, p99, p99.9, max 및 deadline miss count를 기록한다.
7. **고장 없는 정상 경로만 검증하지 않는다.** cable disconnect, node power-off, sensor timeout, process crash, malformed message를 포함한다.

---

## 6. Release / Phase 로드맵

### Spike-C — MP157 보드 타당성 (Release C 착수 전, 1~2주)

Release A/B와 독립적인 구매·검증 gate입니다. 상세: [docs/spikes/mp157-board-feasibility.md](spikes/mp157-board-feasibility.md)

**CAN 경로 기본 방침** — carrier에서 native FDCAN 노출이 공식 회로도와 실제 pinmux로 확인되기 전까지는 **외장 SPI MCP2517FD/2518FD를 기본 계획**으로 견적에 반영합니다. native FDCAN이 확인되면 ADR을 작성하고 경로를 단순화합니다.

---

### Release A — Zonal SDV MVP (Phase 0~3, 약 10~13주)

여기까지만으로도 독립적으로 완결된 프로젝트여야 합니다.

#### Phase 0A — 재현 가능한 빌드·테스트 기반 (하드웨어 불필요, 1~2주)

- ARM cross-build와 host test target을 분리한 CMake 구성
- 단일 명령으로 host test 5종 실행
- GitHub Actions: `pull_request`, `push: main`
- 자체 코드만 `-Wall -Wextra -Werror`, vendor/CubeMX 코드는 경고 기록만
- ARM toolchain 버전 고정
- `.elf`, `.map`, flash/RAM size를 CI artifact로 저장
- [docs/traceability.md](traceability.md) 작성
- 프로젝트 코드 라이선스 정책 확정 — **임의 선택 금지, 후보와 제약을 정리해 승인 요청**

#### Phase 0B — blocking 및 scheduling 실측 (보유 G474 필요, 약 1주)

- 정상 센서 조건과 I2C fault 조건에서 ADC/I2C 호출시간 측정
- 100 ms status 주기의 jitter와 deadline miss 측정
- GPIO toggle + logic analyzer 또는 cycle counter 사용, 측정 방법 명시
- **fault injection 방법을 사전에 확정하고 문서화** (센서 전원 차단 / SDA·SCL 개방 / 미응답 주소 중 택일, 재현 절차 포함)
- 결과에 따라 I2C timeout 단축, recovery, IT/DMA 또는 상태머신 전환 결정
- [ADR-0001](adr/0001-scheduling-model-for-edge-nodes.md) 작성

**측정 전에 scheduling 결론을 작성하지 않습니다.**

**Phase 0 Acceptance**: PR/main push에서 host test와 ARM build 통과 · `.elf`/`.map`/size가 CI artifact로 존재 · 정상·고장 조건 blocking 상한과 deadline miss가 재현 가능하게 기록 · traceability와 라이선스/third-party 정책 존재 · scheduling 선택이 측정 자료와 일치

#### Phase 1 — 물리 CAN FD (약 3주)

- internal loopback → external normal mode, TJA1051T/3 물리계층
- `.dbc` 작성: `0x180 FrontZonalStatus`, `0x200 CentralControlCommand`
- Pi 5에 MCP2518FD + SocketCAN, systemd unit으로 `can0` 시작 (arb 500 kbit/s, data 2 Mbit/s, FD on)
- 기존 중앙 정책 테스트를 Pi 서비스 테스트로 **먼저 이관한 뒤** `central_sim.c/h` 제거

**Acceptance**: `candump can0`에 실제 CAN FD 트래픽 · 케이블 분리 후 500 ms 이내 SAFE 전이, 재연결 후 정의된 조건으로 복구 · CAN trace, 배선도, bit timing, 오류 카운터를 `verification.md`에 보존

#### Phase 2 — Rear Legacy ECU (약 3주)

- `firmware/common/`으로 protocol, time utility, diagnostics, input abstraction 분리
- Rear 앱 구현 (후방 거리, reverse warning 최소 기능)
- Pi 5가 node heartbeat 감시, `REAR_ECU_LOST` 기록
- Front/Rear의 node identity, firmware version, reset reason을 status에 포함

**Acceptance**: Rear 전원 차단 시 Pi가 fault 기록 · Rear 장애가 Front Edge ECU에 영향 없음 · Rear 재투입 시 정의된 절차로 복구 · fault injection 로그와 복구 시간 기록

#### Phase 3 — VSS 추상화와 독립 재배포 (약 3주)

- 최소 VSS tree 정의 (`Vehicle.Front.Distance`, `Vehicle.Rear.Distance` 등)
- KUKSA Databroker + CAN provider, provider가 Phase 1의 DBC를 소비
- transport-specific 정보를 VSS 경계에서 은닉
- 경고 threshold/hysteresis 정책을 Pi 컨테이너 앱으로 이전
- systemd/compose 기반 lifecycle, health check, rollback 절차
- 배포 manifest에 앱 버전, compatible schema version, image digest 기록

**Acceptance**: 200/250 mm 정책을 중앙 앱 이미지 교체만으로 변경 가능 · **중앙 앱 digest는 달라지고 G474 firmware SHA-256은 동일** · 재시작 후 provider와 policy app 자동 복구 · 잘못된 이미지 또는 failed health check에서 직전 동작 버전으로 복귀

---

### Release B — Ethernet Zone Controller와 RTOS 연구 (Phase 4, 약 6~10주)

#### Phase 4A — Zone Controller v1 (H723ZG)

- H723 Ethernet/lwIP bring-up
- Front Edge ECU를 CAN-A로 H723 아래 배치, Rear Legacy ECU는 CAN-B로 Pi 직결
- CAN signal ↔ Ethernet service message 변환, 명시적 routing/source-of-truth table

**Lightweight Service Transport 계약** — protobuf-over-UDP는 SOME/IP라고 부르지 않습니다. 다음 최소 의미만 구현합니다.

- service / instance / message ID, schema version
- discovery 또는 정적 registry 중 하나를 명시적으로 선택
- request/response, event publish/subscribe
- sequence number와 source timestamp
- timeout, duplicate, out-of-order 처리 규칙
- heartbeat/health 상태
- malformed / unknown version 처리

MTU 내 단일 datagram을 기본으로 하며 자체 fragmentation을 만들지 않습니다.

#### Phase 4B — 동일 H723에서 superloop vs FreeRTOS A/B 비교

반드시 같은 보드, 클럭, compiler option, CAN/Ethernet 입력, 측정 GPIO, 부하를 사용합니다. 서로 다른 MCU 간 비교는 교란변수 때문에 무효입니다.

- Build S: H723 superloop / Build R: H723 FreeRTOS
- 측정: WCET, jitter distribution, deadline miss rate, CPU load, queue high-water mark

우선순위 역전은 실제 공유 자원 경합으로 재현합니다.

```text
High    CAN routing task
Medium  Ethernet telemetry task
Low     diagnostics task holding a statistics snapshot lock
```

priority inheritance 적용 전·후 또는 critical path에서 lock 제거 전·후를 같은 입력으로 비교합니다.

**Acceptance**: 차량 앱이 `Vehicle.Front.*`와 `Vehicle.Rear.*`를 같은 API로 사용 · Rear 직접 경로 상실 / Front 간접 경로 상실 / Zone Controller 자체 상실을 구분 검출 · sensor timestamp부터 VSS update까지 경로별 end-to-end latency 측정 · A/B 보고서에 sample 수, percentile, max, deadline miss, 원시 데이터 포함 · ADR-0001 갱신

---

### Release C — MP157 이종코어 Zone Controller (Phase 6, 약 10~16주)

부트로더, Device Tree, Linux kernel, remoteproc/RPMsg, RTOS 중간계층을 학습하는 핵심 Release입니다. **반드시 Spike-C를 통과한 뒤 착수합니다.**

- **6A 부트체인/BSP** — boot flow를 실제 보드 로그로 문서화(ROM → TF-A → OP-TEE(사용 시) → U-Boot → Linux) · Ethernet PHY, pinctrl, clock, reserved-memory, IPCC/mailbox, remoteproc, CAN 경로 DT 구성 · kernel config fragment 관리 · M4 ELF memory layout과 DT reserved-memory 일치 검증 · U-Boot에서 M4 조기 시작은 **공식 BSP 지원 + Spike-C 검증된 경우에만** 적용
- **6B M4 FreeRTOS와 RPMsg** — M4가 CAN 및 로컬 안전 I/O의 단독 소유자 · control channel(routing/config) + telemetry channel(jitter, deadline miss, health) · 모든 IPC는 bounded queue와 timeout 사용 · M4 real-time task는 RPMsg 완료를 기다리지 않음 · Linux 부재 시 last-known-good configuration과 만료 정책 · 복구 후 epoch/version 기반 resynchronization
- **6C peripheral ownership 비교** — (1) Linux가 CAN 직접 소유(SocketCAN) vs (2) M4가 소유하고 A7에 RPMsg로 semantic data 전달. 평균 latency뿐 아니라 jitter, deadline miss, CPU 부하, 재시작 복구시간 비교
- **6D 장애 격리** — service process kill/restart · RPMsg consumer hang · Ethernet disconnect/reconnect · A7/Linux reboot · M4 crash 후 remoteproc recovery · full SoC reset/power cycle 후 safe-state recovery. §1.3의 reset 분류에 따라 기대치를 구분
- **6E Yocto** — `meta-st-stm32mp1` 및 vendor layer · 커스텀 `meta-zonal-sdv` · A7 service, schema, M4 ELF, 설정을 recipe로 패키징 · SPDX SBOM · SDK 생성 · build configuration과 layer revision 고정. **Pi 5까지 동일 distro policy로 통합하는 것은 MP157 이미지 안정화 이후의 확장 목표이며 Release C Acceptance에 포함하지 않습니다.**

**보안 부트 범위** — OP-TEE 또는 secure boot를 이용한 M4 firmware authenticity 검증은 ST 공식 지원 경로와 threat model을 확인한 뒤 별도 ADR로 결정합니다. 공식 경로가 불명확한 상태에서 "OP-TEE가 M4 서명을 검증한다"고 사전 단정하지 않습니다.

**Acceptance**: 고정된 layer revision으로 MP157 image와 M4 firmware 재현 빌드 가능 · remoteproc start/stop/crash recovery와 RPMsg 양방향 통신 재현 가능 · userspace crash 동안 M4 CAN deadline miss 미증가 · A7 reboot 및 full reset 결과가 reset 종류와 함께 측정됨 · Linux-owned vs M4-owned CAN 비교 보고서에 원시 데이터 존재 · DT peripheral ownership이 실제 드라이버 binding과 일치

---

### Release D — 보안·OTA·진단 (Phase 5A/5B/5C/7, 약 12~16주)

일정이 부족하면 축소하거나 후속 프로젝트로 이관할 수 있습니다.

- **5A SecOC-lite 연구** — CAN payload에 truncated CMAC + freshness counter · replay / stale counter / authentication failure 분리 집계 · **데모용 provisioned key 사용과 한계 명시**, production HSM/PKI provisioning은 비범위 · 실시간 CAN frame마다 A7/OP-TEE를 왕복하지 않음 · 키 저장 위치, 공격 모델, reset 후 freshness 동기화, 키 노출 한계를 문서화
- **5B ISO-TP + UDS 다운로드** — ISO-TP segmentation/flow control · Diagnostic Session Control · ECU Reset · Security Access(교육용 범위와 한계 명시) · RequestDownload `0x34` · TransferData `0x36` + block sequence counter · RequestTransferExit `0x37` · negative response와 timeout 처리
- **5C G474 A/B firmware update** — image hash/signature 검증 · update metadata의 power-loss atomicity · boot confirmation과 rollback state machine · power-cut fault injection · CI의 `.map`과 image size로 bank budget 추적 · status frame에 software version과 boot result 포함. **G474RE의 실제 flash bank 구성, bootloader 보호 영역, 옵션 바이트는 사용 MCU revision의 reference manual과 linker map으로 확인합니다("뱅크당 약 256 KB"를 검증 없는 사실로 사용하지 않음).** MP157 Zone Controller 자체의 A/B OTA는 G474 UDS bootloader와 별도 문제이며 같은 Phase에서 하나의 구현처럼 섞지 않습니다.
- **7 진단** — Edge/Rear는 UDS over CAN · Zone Controller/HPVC는 DoIP 최소 진단 경로 또는 명확히 정의한 교육용 Ethernet diagnostic transport · DTC 저장/조회/clear, node·version 식별. **DoIP를 표방한다면 ISO 13400 적용 범위를 명확히 하고 필요한 동작을 검증합니다. 임의 TCP 프로토콜을 DoIP라고 부르지 않습니다.**

**Acceptance**: replay와 authentication failure 구분 · ISO-TP multi-frame 전송과 UDS negative response 검증 · 업데이트 중 power cut 이후 이전 정상 이미지로 부팅 · CAN과 Ethernet 진단 경로에서 최소 DTC 시나리오 통과

---

## 7. 기간과 우선순위

| 구간 | 내용 | 예상 |
|---|---|---:|
| Spike-C | MP157 feasibility | 1~2주 |
| Release A | Zonal SDV MVP | 10~13주 |
| Release B | Ethernet Zone Controller + RTOS A/B | 6~10주 |
| Release C | MP157 heterogeneous platform | 10~16주 |
| Release D | Security + OTA + diagnostics | 12~16주 |
| **전체** | 순차 수행 | **39~57주 (약 9~14개월)** |

학업·취업 준비 병행 시 달력 기준으로 더 길어질 수 있습니다.

우선순위: **A 반드시 완결 → B(차량 Zonal/RTOS 포트폴리오 핵심) → C(부트로더·DT·kernel·이종코어 학습 핵심) → D(확장)**

Release A가 불완전한 상태에서 MP157 또는 OTA로 이동하지 않습니다.

---

## 8. 하드웨어 BOM

| 상태 | 품목 | 역할 |
|---|---|---|
| 보유 | Raspberry Pi 5 | HPVC Surrogate |
| 보유 | NUCLEO-G474RE | Front Edge ECU |
| 구매 완료 | NUCLEO-H723ZG | Zone Controller v1 |
| 구매 필요 | NUCLEO-G474RE | Rear Legacy ECU |
| 구매 필요 | VL53L0X | Rear sensor |
| 구매 필요 | TJA1051T/3 ×3 | CAN transceiver |
| 구매 필요 | MCP2518FD SPI module | Pi CAN-B interface |
| 구매 필요 | 120 Ω termination ×4 | CAN-A/CAN-B 양 끝 |
| 구매 필요 | Ethernet/USB cable, jumper | 연결 |
| 구매 필요 | 독립 출력 또는 충분한 전력의 multi-port supply | 다중 보드 전원 |
| 구매 예정 | Seeed ODYSSEY-STM32MP157C | Zone Controller v2 |
| 조건부 | 외장 ST-LINK/V3 | Spike-C 결과에 따라 |
| 기본 계획 | MCP2517FD/2518FD SPI board + IRQ wiring | MP157 carrier의 native FDCAN 미노출 대비 |
| 조건부 | microSD, USB-UART | MP157 image/console |

모듈의 oscillator, logic voltage, CAN transceiver 포함 여부, termination 내장 여부를 구매 전에 확인합니다.

---

## 9. 작업 규칙

- 기존 `App/Src/*.c` 스타일을 따릅니다. HAL-independent logic과 HAL adapter를 분리합니다.
- 모든 인자를 명시적으로 검증하고 초기 상태를 명확히 설정합니다.
- 하드웨어 독립 로직에는 대응 host unit test를 함께 추가합니다.
- 하드웨어 의존 코드는 HIL test ID와 board regression checklist로 검증합니다.
- protocol parser는 length, range, version, malformed input test를 포함합니다.
- 자체 코드만 `-Werror`를 적용합니다. 생성 코드와 vendor code를 불필요하게 포맷하거나 대량 수정하지 않습니다.
- `main`에 직접 커밋하지 않습니다. feature branch와 PR을 사용합니다.
- 관련 코드와 문서는 같은 PR에서 갱신합니다.
- 요구사항은 기존 ID 체계(`IN-xxx`, `COM-xxx`, `SAFE-xxx`, `APP-xxx`, `SEN-xxx`, `DIAG-xxx`)를 따릅니다.
- 설계 판단은 `docs/adr/`에 기록합니다. **ADR 제목에 결론을 미리 담지 않습니다.**
- 성능 데이터는 요약 그래프뿐 아니라 CSV 등 원시 자료와 생성 방법을 함께 보존합니다.

---

## 10. 명시적 비범위

- 현대·기아 양산 내부 아키텍처의 동일 재현
- RC car/motor 기반 시각 데모
- lockstep core와 ASIL safety mechanism
- ISO 26262 정식 HARA/FMEA/FTA 및 안전 인증
- ASPICE 평가
- AUTOSAR 상용 툴체인
- 정식 SOME/IP 구현
- gPTP/TSN 및 100/1000BASE-T1 physical layer
- 양산급 HSM/PKI key provisioning
- production fleet OTA backend
- 전자식 power distribution
- 두 번째 Zone Controller를 추가한 완전 대칭 Zonal topology

---

## 11. 최종 포트폴리오 서사

구현 결과는 다음 한 문장으로 검증 가능해야 합니다.

> 현대차그룹이 공개한 CODA의 HPVC–Zone Controller 분리 원칙을 개발 보드로 축소 구현하고, Legacy CAN과 Zonal Ethernet이 공존하는 전환 구조에서 signal-to-service 변환, 정책 소프트웨어의 독립 배포, RTOS 결정론 및 Linux–M4 장애 격리를 정량적으로 검증했다.

이 문장은 **구현 완료 항목에 대해서만** 사용합니다. 아직 구현하지 않은 Release는 "계획" 또는 "후속 연구"로 구분합니다.
