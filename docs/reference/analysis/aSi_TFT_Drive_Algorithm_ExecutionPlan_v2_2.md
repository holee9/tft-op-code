# a-Si TFT FPD 구동 알고리즘 실행 계획서 (최종 v2.2)
## Idle 상태 관리 및 Dark/Lag 보정 통합 전략

**작성일**: 2026년 2월 9일  
**버전**: 2.2 (최종 - Idle 구동 모드 완성)  
**대상 시스템**: a-Si TFT 기반 간접변환 X-ray Flat Panel Detector  
**주요 추가사항**: Idle mode L1/L2/L3, t_max 계산 로직, warm-up 프레임 처리

---

## 1. 목표 및 범위

### 1.1 주요 목표

1. **방법론 ①** (250 ms, 연구용):
   - Dark correction efficiency: 30% → 목표 유지
   - Lag correction efficiency: 50% → 목표 유지
   - Idle 환경에서도 성능 안정화 (≥ 90% 유지)

2. **방법론 ②** (5 fps, 임상용):
   - 프레임 레이트 안정: 5 fps ± 5%
   - Dark correction efficiency: 15% → 개선 목표 20%
   - Idle 이후 첫 영상 품질 저하 < 5%

### 1.2 구동 알고리즘 스코프

- **FPGA**: Row/Column 타이밍, RESET/DARK/READ 시퀀스
- **MCU**: 온도 모니터링, Idle timer, t_max 계산, LUT 관리
- **SW (Host)**: Dark LUT 계산, blind pixel 보정, 이미지 후처리

---

## 2. 제약 조건 및 가정

### 2.1 하드웨어 제약

- Panel resolution: 2048 × 2048 (기준)
- Pixel pitch: 100~200 µm
- Line rate: 1~10 MHz (FPGA 가능 범위)
- Storage capacitance: 100~500 pF/픽셀
- TFT off-leakage: 1~10 pA (25°C, -2V 바이어스)

### 2.2 Idle 모드 정의

**Level 1 (L1): Normal Idle**
- 조건: 짧은 대기(< 10분)
- Bias: Normal read bias 유지
- Scan: 없음 (readout도 없음)
- t_max @ 25°C: ~50분
- Power consumption: 고정
- 용도: 빠른 재개 필요 시

**Level 2 (L2): Low-bias Idle**
- 조건: 중간 대기(10~100분)
- Bias: 저바이어스 모드 (V_PD ≈ -0.2V, V_COL_idle)
- Scan: 주기적 dummy reset/read (T_dummy = 30~60초)
- t_max @ 25°C: ~200분 (약 4배 연장)
- Power consumption: 기본 대비 80~90% (여전히 고)
- 용도: 일반 대기(예: 병원 당직 상황)

**Level 3 (L3): Deep Sleep**
- 조건: 장시간 보관(> 1시간)
- Bias: Minimal (gate/ROIC 부분 power down)
- Scan: 없음
- t_max @ 25°C: 거의 무한
- Power consumption: 기본 대비 < 10% (대기 전류만)
- 용도: 야간 보관, 연휴

### 2.3 온도 범위 및 코효 계수

- 동작 온도: 10~40°C (보증 범위)
- Ideal operating: 20~35°C (±5°C 추천)
- Dark drift 온도 계수: ≈ 4 DN/°C
- drift rate 온도 계수: ≈ 2× per 10°C (지수)
- V_TH drift 온도 계수: ≈ 20 mV/°C

### 2.4 허용 오차

- Dark level 최대 증가 허용: ΔD_max = 50 DN (전체 16384 DN 대비 0.3%)
- First frame residual dark: < 100 DN (보정 후)
- Lag residual: < 2% (보정 후)

---

## 3. 시스템 아키텍처

### 3.1 세 계층 구성

