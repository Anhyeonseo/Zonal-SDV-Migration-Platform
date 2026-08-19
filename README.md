# Automotive Zonal SDV Migration Platform

현대차그룹이 공개한 **CODA**(Computing & I/O Domain-based Architecture)의 **HPVC–Zone Controller 분리 원칙**을 상용 개발 보드로 축소해 검증하는 프로젝트입니다.

> 기존 CAN ECU와 Zonal Ethernet 경로가 공존하는 전환 구조에서 CAN 신호를 서비스/VSS 데이터로 추상화하고, 중앙 정책 소프트웨어의 독립 배포와 실시간 I/O 경로의 장애 격리를 정량적으로 검증한다.

Raspberry Pi 5가 **HPVC surrogate**(중앙 차량 컴퓨터) 역할을, STM32 보드들이 **Zone Controller**와 **Edge ECU** 역할을 맡습니다. Front 데이터는 Zone Controller를 거치는 Ethernet 경로로, Rear 데이터는 중앙 컴퓨터에 직접 연결된 legacy CAN 경로로 들어옵니다. 이 **의도적인 비대칭**이 분산형 CAN 구조에서 Zonal 구조로 이행하는 migration architecture를 비교하기 위한 실험 조건입니다.

현재 저장소에는 **NUCLEO-G474RE에서 검증한 Edge ECU superloop baseline**이 구현되어 있습니다.

설계 의도·주장 범위·로드맵의 단일 기준 문서는 [docs/project-brief.md](docs/project-brief.md)입니다.

## Current milestone

`v0.1 — Front Zonal ECU superloop baseline`

현재 구현하고 실제 보드에서 검증한 범위는 다음과 같습니다.

* STM32G474 170 MHz system clock
* ADC 기반 analog input 처리와 0~1000 permille 변환
* 20 ms switch debounce
* VL53L0X non-blocking 거리 측정 서비스
* CAN FD frame encode/decode와 payload validation
* FDCAN internal loopback 및 interrupt 기반 RX
* 100 ms 주기의 Front Zonal Status 전송
* Central Computer Simulator의 200 ms command 전송
* command timeout 발생 시 output 차단 및 `SAFE` 상태 전환
* 거리 센서 오류 발생 시 `DEGRADED` 상태 전환
* CAN sequence, invalid payload, HAL TX failure, timeout 진단 카운터
* 하드웨어 비의존 모듈의 host unit test
* 실제 보드 기반 regression test

현재 단계에서는 Front Zonal ECU 내부에 Central Computer Simulator를 함께 실행해 양방향 CAN 통신, 상태 전이, command timeout과 fault handling을 검증합니다.

## Target system

```mermaid
flowchart TB
    CENTRAL["Raspberry Pi 5<br/>Yocto Linux Central Computer"]

    BUS["CAN FD Bus"]

    FRONT["STM32 Front Zonal ECU"]
    REAR["STM32 Rear Zonal ECU"]

    FRONT_INPUT["Front sensors / switches"]
    FRONT_OUTPUT["Front local outputs"]

    REAR_INPUT["Rear sensors / switches"]
    REAR_OUTPUT["Rear local outputs"]

    CENTRAL <--> BUS
    BUS <--> FRONT
    BUS <--> REAR

    FRONT_INPUT --> FRONT
    FRONT --> FRONT_OUTPUT

    REAR_INPUT --> REAR
    REAR --> REAR_OUTPUT
```

최종적으로는 하나의 Central Computer가 Front / Rear Zonal ECU를 통합 관리하는 구조를 목표로 합니다.

Central Computer는 차량 모드와 상위 제어 명령을 생성하고, 각 Zonal ECU는 자신의 영역에서 필요한 입력 처리와 local control을 독립적으로 수행합니다.

## Development status

