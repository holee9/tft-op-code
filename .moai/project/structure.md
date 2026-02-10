# 시스템 아키텍처 및 구조 (System Architecture & Structure)

## 프로젝트 정보

**프로젝트명**: TFT-Leak-plan
**작성일**: 2026-02-10
**버전**: 3.1 (수정)

---

## 1. 전체 아키텍처 개요 (Overall Architecture)

### 1.1 시스템 계층 구조

```
┌─────────────────────────────────────────────────────────┐
│                    Host PC Layer                         │
│  (향후 별도 프로젝트로 구현 예정)                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Image Processing                                   │ │
│  │  - 2D Dark LUT Management                           │ │
│  │  - Bilinear Interpolation                           │ │
│  │  - Blind Pixel Correction                           │ │
│  │  - Adaptive Lag Correction                          │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  User Interface                                     │ │
│  │  - Configuration Settings                           │ │
│  │  - Status Monitoring                                │ │
│  │  - Diagnostic Display                               │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────┘
                      │ Ethernet (Control + Image Data)
┌─────────────────────▼───────────────────────────────────┐
│                MCU Layer (i.MX8 Plus)                    │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Cortex-A53 (Application Processor)                 │ │
│  │  - .NET Runtime Environment                         │ │
│  │  - Temperature Monitoring                           │ │
│  │  - Arrhenius Model Calculation                      │ │
│  │  - State Machine (L1/L2/L3)                         │ │
│  │  - Metadata Generation                              │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Cortex-M4 (Real-time Core)                         │ │
│  │  - FPGA Real-time Control                           │ │
│  │  - Sensor Acquisition (1 Hz)                        │ │
│  │  - Inter-core Communication (MU)                    │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────┘
                      │ Control Signals, Temperature
┌─────────────────────▼───────────────────────────────────┐
│                   FPGA Layer                             │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Timing Controller                                   │ │
│  │  - Row/Column Clock Generation                       │ │
│  │  - RESET/DARK/READ Sequence                          │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Bias MUX Control                                    │ │
│  │  - 2-bit MUX (Normal/Idle/Low-bias)                 │ │
│  │  - V_PD/V_COL Switching                              │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Dummy Scan Engine                                   │ │
│  │  - Periodic Reset-only Sequence                     │ │
│  │  - No ADC Readout                                    │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  ADC Interface                                       │ │
│  │  - Analog-to-Digital Conversion                      │ │
│  │  - Data Buffering                                    │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────┘
                      │ Gate/Column/Clock Signals
┌─────────────────────▼───────────────────────────────────┐
│                 FPD Panel Layer                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Pixel Array (2048 × 2048)                          │ │
│  │  - a-Si:H TFT Switch                                │ │
│  │  - Storage Node (100~500 pF)                        │ │
│  │  - a-Si:H p-i-n Photodiode                          │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Peripheral Components                               │ │
│  │  - Gate Driver                                      │ │
│  │  - Column Amplifier/ADC                             │ │
│  │  - Bias Generator                                   │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 디렉토리 구조 (Directory Structure)

```
TFT-Leak-plan/
│
├── .moai/                          # MoAI 설정
│   ├── config/                     # 구성 섹션
│   │   ├── sections/               # 모듈별 설정
│   │   │   ├── user.yaml          # 사용자 정보
│   │   │   ├── language.yaml      # 언어 설정 (ko)
│   │   │   ├── project.yaml       # 프로젝트 메타데이터
│   │   │   ├── quality.yaml       # 품질 게이트 설정
│   │   │   └── ...
│   │   └── config.yaml             # 메인 구성 파일
│   ├── project/                    # 프로젝트 문서
│   │   ├── product.md             # 제품 개요
│   │   ├── structure.md           # 아키텍처 문서 (본 파일)
│   │   └── tech.md                # 기술 스택 문서
│   └── announcements/              # 다국어 공지사항
│
├── .claude/                        # Claude Code 설정
│   ├── agents/                     # 에이전트 정의
│   │   └── moai/                   # MoAI 에이전트
│   ├── skills/                     # 스킬 정의
│   │   └── moai/                   # MoAI 스킬
│   ├── rules/                      # 규칙 파일
│   │   └── moai/                   # MoAI 규칙
│   ├── hooks/                      # 훅 스크립트
│   │   └── moai/                   # MoAI 훅
│   └── settings.json               # Claude 설정
│
├── images/                         # 문서용 이미지
│   ├── idle_state.jpg              # Idle 상태 다이어그램
│   ├── dark_signal_drift.png       # Dark 신호 드리프트 그래프
│   ├── aSi_tft_cross-section.jpg   # a-Si TFT 단면 구조
│   ├── stucture.jpg                # 시스템 구조도
│   ├── timining_1.jpg              # 타이밍 다이어그램
│   ├── 2.5fps_clinical_mode.jpg    # 임상 모드 타이밍
│   └── infographic_1.png           # 인포그래픽
│
├── [기술 문서들]                   # 주요 기술 문서
│   ├── nblm_aSi_TFT_plan_v3.0.md                   # 통합 계획서
│   ├── aSi_TFT_Leakage_Implementation_Plan_v3_Final.md
│   ├── aSi_TFT_Drive_Algorithm_ExecutionPlan_v2_2.md
│   ├── aSi_TFT_Idle_Strategy_Report_v3_Final.md
│   ├── aSi_TFT_IdleState_CharacteristicReport.md
│   ├── aSi_TFT_Leakage_v2_Final.md
│   ├── nblm_연구논문_초안.md
│   ├── nblm_연구논문_초안_v1.md
│   └── Vendor_Cooperation_Protocol_Guide.md
│
└── CLAUDE.md                       # 프로젝트 실행 지시문
```

---

## 3. 상태 전환도 (State Transition Diagram)

### 3.1 Idle 모드 상태 머신

```
                    ┌─────────────┐
                    │   ACTIVE    │  (normal readout ongoing)
                    │  (exposure) │
                    └──────┬──────┘
                           │ (readout 완료)
                           ▼
                    ┌─────────────┐
                    │    L1       │  (normal idle)
       ┌────────────│  Idle < 10' │◄─────────────┐
       │            └──────┬──────┘              │
       │                   │ (t_idle > 10 min)   │
       │                   ▼                      │
       │            ┌─────────────┐               │
       │            │    L2       │  (low-bias)   │
       │  ┌─────────│ 10' ≤ Idle  │─────────┐     │
       │  │         │  < 100 min  │         │     │
       │  │         └──────┬──────┘         │     │
       │  │                │ (t_idle > 100 min) │
       │  │                ▼                    │
       │  │         ┌─────────────┐             │
       │  │         │    L3       │  (deep sleep)│
       │  │         │  Idle ≥ 100'│             │
       │  │         └─────────────┘             │
       │  │                                      │
       └──┴── Read Request (Wake-up) ────────────┘
              │
              ▼
         ┌─────────────┐
         │   ACTIVE    │
         └─────────────┘