```
┌─────────────────────────────────────────────┐
│  Host (PC / Embedded Linux)                 │
│  - Image processing                         │
│  - Dark/Lag LUT 관리 및 업데이트             │
│  - Blind pixel 보정                          │
│  - 사용자 인터페이스                         │
└────────────┬────────────────────────────────┘
             │ (USB/Ethernet, 제어 + 이미지 데이터)
┌────────────▼────────────────────────────────┐
│  MCU / Microcontroller                      │
│  - Idle timer / t_max(T) 계산               │
│  - 온도 센서 읽기 (1초 주기)                  │
│  - Idle mode L1/L2/L3 상태 전환             │
│  - Frame 메타데이터 생성 (T, Δt_idle 등)    │
│  - Warm-up 프레임 플래그 설정                │
└────────────┬────────────────────────────────┘
             │ (Control signals, temperature)
┌────────────▼────────────────────────────────┐
│  FPGA / Timing Controller                   │
│  - Row/Column clock & timing                │
│  - RESET / DARK / READ 시퀀스 생성          │
│  - Bias voltage 제어 (MUX 선택)             │
│  - Analog-to-digital conversion             │
└─────────────────────────────────────────────┘
                 │
         ┌───────▼────────┐
         │  FPD Panel     │
         │  (2048×2048)   │
         └────────────────┘
```

### 3.2 신호 경로

- **Control**: MCU → FPGA (idle mode, bias select)
- **Timing**: FPGA → Panel (row/col clock, gate signals)
- **Readback**: Panel → FPGA → ADC → Host
- **Temp**: Temperature sensor → MCU (1 kHz polling)

---

## 4. 상세 구동 알고리즘 (FPGA + MCU)

### 4.1 Idle Mode 상태 전환도

```
┌─────────────┐
│   ACTIVE    │  (normal readout ongoing)
│  (exposure) │
└──────┬──────┘
       │ (readout 완료)
       ▼
┌─────────────┐
│    L1       │  (normal idle)
│  Idle < 10' │
└──────┬──────┘
       │ (t_idle > 10 min)
       ▼
┌─────────────┐
│    L2       │  (low-bias + scan)
│ 10' ≤ Idle │
│  < 100 min  │
└──────┬──────┘
       │ (t_idle > 100 min)
       ▼
┌─────────────┐
│    L3       │  (deep sleep)
│  Idle ≥ 100'│  partial power-down
└─────────────┘
```

### 4.2 각 Idle Level에서의 구동

#### L1: Normal Idle (t_idle < 10 min)

```c
// FPGA 설정
bias_config = NORMAL_READ_BIAS;  // V_PD = -1.5 V, V_COL = -1.0 V
gate_control = ALL_OFF;           // 모든 row gate OFF
dummy_scan_enable = 0;            // scan 없음

// MCU 로직
while (idle_time < 10 * 60) {    // 10분 동안
    if (t_now > (t_last_reset + t_max_L1)) {
        // 미리 보호하기 위해 L2로 진입
        transition_to_L2();
        break;
    }
    sleep(1000);  // 1초 대기
}
```

#### L2: Low-Bias Idle (10 min ≤ t_idle < 100 min)

```c
// FPGA 설정
bias_config = IDLE_LOW_BIAS;      // V_PD = -0.2 V, V_COL_idle = -0.2 V
gate_control = ALL_OFF;
dummy_scan_enable = 1;            // 주기적 scan 활성화
dummy_scan_period = 60;           // 60초마다

// Dummy scan 수행 (FPGA 자동)
for (int i = 0; i < num_rows; i++) {
    trigger_row_reset(i);         // Reset 펄스
    wait(100 us);                 // settle
    // READ는 하지 않음 (storage charge 비우기만)
}

// MCU 로직
last_reset_time = t_now;
while (idle_time < 100 * 60) {   // 100분 동안
    if (t_now > (t_last_reset + t_max_L2)) {
        // L2 최대 시간 초과, L3로 진입
        transition_to_L3();
        break;
    }
    if ((t_now - last_dummy_scan) >= 60) {
        // Dummy scan 수행 신호 전송 (FPGA 실행)
        set_dummy_scan_trigger();
        last_dummy_scan = t_now;
    }
    sleep(1000);
}
```

#### L3: Deep Sleep (t_idle ≥ 100 min)

```c
// FPGA 설정
bias_config = SLEEP_BIAS;         // Minimal bias (거의 0V)
gate_control = DISABLED;          // Gate driver power down
dummy_scan_enable = 0;
adc_enable = 0;

// MCU 로직
enter_low_power_mode();           // MCU도 sleep
// Temperature sensor만 주기적으로 깨어남 (RTC interrupt)
// Idle timer 유지

// 재기동 시:
// - Full initialization sequence 필요
// - Warm-up frame 필수
```

### 4.3 Temperature 기반 t_max 계산 (MCU)

MCU는 1초마다 온도를 읽고, drift rate k(T)를 계산:

