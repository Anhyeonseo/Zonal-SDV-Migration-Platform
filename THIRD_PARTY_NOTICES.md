# Third-Party Notices

이 저장소는 아래 third-party 구성요소를 포함합니다. 각 구성요소는 원저작자의 라이선스를 따르며, 원본 라이선스 파일과 소스 헤더의 고지를 유지합니다.

> **프로젝트 자체 코드의 라이선스는 아직 확정되지 않았습니다.** 이 문서는 third-party 고지만 다룹니다. 자체 코드 라이선스 결정은 `docs/project-brief.md` §6 Phase 0A 항목을 참고하십시오.

---

## STM32Cube HAL / LL Driver for STM32G4

- 경로: `firmware/front-zonal-ecu/Drivers/STM32G4xx_HAL_Driver/`
- 저작권: STMicroelectronics
- 라이선스: **BSD-3-Clause**
- 원본 고지: `firmware/front-zonal-ecu/Drivers/STM32G4xx_HAL_Driver/LICENSE.txt`

> This software component is provided to you as part of a software package and applicable
> license terms are in the Package_license file. If you received this software component
> outside of a package or without applicable license terms, the terms of the BSD-3-Clause
> license shall apply. <https://opensource.org/licenses/BSD-3-Clause>

## ARM CMSIS

- 경로: `firmware/front-zonal-ecu/Drivers/CMSIS/`
- 저작권: ARM Limited 외
- 라이선스: **Apache License 2.0**
- 원본 고지: `firmware/front-zonal-ecu/Drivers/CMSIS/LICENSE.txt`

## CMSIS Device Support — STM32G4xx

- 경로: `firmware/front-zonal-ecu/Drivers/CMSIS/Device/ST/STM32G4xx/`
- 저작권: STMicroelectronics
- 원본 고지: `firmware/front-zonal-ecu/Drivers/CMSIS/Device/ST/STM32G4xx/LICENSE.txt`

## VL53L0X API

- 경로: `firmware/front-zonal-ecu/Drivers/VL53L0X/`
- 저작권: © 2016 STMicroelectronics International N.V. All rights reserved.
- 라이선스: **BSD-3-Clause 형식** (각 소스 파일 헤더에 전문 포함)
- 원본 고지: 각 `.c` / `.h` 파일 상단 주석

주요 조건 요약(전문은 소스 헤더 참조):

- 소스 재배포 시 위 저작권 고지, 조건 목록, 면책 조항을 유지해야 함
- 바이너리 재배포 시 배포물의 문서 등에 동일 고지를 포함해야 함
- STMicroelectronics 및 기여자의 이름을 사전 서면 허가 없이 파생 제품의 보증·홍보에 사용할 수 없음
- 무보증(AS IS), 책임 제한 조항 적용

> `VL53L0X/` 디렉터리에는 별도 `LICENSE.txt`가 없고 각 소스 파일 헤더에 라이선스 전문이 들어 있습니다. 파일 헤더 주석을 제거하거나 축약하지 마십시오.

## STM32CubeMX 생성 코드

- 경로: `firmware/front-zonal-ecu/Src/`, `firmware/front-zonal-ecu/Inc/`, `firmware/front-zonal-ecu/Startup/`
- 저작권: STMicroelectronics (생성 코드), 사용자 작성 부분은 `USER CODE BEGIN/END` 블록 내
- 각 파일 헤더의 라이선스 고지를 따릅니다.

---

## 유지 규칙

- vendor 코드의 라이선스 헤더를 제거하거나 재포맷하지 않습니다.
- 새 third-party 구성요소를 추가할 때 이 문서에 항목을 함께 추가합니다.
- Yocto 단계(Release C)에서 SPDX SBOM을 생성하면 이 문서와 SBOM 내용이 일치하는지 확인합니다.