```

### 3.2 상태별 구성

| 상태 | Bias (V_PD) | Bias (V_COL) | Dummy Scan | Power |
|------|-------------|--------------|------------|-------|
| **ACTIVE** | -1.5V | -1.0V | - | 100% |
| **L1** | -1.5V | -1.0V | 없음 | ~100% |
| **L2** | -0.2V | -0.2V | 60초 주기 | ~85% |
| **L3** | ~0V | ~0V | 없음 | <10% |

---

## 4. 신호 흐름 (Data Flow)

### 4.1 제어 신호 경로

```
┌─────────────┐     control     ┌─────────────┐
│    Host     │───────────────►│    MCU      │
│   (PC/App)  │◄───────────────│  (State M/C)│
└─────────────┘    status      └──────┬──────┘
                                      │ control
                                      ▼
                               ┌─────────────┐
                               │   FPGA      │
                               │ (Timing)    │
                               └──────┬──────┘
                                      │ timing
                                      ▼
                               ┌─────────────┐
                               │   Panel     │
                               │   (FPD)     │
                               └──────┬──────┘
                                      │ analog
                                      ▼
                               ┌─────────────┐
                               │    ADC      │
                               └──────┬──────┘
                                      │ digital
                                      ▼
                               ┌─────────────┐
                               │   Host      │
                               │ (Image Proc)│
                               └─────────────┘