```c
typedef struct {
    float temp;           // 현재 온도 (°C)
    float k_typical;      // k(T) 추정값 (DN/min)
    uint32_t t_max_L1;    // 최대 idle 시간 (sec)
    uint32_t t_max_L2;
} ThermalModel;

float estimate_drift_rate(float T) {
    // E_A = 0.45 eV, k_ref = 1.0 DN/min @ 25°C
    float E_A = 0.45;  // eV
    float k_B = 8.617e-5;  // eV/K
    float T_ref = 298.15;  // 25°C in K
    float T_K = T + 273.15;
    
    float k_ref = 1.0;  // @ 25°C
    float k_T = k_ref * exp((E_A / k_B) * (1.0/T_K - 1.0/T_ref));
    
    return k_T;
}

uint32_t calculate_t_max(float T, uint32_t delta_D_max) {
    // delta_D_max = 50 DN (허용 한계)
    float k_T = estimate_drift_rate(T);
    uint32_t t_max_min = (uint32_t)(delta_D_max / k_T);  // minutes
    return t_max_min * 60;  // convert to seconds
}

// Main loop
while (1) {
    float T_current = read_temperature_sensor();
    
    float k_T = estimate_drift_rate(T_current);
    
    uint32_t t_max_current = calculate_t_max(T_current, 50);  // 50 DN limit
    
    idle_allowed_time = t_max_current;  // 이 시간 넘으면 L2 진입
    
    sleep(1000);  // 1초 주기
}
```

### 4.4 Warm-up Frame 처리 (MCU + Host)

Idle 이후 재개 시:

```c
// MCU 로직
if (t_idle > 30 * 60) {  // 30분 이상 idle
    warmup_frame_required = 1;
} else {
    warmup_frame_required = 0;
}

// Host 수신 (프레임 헤더에 메타데이터)
struct FrameMetadata {
    uint32_t timestamp;
    float temperature;
    uint32_t idle_time_sec;
    uint8_t warmup_required;  // 1 if first frame after long idle
    uint8_t idle_mode;         // L1, L2, L3
};

// Host-side 로직
if (metadata.warmup_required) {
    // Frame 1: DARK 또는 reference로만 사용
    capture_frame_1();
    process_as_dark_reference();
    
    // Frame 2: 첫 clinical frame
    capture_frame_2();
    apply_dark_correction_LUT();
    display_as_clinical();
} else {
    // 일반 frame
    capture_frame();
    apply_dark_correction_LUT();
    display();
}
```

---

## 5. 보정 알고리즘 상세

### 5.1 Dark Correction LUT (Host에서 관리)

**2D LUT 구조**:

```c
typedef struct {
    float temp_celsius;
    uint32_t idle_time_sec;
    uint16_t dark_dn;      // estimated dark level [DN]
} DarkLUT_Entry;

// 최대 6 × 6 = 36개 entry (온도 6단계 × idle_time 6단계)
DarkLUT_Entry dark_lut[6][6];

// 온도: 15, 20, 25, 30, 35, 40°C
// idle_time: 0, 10, 30, 60, 120, 240 min

// Runtime에서:
void apply_dark_correction(
    uint16_t* image_raw,      // 입력 raw 이미지
    uint16_t* image_corrected, // 출력
    float temp,               // 현재 온도
    uint32_t idle_time        // idle 시간
) {
    // 2D Interpolation으로 dark 추정
    float dark_estimate = interpolate_2d(dark_lut, temp, idle_time);
    
    for (int i = 0; i < 2048*2048; i++) {
        int16_t corrected = image_raw[i] - (uint16_t)dark_estimate;
        image_corrected[i] = (corrected < 0) ? 0 : corrected;
    }
}
```

**LUT 생성 절차** (Calibration 단계):

1. 각 온도 T ∈ {15, 20, 25, 30, 35, 40}°C에서:
2. Idle time t ∈ {0, 10, 30, 60, 120, 240} 분에 대해:
3. Dark frame 촬영 → 평균값 기록 → LUT 저장
4. 선형/2차 interpolation으로 중간값 추정

### 5.2 Blind Pixel 기반 Frame-level 보정

**Optical Black Pixel 활용** (패널 주변 차폐 픽셀):

