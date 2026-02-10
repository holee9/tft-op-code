# 기술 스택 및 개발 환경 (Technology Stack)

## 프로젝트 정보

**프로젝트명**: TFT-Leak-plan
**작성일**: 2026-02-10
**버전**: 3.1 (수정)

---

## 1. 하드웨어 스펙 (Hardware Specifications)

### 1.1 FPD 패널 사양

| 항목 | 사양 | 비고 |
|------|------|------|
| 패널 타입 | a-Si TFT 기반 간접변환 Flat Panel Detector | 의료/산업용 표준 |
| 해상도 | 2048 × 2048 픽셀 | Full Resolution |
| 픽셀 피치 | 100~200 µm | 제조사별 상이 |
| 스토리지 커패시턴스 | 100~500 pF/픽셀 | 설계 의존 |
| TFT OFF 누설 전류 | 1~10 pA @ 25°C, -2V | 온도 의존성 큼 |
| 동작 온도 | 10~40°C (보증 범위) | 권장 20~35°C |
| 디지털 출력 | 14-bit ADC | 16384 DN 레인지 |

### 1.2 FPGA 하드웨어

| 항목 | 최소 요구사항 | 권장 사양 |
|------|--------------|-----------|
| FPGA | Xilinx Artix-7 또는 동급 | Kintex-7 이상 |
| 로직 셀 | 50K LUT | 100K LUT |
| DSP 슬라이스 | 50개 | 100개 |
| 블록 RAM | 500 KB | 1 MB |
| I/O | 3.3V LVDS | 고속 인터페이스 지원 |
| 타이밍 정확도 | ±1% | ±0.1% |

### 1.3 MCU 하드웨어 (i.MX8 Plus)

| 항목 | 사양 |
|------|------|
| 플랫폼 | NXP i.MX 8 Plus Applications Processor |
| 아키텍처 | Dual-core: Cortex-A53 (Applications) + Cortex-M4 (Real-time) |
| 클럭 | Cortex-A53: 최대 1.5 GHz, Cortex-M4: 최대 400 MHz |
| 메모리 | 64-bit LPDDR4 지원 |
| ADC | 12-bit, 4채널 이상 (온도 센서용) |
| 통신 | SPI/I2C (FPGA 통신), USB/Ethernet (Host 통신) |
| 타이머 | 32-bit 타이머 2개 이상 |
| 개발 환경 | .NET 지원 가능 |

### 1.4 호스트 PC (Host Software)

| 항목 | 최소 요구사항 |
|------|--------------|
| CPU | Intel Core i5 또는 동급 |
| RAM | 8 GB |
| 저장소 | 500 MB 여유 공간 |
| OS | Windows 10/11, Linux (Ubuntu 20.04+) |
| 통신 | USB 3.0 또는 Gigabit Ethernet |

---

## 2. FPGA 기술 스택 (FPGA Technology Stack)

### 2.1 개발 도구

| 도구 | 용도 | 버전 |
|------|------|------|
| **Vivado** | Xilinx FPGA 개발 | 2025.2 |
| **ModelSim/Questa** | 시뮬레이션 | 10.0 이상 |
| **Git** | 버전 관리 | 2.30 이상 |

### 2.2 RTL 설계 언어

- **주 언어**: SystemVerilog (IEEE 1800-2017)
- **보조**: VHDL (기존 모듈 호환용)

### 2.3 핵심 IP 블록

```
FPGA IP Blocks
│
├── Timing_Generator_IP
│   ├── Row_Clock_Generator (PLL based)
│   ├── Column_Clock_Generator
│   └── Reset_Pulse_Generator
│
├── Bias_Mux_IP
│   ├── 2-bit MUX (Normal/Idle/Low-bias)
│   └── DAC Interface
│
├── ADC_Controller_IP
│   ├── ADC SPI Interface
│   └── Data FIFO Buffer
│
├── Dummy_Scan_Engine_IP
│   ├── Periodic Timer
│   └── Row-only Reset Controller
│
└── USB/Uart_Interface_IP
    ├── Control Register Interface
    └── Metadata Injection
```

### 2.4 타이밍 제약

| 항목 | 값 | 비고 |
|------|-----|------|
| 최대 라인 클럭 | 10 MHz | 100 ns/row |
| 최소 리셋 펄스 | 1 µs | TFT OFF → ON 안정화 |
| 바이어스 전환 시간 | < 10 µs | MUX switching |
| ADC 샘플링 속도 | 20 MSPS | 50 ms readout |

---

## 3. MCU 기술 스택 (MCU Technology Stack)

### 3.1 개발 도구 (i.MX8 Plus)

| 도구 | 용도 | 버전 |
|------|------|------|
| **Yocto Project** | 임베디드 Linux 빌드 | 4.0 이상 |
| **VS Code** | 통합 개발 환경 | 최신 버전 |
| **.NET SDK** | 애플리케이션 개발 | 8.0 이상 |
| **GCC ARM** | Cortex-M4 크로스 컴파일 | 10.3 이상 |
| **NXP MCUXpresso** | i.MX 전용 IDE | 최신 버전 |