```

### 4.2 온도 모니터링 경로

```
┌─────────────┐     1 Hz       ┌─────────────┐
│   Temp      │───────────────►│    MCU      │
│   Sensor    │◄───────────────│  (Polling)  │
└─────────────┘    enable      └──────┬──────┘
                                      │ T(t)
                                      ▼
                               ┌─────────────────────┐
                               │  t_max(T) Calculator │
                               │  Arrhenius Model     │
                               └──────────┬──────────┘
                                          │
                                          ▼
                               ┌─────────────────────┐
                               │  Mode Transition     │
                               │  Decision Logic      │
                               └─────────────────────┘
```

---

## 5. 모듈 관계도 (Module Relationships)

### 5.1 FPGA 모듈 구성

```
FPGA Top Level
│
├── Bias_Mux_Controller
│   ├── Normal_Bias_Generator (V_PD = -1.5V, V_COL = -1.0V)
│   ├── Idle_Low_Bias_Generator (V_PD = -0.2V, V_COL = -0.2V)
│   └── MUX_Select_Logic (2-bit control from MCU)
│
├── Timing_Generator
│   ├── Row_Clock_Generator (1~10 MHz)
│   ├── Column_Clock_Generator
│   └── Reset_Pulse_Generator (10 µs)
│
├── Sequence_Controller
│   ├── Normal_Read_Sequence
│   │   ├── DARK1/DARK2 frames
│   │   ├── LAG1/LAG2 frames
│   │   └── CLINICAL frame
│   └── High_Speed_Sequence (5 fps mode)
│
├── Dummy_Scan_Engine
│   ├── Periodic_Reset_Trigger (T_dummy = 30~60s)
│   ├── Row-wise_Reset_Only
│   └── No_ADC_Readout
│
└── ADC_Interface
    ├── Analog_Front_End_Control
    └── Digital_Data_Buffer
```

### 5.2 MCU 모듈 구성 (i.MX8 Plus)

```
i.MX8 Plus Top Level
│
├── Cortex-A53 (Application Processor - Linux + .NET)
│   ├── .NET Application Layer
│   │   ├── Temperature_Monitor
│   │   │   ├── Sensor_Polling (1 Hz)
│   │   │   ├── ADC_Read
│   │   │   └── Temperature_Filter
│   │   │
│   │   ├── Drift_Model
│   │   │   ├── Arrhenius_Calculator
│   │   │   │   ├── E_A = 0.45 eV
│   │   │   │   ├── k_ref = 1.0 DN/min @ 25°C
│   │   │   │   └── k(T) = k_ref * exp(E_A/kB * (1/T - 1/T_ref))
│   │   │   └── t_max_Calculator
│   │   │       └── t_max = Delta_D_max / k(T)
│   │   │
│   │   ├── State_Machine
│   │   │   ├── Idle_Timer
│   │   │   ├── Mode_L1_Controller
│   │   │   ├── Mode_L2_Controller
│   │   │   ├── Mode_L3_Controller
│   │   │   └── Transition_Logic
│   │   │
│   │   └── Metadata_Generator
│   │       ├── Frame_Header_Writer
│   │       ├── Temperature_Stamp
│   │       ├── Idle_Time_Stamp
│   │       └── Warmup_Flag
│   │
│   └── Communication_Services
│       ├── FPGA_Protocol_Service
│       ├── Ethernet_Service
│       └── Host_Protocol_Service
│
└── Cortex-M4 (Real-time Core - FreeRTOS)
    ├── HAL_Drivers
    │   ├── GPIO_Driver
    │   ├── SPI_Driver (FPGA 통신)
    │   ├── I2C_Driver (센서 통신)
    │   ├── UART_Driver
    │   └── ADC_Driver (온도 센서)
    │
    ├── Real-time_Tasks
    │   ├── FPGA_Control_Task
    │   ├── Sensor_Acquisition_Task
    │   └── Watchdog_Task
    │
    └── Inter-Core_Communication
        ├── MU (Messaging Unit) - A53↔M4 통신
        └── Shared_Memory_Region
