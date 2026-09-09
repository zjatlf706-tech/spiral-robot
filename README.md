# spiral-robot
spiral robot 제작과정을 정리합니다.
# AK40-10 Motor Control with STM32 NUCLEO-F401RE

## 1. 프로젝트 개요

본 프로젝트는 **STM32 NUCLEO-F401RE** 보드를 이용하여 **CubeMars AK40-10 모터를 UART 통신으로 제어**하는 프로젝트입니다.

기존에는 Arduino IDE 환경에서 STM32의 UART를 이용하여 AK40-10 모터를 제어하였으며, 해당 코드가 실제 모터를 정상적으로 구동하는 것을 확인했습니다.

이후 개발 환경을 다음과 같이 변경했습니다.

- Arduino IDE
→ STM32CubeMX
→ Visual Studio Code
→ CMake
→ STM32 HAL

기존 Arduino 코드의 AK40-10 통신 프로토콜은 유지하면서 STM32 HAL 기반 코드로 변환했습니다.


---

# 2. 개발 환경

## Hardware

- MCU Board: STM32 NUCLEO-F401RE
- MCU: STM32F401RET6
- Motor: CubeMars AK40-10
- Debugger: On-board ST-LINK
- Motor Communication: UART
- PC Debug Communication: ST-LINK Virtual COM Port


## Software

- STM32CubeMX
- Visual Studio Code
- STM32CubeIDE for Visual Studio Code
- CMake
- Ninja
- STM32 HAL Driver
- VS Code Serial Monitor


---

# 3. UART 구성

두 개의 UART를 서로 다른 목적으로 사용합니다.

## USART6 - AK40-10

AK40-10 모터와 통신합니다.

| 항목 | 설정 |
|---|---|
| Peripheral | USART6 |
| TX | PA11 |
| RX | PA12 |
| Baud Rate | 921600 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Mode | TX/RX |

배선:

    STM32 PA11 (USART6_TX) → AK40-10 RX
    STM32 PA12 (USART6_RX) ← AK40-10 TX
    STM32 GND              ─ AK40-10 GND


## USART2 - PC Debug

PC Serial Monitor 출력에 사용합니다.

| 항목 | 설정 |
|---|---|
| Peripheral | USART2 |
| TX | PA2 |
| RX | PA3 |
| Baud Rate | 115200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |

NUCLEO-F401RE의 ST-LINK Virtual COM Port를 통해 VS Code Serial Monitor에서 확인할 수 있습니다.


---

# 4. 프로젝트 구조

AK40-10 관련 코드를 `main.c`에 모두 작성하지 않고 별도의 드라이버로 분리했습니다.

    ak_motor_swip/
    │
    ├── Core/
    │   ├── Inc/
    │   │   ├── main.h
    │   │   └── ak40.h
    │   │
    │   └── Src/
    │       ├── main.c
    │       └── ak40.c
    │
    ├── Drivers/
    ├── cmake/
    ├── CMakeLists.txt
    ├── CMakePresets.json
    └── ak_motor_swip.ioc

각 파일의 역할은 다음과 같습니다.

### main.c

- STM32 HAL 초기화
- USART2 초기화
- USART6 초기화
- AK40 드라이버 초기화
- 모터 제어 명령 실행
- PC Serial Monitor 디버깅 출력

### ak40.c

- AK40 UART 패킷 생성
- CRC16 계산
- Position 명령
- Position + Speed + Acceleration 명령
- USART6 데이터 송신

### ak40.h

- AK40 명령 ID
- 함수 prototype
- AK40 드라이버 인터페이스


---

# 5. Arduino 코드에서 STM32 HAL로 변환

기존 Arduino 코드에서는 다음과 같은 방식으로 모터 UART를 사용했습니다.

    Uart motorSerial(PA12, PA11);

    motorSerial.begin(921600);

그리고 패킷 전송은 다음과 같이 수행했습니다.

    motorSerial.write(packet, count);

STM32 HAL에서는 다음과 같이 변경했습니다.

    HAL_UART_Transmit(
        &huart6,
        packet,
        packet_length,
        HAL_MAX_DELAY
    );

Arduino의

    delay(2000);

은 STM32 HAL에서

    HAL_Delay(2000);

으로 변경했습니다.


---

# 6. AK40-10 Position 명령

Position 명령 ID는 다음과 같습니다.

    COMM_SET_POS = 9

위치 값은 다음과 같이 변환합니다.

    position = degrees × 1,000,000