```c
void frame_level_dark_correction(
    uint16_t* frame_raw,
    uint16_t* frame_corrected,
    uint16_t* blind_pixel_rows,   // 상/하 가장자리 row
    int num_blind_rows
) {
    // blind pixel 평균값 계산
    uint32_t blind_sum = 0;
    for (int i = 0; i < num_blind_rows * 2048; i++) {
        blind_sum += blind_pixel_rows[i];
    }
    float blind_mean = blind_sum / (num_blind_rows * 2048);
    
    // Reference dark level (정상 상태)
    float reference_dark = 2500.0;  // 평소 값
    
    // Frame-level offset
    float offset_this_frame = blind_mean - reference_dark;
    
    // Correction 적용
    for (int i = 0; i < 2048*2048; i++) {
        int16_t corrected = frame_raw[i] - offset_this_frame;
        frame_corrected[i] = (corrected < 0) ? 0 : corrected;
    }
}
```

### 5.3 Lag Correction (온도/idle 적응)

**Lag model** (이미 작성된 내용 유지):

\[
E_1^{\text{corr}} = E_1 - \alpha_1(T, \Delta t_{\text{idle}}) \times (L_1 - D_{\text{avg}})
\]

**온도/idle 기반 α₁ 선택**:

```c
float get_lag_coefficient(float temp, uint32_t idle_time) {
    // Calibration에서 온도/idle별로 측정된 α₁ 값
    float alpha_1_lut[6][6];  // 위 dark_lut과 같은 구조
    
    // Interpolation으로 현재 조건에 맞는 α₁ 추출
    float alpha_1 = interpolate_2d(alpha_1_lut, temp, idle_time);
    
    return alpha_1;
}
```

---

## 6. FPGA 타이밍 및 시퀀스

### 6.1 Normal Read Sequence (방법론 ①, 250 ms)

```
Cycle: DARK₁ → DARK₂ → LAG₁ → LAG₂ → CLINICAL

DARK₁:
  RESET_ALL (10 µs)
  WAIT (100 ms)
  READ_ALL (line-by-line, 250 ms) → stored

DARK₂:
  RESET_ALL
  WAIT (100 ms)
  READ_ALL → stored (averaging with DARK₁)

LAG₁:
  RESET_ALL
  WAIT (100 ms)
  EXPOSE (short exposure, X-ray ON, 10 ms)
  READ_ALL → stored

LAG₂:
  RESET_ALL
  WAIT (100 ms)
  EXPOSE (no X-ray, dark) → stored

CLINICAL:
  RESET_ALL
  WAIT (exposure time, ~1000 ms)
  EXPOSE (X-ray ON) → signal acquisition
  READ_ALL
  → Apply corrections (dark, lag) → Output

Total cycle time: ~250 ms × 5 frames = 1.25 sec per cycle
```

### 6.2 5 fps Mode Sequence (방법론 ②)

```
Cycle (200 ms): DARK (async) + EXPOSE + READ

Main loop (5 fps = 200 ms):
  RESET_ALL (10 µs)
  WAIT (integration time, ~150 ms)
  EXPOSE (X-ray ON if needed)
  READ_ALL (line-by-line, ~50 ms) → frame output
  → Apply LUT correction + blind pixel + lag correction
  → Output

Asynchronous DARK capture (every 10 frames, ~2 seconds):
  RESET_ALL
  WAIT (150 ms)
  EXPOSE (no X-ray)
  READ_ALL → DARK_async
  → Update LUT or use for blind pixel offset
```

### 6.3 Idle Mode에서의 Dummy Scan (L2)

```
T_dummy = 60 sec interval:

DUMMY_SCAN (no output):
  for row i in 0..2047:
    SET gate_i = ON
    WAIT (1 µs)
    TRIGGER_RESET for row i (pulse: 1 µs)
    WAIT (100 µs, settle)
    SET gate_i = OFF
  
  → Total dummy scan time: ~2 ms (한 번에 모든 row)

이 스캔은 storage node의 charge를 비우는 역할만 함.
ADC readout 없음 (프레임 생성 안 됨).
```

---

## 7. 성능 지표 (KPI)

### 7.1 방법론 ①(250 ms) 예상 성능

| 지표 | 목표 | 달성 기준 |
|------|------|---------|
| Dark correction efficiency | 30% | residual dark < 750 DN |
| Lag correction efficiency | 50% | residual lag < 1% |
| Read time per frame | 250 ms | ±10 ms |
| Idle 환경 성능 저하 | < 10% | dark correction > 27% |
| Idle 허용 시간 (25°C) | > 50 min | no performance loss |

### 7.2 방법론 ②(5 fps) 예상 성능