### 3.2 펌웨어 구조 (i.MX8 Plus)

```c
// i.MX8 Plus Firmware Structure

// Cortex-A53 (Application Processor - Linux)
├── .NET Application Layer
│   ├── Temperature_Monitor
│   │   ├── Sensor_Polling_Task
│   │   └── Filter_Module
│   ├── Drift_Model
│   │   ├── Arrhenius_Calculator
│   │   └── t_max_Calculator
│   ├── State_Machine
│   │   ├── Idle_Timer
│   │   └── Mode_Controller
│   └── Communication
│       ├── FPGA_Protocol
│       └── Host_Protocol

// Cortex-M4 (Real-time Core)
├── HAL_Drivers
│   ├── GPIO
│   ├── UART/USB
│   ├── SPI/I2C
│   ├── ADC (Temperature)
│   └── TIM (PWM, Timers)

// Real-time Firmware
├── FreeRTOS
│   ├── Task Scheduler
│   ├── Queue Management
│   └── Semaphore

└── Hardware_Interface
    ├── FPGA_Control
    └── Sensor_Interface
```

### 3.3 주요 라이브러리 (i.MX8 Plus)

| 라이브러리 | 용도 | 라이선스 |
|-----------|------|---------|
| .NET Runtime | 애플리케이션 프레임워크 | MIT |
| FreeRTOS | Cortex-M4 실시간 OS | MIT |
| NXP HAL | i.MX8 하드웨어 추상화 | 프로프리어터리 |
| TinyExpr | 수식 파싱 (Arrhenius) | BSD |
| Circular Buffer | 데이터 버퍼링 | MIT |

---

## 4. 호스트 소프트웨어 기술 스택 (Host Software Stack)

> **참고**: 호스트 소프트웨어 구현은 본 프로젝트 범위에 포함되지 않습니다.
> 향후 별도 프로젝트로 구현이 계획되어 있습니다.

### 4.1 예상 기술 스택 (향후 구현용)

| 항목 | 선택 | 버전 |
|------|------|------|
| **주 언어** | Python / .NET | Python 3.9+ / .NET 8.0+ |
| **이미지 처리** | NumPy, SciPy | 1.21+, 1.7+ |
| **시각화** | Matplotlib | 3.4+ |
| **데이터 직렬화** | JSON, msgpack | - |
| **단위 테스트** | pytest / xUnit | 6.2+ |

### 4.2 예상 모듈 구조 (향후 구현용)

```
tft_leakage_package/
│
├── tft_leakage/
│   ├── core/                    # 핵심 알고리즘
│   │   ├── dark_lut.py          # 2D Dark LUT
│   │   ├── interpolation.py      # Bilinear interpolation
│   │   ├── blind_pixel.py       # Blind pixel correction
│   │   └── lag_correction.py    # Adaptive lag correction
│   │
│   ├── models/                  # 데이터 모델
│   │   ├── frame_metadata.py    # Frame metadata structure
│   │   ├── dark_lut_model.py    # LUT data structure
│   │   └── system_state.py      # System state
│   │
│   ├── hardware/                # 하드웨어 인터페이스
│   │   ├── fpga_interface.py    # FPGA communication
│   │   ├── mcu_interface.py     # MCU communication
│   │   └── temperature_sensor.py # Temperature monitoring
│   │
│   └── calibration/             # 보정 도구
│       ├── dark_lut_builder.py  # LUT generation
│       ├── lag_measurement.py   # Lag characterization
│       └── temp_chamber.py      # Chamber interface
```

---

## 5. 물리 모델 및 알고리즘 (Physical Models)

### 5.1 Arrhenius 모델 (Dark Drift)

```
k(T) = k_ref * exp((E_A / k_B) * (1/T - 1/T_ref))

Parameters:
- E_A (Activation Energy): 0.45 eV
- k_B (Boltzmann Constant): 8.617e-5 eV/K
- k_ref (Reference Rate): 1.0 DN/min @ 25°C
- T_ref (Reference Temp): 298.15 K (25°C)
```

### 5.2 VTH 드리프트 모델 (Stretched Exponential)

```
ΔV_TH(t) = ΔV_0 * [1 - exp(-(t/τ)^β)]

Parameters:
- ΔV_0: Saturation threshold shift
- τ: Time constant
- β: Stretched exponent (0 < β ≤ 1)
```

### 5.3 Lag 모델

```
L_1 = A * (E_1 - D_avg) + B
L_2 = C * L_1 + D

Coefficients (α_1, α_2) depend on:
- Temperature (T)
- Idle time history (Δt_idle)
- Previous frame exposure level
```

---

## 6. 개발 환경 설정 (Development Environment Setup)

### 6.1 FPGA 개발 환경