```mermaid
flowchart LR
    subgraph CURRENT["현재 구현"]
        INPUT["ADC / switch / VL53L0X"]
        FRONT_APP["Front Zonal Application"]
        OUTPUT["LD2 / warning LED"]
        STATUS["Front Zonal Status<br/>100 ms"]
        FDCAN["FDCAN Internal Loopback"]
        SIM["Central Computer Simulator"]
        COMMAND["Central Command<br/>200 ms"]

        INPUT --> FRONT_APP
        FRONT_APP --> OUTPUT
        FRONT_APP --> STATUS
        STATUS --> FDCAN
        FDCAN --> SIM
        SIM --> COMMAND
        COMMAND --> FDCAN
        FDCAN --> FRONT_APP
    end

    subgraph NEXT["다음 통합 단계"]
        PI["Raspberry Pi 5<br/>Linux Central Computer"]
        CANBUS["External CAN FD Bus"]
        FRONT_ECU["Front Zonal ECU"]
        REAR_ECU["Rear Zonal ECU"]

        PI <--> CANBUS
        CANBUS <--> FRONT_ECU
        CANBUS <--> REAR_ECU
    end

    CURRENT -.-> NEXT
```

현재 Central Computer Simulator는 Raspberry Pi가 연결되기 전까지 CAN contract와 ECU application 동작을 독립적으로 검증하기 위한 임시 구성입니다.

외부 CAN FD 통합 이후에는 simulator를 제거하고 Raspberry Pi의 Linux service가 실제 command를 전송합니다.

## Responsibility split

| Component        | Responsibility                                                                               |
| ---------------- | -------------------------------------------------------------------------------------------- |
| Central Computer | vehicle mode 관리, 상위 제어 정책, command 생성, ECU 상태 수집, heartbeat 감시, logging 및 diagnostics        |
| Front Zonal ECU  | 전방 영역의 sensor 및 switch 입력, local output 제어, 주기 상태 전송, command timeout 기반 safe output         |
| Rear Zonal ECU   | 후방 영역의 sensor 및 switch 입력, local output 제어, 주기 상태 전송, command timeout 기반 safe output         |
| CAN contract     | sender/receiver, CAN ID, payload format, cycle time, alive counter, validity 및 timeout 규칙 정의 |

## ECU concept

### Front Zonal ECU

Front Zonal ECU는 차량 전방 영역에 연결된 입력과 출력을 담당합니다.

현재 baseline에서는 다음 기능을 구현합니다.

* analog input
* switch input
* front distance sensor
* warning output
* periodic status transmission
* command timeout detection
* sensor fault detection
* `NORMAL`, `DEGRADED`, `SAFE` 상태 전이

### Rear Zonal ECU

Rear Zonal ECU는 Front Zonal ECU에서 검증한 공통 architecture를 재사용해 개발할 예정입니다.

예정된 역할은 다음과 같습니다.

* rear distance sensor
* rear switch input
* reverse warning output
* rear lighting output
* periodic status transmission
* command timeout detection
* sensor 및 communication fault handling

Rear Zonal ECU는 Front firmware를 단순 복사하는 방식이 아니라, 공통 service와 protocol layer를 재사용하고 ECU별 application 및 hardware configuration을 분리하는 방향으로 설계합니다.

### Central Computer

Central Computer는 Raspberry Pi 5에서 동작하는 Linux 기반 상위 제어기입니다.

예정된 주요 기능은 다음과 같습니다.

* vehicle mode 관리
* Front / Rear Zonal ECU command 생성
* ECU status 및 heartbeat 감시
* communication timeout 검출
* degraded operation 관리
* CAN message logging
* fault event 및 diagnostic data 저장
* Linux service 자동 시작 및 복구
* SocketCAN 기반 CAN FD communication

최종 단계에서는 Device Tree와 Yocto를 이용해 프로젝트 전용 Linux image를 구성할 예정입니다.

## Multi-ECU fault scenario

두 개의 Zonal ECU를 구성하는 목적은 단순히 STM32 보드 수를 늘리는 것이 아니라, 다중 ECU 환경에서 발생하는 통신과 fault handling을 검증하는 것입니다.

대표적인 목표 시나리오는 다음과 같습니다.

```text
Rear Zonal ECU heartbeat timeout
→ Central Computer가 REAR_ZONE_LOST fault 검출
→ Rear 관련 command 전송 중단
→ Rear output을 safe state로 전환
→ Front Zonal ECU 기능은 계속 유지
→ Central Computer가 degraded operation 기록
```

이를 통해 한 ECU의 장애가 전체 시스템 정지로 이어지지 않도록 역할 분리, 장애 감지, 기능 격리와 상태 복구를 검증합니다.

## Repository layout

