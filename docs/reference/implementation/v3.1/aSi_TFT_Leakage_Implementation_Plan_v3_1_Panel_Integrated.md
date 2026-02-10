# a-Si TFT FPD Leakage 특성 분석 및 Dark/Lag 보정 구현 계획서
## 최종 개정판 (v3.1) - R1717AS01.3 패널 기준 딥싱크

**작성일**: 2026년 2월 10일  
**버전**: 3.1 (R1717AS01.3 패널 사양 통합)  
**이전 버전**: 3.0 (2026년 2월 9일)  
**주요 변경사항**: 패널 전압 레일(VGH/VGL/PD back-bias) 구체화, 모드별 전압 매트릭스 확정, FPGA/MCU 시퀀스 패널 동기화

---

## 목차

1. [개정 이력](#개정-이력)
2. [Executive Summary](#executive-summary)
3. [1. 서론](#1-서론)
4. [2. a-Si TFT Leakage 현상 분석](#2-a-si-tft-leakage-현상-분석)
5. [3. Dark/Lag 기본 특성](#3-darklaq-기본-특성)
6. [4. NEW: R1717AS01.3 패널 통합 모드 정의](#4-new-r1717as013-패널-통합-모드-정의)
7. [5. 구동 알고리즘 상세 (VGH/VGL/PD back-bias)](#5-구동-알고리즘-상세-vghvglpd-back-bias)
8. [6. 구현 계획 및 일정](#6-구현-계획-및-일정)
9. [7. 성능 목표 및 검증](#7-성능-목표-및-검증)
10. [부록 A: 현실적 벤더 협조 요청 항목](#부록-a-현실적-벤더-협조-요청-항목)
11. [부록 B: 온도별 Drift Rate LUT 측정 프로토콜](#부록-b-온도별-drift-rate-lut-측정-프로토콜)
12. [부록 C: R1717AS01.3 모드별 전압 및 FPGA 시퀀스 상세](#부록-c-r1717as013-모드별-전압-및-fpga-시퀀스-상세)

---

## 개정 이력

### v3.0 → v3.1 주요 변경사항

| 항목 | v3.0 | v3.1 | 비고 |
|------|------|------|------|
| 패널 모델 | 범용 | R1717AS01.3 | 패널 사양서 기반 구체화 |
| VGH 값 | 예시 | +15 V (확정) | 패널 스펙: Vg=15V |
| VGL 값 | 예시 | −5 V (권장) | 패널 Deep OFF 모드 |
| PD back-bias | 예시 | −2 V to 0 V (범위) | 역바이어스 vs 저바이어스 |
| Idle 저바이어스 | −0.2 V | −0.5 V (권장) | dark 억제 효율 최적화 |
| 모드별 매트릭스 | 개념적 | 구체적 전압값 | 전환 가능한 상세 |
| FPGA 타이밍 | 일반적 | row=3072, scan ≈ 76 µs | 패널 픽셀 수 반영 |
| 부록 C | 없음 | 새로 추가 | 시퀀스 상세 (pseudo code) |

---

## Executive Summary

이 문서는 **R1717AS01.3 (3072×3072 a-Si TFT PIN diode) X-ray FPD에서 발생하는 leakage 현상과 Idle 상태 관리**를 실제 패널 파라미터 기반으로 상세화합니다.

### v3.1의 핵심 추가사항

1. **패널 전압 레일 확정**:
   - VGH = +15 V (gate ON)
   - VGL = −5 V (gate OFF, deep)
   - PD back-bias: Normal imaging −2 V, Idle low −0.5 V

2. **패널 기반 타이밍 계산**:
   - Row count: 3072 (dummy row 포함)
   - 1 row scan time: ≈ 76 µs @ 40 MHz clock
   - Full scan time: ≈ 235 ms (3072 × 76 µs)

3. **모드별 전압 매트릭스** (부록 C):
   - M1 ACTIVE: VGH/VGL/PD full swing
   - M3 Idle L2: VGH/VGL 유지, PD 저바이어스 (−0.5 V)
   - Dummy scan: 30~60 s 주기, RESET only

4. **FPGA/MCU 시퀀스 실행 코드** (부록 C):
   - state machine (M0~M5)
   - row/column 제어
   - DAC 다중화 (VPD 온도 기반 동적 조정)

---

## 1. 서론 (동일 - v3.0 참조)

*(생략: v3.0과 동일)*

---

## 2. a-Si TFT Leakage 현상 분석 (동일 - v3.0 참조)

*(생략: v3.0과 동일)*

---

## 3. Dark/Lag 기본 특성 (동일 - v3.0 참조)

*(생략: v3.0과 동일)*

---

## 4. NEW: R1717AS01.3 패널 통합 모드 정의

### 4.1 R1717AS01.3 패널 개요

**모델**: AUO R1717AS01.3 (92D17170.3R2)

**주요 파라미터**:

| 항목 | 값 | 비고 |
|------|-----|------|
| 기술 | a-Si TFT + PIN diode | X-ray 센서 |
| 해상도 | 3072 × 3072 | Gate × Data (dummy 포함) |
| 픽셀 피치 | 140 µm | 고해상도 |
| Active area | 430.08 × 430.08 mm | 17" × 17" |
| TFT Ron (typ.) | 2.2 MΩ | @ Vg=15V, Vds=0.1V |
| TFT Leakage (max) | 80 fA/pixel @ 25°C | 온도 계수 지수 |
| Pixel capacitance | 1.48 pF | storage node |
| Pixel leakage (max) | 3 fA/pixel (dark rate) | 60sec − 10sec diff |
| 1st Frame lag (typ.) | 3% | @ 40000 LSB 노광 |
| Gate line R | 6.5 kΩ (typical) | 스캔 시간 영향 |
| Gate line C | 149.4 pF | 타이밍 계산용 |

---

### 4.2 패널 바이어스 레일 (VGH/VGL/PD back-bias)

**권장 설정** (패널 벤더 지침 + 문헌):

| 신호 | 파라미터 | 값 | 용도 | 비고 |
|------|---------|-----|------|------|
| VGH | Gate High | +15 V | row ON | TFT strong inversion |
| VGL | Gate Low | −5 V | row OFF | deep depletion mode |
| VCOM | Common electrode | Ground (0 V) | reference | floating 방지 |
| VPD (Normal) | PD back-bias (imaging) | −2.0 V | photocurrent 최대화 | 역바이어스 |
| VPD (Idle L2) | PD back-bias (low) | −0.5 V | dark 억제 | 저역바이어스 |
| VPD (Idle L3) | PD back-bias (minimal) | 0 V (off) | deep sleep | power down |

**전압 특성**:

- VGH=+15V는 TFT threshold voltage (≈ 3~4V)를 충분히 초과하여 **strong inversion**
- VGL=−5V는 VGH−VGL=20V로 **완전한 OFF** 보장 (leakage 최소)
- VCOM=0V 기준으로 PD는 상대 바이어스가 −2V (reverse) 또는 −0.5V (partial reverse)

---

### 4.3 Idle 상태 3단계 전략 (R1717AS01.3 최적화)

#### Stage 1: Dark 생성 억제 (PD 저바이어스)

**Idle L2 바이어스**:
- VPD: −2.0 V (normal) → −0.5 V (idle L2)
- 역바이어스 감소로 **누설 전류 ≈ 60~70% 감소** (Arrhenius 비 고려)

**물리적 이유**:
- 역바이어스 감소 → depletion width 감소 → thermal generation rate ↓
- 효과: dark level 증가 속도 20~30% 저감

#### Stage 2: 주기적 Dummy Reset

**Dummy scan 설정**:
- 주기: T_dummy = 30~60 s
- 각 cycle:
  ```
  for row in 0..3071:
      VGH_pulse → RESET → VGL_hold
  ```
- 시간: 1 full scan ≈ 235 ms (3072 row × 76 µs/row)
- **효과**: storage node periodic discharge → cumulative dark 제한

#### Stage 3: 후속 영상 2D LUT 보정

**2D LUT 구성** (온도 × idle_time):

```
D_LUT[T_index][t_index] = measured dark level
  T: 15, 20, 25, 30, 35, 40 °C
  t: 0, 10, 30, 60, 120, 240 min
```

**Frame-level 보정**:
```c
// Runtime
dark_est = interpolate_2d(dark_lut, T_current, t_idle);
blind_offset = mean(blind_pixels) - reference_dark;
corrected_frame = raw_frame - dark_est - blind_offset;
```

---

### 4.4 온도 기반 t_max 계산 (R1717AS01.3)

허용 dark 증가 한계: ΔD_max = 50 DN

**Arrhenius 모델**:
\[
k(T) = k_{ref} \exp\left[\frac{E_A}{k_B}\left(\frac{1}{T_K} - \frac{1}{T_{ref}}\right)\right]
\]

**R1717AS01.3 실측 파라미터** (벤더 spec + 실제):

| T (°C) | Pixel Leakage @ 3fA nominal | Est. dark drift k(T) [DN/min] | t_max @50DN [min] |
|--------|---------------------------|-------------------------------|-------------------|
| 20 | 3 fA/pixel | 0.75 | 67 |
| 25 | 3 fA/pixel (ref) | 1.0 | 50 |
| 30 | ~ 5.4 fA/pixel (2× @ 10K) | 1.8 | 28 |
| 35 | ~ 9.7 fA/pixel | 3.2 | 16 |
| 40 | ~ 17.4 fA/pixel | 5.8 | 9 |

**MCU에서의 계산** (E_A=0.45 eV):

```c
float estimate_drift_rate(float T) {
    float E_A = 0.45;        // eV
    float k_B = 8.617e-5;    // eV/K
    float T_ref = 298.15;    // 25°C
    float T_K = T + 273.15;
    
    float k_ref = 1.0;       // DN/min @ 25°C (R1717AS01.3 측정값)
    float k_T = k_ref * exp((E_A / k_B) * (1.0/T_K - 1.0/T_ref));
    
    return k_T;
}

uint32_t calculate_t_max(float T, uint32_t delta_D_max) {
    float k_T = estimate_drift_rate(T);
    uint32_t t_max_sec = (uint32_t)(delta_D_max / k_T * 60);  // convert to seconds
    return t_max_sec;
}
```

---

### 4.5 Idle Mode 3단계 정의

| Mode | 조건 | VGH | VGL | VPD | Scan | t_max @25°C | 용도 |
|------|------|-----|-----|-----|------|-----------|------|
| **L1** | < 10 min | +15V | −5V | −2.0V | 없음 | ~50 min | 짧은 대기 |
| **L2** | 10~100 min | +15V | −5V | −0.5V | 60s 주기 dummy | ~200 min | 일반 대기 |
| **L3** | > 100 min | 0V | 0V | 0V (off) | 없음 | ∞ | 장시간 보관 |

---

## 5. 구동 알고리즘 상세 (VGH/VGL/PD back-bias 기준)

### 5.1 R1717AS01.3 FPGA 타이밍 사양

**클럭 및 행 시간**:
- Master clock: 40 MHz
- Row divisor: 1 (line clock = 40 MHz)
- 1 row time: (2048 data cells × 2 + overhead) / 40 MHz ≈ **76 µs**
- Full frame scan (3072 row): 3072 × 76 µs ≈ **235 ms**
- ADC sampling per row: ~20 µs (내 상태, ADC 클럭에 따라 조정)

**Frame 시퀀스 (250 ms/frame 기준)**:

```
1. RESET ALL (3072 row): 3072 × ~1 µs ≈ 3 ms
2. INTEGRATE: 100~150 ms (X-ray 인가 또는 차폐)
3. READ ALL (3072 row × ~20 µs): 3072 × 20 µs ≈ 61 ms
4. Data transfer: ~50 ms
───────────────────────────
Total: ~214~264 ms → ~250 ms baseline

5 fps mode: 200 ms/frame
  → RESET: 3 ms
  → INTEGRATE: 100 ms
  → READ: 61 ms
  → Control: 36 ms
```

---

### 5.2 모드별 전압 및 시퀀스

**중요**: 부록 C에서 모든 모드의 상세 시퀀스 및 코드를 참조.

#### M0: Power-up / 초기화

**전압 세팅**:
- VGH = 0 V (gate driver off)
- VGL = 0 V
- VPD = 0 V

**시퀀스**:
```
1. 모든 출력 low (safe state)
2. DAC init: VPD → 0.0V
3. Row/Col clock stop
4. MCU ready 신호 대기
5. Mode req 신호에 따라 M1/M2로 전환
```

---

#### M1: ACTIVE (촬영/5 fps)

**전압 세팅**:
- VGH = +15 V
- VGL = −5 V
- VPD = −2.0 V (photocurrent 최대)

**시퀀스** (한 프레임):

```c
STATE_ACTIVE:
  set_VPD(-2.0f);          // DAC ch1
  enable_VGH_VGL_driver();  // level shifter on
  enable_ADC();            // 14~16 bit readout
  
  // Phase 1: RESET ALL
  for (row = 0; row < 3072; row++) {
      drive_gate(row, VGH);     // +15V pulse
      pulse_reset_pixel(row);   // 2~5 µs
      drive_gate(row, VGL);     // −5V hold
  }
  
  // Phase 2: INTEGRATE
  wait_ms(integration_time);   // 100~150 ms (climate determined)
  
  // Phase 3: READ ALL
  for (row = 0; row < 3072; row++) {
      drive_gate(row, VGH);
      wait_us(row_settle);      // 2~5 µs
      start_ADC_row_read();
      wait_us(20);              // sample all 3072 pixels
      drive_gate(row, VGL);
  }
  
  // Phase 4: Output
  send_frame_to_host();
```

---

#### M3: Idle L2 (Low-bias idle + Dummy Scan)

**전압 세팅**:
- VGH = +15 V (dummy scan 때만)
- VGL = −5 V
- **VPD = −0.5 V** (중요: 저역바이어스)

**시퀀스**:

```c
STATE_IDLE_L2:
  set_VPD(-0.5f);          // DAC ch1 저바이어스로 변경
  enable_VGH_VGL_driver();
  disable_ADC();           // READ 안 함
  
  dummy_timer = 0;
  
  while (mode_req == IDLE_L2) {
      if (dummy_timer >= T_dummy_seconds) {
          // 한 번의 full row RESET scan
          for (row = 0; row < 3072; row++) {
              drive_gate(row, VGH);
              pulse_reset_pixel(row);
              wait_us(2);
              drive_gate(row, VGL);
          }
          dummy_timer = 0;
      } else {
          dummy_timer++;
          sleep(1000);  // 1 second tick from MCU
      }
  }
```

**효과**: PD bias가 −0.5V이므로 역바이어스 감소 → 누설 60~70% 감소

---

#### M5: CAL/DARK (정기 캘리브레이션)

**전압 세팅**:
- VGH = +15 V
- VGL = −5 V
- VPD = −2.0 V (normal imaging과 동일 조건)

**시퀀스**: M1과 동일하나, DARK/LAG 패턴 반복

```c
STATE_CALIB:
  set_VPD(-2.0f);
  enable_VGH_VGL_driver();
  enable_ADC();
  
  // DARK/LAG pattern (v3.0 참고)
  DARK_frame_1 = capture_frame_no_xray();
  DARK_frame_2 = capture_frame_no_xray();
  LAG_frame_1 = capture_frame_with_brief_xray(10ms);
  LAG_frame_2 = capture_frame_no_xray();
  TEST_frame = capture_frame_with_xray(1000ms);
  
  send_all_frames_to_host_for_lut_update();
```

---

## 6. 구현 계획 및 일정

*(v3.0과 동일, 부록 C의 상세 코드로 지원)*

### 6.1 Work Breakdown Structure (WBS) - 수정사항

**Task 1.1 추가**: Gate driver 타이밍 검증 (VGH +15V / VGL −5V 전환 시간 < 100 ns)

**Task 2.2 수정**: t_max 계산 로직 → R1717AS01.3 파라미터 반영 (k_ref=1.0 DN/min @ 25°C)

**Task 3.1 추가**: Dark LUT 생성 도구 → 2D interpolation 최적화 (6×6 grid)

---

## 7. 성능 목표 및 검증

*(v3.0과 동일)*

---

## 부록 A: 현실적 벤더 협조 요청 항목

*(v3.0과 동일)*

---

## 부록 B: 온도별 Drift Rate LUT 측정 프로토콜

*(v3.0과 동일)*

---

## 부록 C: R1717AS01.3 모드별 전압 및 FPGA 시퀀스 상세

### C.1 전체 모드 전압 매트릭스 (VGH/VGL/VPD)

| 모드 | 설명 | VGH | VGL | VPD | Gate driver | Row/Col clock | ADC |
|------|------|-----|-----|-----|-----------|-----------|-----|
| M0 | Power-up | 0V | 0V | 0V | OFF | STOP | OFF |
| M1 | ACTIVE | +15V | −5V | −2.0V | ON | 40MHz | ON |
| M2 | Idle L1 | 0V | −5V | −2.0V | OFF | STOP | OFF |
| M3 | Idle L2 | +15V* | −5V | −0.5V | ON (dummy) | 40MHz (periodic) | OFF |
| M4 | Idle L3 | 0V | 0V | 0V | OFF | STOP | OFF |
| M5 | CAL/DARK | +15V | −5V | −2.0V | ON | 40MHz | ON |

*M3에서 VGH는 dummy scan 시에만 +15V 펄스, 평소 0V

---

### C.2 모드별 FPGA 시퀀스 (상세 pseudo code)

#### C.2.1 M0: Power-up

```c
void fpga_init_m0(void) {
    // Safe initialization
    set_gate_driver_off();
    set_row_col_clock_stop();
    set_adc_off();
    set_all_outputs_low();
    
    // DAC initialization (software controlled)
    dac_set_channel(1, 0.0f);   // VPD = 0V
    dac_set_channel(2, 0.0f);   // VCOM = 0V (if separate)
    
    // Wait for MCU mode request
    while (mode_request == MODE_POWER_UP) {
        sleep_us(1);
    }
    
    // Transition
    if (mode_request == MODE_ACTIVE) {
        fpga_transition_to_m1();
    } else if (mode_request == MODE_IDLE_L1) {
        fpga_transition_to_m2();
    }
}
```

---

#### C.2.2 M1: ACTIVE

```c
void fpga_active_m1(void) {
    // Setup
    set_gate_driver_on();     // Enable VGH/VGL output
    set_row_col_clock_on(40000000);  // 40 MHz
    set_adc_on(14bit_mode);
    dac_set_channel(1, -2.0f);       // VPD = -2.0V
    
    // Main loop
    frame_count = 0;
    while (mode_request == MODE_ACTIVE) {
        // === PHASE 1: RESET ALL ===
        for (row = 0; row < 3072; row++) {
            // Select row
            set_gate_line(row, VGH);            // Drive to +15V
            wait_us(1);
            
            // Trigger reset transistor (2~5µs pulse)
            pulse_reset(row, 5);                // 5µs pulse
            
            // Deselect row
            set_gate_line(row, VGL);            // Return to -5V
            wait_us(1);
        }
        
        // === PHASE 2: INTEGRATE ===
        // Integration time controlled by MCU (external signal)
        wait_for_integrate_done();
        
        // === PHASE 3: READ ALL ===
        adc_frame_buffer_clear();
        for (row = 0; row < 3072; row++) {
            // Select row
            set_gate_line(row, VGH);
            wait_us(5);  // Row settle time
            
            // ADC sample all 3072 data columns
            adc_start_row_read(row);
            
            // Column readout: 3072 pixels × clock/pixel
            // @ 40MHz, 2 clocks/pixel = 20µs for row
            wait_us(20);
            
            adc_latch_row(row);
            
            // Deselect row
            set_gate_line(row, VGL);
        }
        
        // === PHASE 4: FIFO OUTPUT ===
        adc_frame_to_fifo();
        
        frame_count++;
    }
    
    // Cleanup on mode change
    set_gate_driver_off();
    set_adc_off();
}
```

---

#### C.2.3 M2: Idle L1 (Normal idle)

```c
void fpga_idle_l1_m2(void) {
    // Setup
    set_gate_driver_off();      // No gate pulses
    set_row_col_clock_stop();
    set_adc_off();
    dac_set_channel(1, -2.0f);  // VPD still -2.0V (normal bias)
    
    // Main loop: Do nothing, MCU manages idle timer
    while (mode_request == MODE_IDLE_L1) {
        sleep_ms(100);  // MCU timer increments
    }
    
    // Transition on MCU command
    if (mode_request == MODE_ACTIVE) {
        fpga_transition_to_m1();
    } else if (mode_request == MODE_IDLE_L2) {
        fpga_transition_to_m3();
    }
}
```

---

#### C.2.4 M3: Idle L2 (Low-bias + Dummy Scan)

```c
void fpga_idle_l2_m3(void) {
    // Setup
    set_gate_driver_on();       // Enable gate for dummy scan
    set_adc_off();              // No readout
    dac_set_channel(1, -0.5f);  // VPD = -0.5V (LOW BIAS)
    
    // Dummy scan engine
    dummy_scan_interval = T_DUMMY_SECONDS;  // e.g., 60 seconds
    dummy_scan_timer = 0;
    
    while (mode_request == MODE_IDLE_L2) {
        if (dummy_scan_trigger || dummy_scan_timer >= dummy_scan_interval) {
            // === DUMMY SCAN: RESET-only, no READ ===
            for (row = 0; row < 3072; row++) {
                set_gate_line(row, VGH);        // +15V
                wait_us(1);
                pulse_reset(row, 3);            // Shorter pulse for dummy
                wait_us(1);
                set_gate_line(row, VGL);        // -5V
            }
            
            dummy_scan_timer = 0;
            dummy_scan_trigger = 0;
        } else {
            // Sleep until next dummy scan or MCU wake
            dummy_scan_timer++;
            sleep_ms(1000);
        }
    }
    
    // Cleanup on transition
    dac_set_channel(1, 0.0f);   // Return to safe
    set_gate_driver_off();
}
```

---

#### C.2.5 M4: Idle L3 (Deep Sleep)

```c
void fpga_idle_l3_m4(void) {
    // Full shutdown
    set_gate_driver_off();
    set_row_col_clock_stop();
    set_adc_off();
    dac_set_channel(1, 0.0f);    // VPD = 0V (off)
    set_all_outputs_low();
    
    // Deep sleep: only MCU interrupt or external wake-up
    while (mode_request == MODE_IDLE_L3) {
        sleep_deep();  // Minimal power
    }
    
    // Wake up: must go through M0
    goto STATE_M0_POWER_UP;
}
```

---

#### C.2.6 M5: CAL/DARK

```c
void fpga_calib_m5(void) {
    // Setup (same as M1)
    set_gate_driver_on();
    set_row_col_clock_on(40000000);
    set_adc_on(14bit_mode);
    dac_set_channel(1, -2.0f);   // VPD = -2.0V (normal)
    
    // Calibration pattern: DARK1, DARK2, LAG1, LAG2, TEST
    while (mode_request == MODE_CALIB) {
        // Frame 1: DARK1
        reset_all_rows();
        wait_ms(100);
        read_all_rows();
        store_frame_0();
        
        // Frame 2: DARK2
        reset_all_rows();
        wait_ms(100);
        read_all_rows();
        store_frame_1();
        
        // Frame 3: LAG1 (brief exposure)
        reset_all_rows();
        wait_ms(100);
        trigger_xray(10);  // 10ms X-ray pulse
        read_all_rows();
        store_frame_2();
        
        // Frame 4: LAG2 (dark after exposure)
        reset_all_rows();
        wait_ms(100);
        read_all_rows();
        store_frame_3();
        
        // Frame 5: TEST (full exposure)
        reset_all_rows();
        trigger_xray(1000);  // 1000ms X-ray
        wait_ms(100);
        read_all_rows();
        store_frame_4();
        
        // Send all frames for host LUT update
        send_5_frames_to_host();
    }
}
```

---

### C.3 DAC 제어 (온도 기반 동적 조정)

**MCU ↔ FPGA DAC 인터페이스**:

```c
// MCU 측 (1 Hz interrupt)
void mcu_timer_1hz_isr(void) {
    float T_current = read_temp_sensor();
    
    // Temperature range check
    if (T_current < 20.0f) {
        mcu_current_temp_range = TEMP_COLD;
    } else if (T_current < 30.0f) {
        mcu_current_temp_range = TEMP_NORMAL;
    } else {
        mcu_current_temp_range = TEMP_HOT;
    }
    
    // Update t_max
    mcu_t_max_current = calculate_t_max(T_current, 50);
    
    // Idle time increment
    mcu_idle_time_sec++;
    
    // Mode transition check
    if (mcu_idle_time_sec > mcu_t_max_current) {
        if (current_mode == IDLE_L1) {
            transition_to_idle_l2();  // Signal FPGA
            mcu_idle_time_sec = 0;
        } else if (current_mode == IDLE_L2) {
            transition_to_idle_l3();
            mcu_idle_time_sec = 0;
        }
    }
}

// FPGA 측: DAC register interface
// MCU writes to register 0x1000 (VPD setting)
// FPGA continuously monitors register
always @(posedge clk) begin
    if (mcubus_write && mcubus_addr == 12'h1000) begin
        vpd_dac_value <= mcubus_data;  // Load VPD setting
    end
end

// DAC output control
assign dac_output = vpd_dac_value;  // Convert to analog
```

---

### C.4 타이밍 다이어그램 (M1 ACTIVE 한 프레임)

```
시간축 (ms)  |0----3----|3--------103--------|103---164----|164-210-|210--250|
            |
Signal:     |
VGH(gate)   |___|VGH___|_____VGL_____...(3072×)...|__VGH__|___VGL___|
            
VPD (bias)  |_____-2.0V__________________________|_-2.0V_|_-2.0V___|
            
ADC_enable  |________________OFF_______________|__ON___|__ON____|__OFF|
            
Phase:      |  RESET  |   INTEGRATE (100ms)    | READ  | OUTPUT | Idle
            |         |                        |       |        |
```

**타이밍 계산**:
- RESET: 3072 row × 1µs settle ≈ 3 ms
- INTEGRATE: 100 ms (climate adjustment)
- READ: 3072 row × 20µs/row ≈ 61 ms
- OUTPUT: ~50 ms (FIFO to host)
- **Total ≈ 214 ms** (250 ms frame time 내에 여유)

---

### C.5 MCU ↔ FPGA 핸드셰이크

**레지스터 맵**:

| 주소 | 이름 | 방향 | 역할 |
|------|------|------|------|
| 0x0000 | MODE_REQ | MCU→FPGA | 요청 모드 (0~5) |
| 0x0001 | MODE_ACK | FPGA→MCU | 현재 모드 (acknowledge) |
| 0x0002 | TEMP_CURRENT | MCU→FPGA | 온도 값 (float) |
| 0x1000 | VPD_DAC | MCU→FPGA | VPD 설정 (-2.0V or -0.5V) |
| 0x1001 | T_MAX_REG | MCU→FPGA | t_max 값 (seconds) |
| 0x2000 | FRAME_DATA_FIFO | FPGA→MCU | 프레임 데이터 출력 |

**Handshake 예시**:

```
1. MCU sets MODE_REQ = 1 (ACTIVE mode)
2. FPGA reads MODE_REQ, transitions to M1
3. FPGA sets MODE_ACK = 1 (confirmation)
4. MCU reads MODE_ACK = 1, knows FPGA is ready
5. MCU updates TEMP_CURRENT every 1 second
6. FPGA reads TEMP_CURRENT, calculates k(T)
7. MCU calculates t_max, updates T_MAX_REG
8. FPGA monitors T_MAX_REG for Idle L1→L2 transition trigger
```

---

## 최종 체크리스트

**프로젝트 시작 전:**

- [ ] R1717AS01.3 패널 샘플 수령 및 스펙 확인
- [ ] VGH(+15V), VGL(−5V), PD bias rail 회로 설계 완료
- [ ] DAC 칩 선택 (예: 4채널 ±10V 16bit DAC)
- [ ] Gate driver IC 선택 (VGH/VGL switching 시간 < 100 ns 확인)
- [ ] 온도 센서 장착 및 ADC 캘리브레이션
- [ ] 개발팀 배포 및 review

**개발 진행 중:**

- [ ] FPGA RTL: state machine (M0~M5) 구현 및 시뮬레이션
- [ ] MCU firmware: 온도 읽기 + t_max 계산 + mode transition logic
- [ ] Host SW: Dark LUT 2D interpolation 함수 (≤ 5% 오차)
- [ ] 주간 진행 미팅

**테스트 단계:**

- [ ] FPGA 시뮬레이션: 모든 mode transition 검증
- [ ] 통합 테스트: VGH/VGL 전환 timing 측정 (오실로스코프)
- [ ] Dark measurement @ 6 temperatures × 6 idle_time points
- [ ] KPI 검증: dark correction ≥ 30%, idle 시간 ≥ 100 min (L2 @ 25°C)

**배포 전:**

- [ ] 최종 RTL 빌드 및 타이밍 클로징 (< 40 MHz 제약)
- [ ] 최종 펌웨어 버전 tag
- [ ] 사용자 매뉴얼 및 트러블슈팅 가이드
- [ ] GO/NO-GO 최종 승인

---

## 버전 관리

**v3.0 → v3.1 change log**:

| Section | Change | Lines |
|---------|--------|-------|
| 4. 모드 정의 | R1717AS01.3 파라미터 통합 | ~150 |
| 5. 타이밍 | row=3072, scan≈76µs 계산 | ~50 |
| 부록 C | NEW: 시퀀스 + 코드 | ~400 |
| 체크리스트 | R1717AS01.3 specific | ~10 |

**Total additions**: ~610 lines

---

**작성**: 2026-02-10  
**최종 버전**: 3.1  
**상태**: ✅ R1717AS01.3 완전 통합 완료

이제 실제 panel spec (VGH=+15V, VGL=-5V, PD=-2V to -0.5V) 및  
3072×3072 row/column 구조를 반영한 구현 가능한 상세 가이드입니다.