```bash
# Vivado 설치 후 경로 설정
export VIVADO_PATH=/path/to/Xilinx/Vivado/2025.2

# 프로젝트 생성
vivado -mode tcl -source create_project.tcl

# 시뮬레이션
cd sim
make sim

# 빌드
cd ../synth
make synth

# 비트스트림 생성
make bitstream
```

### 6.2 MCU 개발 환경 (i.MX8 Plus)

```bash
# Yocto 프로젝트 설정
source oe-init-build-env

# 이미지 빌드
bitbake core-image-minimal

# Cortex-M4 펌웨어 크로스 컴파일
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -o firmware.elf

# FPGA 통신 테스트 (Cortex-A53)
dotnet run --project FpgaCommunication
```

### 6.3 .NET 개발 환경

```bash
# .NET 프로젝트 생성
dotnet new console -n TftLeakageController

# 종속성 추가
dotnet add package System.IO.Ports
dotnet add package Microsoft.Extensions.Logging

# 빌드 및 실행
dotnet build
dotnet run
```

```bash
# 가상 환경 생성
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# 패키지 설치
pip install -r requirements.txt

# 개발 모드 설치
pip install -e .

# 테스트 실행
pytest tests/ --cov=tft_leakage --cov-report=html
```

---

## 7. 빌드 및 배포 (Build & Deployment)

### 7.1 FPGA 배포

| 산출물 | 설명 |
|--------|------|
| `bitstream.bit` | FPGA 바이어스트림 파일 |
| `debug.ltx` | 디버크 라이선스 파일 |
| `mcs.bin` | SPI 플래시 부팅 이미지 |

### 7.2 MCU 펌웨어 배포

| 산출물 | 설명 |
|--------|------|
| `firmware.hex` | Intel HEX 형식 |
| `firmware.bin` | 바이너리 형식 |
| `firmware.elf` | 디버깅용 ELF |

### 7.3 호스트 SW 배포 (향후 구현)

> 호스트 소프트웨어는 별도 프로젝트로 구현 예정

```bash
# .NET 배포 예시
dotnet publish -c Release -r linux-x64 --self-contained

# 또는 NuGet 배포
dotnet pack
dotnet nuget push package.nupkg
```

---

## 8. 테스트 전략 (Testing Strategy)

### 8.1 단위 테스트 (Unit Tests)

| 모듈 | 테스트 항목 | 커버리지 목표 |
|------|------------|--------------|
| Dark LUT | Interpolation 정확도 | > 90% |
| Blind Pixel | Offset 계산 | > 85% |
| Lag Correction | 계수 선택 로직 | > 85% |
| i.MX8 State Machine | 상태 전환 | > 95% |
| FPGA Timing | 타이밍 위배 검사 | 100% |
| .NET Services | 비즈니스 로직 | > 85% |

### 8.2 통합 테스트 (Integration Tests)

- FPGA-i.MX8 Plus 통신 검증
- Cortex-A53 ↔ Cortex-M4 통신 (MU) 검증
- 전체 파이프라인 데이터 흐름

### 8.3 시스템 테스트 (System Tests)

- 8시간 연속 운전
- 온도 스텝 응답 (±5°C)
- Idle 모드 전환 테스트
- Inter-core communication 안정성

---

## 9. CI/CD 파이프라인 (CI/CD Pipeline)

### 9.1 버전 관리

- **Git 브랜치 전략**: GitFlow
  - `main`: 안정 릴리스
  - `develop`: 개발 중
  - `feature/*`: 기능 개발
  - `hotfix/*`: 긴급 수정

### 9.2 CI 작업

```yaml
# .github/workflows/ci.yml (예시)

name: CI

on: [push, pull_request]

jobs:
  fpga-test:
    runs-on: [self-hosted, fpga]
    steps:
      - uses: actions/checkout@v2
      - name: Run simulation
        run: make sim

  imx8-test:
    runs-on: [self-hosted, imx8]
    steps:
      - uses: actions/checkout@v2
      - name: Build .NET application
        run: dotnet build --configuration Release
      - name: Run unit tests
        run: dotnet test --no-build --configuration Release
      - name: Build Cortex-M4 firmware
        run: make -f Makefile.imx8

  dotnet-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      - name: Restore dependencies
        run: dotnet restore
      - name: Build
        run: dotnet build --no-restore
      - name: Test
        run: dotnet test --no-build --verbosity normal
```

---

## 10. 보안 및 라이선스 (Security & Licensing)

### 10.1 라이선스

| 구성 요소 | 라이선스 |
|----------|---------|
| FPGA RTL | 프로프리어터리 (사내 전용) |
| i.MX8 Plus 펌웨어 (.NET) | 프로프리어터리 (사내 전용) |
| 호스트 SW (향후 구현) | MIT / Apache 2.0 |
| 문서 | CC BY-NC-SA 4.0 |

### 10.2 보안 고려사항

- 민감한 환자 데이터 처리 시 암호화 적용
- USB 통신 시 인증 절차
- 펌웨어 업데이트 시 서명 검증

---