```text
automotive-zonal-control-system/
├── README.md
├── THIRD_PARTY_NOTICES.md
├── docs/
│   ├── project-brief.md        # 포지셔닝·주장 규율·로드맵의 단일 기준
│   ├── architecture.md
│   ├── requirements.md
│   ├── traceability.md         # 요구사항 ↔ 검증 증거 매핑
│   ├── hardware-setup.md
│   ├── can-protocol.md
│   ├── verification.md
│   ├── development-log.md
│   ├── adr/                    # 설계 판단 기록
│   └── spikes/                 # 타당성 검증 (Spike-C 등)
├── firmware/
│   ├── common/                 # 공통 layer (Phase 2에서 분리)
│   ├── front-zonal-ecu/        # Front Edge ECU — G474RE (현재 baseline)
│   │   ├── App/                #   직접 설계한 application 및 service 코드
│   │   ├── Inc/                #   CubeMX generated header
│   │   ├── Src/                #   CubeMX generated code와 main integration
│   │   ├── Drivers/            #   STM32 HAL, CMSIS, VL53L0X API
│   │   ├── Startup/            #   MCU startup assembly
│   │   └── Tests/              #   host unit test
│   ├── rear-legacy-ecu/        # Rear Legacy ECU — G474RE (Phase 2)
│   └── zone-controller-v1/     # Zone Controller v1 — H723ZG (Phase 4)
├── central-computer/           # HPVC surrogate — Raspberry Pi 5
│   ├── vehicle-app/            #   정책 로직 (컨테이너, Phase 3)
│   ├── providers/              #   CAN / service → VSS provider
│   └── vss/                    #   VSS 신호트리 정의
├── platform/
│   └── yocto/                  # meta-zonal-sdv 등 (Release C)
└── tools/
```

`firmware/front-zonal-ecu/`는 현재 명칭을 유지합니다. Front **Edge** ECU로의 디렉터리 rename은 참조와 문서를 함께 갱신하는 별도 PR에서 처리합니다. 자세한 내용은 [project-brief.md](docs/project-brief.md) §2를 참고하십시오.

## Firmware modules

| Module                    | Role                                                |
| ------------------------- | --------------------------------------------------- |
| `can_protocol`            | CAN payload encode/decode와 값 범위 검증                  |
| `can_service`             | FDCAN 초기화, 주기 송신, interrupt RX와 통신 진단               |
| `central_sim`             | Raspberry Pi 통합 전 Central Computer command 생성       |
| `front_zonal_app`         | Front ECU 상태 전이, command timeout과 output policy     |
| `input_service`           | ADC 변환, input validity와 switch debounce             |
| `input_service_stm32`     | HAL ADC/GPIO hardware adapter                       |
| `distance_sensor_service` | VL53L0X 초기화와 non-blocking measurement state machine |
| `time_utils`              | tick rollover와 interrupt context를 고려한 경과 시간 계산      |

Rear Zonal ECU 개발 단계에서는 CAN, time, diagnostics와 hardware-independent input module을 공통 layer로 분리하고, ECU별 application과 hardware adapter를 별도로 구성할 예정입니다.

## Documentation

상세 설계와 검증 기록은 `docs/`에서 관리합니다.

* **[Project brief](docs/project-brief.md)** — 포지셔닝, 주장 규율, 로드맵의 단일 기준
* [Architecture](docs/architecture.md)
* [Requirements](docs/requirements.md)
* [Traceability](docs/traceability.md) — 요구사항 ↔ 검증 증거 매핑
* [Hardware setup](docs/hardware-setup.md)
* [CAN protocol](docs/can-protocol.md)
* [Verification](docs/verification.md)
* [Development log](docs/development-log.md)
* [ADR](docs/adr/) — 설계 판단 기록
* [Spikes](docs/spikes/) — 타당성 검증

## Build

### STM32 firmware

1. STM32CubeIDE에서 `firmware/front-zonal-ecu`를 existing project로 import합니다.
2. `Debug` configuration을 선택합니다.
3. Project의 Clean과 Build를 실행합니다.
4. ST-LINK를 통해 NUCLEO-G474RE에 firmware를 다운로드합니다.

현재 기준 전체 ARM build 결과는 다음과 같습니다.

```text
0 errors, 0 warnings
```

### Host unit tests