```

### 5.3 Host SW 모듈 구성 (향후 구현)

> **참고**: 호스트 소프트웨어는 본 프로젝트 범위에 포함되지 않습니다.
> 향후 별도 프로젝트로 구현이 계획되어 있습니다.

```
Host Application (Future Implementation)
│
├── Image_Processing
│   ├── 2D_LUT_Manager
│   │   ├── Dark_LUT_2D[T][t_idle]
│   │   └── Bilinear_Interpolation
│   ├── Blind_Pixel_Corrector
│   │   ├── Region_Definition
│   │   └── Offset_Calculator
│   └── Lag_Corrector
│       └── Adaptive_Coefficient_Selector
│
├── Device_Interface
│   ├── Ethernet_Driver
│   ├── Frame_Receiver
│   └── Metadata_Parser
│
├── Calibration_Tool
│   ├── Dark_LUT_Builder
│   ├── Lag_Measurement
│   └── Temperature_Chamber_Interface
│
└── User_Interface
    ├── Status_Display
    ├── Configuration_Panel
    └── Diagnostic_View
```

---

## 6. 외부 시스템 통합 (External Integration)

### 6.1 벤더 협조 인터페이스

| 항목 | 인터페이스 형태 | 목적 |
|------|----------------|------|
| 타이밍 스펙 | 데이터시트/기술 문의 | Frame time 최적화 |
| Dark vs Temperature | 그래프/테이블 | LUT 보정 |
| Standby 권장사항 | Application Note | Idle 모드 설계 |
| Blind Pixel 정보 | 픽셀 맵 | Frame 레벨 보정 |

### 6.2 자체 측정 프로토콜

```
┌─────────────────────────────────────────────────────┐
│           Temperature Chamber                       │
│  (Control: 15~40°C, ±2°C accuracy)                  │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│           Measurement Protocol                       │
│  1. 15 min thermal stabilization                     │
│  2. Dark capture at t = 0, 30, 60, 120, 240 min     │
│  3. Extract k(T) from D(t) = D0 + k(T) * t          │
│  4. Build 2D LUT                                    │
└─────────────────────────────────────────────────────┘
```

---

## 7. 타이밍 다이어그램 (Timing Diagrams)

### 7.1 Normal Read Sequence (250ms mode)

```
DARK1:     RESET──┐          ┌──READ (250ms)──┐
           │          │                      │
           └──100ms───┘                      │

DARK2:     ─────────RESET──┐          ┌──READ (250ms)──┐
                          │          │                 │
                          └──100ms───┘                 │

LAG1:      ─────────────────────RESET──┐──EXPOSE(10ms)──READ (250ms)──┐
                                    │                              │
                                    └──────────────────────────────┘

LAG2:      ───────────────────────────────RESET──┐          ┌──READ──┐
                                             │          │         │
                                             └──100ms───┘         │

CLINICAL:  ───────────────────────────────────RESET──┐──EXPOSE(1000ms)──READ──┐
                                                   │                        │
                                                   └────────────────────────┘

Total cycle time: ~1.5 sec
```

### 7.2 High-Speed Mode (5 fps)

```
Frame N:   RESET──┐──EXPOSE──┐──READ (50ms)──┐
           │      │         │               │
           └─150ms┘         └───┬───────────┘
                                  │

Frame N+1: ─────────RESET──┐──EXPOSE──┐──READ──┐
                          │         │        │
                          └─150ms───┘        │

Frame period: 200 ms (5 fps)

Async Dark: (every 2 sec, ~10 frames)
            ─────────RESET──┐──READ──┐
                            │        │
                            └─150ms──┘
```

### 7.3 Dummy Scan (L2 mode)

```
Idle time accumulates...

Every 60 seconds:
┌─────────────────────────────────────────┐
│ DUMMY_SCAN (No Output)                  │
│                                         │
│ for row = 0 to 2047:                   │
│   SET gate[row] = ON                   │
│   TRIGGER_RESET pulse (1 µs)            │
│   WAIT settle (100 µs)                  │
│   SET gate[row] = OFF                  │
│   (NO READOUT)                          │
│                                         │
│ Total: ~2 ms for all rows               │
└─────────────────────────────────────────┘

Dark accumulation resets every 60 seconds
```

---

## 8. 데이터 구조 (Data Structures)

### 8.1 프레임 메타데이터

```c
typedef struct {
    uint32_t timestamp;           // Unix timestamp
    float temperature;            // Celsius
    uint32_t idle_time_sec;       // Seconds since last reset
    uint8_t idle_mode;            // L1=0, L2=1, L3=2
    uint8_t warmup_required;      // 1 if first frame after long idle
    uint8_t frame_type;           // DARK1=0, DARK2=1, LAG1=2, LAG2=3, CLINICAL=4
    uint16_t frame_number;        // Sequence number
    uint32_t exposure_time_ms;    // Integration time
} FrameMetadata;
```

### 8.2 2D Dark LUT 구조

```c
// Temperature grid: 15, 20, 25, 30, 35, 40°C (6 points)
// Idle time grid: 0, 10, 30, 60, 120, 240 min (6 points)
// Total: 6x6 = 36 entries