## 11. 성능 최적화 (Performance Optimization)

### 11.1 FPGA 최적화

- 파이프라이닝으로 처리량 증가
- DSP 블록 활용 for 곱셈 연산
- 블록 RAM 활용 for 버퍼링

### 11.2 MCU 최적화

- FPU 활용 for 부동소수점 연산
- DMA 활용 for 데이터 전송
- 인터럽트 기반 이벤트 처리

### 11.3 .NET 최적화

- SIMD 활용 (System.Numerics.Vectors)
- Span<T> 활용 for 효율적인 메모리 관리
- async/await 패턴 for I/O 작업
- Memory Pooling for 빈번한 할당

---

## 12. 의존성 관리 (Dependency Management)

### 12.1 .NET 패키지 버전 고정

```xml
<!-- 핵심 의존성 버전 고정 -->
<PackageReference Include="System.IO.Ports" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Logging" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
```

### 12.2 MCU 라이브러리 (i.MX8 Plus)

- NXP HAL Driver (최신 버전)
- FreeRTOS Kernel (LTS)
- .NET IoT Libraries

---

## 13. .NET 크로스 플랫폼 개발 가이드 (.NET Cross-Platform Development)

### 13.1 .NET 지원 현황 (i.MX8 Plus / Yocto)

#### 지원되는 .NET 버전

| .NET 버전 | ARM64 Linux 지원 | LTS 상태 | 권장 대상 |
|-----------|------------------|----------|----------|
| .NET 8.0 | 완전 지원 | LTS (2026-11까지) | 권장 |
| .NET 9.0 | 완전 지원 | STS (2026-05까지) | 최신 기능 필요 시 |
| .NET 6.0 | 완전 지원 | 지원 종료 (2024-11) | 마이그레이션 권장 |

**참고**: .NET 8.0이 현재(2026년 기준) 가장 안정적인 LTS 버전으로, 프로덕션 환경에 권장됩니다.

#### Runtime Identifier (RID)

```bash
# i.MX8 Plus (Cortex-A53, ARM64 Linux)
RID: linux-arm64

# 빌드 시 RID 지정
dotnet publish -c Release -r linux-arm64 --self-contained true
```

#### Yocto Project에서의 .NET 지원

**meta-dotnet Layer**: OpenEmbedded 커뮤니티에서 제공하는 .NET 레이어

```bash
# bblayers.conf에 추가
BBLAYERS += "${TOPDIR}/layers/meta-dotnet"

# local.conf 또는 이미지 레시피에 추가
IMAGE_INSTALL:append = " dotnet"
```

**NXP i.MX8 Plus BSP와의 호환성**:
- NXP Yocto BSP (meta-imx)와 .NET 8.0 호환성 확인 완료
- Wayland/X11 기반 GUI 애플리케이션 지원
- Systemd 서비스로 등록 가능

---

### 13.2 Windows → ARM64 Linux 포팅 가이드

#### 13.2.1 프로젝트 파일 수정 (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <!-- 크로스 플랫폼 빌드 지원 -->
    <RuntimeIdentifiers>win-x64;linux-arm64</RuntimeIdentifiers>
    <!-- nullable 참조 타입 -->
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <!-- 플랫폼별 조건부 컴파일 -->
  <PropertyGroup Condition="'$(TargetPlatform)' == 'linux-arm64'">
    <DefineConstants>LINUX;ARM64</DefineConstants>
  </PropertyGroup>

  <PropertyGroup Condition="'$(TargetPlatform)' == 'win-x64'">
    <DefineConstants>WINDOWS</DefineConstants>
  </PropertyGroup>

</Project>
```

#### 13.2.2 경로 구분자 처리

```csharp
// 잘못된 방법 (Windows 전용)
string path = "C:\\Data\\config.json";

// 올바른 방법 (크로스 플랫폼)
using System.IO;

// 방법 1: Path.Combine 사용
string path = Path.Combine("data", "config.json");

// 방법 2: Path.DirectorySeparatorChar 사용
string path = $"data{Path.DirectorySeparatorChar}config.json";

// 방법 3: 경로 결합
string basePath = Path.GetFullPath("..");  // 상대 경로 해결
string configPath = Path.Combine(basePath, "data", "config.json");
```

#### 13.2.3 줄바꿈 문자 처리

```csharp
// 잘못된 방법
string text = "Line1\r\nLine2";  // Windows 전용

// 올바른 방법
string text = "Line1" + Environment.NewLine + "Line2";

// 또는
string text = $"Line1{Environment.NewLine}Line2";
```

#### 13.2.4 파일 시스템 권한 처리

```csharp
using System.IO;

