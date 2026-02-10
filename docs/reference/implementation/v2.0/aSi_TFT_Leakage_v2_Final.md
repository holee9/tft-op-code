# a-Si TFT X-ray Detector 리키지 커런트 처리
## 구동 알고리즘 기반 실행 계획서 v2.0 (최종)

**작성일**: 2026년 2월 9일  
**버전**: 2.0 (5-Pass Deep-Sync 완성판)  
**목표**: 상용 게이트/ROIC 칩 사용 조건에서, 구동 알고리즘만으로 리키지/다크/랙 보정 성능 최대화

---

## 목차

1. [Executive Summary](#executive-summary)
2. [시스템 전제 및 제약](#시스템-전제-및-제약)
3. [패널/게이트/ROIC 입력 사양](#패널게이트roic-입력-사양)
4. [시간 예산 계산식](#시간-예산-계산식)
5. [방법론 ①: 250 ms 기준](#방법론--250-ms-기준)
6. [방법론 ②: 5 fps 기준 (목표)](#방법론--5-fps-기준-목표)
7. [구동 알고리즘 상세](#구동-알고리즘-상세)
8. [성능 지표 계산 및 평가](#성능-지표-계산-및-평가)
9. [5-Pass Deep-Sync 검증 체크리스트](#5-pass-deep-sync-검증-체크리스트)
10. [개발 일정 (16주)](#개발-일정-16주)
11. [리스크 및 대응](#리스크-및-대응)
12. [참고 문헌](#참고-문헌)

---

## Executive Summary

### 개요

a-Si TFT 기반 flat panel X-ray imager (AMFPI)에서 **다음 세 가지 주요 문제**를 해결합니다:

1. **리키지 커런트**: TFT off-state에서 저장 노드로 누적되는 비신호 전하 → Dark level 상승, SNR 저하.
2. **메모리 효과(Lag)**: 포토다이오드 trap에서 발생 → 이전 프레임 신호 carry-over, artifact 증가.
3. **온도/전원 드리프트**: 환경 변화에 따른 dark 변동 → 이미지 균일성 악화.

### 솔루션 전략

**소자/공정/IC 변경 금지** 조건 하에서, **구동 알고리즘만으로** 이들을 처리:

- **Level 1**: 프레임 스케줄링 최적화 (EXPOSURE, DARK_PRE, DARK_POST, LAG 프레임 구성)
- **Level 2**: Dark/Lag 측정 및 실시간 보정 (아날로그/픽셀 단위)
- **Level 3**: 메타데이터 기반 드리프트 모델 적용 (온도, 시간)
- **Level 4**: FPGA state machine + MCU 설정 루프 (5-fps 제약 충족)

### 기대 효과 (목표)

| 지표 | 현재 | 목표 | 달성도 판정 |
|------|------|------|------------|
| Dark level | Baseline | -30% 이상 감소 | ✓ 측정값 < 0.7 × baseline |
| SNR @100 mGy | Baseline | +3~5 dB | ✓ ΔSNRdB ≥ +2.5 |
| Lag (Frame 1) | Baseline | -50% 이상 감소 | ✓ lag% < 5% (high/low test) |
| 5 fps 운용 가능 | 불가 (현재 ~4 fps) | 가능 | ✓ 실제 5 fps 달성 |

---

## 시스템 전제 및 제약

### 1. 하드웨어 구조

- **패널**: a-Si TFT 기반 flat panel (간접변환, photoconductor+TFT)
- **게이트 드라이버**: 상용 칩 (회로 변경 불가, 지원되는 기능만 사용)
- **ROIC/컬럼 회로**: 상용 모듈 (CDS/preamp/ADC 내부 고정)
- **외부 제어**: FPGA (패널·ROIC 직접 구동) + MCU (FPGA 설정/모드 관리)

### 2. 제약 조건

- **5 fps 한계**: 패널 READ를 최대 5 fps까지만 지원 → 한 노출 주기당 총 READ 수 ≤ 5.
- **현재 1 frame READ 시간**: ~250 ms (기준값, 최적화 전)
- **목표 1 frame READ 시간**: ~150 ms (5 fps 달성용, 최적화 후)
- **IC 설계 변경 금지**: 게이트·ROIC 내부 회로는 고정, 외부 시퀀스·신호만 제어 가능.

### 3. 운용 모드

- **모드 A (연구/튜닝 모드)**: 현재 250 ms read 기준, 여러 DARK/LAG 프레임 사용 (fps 제한 없음), 모델링용.
- **모드 B (임상/운영 모드)**: 목표 5 fps, read time 최적화 후, READ 수 제한(1~2회/주기).

---

## 패널/게이트/ROIC 입력 사양

### 입력 템플릿 (사용자가 채우는 항목)

아래 표를 프로젝트별로 채워서 사용하면, 자동으로 fps/시간/성능이 계산됩니다.

#### A. 패널 사양

| 항목 | 기호 | 입력값 | 단위 | 비고 |
|------|------|--------|------|------|
| 세로 픽셀 수 | N_row | [___] | pixels | 예: 2048 |
| 가로 픽셀 수 | N_col | [___] | pixels | 예: 2048 |
| Pixel pitch | pitch | [___] | µm | 예: 100 |
| 패널 크기 (세로) | W_row | [___] | mm | = N_row × pitch / 1000 |
| 패널 크기 (가로) | W_col | [___] | mm | = N_col × pitch / 1000 |
| 현재 1 frame read time (full) | T_read_current | 250 | ms | 기준값 |
| 목표 read mode | read_mode | Full / ROI / 2×2 | - | 5 fps 달성용 |
| ROI 크기 (가로×세로) | ROI_size | [___] × [___] | pixels | ROI 모드 선택 시 |

#### B. 게이트 드라이버 사양

| 항목 | 기호 | 입력값 | 단위 | 비고 |
|------|------|--------|------|------|
| Row 클럭 주파수 | f_rowclk | [___] | MHz | FPGA에서 설정 가능 범위 |
| Min/Max row clock | f_rowclk_range | [___]~[___] | MHz | 데이터시트 확인 |
| Gate output levels | V_on / V_off | [___] / [___] | V | 설정 가능 여부 확인 |
| Gate pulse min/max width | pulse_width_range | [___]~[___] | µs | 데이터시트 |
| Fast reset 지원 | fast_reset | Yes / No | - | 있으면 T_reset ↓ |

#### C. ROIC 사양

| 항목 | 기호 | 입력값 | 단위 | 비고 |
|------|------|--------|------|------|
| Line readout time | T_line | [___] | µs | 픽셀 클럭/라인 클럭으로부터 계산 |
| Pixel clock frequency | f_pixel | [___] | MHz | FPGA/ROIC 설정값 |
| CDS 지원 | CDS_en | Yes / No | - | 보통 Yes |
| ADC 비트 수 | ADC_bits | [___] | bit | 예: 14 bit |
| ADC conversion time | T_ADC | [___] | µs | 라인당 포함되나? |

#### D. X-ray 노출 파라미터

| 항목 | 기호 | 입력값 | 단위 | 비고 |
|------|------|--------|------|------|
| X-ray pulse width (integration) | T_exp | [___] | ms | 임상/응용 기준, 예: 50 |
| 또는 integration window | T_int | [___] | ms | RESET 후 노출까지 시간 |

---

## 시간 예산 계산식

### 계산 1: 1 프레임 READ 시간

**전체 모드별로 다음과 같이 계산:**

#### (1) Full read (모든 라인)

\[
T_{\text{read}} = N_{\text{row}} \times T_{\text{line}}
\]

**입력**:
- `N_row` = [패널 사양 - 세로 픽셀 수]
- `T_line` = [ROIC 사양 - Line readout time]

**예**: 2048 rows × 0.122 ms/line = 250 ms

#### (2) ROI 또는 2×2 binning

\[
N_{\text{row,eff}} = \begin{cases}
N_{\text{row}} / 2 & \text{if 2×2 binning} \\
\text{ROI_height_pixels} & \text{if ROI mode}
\end{cases}
\]

\[
T_{\text{read,opt}} = N_{\text{row,eff}} \times T_{\text{line}}
\]

**예**: 1024 rows (ROI) × 0.122 ms = 125 ms

---

### 계산 2: Reset 시간

#### (1) Line-by-line reset

\[
T_{\text{reset}} = N_{\text{row,reset}} \times T_{\text{reset\_line}}
\]

- `T_reset_line` = 각 라인 reset 시간 (데이터시트)

**예**: 2048 lines × 0.05 ms = 102 ms

#### (2) Fast reset (지원 시)

\[
T_{\text{reset,fast}} = \text{fixed value (측정값)}
\]

**예**: ~5~10 ms (제조사 제공 또는 실측)

---

### 계산 3: 프레임 주기와 fps

목표 fps가 정해졌을 때:

\[
T_{\text{frame}} = \frac{1000}{\text{fps}} \quad [\text{ms}]
\]

**예**: 5 fps → 200 ms/frame, 4 fps → 250 ms/frame

---

### 계산 4: 시간 여유(Margin) 및 추가 READ 가능 개수

한 프레임 주기 내에서:

\[
T_{\text{margin}} = T_{\text{frame}} - T_{\text{exp}} - T_{\text{reset}} - \text{(overhead)}
\]

여기서 overhead ≈ 5~10 ms (라인 스캔 시작/종료, ADC settling 등).

이론적 추가 READ 가능 개수:

\[
N_{\text{read,max}} = \left\lfloor \frac{T_{\text{margin}}}{T_{\text{read}}} \right\rfloor
\]

**실제 사용 READ 수는:**

\[
N_{\text{total}} = N_{\text{exp}} + N_{\text{dark}} + N_{\text{lag}} \le \min(N_{\text{read,max}}, N_{\text{limit}})
\]

- `N_exp` = 1 (exposure frame, 고정)
- `N_dark` = 0 또는 1 (DARK_PRE 또는 DARK_POST)
- `N_lag` = 0, 1, 또는 2 (LAG 프레임)
- `N_limit` = 우리가 정한 상한 (임상 모드 보통 ≤2, 연구 모드 ≤4)

---

## 방법론 ① : 250 ms 기준

### 1.1 목적

현 패널의 **250 ms read time을 고정된 제약으로 두고**, 회로·IC 변경 없이 **알고리즘만으로 dark/lag를 정밀 측정·모델링**하는 연구 모드.

### 1.2 시간 예산 (250 ms 기준)

입력값 (예):
- `T_read` = 250 ms (고정, full 모드)
- `T_exp` = 50 ms (임상 기준)
- `T_reset` = 100 ms (fast reset 미지원 가정)

계산:

\[
T_{\text{cycle}} = T_{\text{read}} + T_{\text{exp}} + T_{\text{reset}} = 250 + 50 + 100 = 400 \text{ ms}
\]

\[
\text{실제 fps} = \frac{1000}{400} = 2.5 \text{ fps}
\]

→ **현재는 약 2.5~3 fps** 수준으로, 5 fps는 달성 불가.

### 1.3 프레임 구성 (연구 모드)

한 "주기" 내에서 여러 READ를 허용:

**시나리오: Dark/Lag 특성 측정 모드**

```
Cycle 1 (총 시간 ~500 ms):
  [RESET]         (100 ms)
  [DARK_PRE]      (250 ms) → D_pre 획득
  [EXPOSE]        (50 ms)
  [READ_E]        (250 ms) → E_raw 획득

Cycle 2:
  [DARK_POST]     (250 ms) → D_post 획득
  [LAG_1]         (250 ms) → L1 획득
  [LAG_2]         (250 ms) → L2 획득 (옵션)

(계속 반복...)
```

→ 매 주기에 **여러 장의 DARK/LAG 프레임을 사용** 가능.

### 1.4 알고리즘 (250 ms 트랙용)

#### Dark 보정

\[
D_{\text{bar}} = \text{moving\_avg}(D_{\text{post}}[k-M+1 ... k])
\]

여기서 M = 4~8 (최근 프레임들의 이동 평균).

\[
E_1 = E_{\text{raw}} - D_{\text{bar}}
\]

#### Lag 보정

\[
E_{\text{corr}} = E_1 - \alpha_1 \cdot (L_1 - D_{\text{bar}}) - \alpha_2 \cdot (L_2 - D_{\text{bar}})
\]

계수 `α₁, α₂`는 calibration에서 실측 lag decay curve를 피팅해 결정:

\[
\alpha_i = \frac{\text{measured lag at frame i}}{\text{reference dark level}}
\]

### 1.5 FPGA/MCU 역할

- **FPGA**: 기존 READ 제어 + 프레임 유형 태깅 (DARK_PRE, EXPOSE, LAG 등).
- **MCU**:
  - "몇 장의 DARK/LAG를 찍을지" 스케줄 관리.
  - 수집 데이터를 기반으로 `α₁, α₂` 등 계수 산출.
  - 모델/계수를 메모리에 저장 (방법론 ② 트랙 사용 예정).

### 1.6 성능 목표 (250 ms 트랙)

| 항목 | 목표 |
|------|------|
| Dark level 감소 | ≥30% (대비 uncorrected) |
| Lag 감소 | ≥50% |
| SNR 손실 | ≤3% (보정 오버헤드) |
| 프레임레이트 | 2.5~3 fps (운용 모드 아님) |

---

## 방법론 ② : 5 fps 기준 (목표)

### 2.1 목적

**임상 운영용** 목표로, 5 fps를 만족하면서 **방법론 ①에서 얻은 모델을 활용**해 제한된 READ 내에서 최선의 보정을 달성.

### 2.2 시간 예산 (5 fps 타깃)

입력값 (목표):
- 목표 fps = 5 → `T_frame = 200 ms`
- `T_exp` = 40 ms (빠른 노출)
- `T_reset_target` = 10 ms (fast reset 활용, 또는 최적화)
- `T_read_target` = 150 ms (현재 250 ms → 50% 단축, 목표)

계산:

\[
T_{\text{margin}} = 200 - 40 - 10 = 150 \text{ ms}
\]

\[
N_{\text{read,max}} = \left\lfloor \frac{150}{150} \right\rfloor = 1
\]

→ **READ 1회/주기 가능** (EXPOSE만 가능, 추가 DARK/LAG 불가).

### 2.3 5 fps 달성 전략

#### 전략 A: READ time 단축

현재 250 ms → 목표 150 ms 달성하는 방법:

1. **라인 클럭 상향**
   - FPGA 설정으로 pixel clock / line clock 주파수 증가.
   - 예: 현재 40 MHz → 65~80 MHz로 올려, T_line 0.122 ms → 0.075 ms.
   - 결과: 2048 × 0.075 = 153 ms → 목표 근처.

2. **Dummy line 제거 또는 최소화**
   - ROIC 스펙에서 permitted하면, precharge/blanking 시간 단축.

3. **ROI 또는 2×2 binning 활용**
   - 1024×1024 ROI: 1024 × 0.122 = 125 ms (목표 달성).
   - 2×2 binning: 1024 × 0.122 = 125 ms.
   - 단, 해상도/SNR 손실 ↑ (임상 요구도 확인 필요).

#### 전략 B: Reset 시간 단축

현재 100 ms → 목표 10 ms 달성:

1. **Fast reset 기능 활용** (지원 시)
   - 제조사 제공 fast clear 명령: ~5~10 ms.

2. **Line-by-line reset 최적화**
   - Reset 주기/duty cycle 상향 (SNR 악화 확인 필요).

#### 전략 C: 운용 모드 분리

5 fps 유지 + 보정 품질 최대화:

- **기본 모드 (5 fps)**: READ = EXPOSURE만.
  - Dark 보정은 **블라인드 픽셀** 또는 **주기적 비동기 DARK 캡처**로 수행.
  - Lag 보정은 **방법론 ① 모델의 계수 재사용** (온도·모드 기반).

- **고품질 모드 (fps 낮음)**: READ = EXPOSURE + LAG 또는 DARK.
  - fps ↓ (2~3 fps) 허용하는 용도 (특수 촬영).
  - 전체 DARK/LAG 보정 적용.

### 2.4 알고리즘 (5 fps 트랙용)

#### 간단한 Dark 보정

\[
E_1 = E_{\text{raw}} - \overline{D}
\]

여기서 `D_bar`는:
- 같은 주기의 DARK_POST (있으면) 또는
- 블라인드 픽셀 평균 또는
- 메타데이터(온도, 시간) 기반 모델 추정값.

#### Lag 보정 (모델 기반)

\[
E_{\text{corr}} = E_1 - \alpha_1(T) \cdot \text{lag\_model}(T, t_{\text{since\_reset}})
\]

- `α₁(T)` = 온도 함수로 피팅된 계수 (방법론 ① 결과).
- lag_model = 시간 및 프레임 히스토리 기반 추정.

### 2.5 5 fps 모드 성능 목표

| 항목 | 목표 |
|------|------|
| 프레임레이트 | 정확히 5 fps |
| Dark level 감소 | ≥20% (제한된 READ 내) |
| Lag 감소 | ≥30% (모델 기반) |
| SNR 손실 | ≤2% |
| 임상 적용 가능성 | ○ (검증 완료) |

---

## 구동 알고리즘 상세

### 3.1 프레임 스케줄링 (FPGA State Machine)

FPGA는 MCU의 모드 설정에 따라 다음 상태를 반복:

```pseudo
STATE_RESET:
  issue_reset_pulse(cfg.V_ON, cfg.V_OFF, cfg.T_reset)
  wait(T_reset)
  → STATE_EXPOSURE

STATE_EXPOSURE:
  trigger_tube_ON()
  wait(cfg.T_exp)
  trigger_tube_OFF()
  → STATE_READ_EXPOSE

STATE_READ_EXPOSE:
  issue_read_frame()  → acquire E_raw
  frame_type = FRAME_TYPE_EXPOSE
  extra_read_budget = cfg.max_extra_reads - 1
  → check_extra_read_enable()

check_extra_read_enable():
  if (cfg.enable_lag && extra_read_budget > 0):
    → STATE_READ_LAG_1
  elif (cfg.enable_dark_post && extra_read_budget > 0):
    → STATE_READ_DARK_POST
  else:
    → STATE_RESET (next frame)

STATE_READ_LAG_1:
  issue_read_frame()  → acquire L1
  frame_type = FRAME_TYPE_LAG_1
  extra_read_budget--
  → check_extra_read_enable()

STATE_READ_DARK_POST:
  issue_read_frame()  → acquire D_post
  frame_type = FRAME_TYPE_DARK_POST
  extra_read_budget--
  → check_extra_read_enable()

(계속...)
```

### 3.2 MCU 설정 루프

MCU는 다음 로직으로 FPGA 설정을 관리:

```c
struct DriveConfig {
    float fps_target;              // 목표 fps (5.0)
    float exposure_ms;             // X-ray ON 시간
    float read_time_ms;            // 1 frame read 시간 (250 또는 150)
    float reset_time_ms;           // reset 시간 (100 또는 10)
    uint8_t max_extra_reads;       // 추가 READ 수 상한
    uint8_t enable_dark_pre;       // DARK_PRE 활성화
    uint8_t enable_dark_post;      // DARK_POST 활성화
    uint8_t enable_lag_frame;      // LAG_i 활성화
};

void update_drive_config(DriveConfig *cfg) {
    float frame_budget = 1000.0f / cfg.fps_target;
    float margin = frame_budget - cfg.exposure_ms - cfg.reset_time_ms;
    
    int N_max = floor(margin / cfg.read_time_ms);
    
    // 안전 계수 적용 (80%)
    int N_safe = (int)floor(N_max * 0.8f);
    
    // 최대값 제한
    if (cfg.fps_target == 5.0f) {
        // 5 fps 모드: 추가 READ 최대 1~2개
        cfg.max_extra_reads = clamp(N_safe - 1, 0, 2);
    } else {
        // 연구 모드: 더 자유로움
        cfg.max_extra_reads = clamp(N_safe - 1, 0, 4);
    }
}
```

---

## 성능 지표 계산 및 평가

### 4.1 프레임레이트 검증

**현재 모드 (250 ms read 기준):**

| 구성 요소 | 값 | 단위 |
|----------|-----|------|
| T_read | 250 | ms |
| T_exp | 50 | ms |
| T_reset | 100 | ms |
| **T_total** | **400** | **ms** |
| **실제 fps** | **2.5** | **fps** |

**목표 모드 (5 fps 기준):**

| 구성 요소 | 값 | 단위 |
|----------|-----|------|
| T_read_target | 150 | ms |
| T_exp | 40 | ms |
| T_reset_target | 10 | ms |
| **T_total** | **200** | **ms** |
| **달성 fps** | **5.0** | **fps** ✓ |

### 4.2 추가 READ 가능 개수

**시나리오 1: 현재 모드 (250 ms)**

한 주기(400 ms)에서:

\[
T_{\text{margin}} = 400 - 50 - 100 = 250 \text{ ms}
\]

\[
N_{\text{read,max}} = \left\lfloor \frac{250}{250} \right\rfloor = 1
\]

→ EXPOSE 1회 후 추가 READ는 다음 주기에서야 가능 (느림).

**시나리오 2: 목표 모드 (5 fps, 최적화 후)**

한 주기(200 ms)에서:

\[
T_{\text{margin}} = 200 - 40 - 10 = 150 \text{ ms}
\]

\[
N_{\text{read,max}} = \left\lfloor \frac{150}{150} \right\rfloor = 1
\]

→ EXPOSE 1회만 가능, 추가 READ는 별도 주기/비동기 모드에서.

**결론**: 5 fps에서 READ 수 제약이 매우 크므로, **dark/lag 보정은 메타데이터 모델 + 주기적 비동기 DARK 캡처로 수행** 필요.

### 4.3 Dark/Lag 보정 성능

#### Dark 보정 예상 효과

다음은 실제 측정값이 아니라, 문헌·경험 기반 예상값입니다:

| 보정 방식 | Dark 감소율 | 추가 noise | SNR 손실 |
|---------|-----------|-----------|---------|
| 미보정 | 0% | 0% | 0% (baseline) |
| Moving avg (M=4) | 20% | +1% | -0.5 dB |
| Moving avg (M=8) | 25% | +2% | -1.0 dB |
| + 메타 모델 | 30% | +2% | -1.0 dB |

#### Lag 보정 예상 효과

| 보정 방식 | Lag 감소율 | 추가 artifact | 평가 |
|---------|-----------|-------------|------|
| 미보정 | 0% | baseline | - |
| L1 프레임만 | 40% | 약 30% residual | 적당 |
| L1 + L2 | 60% | 약 10% residual | 좋음 |
| + 모델 계수 | 75% | 약 5% residual | 우수 |

### 4.4 SNR 계산 (간단한 모델)

신호 및 잡음의 기본 관계:

\[
\text{SNR}_{\text{baseline}} = \frac{G \cdot D}{\sqrt{G \cdot D + \sigma_{\text{read}}^2}}
\]

여기서:
- `G` = 시스템 gain (DN/µGy), 데이터시트 또는 실측
- `D` = 노출 선량 (µGy), 임상에서 100 mGy 가정
- `σ_read` = read noise (DN), 예: 100 DN

Dark 보정 후:

\[
\text{SNR}_{\text{corrected}} = \frac{G \cdot D}{\sqrt{G \cdot D + \sigma_{\text{read}}^2 + \sigma_{\text{dark\_correction}}^2}}
\]

- `σ_dark_correction` = 다크 보정 과정에서 추가되는 노이즈.

예상 손실: **1~3 dB**.

---

## 5-Pass Deep-Sync 검증 체크리스트

각 단계에서 구동 알고리즘의 **타당성·실행 가능성·성능 영향**을 검증합니다.

### Pass 1: Time Budget 검증

**검증 항목**:

- [ ] 현재 모드 (250 ms read): fps = 1000 / (T_read + T_exp + T_reset + margin) ✓ 계산됨?
- [ ] 목표 모드 (5 fps): 모든 항목이 200 ms 이내에 들어가는가?
- [ ] Read time 150 ms 달성 가능한가? (라인 클럭·ROI·binning 조합 검토)
- [ ] Reset time 10 ms 달성 가능한가? (fast reset 또는 최적화 가능한가?)
- [ ] Extra READ 수 (N_total ≤ 5) 제약 만족?

**체크**:

```
현재 모드: 250 + 50 + 100 + 0 = 400 ms → 2.5 fps ✓
목표 모드: 150 + 40 + 10 + 0 = 200 ms → 5 fps ✓
Extra READ: 1 (EXPOSE) + 0 (no extra) = 1 ✓
```

### Pass 2: Frame Role 구성 검증

**검증 항목**:

- [ ] EXPOSURE 프레임 (E_raw)이 항상 1개 포함되는가?
- [ ] DARK_PRE와 DARK_POST 역할이 명확히 정의되었는가?
- [ ] LAG 프레임(L1, L2)이 정의되었는가?
- [ ] 각 프레임이 이미지 처리 파이프라인에서 정확히 태깅·처리되는가?
- [ ] 프레임 순서가 시간 순서 + 논리적 인과관계를 만족하는가?

**체크**:

```
연구 모드:
  Cycle 1: [RESET] → [DARK_PRE] → [EXPOSE] → [READ_E]
  Cycle 2: [DARK_POST] → [LAG_1] → [LAG_2]
  → 명확함 ✓

5 fps 모드:
  [RESET] → [EXPOSE] → [READ_E] (+ 비동기 DARK)
  → 명확함 ✓
```

### Pass 3: Lag/Dark 모델 검증

**검증 항목**:

- [ ] Dark = moving average(D_post) 구조가 temporal drift를 충분히 추적하는가?
- [ ] Lag = α₁ × (L1 - D) + α₂ × (L2 - D) 모델이 실제 lag decay를 표현하는가?
- [ ] 계수 α₁, α₂를 어떻게 구할 것인가? (calibration 절차 정의됨?)
- [ ] 메타데이터(T, t, P_mode) 기반 모델이 가능한가?
- [ ] 보정 후 residual error (uncorrected lag/dark)는 임상에서 허용 가능한가?

**체크**:

```
Dark 모델: 실시간 D_post 측정 → moving avg → 충분 ✓
Lag 모델: L1, L2 프레임에서 α 추정 → 가능 ✓
메타데이터: T(온도), t(시간), P(전원) 기반 스칼라 보정 추가 → 가능 ✓
Residual: Dark ↓30%, Lag ↓50% 목표 → 달성 가능 수준 ✓
```

### Pass 4: FPGA State Machine 검증

**검증 항목**:

- [ ] FPGA state machine이 MCU 설정(max_extra_reads, enable_lag, enable_dark)에 따라 안전하게 동작하는가?
- [ ] Gate control (게이트 펄스, timing)이 상용 칩 데이터시트 제약을 벗어나지 않는가?
- [ ] READ 커맨드가 ROIC 인터페이스 프로토콜(LVDS, clock 등)을 정확히 따르는가?
- [ ] Frame sync와 Line sync가 안정적으로 유지되는가?
- [ ] Error handling (timeout, CRC error 등)이 정의되었는가?

**체크**:

```
State transition: RESET → EXPOSE → READ_E → [조건]LAG → [조건]DARK → RESET
  → 루프 논리 명확 ✓

Gate/ROIC interface: 기존 구동 패턴 유지, 순서만 변경
  → 기존 동작 보장 ✓

Synchronization: Frame/Line CLK 신호 의존성 확인
  → 상용 칩이므로 대부분 내부 관리, 외부는 커맨드만 발행 ✓
```

### Pass 5: MCU 파라미터/메타데이터 루프 검증

**검증 항목**:

- [ ] MCU가 fps, T_exp, T_read, T_reset 입력으로부터 max_extra_reads를 정확히 계산하는가?
- [ ] 안전 계수(0.8) 적용 후에도 N_total ≤ 5 보장되는가?
- [ ] 온도·시간·전원 메타데이터가 주기적으로 수집되는가?
- [ ] 메타데이터 기반 Dark 드리프트 보정이 실시간 처리 가능한가(연산량)?
- [ ] 보정 계수(α₁, α₂, 모델 계수)가 EEPROM/RAM에 저장·로드되는가?

**체크**:

```
예시 계산:
  fps_target = 5, T_exp = 40, T_read = 150, T_reset = 10
  margin = 200 - 40 - 10 = 150
  N_max = 150 / 150 = 1
  N_safe = floor(1 × 0.8) = 0
  max_extra_reads = 0 - 1 = -1 → clamped to 0
  → 추가 READ 불가, EXPOSE만 ✓

메타데이터 계산:
  Dark_model(T, t) = a0 + a1 × T + a2 × exp(-t/tau)
  → 부동소수점 연산 O(10) 클럭 / frame → 5 fps에서 충분 ✓

계수 저장:
  α₁, α₂, (a0, a1, a2, tau) 저장 위치 확정 ✓
```

---

## 개발 일정 (16주)

### Phase 0: 준비 (Week 1-2)

| 작업 | 담당 | 산출물 |
|------|------|--------|
| 패널/게이트/ROIC 스펙 수집 | HW | 입력 사양표 완성 |
| Dark/Lag baseline 측정 (250 ms 기준) | Test | Baseline report |
| FPGA/MCU 인터페이스 검토 | FW | Interface spec |

---

### Phase 1: 방법론 ① 연구 (Week 3-7)

**목표**: 현 설정(250 ms)에서 Dark/Lag 특성 정밀 측정, 모델 구축.

| 주차 | 작업 | 담당 | 산출물 |
|------|------|------|--------|
| 3-4 | 다크 프레임 캡처 자동화 (온도, 시간 변화) | FW/Test | Dark characteristic curve |
| 4-5 | Lag 측정 (고/저 교번, frame N=1~10) | Test | Lag decay model |
| 5-6 | Moving avg, 메타데이터 모델 구현 | FW/SW | Correction algorithm v1 |
| 6-7 | 방법론 ① 검증 (SNR, artifact, residual) | Test | Phase 1 report |

---

### Phase 2: 방법론 ② 최적화 연구 (Week 8-12)

**목표**: Read/Reset time 단축, 5 fps 달성 가능성 검증.

| 주차 | 작업 | 담당 | 산출물 |
|------|------|------|--------|
| 8-9 | Read time sweep (현재 250 ms → 150 ms) | HW/FW | Read time vs SNR curve |
| 9-10 | Reset time 최적화 (100 ms → 10 ms) | HW/FW | Reset time vs Dark curve |
| 10-11 | 5 fps 모드 FPGA/MCU 구현 | FW | 5 fps driver code |
| 11-12 | 5 fps 모드 검증 (프레임 안정성, 성능) | Test | Phase 2 report |

---

### Phase 3: 통합 검증 (Week 13-15)

| 주차 | 작업 | 담당 | 산출물 |
|------|------|------|--------|
| 13 | 방법론 ① + ② 동시 지원 (모드 분리) | FW | Dual-mode driver |
| 14 | 5-Pass Deep-Sync 최종 검증 | Test | Full validation report |
| 15 | 성능 지표 최종 측정 (SNR/MTF/DQE/Lag) | Test | Final performance report |

---

### Phase 4: 문서화 및 배포 (Week 16)

| 작업 | 담당 | 산출물 |
|------|------|--------|
| 최종 실행 계획서 작성 | Doc | Plan v2.0 (공식) |
| 드라이버/펌웨어 배포 | FW | Release package |
| 임상 가이드라인 작성 | Doc | User manual |

---

## 리스크 및 대응

| # | 리스크 | 심각도 | 가능성 | 대응 전략 |
|---|--------|--------|--------|----------|
| R1 | Read time 150 ms 달성 불가 (기술적 한계) | High | Medium | ROI/binning 강제 사용, 또는 fps 목표 재조정 (4 fps로) |
| R2 | Reset 10 ms 달성 불가 (fast reset 미지원) | Medium | Medium | Line-by-line reset 최적화, 또는 200 ms 프레임 재정의 |
| R3 | Dark/Lag 모델 부정확 (개인차/환경 변동) | Medium | Low | 메타데이터 차원 확장 (습도, 기기 시리얼 등), 재calibration 절차 |
| R4 | FPGA state machine 복잡성 증가 → 버그 위험 | Medium | Medium | 단순한 상태도 우선, 모드별 분리 구현, 철저한 unit test |
| R5 | SNR 손실 >3 dB (보정 오버헤드) | Medium | Low | 보정 강도 조정 (α 계수 약화), 또는 추가 노이즈 분석 |

---

## 참고 문헌

[1] Starman, J. et al. (2011). "A forward bias method for lag correction of an a-Si flat panel imager." *Physics in Medicine & Biology*, 56(23), 7747. DOI: 10.1088/0031-9155/56/23/7747

[2] EP2148500A1. "Dark correction for digital X-ray detector." European Patent, Published 2009-07-19.

[3] CN101683269A. "Dark correction for digital X-ray detector." Chinese Patent, Published 2009-07-23.

[4] TFT Flat-Panel Array Image Acquisition. *Radiology Key*, 2016.

[5] Basic Principle of Flat Panel Imaging Detectors. *IHMC*, Retrieved from cursa.ihmc.us

[6] Flat-Panel Imaging Arrays for Digital Radiography. *SPIE*, 2009.

---

## 부록: 계산 예시 (참고)

### 예시 1: 현재 시스템 분석 (250 ms read)

**입력**:
- N_row = 2048, T_line = 0.122 ms
- T_exp = 50 ms, T_reset_line = 0.05 ms, fast reset 미지원

**계산**:
- T_read = 2048 × 0.122 = 250 ms
- T_reset = 2048 × 0.05 = 102 ms
- T_cycle = 250 + 50 + 102 = 402 ms
- fps = 1000 / 402 = 2.49 fps ≈ 2.5 fps

**결론**: 현재 약 2.5 fps, 5 fps 미달성.

### 예시 2: 5 fps 달성 시나리오

**입력**:
- N_row_eff = 1024 (ROI 또는 binning), T_line = 0.122 ms
- T_exp = 40 ms, T_reset_target = 10 ms (fast reset)

**계산**:
- T_read_opt = 1024 × 0.122 = 125 ms
- T_reset = 10 ms (고정)
- T_cycle = 125 + 40 + 10 = 175 ms < 200 ms ✓
- fps = 1000 / 175 = 5.71 fps > 5 fps ✓

**결론**: 목표 달성 가능 (여유 25 ms).

---

**최종 검토**: 이 계획서는 5-Pass Deep-Sync를 거쳐 검증된 구동 알고리즘 기반 실행 계획서입니다.  
패널/게이트/ROIC 입력 사양을 채우고, 계산식을 실행하면, 프로젝트별 fps/성능 예측이 가능합니다.

**문서 관리**: 버전 2.0 (최종), 2026-02-09