typedef struct {
    float temp_celsius;           // Temperature in °C
    uint32_t idle_time_min;       // Idle time in minutes
    uint16_t dark_dn;             // Estimated dark level [DN]
    uint16_t sigma_dn;            // Standard deviation
} DarkLUT_Entry;

DarkLUT_Entry dark_lut[6][6];

// Runtime interpolation
float estimate_dark_level(float T, uint32_t t_idle, DarkLUT_Entry lut[6][6]);
```

---

## 9. 아키텍처 결정 배경 (Architecture Decisions)

### 9.1 3계층 분산 아키텍처 채택 이유

- **FPGA**: 하드웨어 타이밍 제어 (µs 단위 정밀도 필요)
- **MCU**: 상태 관리 및 온도 모델링 (1Hz 폴링 충분)
- **Host**: 대용량 LUT 및 복잡한 보정 연산

### 9.2 3단계 Idle 모드 계층화 이유

- **L1**: 짧은 대기 시 빠른 응답 필요
- **L2**: 일반 대기 시 dark 억제와 전력 절감 균형
- **L3**: 장시간 보관 시 최소 전력 소모

### 9.3 2D LUT 채택 이유

- 1D LUT (온도만)으로는 idle 시간 효과 반영 불가
- 물리적 모델 복잡도 고려하여 측정 기반 LUT 선택
- 6×6 = 36개 포인트로 실용적인 메모리 사용량

---

## 10. 기술적 제약사항 (Technical Constraints)

| 항목 | 제약 | 영향 |
|------|------|------|
| 패널 해상도 | 2048 × 2048 | Read time 최소 50ms |
| 온도 센서 | 1 Hz 폴링 | 온도 변화 응답 지연 |
| FPGA 리소스 | 제한된 DSP 블록 | 보정 알고리즘은 Host로 이동 |
| MCU 연산력 | FPU 없을 수 있음 | Arrhenius 모델 근사화 필요 |
| Ethernet 대역폭 | ~1 Gbps | 5fps 이상 전송 가능 |

---

## 11. .NET 크로스 플랫폼 개발 아키텍처

### 11.1 개발 플로우 (Windows → i.MX8 Plus)

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows 개발 환경                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Visual Studio / VS Code + C# DevKit                 │  │
│  │  - .NET 8.0 SDK                                      │  │
│  │  - 단위 테스트 (xUnit)                                │  │
│  │  - 하드웨어 모의 (Moq)                                │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ 크로스 컴파일                  │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  dotnet publish -c Release -r linux-arm64            │  │
│  │  └── ARM64 실행 파일 생성                              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ 배포 (SSH/SCP/FTP)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    i.MX8 Plus 장치                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Yocto Linux (ARM64)                                  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  .NET 8.0 Runtime (meta-dotnet)                │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │  TftLeakageController                     │  │  │  │
│  │  │  │  ├── Temperature_Monitor                  │  │  │  │
│  │  │  │  ├── Arrhenius_Calculator                 │  │  │  │
│  │  │  │  ├── State_Machine                        │  │  │  │
│  │  │  │  └── Hardware_Interface                   │  │  │  │
│  │  │  │      ├── System.Device.Gpio (GPIO)        │  │  │  │
│  │  │  │      ├── System.Device.Spi (FPGA)         │  │  │  │
│  │  │  │      ├── System.IO.Ports (UART)           │  │  │  │
│  │  │  │      └── P/Invoke (NXP HAL)               │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Systemd 서비스                                  │  │  │
│  │  │  - 자동 시작                                     │  │  │
│  │  │  - 재시작 정책                                   │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 .NET 애플리케이션 모듈 구조

```
TftLeakageController/
│
├── TftLeakageController.csproj    # 프로젝트 파일
│   ├── <TargetFramework>net8.0</TargetFramework>
│   ├── <RuntimeIdentifiers>win-x64;linux-arm64</RuntimeIdentifiers>
│   └── 조건부 컴파일 정의
│
├── Core/                           # 플랫폼 독립적 코어
│   ├── Models/                     # 데이터 모델
│   │   ├── FrameMetadata.cs
│   │   ├── TemperatureData.cs
│   │   ├── IdleState.cs
│   │   └── ArrheniusModel.cs
│   │
│   ├── Services/                   # 비즈니스 로직
│   │   ├── ITemperatureMonitor.cs
│   │   ├── TemperatureMonitor.cs
│   │   ├── IArrheniusCalculator.cs
│   │   ├── ArrheniusCalculator.cs
│   │   ├── IStateMachine.cs
│   │   ├── IdleStateMachine.cs
│   │   └── MetadataGenerator.cs
│   │
│   └── Interfaces/                 # 하드웨어 추상화
│       ├── IGpioController.cs
│       ├── ISpiDevice.cs
│       ├── II2cDevice.cs
│       └── IUartDevice.cs
│
├── Hardware/                       # 플랫폼별 하드웨어 구현
│   ├── Linux/                      # Linux 전용 구현
│   │   ├── GpioControllerImpl.cs   # System.Device.Gpio
│   │   ├── SpiDeviceImpl.cs        # System.Device.Spi
│   │   ├── I2cTemperatureSensor.cs # System.Device.I2c
│   │   ├── UartFpgaComm.cs         # System.IO.Ports
│   │   └── NativeHal.cs            # P/Invoke (libnxphal.so)
│   │
│   └── Windows/                    # Windows 시뮬레이션
│       ├── MockGpioController.cs
│       ├── MockSpiDevice.cs
│       └── SimulatorUart.cs
│
├── InterCore/                      # A53↔M4 통신
│   ├── MessagingUnitClient.cs      # MU 드라이버 래퍼
│   ├── SharedMemoryRegion.cs       # 공유 메모리 관리
│   └── M4Protocol.cs               # M4 통신 프로토콜
│
├── Communication/                  # 외부 통신
│   ├── FpgaProtocolService.cs      # FPGA 제어 프로토콜
│   ├── HostProtocolService.cs      # Host 통신 프로토콜
│   └── EthernetServer.cs           # Ethernet 서버
│
├── Infrastructure/                 # 인프라스트럭처
│   ├── Logging/                    # 로깅
│   │   ├── ILoggerService.cs
│   │   └── SerilogLogger.cs
│   │
│   ├── Configuration/              # 설정 관리
│   │   ├── AppConfig.cs
│   │   └── HardwareConfig.cs
│   │
│   └── Monitoring/                 # 모니터링
│       ├── HealthCheck.cs
│       └── Diagnostics.cs
│
└── Program.cs                      # 진입점
    ├── CreateHostBuilder()
    ├── ConfigureServices()
    └── ConfigureLogging()