// Linux에서는 파일 권한이 중요
public static void EnsureDirectoryExists(string path)
{
    if (!Directory.Exists(path))
    {
        Directory.CreateDirectory(path);

        // Linux만 해당 (디렉토리 생성 권한 755)
        if (OperatingSystem.IsLinux())
        {
            var dirInfo = new DirectoryInfo(path);
            dirInfo.UnixFileMode = UnixFileMode.UserRead |
                                   UnixFileMode.UserWrite |
                                   UnixFileMode.UserExecute |
                                   UnixFileMode.GroupRead |
                                   UnixFileMode.GroupExecute |
                                   UnixFileMode.OtherRead |
                                   UnixFileMode.OtherExecute;
        }
    }
}
```

#### 13.2.5 환경 변수 사용

```csharp
// 플랫폼별 기본 경로 설정
public static string GetDefaultDataPath()
{
    if (OperatingSystem.IsWindows())
    {
        return Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData),
            "TftLeakageController");
    }
    else if (OperatingSystem.IsLinux())
    {
        // Linux 표준: /var/lib 또는 /opt
        return "/var/lib/tft-leakage-controller";
    }

    throw new PlatformNotSupportedException();
}
```

---

### 13.3 조건부 컴파일 패턴

#### 13.3.1 전처리기 지시어 사용

```csharp
public class HardwareInterface
{
    public void InitializeGpio()
    {
#if LINUX
        // Linux (i.MX8 Plus) 전용 GPIO 제어
        // Sysfs 또는 libgpiod 사용
        InitializeGpioLinux();
#elif WINDOWS
        // Windows 개발용 시뮬레이션
        InitializeGpioSimulation();
#else
        throw new PlatformNotSupportedException();
#endif
    }

#if LINUX
    private void InitializeGpioLinux()
    {
        // 실제 하드웨어 제어 코드
        // 예: /sys/class/gpio/ 접근
    }
#elif WINDOWS
    private void InitializeGpioSimulation()
    {
        // 시뮬레이션용 모의 객체
    }
#endif
}
```

#### 13.3.2 런타임 플랫폼 확인

```csharp
using System.Runtime.InteropServices;

public class PlatformService
{
    public static bool IsRunningOnImx8()
    {
        if (!RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
            return false;

        // CPU 정보 확인 (ARM64)
        var cpuInfo = File.ReadAllText("/proc/cpuinfo");
        return cpuInfo.Contains("ARMv8") || cpuInfo.Contains("aarch64");
    }

    public static string GetPlatformInfo()
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
            return $"Windows {RuntimeInformation.OSArchitecture}";
        }
        else if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
        {
            var osRelease = File.ReadAllText("/etc/os-release");
            return $"Linux {RuntimeInformation.OSArchitecture}";
        }

        return "Unknown Platform";
    }
}
```

---

### 13.4 하드웨어 접근 라이브러리

#### 13.4.1 GPIO 제어 (System.Device.Gpio)

```csharp
// NuGet 패키지: System.Device.Gpio (1.5.0+)
using System.Device.Gpio;

public class GpioControllerService : IDisposable
{
    private readonly GpioController _controller;
    private readonly int _resetPin = 17;  // 예: GPIO17

    public GpioControllerService()
    {
        _controller = new GpioController();
        _controller.OpenPin(_resetPin, PinMode.Output);
    }

    public void TriggerReset()
    {
        _controller.Write(_resetPin, PinValue.High);
        Thread.Sleep(1);  // 1µs 펄스
        _controller.Write(_resetPin, PinValue.Low);
    }

    public void Dispose()
    {
        _controller?.Dispose();
    }
}
```

#### 13.4.2 SPI 통신

```csharp
// NuGet 패키지: System.Device.Spi
using System.Device.Spi;

public class SpiService : IDisposable
{
    private readonly SpiDevice _spiDevice;

    public SpiService()
    {
        var settings = new SpiConnectionSettings()
        {
            BusId = 0,        // SPI0
            ChipSelectLine = 0,
            Mode = SpiMode.Mode0,
            DataFlow = DataFlow.MsbFirst,
            ClockFrequency = 1000000  // 1 MHz
        };

        _spiDevice = SpiDevice.Create(settings);
    }

    public byte[] Transfer(byte[] data)
    {
        _spiDevice.Write(data);
        var readBuffer = new byte[data.Length];
        _spiDevice.Read(readBuffer);
        return readBuffer;
    }

    public void Dispose()
    {
        _spiDevice?.Dispose();
    }
}
```

#### 13.4.3 I2C 통신

```csharp
// NuGet 패키지: System.Device.I2c (or Iot.Device.Bindings)
using System.Device.I2c;

public class I2cTemperatureSensor : IDisposable
{
    private readonly I2cDevice _i2cDevice;
    private const byte DeviceAddress = 0x48;  // TMP100 예시

    public I2cTemperatureSensor(int busId = 1)
    {
        var settings = new I2cConnectionSettings(busId, DeviceAddress);
        _i2cDevice = I2cDevice.Create(settings);
    }

    public float ReadTemperature()
    {
        _i2cDevice.WriteByte(0x00);  // Temperature 레지스터
        byte[] data = new byte[2];
        _i2cDevice.Read(data);

        // 데이터 파싱 (센서별로 다름)
        short rawValue = (short)((data[0] << 8) | data[1]);
        return rawValue / 256.0f;
    }