예를 들어 90도의 경우:

    90.0 × 1,000,000
    = 90,000,000

변환된 `int32_t` 데이터는 Big-Endian 순서로 payload에 저장합니다.

Position 패킷 구조:

    0x02
    Payload Length
    COMM_SET_POS
    Position Byte 3
    Position Byte 2
    Position Byte 1
    Position Byte 0
    CRC High
    CRC Low
    0x03


---

# 7. Position + Speed 명령

사용한 명령 ID:

    COMM_SET_POS_SPD = 91

Payload는 총 13 bytes입니다.

    Command       1 byte
    Position      4 bytes
    Speed         4 bytes
    Acceleration  4 bytes

각 값의 scale은 기존에 정상 동작했던 Arduino 코드와 동일하게 유지했습니다.

    Position     = degree × 1,000,000
    Speed        = speed × 1,000
    Acceleration = acceleration × 1,000

모든 32-bit 데이터는 Big-Endian으로 전송합니다.


---

# 8. CRC16

AK40-10 패킷에는 CRC16을 추가합니다.

사용한 CRC polynomial:

    0x1021

CRC는 payload에 대해 계산하고 다음 순서로 패킷에 추가합니다.

    CRC High Byte
    CRC Low Byte


---

# 9. 모터 동작 테스트

다음 순서로 테스트했습니다.

    90°
     ↓
    180°
     ↓
    270°
     ↓
    360°
     ↓
    Position + Speed Control
     ↓
    180°
     ↓
    0°

예:

    AK40_SetPosition(90.0f);
    HAL_Delay(2000);

    AK40_SetPosition(180.0f);
    HAL_Delay(2000);

    AK40_SetPosition(270.0f);
    HAL_Delay(2000);

    AK40_SetPosition(360.0f);
    HAL_Delay(2000);

Position + Speed 제어:

    AK40_SetPositionSpeed(
        180.0f,
        500.0f,
        1000.0f
    );

그리고 빠르게 0도로 복귀:

    AK40_SetPositionSpeed(
        0.0f,
        2000.0f,
        3000.0f
    );


---

# 10. 문제 1 - AK40 함수 Undefined Reference

## 증상