```

### 11.3 플랫폼별 구현 패턴

#### 11.3.1 추상화 팩토리 패턴

```csharp
// 하드웨어 추상화를 위한 팩토리
public interface IHardwareFactory
{
    IGpioController CreateGpioController();
    ISpiDevice CreateSpiDevice(SpiConnectionSettings settings);
    II2cDevice CreateI2cDevice(I2cConnectionSettings settings);
}

// Linux 구현
public class LinuxHardwareFactory : IHardwareFactory
{
    public IGpioController CreateGpioController()
    {
        return new GpioControllerImpl();  // System.Device.Gpio
    }

    // ... 기타 구현
}

// Windows 구현 (시뮬레이션)
public class WindowsHardwareFactory : IHardwareFactory
{
    public IGpioController CreateGpioController()
    {
        return new MockGpioController();
    }

    // ... 기타 구현
}

// 팩토리 선택
public static class HardwareFactoryProvider
{
    public static IHardwareFactory CreateFactory()
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
        {
            return new LinuxHardwareFactory();
        }
        else
        {
            return new WindowsHardwareFactory();
        }
    }
}
```

#### 11.3.2 의존성 주입 설정

```csharp
// Program.cs
public static IHost CreateHostBuilder(string[] args)
{
    return Host.CreateDefaultBuilder(args)
        .ConfigureServices((context, services) =>
        {
            // 플랫폼에 따른 서비스 등록
            if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
            {
                services.AddSingleton<IGpioController, GpioControllerImpl>();
                services.AddSingleton<ISpiDevice, SpiDeviceImpl>();
                services.AddSingleton<II2cDevice, I2cTemperatureSensor>();
            }
            else
            {
                // Windows 개발 환경: 모의 객체
                services.AddSingleton<IGpioController, MockGpioController>();
                services.AddSingleton<ISpiDevice, MockSpiDevice>();
                services.AddSingleton<II2cDevice, MockI2cDevice>();
            }

            // 플랫폼 독립적 서비스
            services.AddSingleton<ITemperatureMonitor, TemperatureMonitor>();
            services.AddSingleton<IArrheniusCalculator, ArrheniusCalculator>();
            services.AddSingleton<IStateMachine, IdleStateMachine>();

            // 호스팅 서비스
            services.AddHostedService<TemperatureMonitoringService>();
            services.AddHostedService<StateManagementService>();
        })
        .ConfigureLogging(logging =>
        {
            logging.ClearProviders();
            logging.AddConsole();
            logging.AddDebug();
        })
        .BuildServiceProvider();
}
```

### 11.4 Inter-Core 통신 아키텍처 (A53 ↔ M4)

```
┌─────────────────────────────────────────────────────────────┐
│              Cortex-A53 (Linux + .NET)                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  .NET Application                                     │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  MessagingUnitClient                            │  │  │
│  │  │  - Open MU device (/dev/mu)                     │  │  │
│  │  │  - Send message to M4                           │  │  │
│  │  │  - Receive message from M4                      │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │ MU Hardware
                              │ (Memory Mapped Register)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Messaging Unit (MU) Hardware                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   TX Ring   │  │   RX Ring   │  │   Status Register   │  │