    public void Dispose()
    {
        _i2cDevice?.Dispose();
    }
}
```

#### 13.4.4 시리얼 포트 (System.IO.Ports)

```csharp
// .NET 8.0에서 기본 포함
using System.IO.Ports;
using System.Threading;

public class FpgaSerialCommunication : IDisposable
{
    private readonly SerialPort _serialPort;

    public FpgaSerialCommunication(string portName = "/dev/ttyUSB0")
    {
        _serialPort = new SerialPort(portName)
        {
            BaudRate = 115200,
            Parity = Parity.None,
            DataBits = 8,
            StopBits = StopBits.One,
            Handshake = Handshake.None
        };

        _serialPort.Open();
    }

    public void SendCommand(byte[] command)
    {
        _serialPort.Write(command, 0, command.Length);
    }

    public byte[] ReceiveResponse(int expectedLength)
    {
        var buffer = new byte[expectedLength];
        int bytesRead = 0;

        while (bytesRead < expectedLength)
        {
            bytesRead += _serialPort.Read(buffer, bytesRead,
                                          expectedLength - bytesRead);
        }

        return buffer;
    }

    public void Dispose()
    {
        if (_serialPort.IsOpen)
            _serialPort.Close();
        _serialPort?.Dispose();
    }
}
```

---

### 13.5 네이티브 인터톱 (P/Invoke)

#### 13.5.1 C 라이브러리 호출

```csharp
using System.Runtime.InteropServices;

public class NativeHal
{
    // NXP HAL 라이브러리 (공유 객체)
    private const string HalLibrary = "libnxphal.so";

    // 함수 시그니처 정의
    [DllImport(HalLibrary, CallingConvention = CallingConvention.Cdecl)]
    public static extern int HAL_Initialize();

    [DllImport(HalLibrary, CallingConvention = CallingConvention.Cdecl)]
    public static extern void HAL_SetGpio(int pin, int value);

    [DllImport(HalLibrary, CallingConvention = CallingConvention.Cdecl)]
    public static extern int HAL_ReadAdc(int channel);

    public static int Initialize()
    {
        if (!RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
        {
            return -1;  // Windows에서는 시뮬레이션 모드
        }

        return HAL_Initialize();
    }
}
```

#### 13.5.2 Cortex-M4 통신 (Messaging Unit)

```csharp
// i.MX8 Plus에서 A53↔M4 통신을 위한 MU 드라이버 래퍼
public class InterCoreCommunication
{
    private const string MuDevicePath = "/dev/mu_device";  // 가상 경로

    // 메시지 구조체 (C 구조체와 일치해야 함)
    [StructLayout(LayoutKind.Sequential)]
    public struct MuMessage
    {
        public uint MessageId;
        public uint DataLength;
        public uint Data1;
        public uint Data2;
    }

    [DllImport("libmu.so", CallingConvention = CallingConvention.Cdecl)]
    private static extern int MU_SendMessage(ref MuMessage msg);

    [DllImport("libmu.so", CallingConvention = CallingConvention.Cdecl)]
    private static extern int MU_ReceiveMessage(ref MuMessage msg);

    public bool SendToFpgaControlCommand(uint command, uint parameter)
    {
        var msg = new MuMessage
        {
            MessageId = 0x01,  // FPGA 제어 명령 ID
            DataLength = 2,
            Data1 = command,
            Data2 = parameter
        };

        int result = MU_SendMessage(ref msg);
        return result == 0;
    }
}
```

---

### 13.6 빌드 및 배포 프로세스

#### 13.6.1 크로스 컴파일 (Windows 호스트)

```bash
# 1. ARM64 런타임 지정하여 빌드
dotnet publish -c Release \
    -r linux-arm64 \
    --self-contained true \
    -p:PublishTrimmed=true \
    -p:TrimMode=link \
    -p:PublishSingleFile=false

# 2. 출력물 위치: bin/Release/net8.0/linux-arm64/publish/

# 3. Yocto 레시피에서 참조할 수 있도록 아카이브 생성
cd bin/Release/net8.0/linux-arm64/publish/
tar czf tft-leakage-controller-arm64.tar.gz *
```

#### 13.6.2 Yocto 레시피 예시

```bitbake
# recipes-core/tft-leakage/tft-leakage-controller_1.0.bb

SUMMARY = "TFT Leakage Controller for i.MX8 Plus"
LICENSE = "CLOSED"
SRC_URI = "file://tft-leakage-controller-arm64.tar.gz"

S = "${WORKDIR}"

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${S}/TftLeakageController ${D}${bindir}/

    install -d ${D}${sysconfdir}/tft-leakage
    install -m 0644 ${S}/config.json ${D}${sysconfdir}/tft-leakage/

    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${S}/tft-leakage.service ${D}${systemd_system_unitdir}/
}

FILES_${PN} += "${bindir}/* ${sysconfdir}/* ${systemd_system_unitdir}/*"