| 지표 | 목표 | 달성 기준 |
|------|------|---------|
| Frame rate | 5 fps | ±5% |
| Read time | 50 ms | ±5 ms |
| Dark correction efficiency | > 20% | residual dark < 1000 DN |
| Lag correction efficiency | > 30% | residual lag < 2% |
| Idle 후 첫 프레임 dark | < 100 DN (보정 후) | residual < 5% |
| Warm-up 오버헤드 | < 500 ms | 2 frames @ 200 ms |

### 7.3 Idle 관련 KPI

| 지표 | 목표 | 측정 방법 |
|------|------|---------|
| L1 허용 idle 시간 | > 10 min @ 25°C | No performance loss |
| L2 허용 idle 시간 | > 100 min @ 25°C | Dark correction > 90% of normal |
| Dummy scan 주기 | 30~60 sec (L2) | No accumulated dark > 50 DN |
| Temperature stability | ±3°C | Thermal control requirement |
| Warm-up frame overhead | < 0.5 sec | First good frame within 200 ms |

---

## 8. 테스트 계획

### 8.1 Unit Test

1. **Drift rate 모델 검증**:
   - 온도별 dark drift vs 모델 오차 < 10%
   
2. **t_max 계산 정확도**:
   - 실측 idle 시간과 예측 t_max 비교
   - Margin: 20% (안전 팩터)

3. **LUT Interpolation**:
   - 온도/idle time 중간값에서 보정 오차 < 5%

### 8.2 Integration Test

1. **L1/L2/L3 전환**:
   - 각 모드로 전환 시 bias 제어 확인
   - Dummy scan 주기 정확도 ±10%

2. **Warm-up 시퀀스**:
   - Idle 30분/1시간/2시간 후 warm-up 첫 프레임 품질 확인
   - Residual dark < 100 DN 검증

3. **Blind pixel 보정**:
   - Blind pixel 존재 시 frame-level 오프셋 정확도
   - 신호 손실 없음 확인

### 8.3 System Test

1. **연속 운전** (8시간+):
   - 온도 변화(±5°C) 환경에서 dark drift 추적
   - KPI 유지 확인

2. **온도 스텝 응답**:
   - 5°C jump → t_max 변화 추적
   - MCU 반응 시간 < 10 sec

3. **임상 모드 (5 fps)**:
   - 연속 200 프레임 촬영
   - Frame rate 5.0 ± 0.25 fps 유지
   - Lag/dark residual 안정성

---

## 9. 배포 및 문서화

### 9.1 펌웨어 배포 (FPGA + MCU)

**FPGA image**:
- Idle mode bias select logic
- Dummy scan sequence generator
- Temperature-compensated timing

**MCU firmware**:
- Temperature polling (1 Hz)
- t_max calculation
- Idle state machine (L1/L2/L3)
- Frame metadata generation

### 9.2 호스트 SW (Python/C++)

- Dark LUT 계산 및 저장
- 2D interpolation 함수
- Frame-level blind pixel 보정
- Lag correction 적응 알고리즘

### 9.3 사용자 문서

- Idle mode 사용 설명서
- 온도 관리 권장사항
- Warm-up 절차
- 트러블슈팅 가이드

---

## 10. 일정 및 마일스톤

| 항목 | 기간 | 산출물 |
|------|------|--------|
| FPGA 구현 (Idle mode + dummy scan) | 2주 | RTL code, simulation |
| MCU 펌웨어 (temp model + state machine) | 1.5주 | Compiled firmware |
| 호스트 SW (LUT + correction) | 2주 | Python/C++ library |
| 통합 테스트 | 2주 | Test report |
| 배포 및 문서화 | 1주 | User manual + SW release |
| **Total** | **~8.5주** | |

---

## 최종 체크리스트

- [ ] FPGA: Idle bias 선택 MUX 회로 검증
- [ ] MCU: 온도 센서 calibration 완료
- [ ] MCU: t_max(T) drift rate 테이블 작성
- [ ] Host: Dark LUT 2D interpolation 함수 구현 및 단위 테스트
- [ ] Host: Blind pixel row/col 위치 확인 (벤더로부터)
- [ ] Integration: 모든 모드 전환 성공
- [ ] System: 8시간 연속 운전 테스트 완료
- [ ] Documentation: 사용자 매뉴얼 완성
- [ ] Release: 최종 SW 버전 태깅 및 배포

---

**작성 및 검토**: 2026-02-09  
**최종 버전**: 2.2 (Idle 모드 완성)  
**상태**: 구현 준비 완료