│  │  (A53→M4)   │  │  (M4→A53)   │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │ MU Hardware
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Cortex-M4 (FreeRTOS)                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Firmware Task                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  MU ISR Handler                                 │  │  │
│  │  │  - Check RX register                            │  │  │
│  │  │  - Parse message                                │  │  │
│  │  │  - Execute command                              │  │  │
│  │  │  - Send response                                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Real-time Tasks                                │  │  │
│  │  │  - FPGA Control Task                            │  │  │
│  │  │  - Sensor Acquisition Task                      │  │  │
│  │  │  - Watchdog Task                                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 11.5 Yocto 레이어 구조

```
meta-imx-sdk/                     # NXP 공식 SDK
├── recipes-core/
│   └── dotnet/
│       └── dotnet_8.0.bb         # .NET 8.0 레시피
│
meta-tft-leakage/                 # 프로젝트 전용 레이어
├── recipes-core/
│   └── tft-leakage-controller/
│       ├── tft-leakage-controller_1.0.bb
│       └── files/
│           ├── tft-leakage.service
│           └── config.json
│
├── recipes-kernel/
│   └── tft-leakage/
│       └── kernel-module/
│           └── mu-driver_1.0.bb  # MU 커널 드라이버
│
└── conf/
    └── layer.conf
```

### 11.6 배포 아티팩트

```
배포 패키지 구조:
tft-leakage-controller-arm64.tar.gz
├── TftLeakageController           # 메인 실행 파일
├── TftLeakageController.dll        # 어셈블리
├── TftLeakageController.deps.json  # 의존성 맵
├── TftLeakageController.runtimeconfig.json
├── System.Device.Gpio.dll
├── System.Device.Spi.dll
├── System.IO.Ports.dll
├── ... (기타 의존성)
├── config.json                     # 설정 파일
├── tft-leakage.service             # Systemd 서비스 파일
└── README.deploy.txt               # 배포 가이드
```

---

**문서 버전**: 3.2
**최종 수정일**: 2026-02-10
**상태**: .NET 크로스 플랫폼 아키텍처 섹션 추가 완료
**변경 내역**:
- MCU 아키텍처: i.MX8 Plus (Cortex-A53 + Cortex-M4)로 업데이트
- 호스트 SW: 향후 별도 프로젝트로 구현 예정 표기 추가
- **[NEW]** 섹션 11 추가: .NET 크로스 플랫폼 개발 아키텍처
  - Windows → i.MX8 Plus 개발 플로우
  - .NET 애플리케이션 모듈 구조
  - 플랫폼별 구현 패턴 (추상화 팩토리)
  - Inter-Core 통신 (A53↔M4)
  - Yocto 레이어 구조
  - 배포 아티팩트 구성