SYSTEMD_SERVICE_${PN} = "tft-leakage.service"

RDEPENDS_${PN} = "dotnet (>= 8.0) bash"
```

#### 13.6.3 Systemd 서비스 설정

```ini
# tft-leakage.service
[Unit]
Description=TFT Leakage Controller
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/TftLeakageController
Restart=always
RestartSec=10

# 환경 변수
Environment="DOTNET_ENVIRONMENT=Production"
Environment="ASPNETCORE_URLS=http://0.0.0.0:5000"

# 보안 하드닝
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/tft-leakage-controller /var/log

[Install]
WantedBy=multi-user.target
```

---

### 13.7 원격 디버깅 설정

#### 13.7.1 SSH 터널링을 통한 디버깅

```bash
# i.MX8 장치에서 SSH 서버 활성화
systemctl enable sshd
systemctl start sshd

# Windows에서 VS Code Remote SSH 확장 설치 후
# ~/.ssh/config 설정:
Host imx8plus
    HostName 192.168.1.100
    User root
    IdentityFile ~/.ssh/id_rsa_imx8
```

#### 13.7.2 launch.json 설정 (VS Code)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Remote Attach (i.MX8)",
            "type": "coreclr",
            "request": "attach",
            "program": "/usr/bin/TftLeakageController",
            "processId": "${command:pickRemoteProcess}",
            "pipeTransport": {
                "pipeCwd": "${workspaceFolder}",
                "pipeProgram": "ssh",
                "pipeArgs": ["root@192.168.1.100"],
                "debuggerPath": "/vsdbg/vsdbg"
            }
        }
    ]
}
```

#### 13.7.3 CLI 기반 원격 디버깅

```bash
# i.MX8 장치에서 .NET 애플리케이션 디버그 모드 실행
dotnet run --project TftLeakageController.csproj -- --debug

# 또는
DOTNET_USE_POLLING_FILE_WATCHER=1 dotnet watch run

# Windows에서 SSH 터널링 후 dotnet-trace 사용
dotnet-trace collect --process-id 1234 --profile cpu-sampling --output trace.netperf
```

---

### 13.8 의존성 관리

#### 13.8.1 크로스 플랫폼 호환 NuGet 패키지

| 패키지 | 용도 | Linux ARM64 지원 |
|--------|------|------------------|
| `System.Device.Gpio` | GPIO 제어 | 지원 |
| `System.Device.Spi` | SPI 통신 | 지원 |
| `System.IO.Ports` | 시리얼 통신 | 지원 |
| `Microsoft.Extensions.Hosting` | 호스팅 | 지원 |
| `Microsoft.Extensions.Logging` | 로깅 | 지원 |
| `Serilog` | 구조화 로깅 | 지원 |
| `MQTTnet` | MQTT 클라이언트 | 지원 |
| `Newtonsoft.Json` | JSON 처리 | 지원 |

#### 13.8.2 플랫폼별 패키지 참여

```xml
<!-- Windows 전용 패키지 -->
<PackageReference Include="WindowsFirewallHelper" Version="3.0.0"
                  Condition="$([MSBuild]::IsOSPlatform('Windows'))" />

<!-- Linux 전용 패키지 -->
<PackageReference Include="LibgpiodCsharp" Version="1.0.0"
                  Condition="$([MSBuild]::IsOSPlatform('Linux'))" />
```

---

### 13.9 테스트 전략

#### 13.9.1 단위 테스트 (크로스 플랫폼)

```csharp
// xUnit을 사용한 플랫폼 독립적 테스트
using Xunit;
using Moq;

public class TemperatureMonitorTests
{
    [Fact]
    public void CalculateDriftRate_ValidTemperature_ReturnsCorrectRate()
    {
        // Arrange
        var calculator = new ArrheniusCalculator();
        double temperature = 298.15;  // 25°C in Kelvin

        // Act
        double rate = calculator.CalculateDriftRate(temperature);

        // Assert
        Assert.Equal(1.0, rate);  // k_ref = 1.0 DN/min @ 25°C
    }

    [Fact]
    public void CalculateDriftRate_HigherTemperature_ReturnsHigherRate()
    {
        var calculator = new ArrheniusCalculator();
        double temp25 = 298.15;
        double temp35 = 308.15;

        double rate25 = calculator.CalculateDriftRate(temp25);
        double rate35 = calculator.CalculateDriftRate(temp35);

        Assert.True(rate35 > rate25);
    }
}
```

#### 13.9.2 하드웨어 모의 (Mocking)

```csharp
// Moq를 사용한 하드웨어 모의
using Moq;

public class StateMachineTests
{
    [Fact]
    public void IdleTimerExceedsThreshold_TransitionsToL2()
    {
        // Arrange
        var gpioMock = new Mock<IGpioController>();
        var stateMachine = new IdleStateMachine(gpioMock.Object);

        // Act
        stateMachine.StartIdleTimer();
        stateMachine.SimulateTimePassage(TimeSpan.FromMinutes(15));

        // Assert
        Assert.Equal(IdleMode.L2, stateMachine.CurrentMode);
        gpioMock.Verify(x => x.WritePin(It.IsAny<int>(), PinValue.Low),
                       Times.Once);  // Low-bias 전환 확인
    }
}
```

