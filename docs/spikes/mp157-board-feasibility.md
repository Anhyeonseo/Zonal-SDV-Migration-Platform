# Spike-C — ODYSSEY-STM32MP157C 보드 타당성

- Status: **Not started** — 보드 미수령
- Phase: Spike-C (Release C 착수 전 gate)
- 예상 기간: 1~2주

## 목적

Release C(MP157 이종코어 Zone Controller)의 설계를 확정하기 전에, 보드가 실제로 그 설계를 지원하는지 검증합니다. **하나라도 막히면 Phase 6을 억지로 진행하지 않고 보드 또는 CAN 경로를 재선정합니다.**

Release A/B와 독립적인 gate입니다. Release A·B 진행 중에 병행할 수 있습니다.

## 대상 하드웨어

| 항목 | 내용 |
|---|---|
| 보드 | Seeed ODYSSEY-STM32MP157C (SoM + carrier) |
| SoC | STM32MP157C — Cortex-A7 ×2 + Cortex-M4 |
| RAM / Flash | 512 MB DDR3 / 4 GB eMMC |
| 네트워크 | Gigabit Ethernet (carrier 탑재) |
| 확장 | Raspberry Pi 40-pin 호환 헤더, Grove (GPIO/I2C) |

## 확인 항목

### 1. FDCAN 핀 노출 여부 — **최우선**

STM32MP157C SoC는 FDCAN1/FDCAN2를 보유합니다(ST 데이터시트 확인). 확인 대상은 **carrier board가 해당 TX/RX 핀을 외부로 노출하는가**입니다.

- [ ] SoM 회로도에서 FDCAN1/FDCAN2 핀의 B2B 커넥터 할당 확인
- [ ] Carrier 회로도에서 해당 핀의 외부 노출 여부 확인
- [ ] 기존 주변장치(Ethernet, SDMMC, DSI, DVP 등)와의 pinmux 충돌 확인

**이 항목의 결과가 Release C 작업량을 크게 좌우합니다.**

| 결과 | 영향 |
|---|---|
| native FDCAN 노출됨 | ADR 작성 후 경로 단순화. SPI 드라이버 작업 불필요 |
| 노출 안 됨 | 아래 "SPI 경로" 작업이 추가됨 |

#### CAN 경로 기본 방침

**native FDCAN 노출이 공식 회로도와 실제 pinmux로 확인되기 전까지는 외장 SPI MCP2517FD/2518FD를 기본 계획으로 견적에 반영합니다.**

SPI 경로를 사용할 경우 추정에 포함할 작업:

- M4용 MCP2517FD/2518FD 드라이버
- interrupt GPIO 배선과 오류 처리
- SPI/GPIO의 M4 독점 소유권 설정
- Linux Device Tree에서 동일 주변장치 비활성화
- TF-A/U-Boot 단계의 pinctrl·clock 준비 여부 확인

### 2. 핀 충돌

- [ ] SPI 핀 사용 가능 여부와 충돌
- [ ] IRQ용 GPIO 확보
- [ ] SWD / UART 콘솔 핀 확보 및 충돌

### 3. 벤더 이미지 부팅

- [ ] Seeed 제공 이미지로 부팅 성공
- [ ] UART 콘솔 접근
- [ ] Ethernet 링크 및 IP 통신 확인

### 4. remoteproc / RPMsg 기본 동작

- [ ] `remoteproc`으로 M4 firmware 로드·기동·정지
- [ ] M4 blink 예제 동작
- [ ] RPMsg echo 양방향 통신

### 5. reset 계층 실측 — **설계 전제 검증**

`project-brief.md` §1.3의 reset 분류가 이 보드에서 실제로 어떻게 동작하는지 측정합니다. **결과를 사전 가정하지 않습니다.**

| 시나리오 | 측정할 것 | 결과 |
|---|---|---|
| Linux userspace 프로세스 kill/restart | M4 실행 지속 여부, 중단 시간 | |
| A7 / Linux reboot | M4 유지 여부, 중단 시간 | |
| full SoC reset / power cycle | 복구 시간, 안전 출력 상태 | |

### 6. 디버깅 경로 확정

- [ ] 외장 ST-LINK/V3 연결 가능 여부 (SWD 핀 접근)
- [ ] 또는 remoteproc 중심 디버깅으로 대체 가능한지
- [ ] M4 firmware 로딩/재시작 반복 개발 사이클 확인

## Acceptance Criteria

- 이 문서에 **회로도 근거, 테스트 명령, 결과, 사진/로그**가 기록됨
- CAN 경로 / 디버깅 경로 / reset 의미가 ADR로 확정됨
- 실패 항목이 있으면 Phase 6 설계 변경 또는 보드·CAN 경로 재선정 결론이 명시됨

## 결과

> 보드 수령 후 작성.

## References

- STM32MP157C/F 데이터시트 <https://www.st.com/resource/en/datasheet/stm32mp157c.pdf>
- ODYSSEY-STM32MP157C wiki <https://wiki.seeedstudio.com/ODYSSEY-STM32MP157C/>
- `Seeed-Studio/meta-st-odyssey` <https://github.com/Seeed-Studio/meta-st-odyssey>