`Tests/`는 HAL에 의존하지 않는 firmware module을 PC C compiler에서 검증합니다.

현재 검증 대상은 다음과 같습니다.

* CAN protocol
* Central Computer Simulator
* Front Zonal Application
* Input Service
* time utility

반복 실행용 test script와 CI pipeline은 후속 단계에서 추가합니다.

## Roadmap

상세 내용과 Acceptance Criteria는 [docs/project-brief.md](docs/project-brief.md) §6에 있습니다.

**완료 — Baseline**

* [x] STM32 peripheral and sensor bring-up
* [x] Edge ECU superloop baseline
* [x] CAN FD internal loopback and interrupt RX
* [x] Protocol, service, application modularization
* [x] Host unit tests and board regression test

**Release A — Zonal SDV MVP**

* [ ] Phase 0A: CMake/CI, traceability, license policy
* [ ] Phase 0B: ADC/I2C blocking 실측 → [ADR-0001](docs/adr/0001-scheduling-model-for-edge-nodes.md) 결정
* [ ] Phase 1: 물리 CAN FD, DBC, SocketCAN, `CentralSimulator` 제거
* [ ] Phase 2: Rear Legacy ECU, 공통 firmware layer 분리, fault isolation
* [ ] Phase 3: VSS/KUKSA 추상화, 정책 앱 컨테이너화, 독립 재배포 검증

**Release B — Ethernet Zone Controller와 RTOS 연구**

* [ ] Phase 4A: Zone Controller v1 (H723ZG), lightweight service transport
* [ ] Phase 4B: 동일 보드 superloop vs FreeRTOS A/B 비교, 우선순위 역전 재현

**Release C — 이종코어 Zone Controller**

* [ ] Spike-C: [MP157 보드 타당성 검증](docs/spikes/mp157-board-feasibility.md)
* [ ] Phase 6A: 부트체인(TF-A/U-Boot/Linux), Device Tree, kernel config
* [ ] Phase 6B: M4 FreeRTOS + remoteproc/RPMsg
* [ ] Phase 6C: peripheral ownership 비교 (Linux-owned vs M4-owned CAN)
* [ ] Phase 6D: 장애 격리 (userspace crash / A7 reboot / full reset)
* [ ] Phase 6E: Yocto custom layer, SPDX SBOM

**Release D — 보안·OTA·진단**

* [ ] Phase 5A: SecOC-lite
* [ ] Phase 5B: ISO-TP + UDS download
* [ ] Phase 5C: A/B firmware update, power-cut rollback
* [ ] Phase 7: UDS over CAN, Ethernet 진단 경로

## Engineering boundary

이 저장소는 automotive-style E/E architecture와 embedded software development process를 학습하고 검증하기 위한 prototype입니다.

다음 항목의 준수를 주장하지 않습니다.

* AUTOSAR compliance
* ISO 26262 compliance
* ASIL qualification / lockstep core
* production-ready automotive ECU
* production vehicle network security
* ASPICE 평가
* 정식 SOME/IP 구현
* 양산급 HSM/PKI key provisioning 및 fleet OTA backend

### 주장 규율

공개 아키텍처와의 관계를 서술할 때는 현대차그룹·계열사의 **공식 공개 자료만** 근거로 사용하고, 공개되지 않은 양산 내부 구현은 추정하지 않습니다. 다음 표현은 사용하지 않습니다.

* "현대·기아 양산 E/E 아키텍처를 재현했다"
* "현대차그룹이 KUKSA/VSS 또는 이 프로젝트의 프로토콜을 사용한다"
* "S32G/R-Car 또는 AUTOSAR를 재현했다"
* "컨테이너 교체만으로 양산급 OTA를 구현했다"

정확한 표현은 [docs/project-brief.md](docs/project-brief.md) §0을 따릅니다. 생산 환경과의 차이(Automotive Ethernet, 차량용 HPVC 하드웨어, SOME/IP, AUTOSAR Classic OS, 양산 OTA)는 같은 문서 §4.3에 정리돼 있습니다.

대신 명확한 responsibility split, CAN contract, timeout 처리, fault state, modular firmware, host test와 hardware verification을 통해 차량 ECU software architecture의 핵심 개념을 단계적으로 구현하는 것을 목표로 합니다.