---

### 13.10 성능 최적화 (ARM64)

#### 13.10.1 SIMD 활용

```csharp
using System.Numerics;

public class ImageProcessor
{
    // SIMD를 활용한 픽셀 처리 (ARM64 NEON)
    public static void SubtractDarkLevel(short[] pixelData, short darkLevel)
    {
        int vectorSize = Vector<short>.Count;
        int i = 0;

        // 벡터 처리
        Vector<short> darkVector = new Vector<short>(darkLevel);
        for (; i <= pixelData.Length - vectorSize; i += vectorSize)
        {
            Vector<short> data = new Vector<short>(pixelData, i);
            Vector<short> result = data - darkVector;
            result.CopyTo(pixelData, i);
        }

        // 나머지 처리
        for (; i < pixelData.Length; i++)
        {
            pixelData[i] -= darkLevel;
        }
    }
}
```

#### 13.10.2 메모리 관리

```csharp
using System.Buffers;

public class FrameBuffer
{
    // ArrayPool을 사용한 메모리 재사용
    public byte[] ProcessFrame(int frameSize)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(frameSize);

        try
        {
            // 처리 로직
            ProcessFrameInternal(buffer);

            var result = new byte[frameSize];
            Buffer.BlockCopy(buffer, 0, result, 0, frameSize);
            return result;
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}
```

---

### 13.11 CI/CD 파이프라인 (크로스 플랫폼)

```yaml
# .github/workflows/cross-platform-build.yml
name: Cross-Platform Build

on:
  push:
    branches: [ main, develop ]
  pull_request:

jobs:
  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - name: Build
        run: dotnet build -c Release -r win-x64
      - name: Test
        run: dotnet test --no-build

  build-linux-arm64:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - name: Cross-compile for ARM64
        run: |
          dotnet publish -c Release \
            -r linux-arm64 \
            --self-contained true \
            -o publish/linux-arm64
      - name: Archive artifacts
        uses: actions/upload-artifact@v4
        with:
          name: linux-arm64-binary
          path: publish/linux-arm64/
```

---

### 13.12 문제 해결 가이드

#### 13.12.1 일반적인 포팅 이슈

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| `DllNotFoundException` | 네이티브 라이브러리 누락 | Yocto 레시피에 `RDEPENDS` 추가 |
| `Permission Denied` | GPIO/I2C 장치 접근 권한 | 사용자를 `gpio`, `i2c` 그룹에 추가 |
| 경로 관련 오류 | 하드코딩된 Windows 경로 | `Path.Combine`, `Path.GetFullPath` 사용 |
| 인코딩 문제 | 다른 기본 문자 인코딩 | UTF-8 명시적 지정 |
| 성능 저하 | ARM64 최적화 부족 | SIMD, Span<T> 활용 |

#### 13.12.2 장치 접근 권한 설정

```bash
# i.MX8 Plus에서 udev 규칙 추가
# /etc/udev/rules.d/99-tft-leakage.rules

# GPIO 장치
KERNEL=="gpiochip*", MODE="0660", GROUP="gpio"

# I2C 장치
KERNEL=="i2c-[0-9]*", MODE="0660", GROUP="i2c"

# SPI 장치
KERNEL=="spidev*", MODE="0660", GROUP="spi"

# 시리얼 포트
KERNEL=="ttyUSB*", MODE="0660", GROUP="dialout"

# 사용자 그룹 추가
usermod -aG gpio,i2c,spi,dialout root
```

---

**문서 버전**: 3.2
**최종 수정일**: 2026-02-10
**상태**: .NET 크로스 플랫폼 섹션 추가 완료
**변경 내역**:
- Vivado 버전: 2022.1+ → 2025.2
- MCU 플랫폼: ARM Cortex-M4 → NXP i.MX8 Plus (Cortex-A53 + Cortex-M4)
- 개발 환경: STM32CubeIDE/Keil → Yocto Project + .NET 8.0+
- 호스트 SW: Python/PyQt5 → 별도 프로젝트로 이동 (범위에서 제외)
- **[NEW]** 섹션 13 추가: Windows → i.MX8 Plus (ARM64 Linux) .NET 포팅 가이드
  - .NET 8.0 ARM64 지원 현황
  - Yocto meta-dotnet 레이어 통합
  - 크로스 플랫폼 코드 패턴
  - 하드웨어 접근 라이브러리 (GPIO, SPI, I2C, UART)
  - P/Invoke를 통한 네이티브 인터톱
  - 빌드 및 배포 프로세스
  - 원격 디버깅 설정
  - ARM64 성능 최적화
  - CI/CD 파이프라인 예시