빌드 과정에서 다음 오류가 발생했습니다.

    undefined reference to `AK40_Init'

    undefined reference to `AK40_SetPosition'

    undefined reference to `AK40_SetPositionSpeed'

컴파일 자체는 진행되었지만 최종 ELF 생성 단계에서 실패했습니다.


## 원인

`ak40.h`는 정상적으로 include되어 있었기 때문에 `main.c`는 함수 선언을 알고 있었습니다.

하지만 새로 생성한

    Core/Src/ak40.c

파일이 CMake의 build source에 등록되지 않았습니다.

따라서 컴파일/링크 과정에 `ak40.c`가 포함되지 않았습니다.


## 해결

프로젝트 최상위 `CMakeLists.txt`의 다음 부분을 수정했습니다.

기존:

    target_sources(${CMAKE_PROJECT_NAME} PRIVATE
        # Add user sources here
    )

수정:

    target_sources(${CMAKE_PROJECT_NAME} PRIVATE
        Core/Src/ak40.c
    )

또한 include 경로를 명시했습니다.

    target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        ${CMAKE_SOURCE_DIR}/Core/Inc
    )

이후 다시 Build하여 `ak40.c`가 정상적으로 컴파일되고 ELF가 생성되는 것을 확인했습니다.


---

# 11. 문제 2 - 잘못된 STM32 VS Code Extension

처음에는 다음 VS Code Extension이 동작했습니다.

    STM32 for VSCode
    Publisher: bmd

해당 Extension에서 다음과 같이 OpenOCD를 설치하려는 동작이 나타났습니다.

    xpm install --global @xpack-dev-tools/openocd@latest

이번 프로젝트에서는 STMicroelectronics의 공식 STM32 VS Code 환경을 사용하기로 했기 때문에 해당 Extension을 사용하지 않았습니다.

사용한 Extension:

    STM32CubeIDE for Visual Studio Code
    Publisher: STMicroelectronics

NUCLEO-F401RE에 내장된 ST-LINK를 이용하여 프로그램 다운로드 및 디버깅을 진행합니다.


---

# 12. STM32 업로드

빌드가 성공하면 다음 ELF 파일이 생성됩니다.

    ak_motor_swip.elf

정상적인 빌드 결과:

    Linking C executable ak_motor_swip.elf

    Build finished with exit code 0

VS Code의 Run and Debug에서 NUCLEO의 ST-LINK를 사용하기 위해 다음 항목을 선택합니다.

    STM32Cube: STM32 Launch STLink GDB Server

J-Link를 사용하지 않으므로 다음 항목은 선택하지 않습니다.

    STM32Cube: STM32 Launch JLink GDB Server


---

# 13. Serial Monitor

PC 디버깅 출력은 USART2를 사용합니다.

VS Code Serial Monitor 설정:

    Port      : ST-LINK Virtual COM Port
    Baud Rate : 115200
    Data Bits : 8
    Parity    : None
    Stop Bits : 1

Windows에서는 장치 관리자에서 다음과 비슷한 장치를 확인할 수 있습니다.

    STMicroelectronics STLink Virtual COM Port (COMx)

주의:

    USART6 = 921600 → AK40-10

    USART2 = 115200 → PC Serial Monitor

따라서 PC Serial Monitor는 921600이 아니라 115200으로 연결합니다.


---

# 14. Serial Monitor 결과

실제 실행 시 다음과 같은 로그를 확인했습니다.

    [MOTOR] Moving to 270 degrees...
    [TX] Position 270 : OK

    [MOTOR] Moving to 360 degrees...
    [TX] Position 360 : OK

    [MOTOR] Moving to 180 degrees slowly...
    [TX] Position-Speed 180 : OK

이는 STM32 프로그램이 정상적으로 실행되고 있으며 `AK40_SetPosition()` 및 `AK40_SetPositionSpeed()`에서 USART6 송신이 성공했다는 것을 의미합니다.


---

# 15. TX OK의 의미

다음 메시지는:

    [TX] Position 180 : OK

AK40-10이 명령을 정상적으로 처리했다는 ACK를 의미하는 것은 아닙니다.

현재 코드에서는 `HAL_UART_Transmit()`의 반환값을 확인합니다.

따라서 `TX OK`가 의미하는 것은:

    STM32
      ↓
    AK40 packet 생성
      ↓
    USART6
      ↓
    HAL_UART_Transmit()
      ↓
    HAL_OK

즉 **STM32가 USART6를 통해 패킷 송신을 성공적으로 완료했다는 의미**입니다.

모터의 실제 응답까지 검증하려면 USART6 RX 처리가 추가로 필요합니다.


---

# 16. 현재까지 확인된 사항

- STM32CubeMX 프로젝트 생성
- VS Code CMake Build 성공
- `ak40.c` 별도 드라이버 구성
- Arduino AK40 UART 패킷 STM32 HAL로 변환
- CRC16 적용
- USART6 921600 bps 송신
- USART2 115200 bps PC 디버깅
- ST-LINK를 이용한 프로그램 실행
- Position 명령 송신
- Position + Speed 명령 송신
- VS Code Serial Monitor 출력 확인


---

# 17. 다음 개발 목표

현재는 TX 중심의 모터 제어가 구현되어 있습니다.

다음 단계에서는 USART6 RX를 구현하여 AK40-10의 실제 응답을 분석할 예정입니다.

목표 구조:

    STM32
       │
       │ USART6 TX
       ▼
    AK40-10
       │
       │ USART6 RX
       ▼
    STM32
       │
       ├── Frame Detection
       ├── Payload Length Check
       ├── CRC16 Check
       ├── Packet Parsing
       │
       ▼
    USART2
       │
       ▼
    VS Code Serial Monitor

추가 예정 기능:

- USART6 RX
- DMA 또는 Interrupt 기반 수신
- RAW RX HEX 출력
- Frame parsing
- CRC validation
- Motor position 확인
- Motor speed 확인
- Motor current 확인
- Motor temperature 확인
- Fault code 확인


---

# 18. 요약

이 프로젝트에서는 Arduino IDE에서 정상 동작하던 CubeMars AK40-10 UART 제어 코드를 STM32 HAL 기반 프로젝트로 변환했습니다.

핵심 변환은 다음과 같습니다.

    Arduino Uart
        ↓
    STM32 HAL UART

    motorSerial.write()
        ↓
    HAL_UART_Transmit()

    delay()
        ↓
    HAL_Delay()

    Arduino loop()
        ↓
    STM32 while(1)

그리고 AK40-10의 기존 패킷 구조, CRC16, Command ID 및 데이터 scale은 유지했습니다.

현재 STM32 NUCLEO-F401RE에서 USART6를 이용한 AK40-10 명령 송신과 USART2를 이용한 PC Serial Monitor 디버깅까지 구현된 상태입니다.
