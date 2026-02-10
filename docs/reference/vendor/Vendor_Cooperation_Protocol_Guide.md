# a-Si FPD 벤더 협조 요청서 및 측정 프로토콜
## 현실적 확보 가능 범위에서의 기술 정보 수집 전략

**작성일**: 2026년 2월 9일  
**버전**: 1.0 (최종)  
**대상**: Panel 제조사 기술 지원팀  
**목적**: Idle 상태 관리 및 구동 알고리즘 최적화를 위한 정보 수집

---

## 1. 벤더 협조 전략

### 1.1 요청 우선순위

| 우선순위 | 범주 | 특성 | 기대 응답률 |
|---------|------|------|-----------|
| **필수** | 공개 스펙 | 데이터시트/애플리케이션 노트 수준 | 95% |
| **중요** | 운용 권장사항 | "이 조건에서 사용하세요" 수준 | 60% |
| **참고** | 내부 평가 | 대략적인 수치/경향 | 30% |
| **포기** | IP/공정 정보 | 제조공정, 디바이스 파라미터 | < 5% |

### 1.2 요청 접근 방식

**권장**:
- 명확하고 구체적인 기술 질문 (Open-ended 아님)
- "왜 필요한가"를 명시
- 기술 수준을 맞춰 작성 (marketing 담당이 아니라 engineering)
- NDA 협의 (필요시 명시적 제안)

**피할 것**:
- "모든 스펙을 알려 달라" (너무 광범위)
- 부정확한 설명으로 신뢰 손상
- Reverse engineering 의도 표현
- 경쟁업체 정보와 비교 요청

---

## 2. 필수 요청 항목 (1순위)

### 2.1 항목: 타이밍 및 모드 스펙

#### 요청 내용

> **제목**: Full Read Mode의 최대 프레임 레이트 및 최소 라인 타이밍 정보 요청
>
> **내용**:
> - 현재 시스템에서 250 ms/frame 기준 구동 중이며, 이를 단축 가능한지 기술 평가 중입니다.
> - 다음 정보를 알려 주실 수 있을까요?
>   1. **Full resolution read 모드에서 최대 fps 또는 minimum frame time (ms)**
>      - "Spec 상" 값과 "실제 테스트됨" 값 (있으면)
>   2. **지원하는 ROI 또는 binning 모드**
>      - 2×2 binning 모드의 typical frame time
>      - ROI 해상도별 read time (예: 1024×1024)
>   3. **Minimum line time 또는 row clock 상한 주파수**
>      - 예: "line time ≥ 10 µs" 또는 "row clock ≤ 10 MHz"
>
> **이유**: 250 ms를 150 ms로 단축할 때 패널이 안정적으로 지원할 수 있는 최대 속도 파악 필요

#### 예상 응답 형식

```
Full Resolution (2048 × 2048):
- Typical frame time: 200 ms (recommended)
- Minimum frame time: 150 ms (tested, margin 15%)
- Maximum line clock: 12 MHz (corresponding to 167 ns line time)

2×2 Binning (1024 × 1024):
- Typical frame time: 50 ms
- Maximum fps: 20 fps (verified)

ROI Support:
- Arbitrary ROI not supported
- Pre-defined ROI only: Full, Half-H, Half-V, Quarter
```

#### 도움이 안 될 경우의 대체 방법

1. **Datasheet 확인**: 보통 "read time" 또는 "frame rate" 스펙 있음
2. **자체 측정**: 실제 패널로 line time sweep 실험
3. **Reference 시스템**: 같은 패널 쓰는 기존 시스템 참조

---

### 2.2 항목: Dark Signal vs Temperature 기본 데이터

#### 요청 내용

> **제목**: 온도에 따른 Dark Signal 특성 정보 요청
>
> **내용**:
> - Idle 상태 관리를 위해 온도별 dark level 변화를 모델링하고 있습니다.
> - 다음 정보를 주실 수 있을까요?
>   1. **Typical dark signal vs temperature (간단한 표 또는 그래프)**
>      - 측정 조건: Integration time = 100 ms 고정
>      - 온도: 15, 20, 25, 30, 35, 40°C
>      - 출력 형식: [DN] (digital number) 또는 [e⁻/pixel/s]
>      - "Typical" 값으로 충분 (σ 불필요)
>   2. **보장된 동작 온도 범위**
>      - "이 범위 내에서 image quality가 guaranteed인가?"
>   3. **온도에 따른 성능 저하의 특성**
>      - 지수 증가인지, 선형인지 (대략적으로)
>
> **이유**: 우리 시스템에서 온도 기반 dark 보정 알고리즘을 설계 중

#### 예상 응답 형식

```
Dark Signal vs Temperature (Integration time = 100 ms)

Temperature (°C) | Typical Dark (DN) | Typical Dark (e⁻/pixel/s)
15               | 2480              | 0.12
20               | 2510              | 0.15
25               | 2550              | 0.20
30               | 2610              | 0.28
35               | 2680              | 0.40
40               | 2750              | 0.58

Note: Dark level increases ~40 DN per 5°C (linear approximation)

Guaranteed operating temperature: 10~40°C
- Image quality within specified ranges: 15~35°C
- Performance degraded outside range (not recommended)
```

#### 도움이 안 될 경우의 대체 방법

1. **자체 측정**: 온도챔버에서 여러 온도에서 dark frame 촬영
2. **Datasheet**: "Dark current vs temperature" 그래프 찾기
3. **문헌**: 같은 a-Si 기술 사용한 다른 시스템 참조

---

### 2.3 항목: Standby/Idle 운용 권장사항

#### 요청 내용

> **제목**: Stand-by 모드에서의 Bias 설정 권장사항 요청
>
> **내용**:
> - 전원 인가 상태에서 오랜 기간 readout이 없을 때(Stand-by) dark 증가와 열화를 최소화하기 위한 권장 구동 방식이 있는지 알고 싶습니다.
> - Application note 수준의 정보면 충분합니다:
>   1. **Stand-by 전용 모드 또는 바이어스 세팅**
>      - 예: "Gate를 OV, Column을 -0.2V로 유지하면 leakage가 최소"
>      - 또는: "이 레지스터 비트를 set하면 low-leakage mode 진입"
>   2. **Stand-by 중 주기적 scan이 필요한가?**
>      - "아무 조치 없으면 몇 시간까지 안전한가?"
>      - "주기적 refresh가 필요하면 권장 주기는?"
>   3. **Stand-by 상태에서 특정 신호는 반드시 끄야 하는가?**
>      - 예: "HV bias는 반드시 OFF"
>
> **이유**: 수 시간 idle이 흔한 의료 영상 시스템에서 dark drift를 관리하기 위함

#### 예상 응답 형식

```
Standard Standby Mode (FPD datasheet Section 7.2):

Gate voltage: 0 V (OFF state)
Storage node: Column reference voltage (-1.0 V typical)

For extended idle (> 30 minutes):
1. Recommended: Switch to Low-Power mode
   - Gate: -2 V (deeper OFF)
   - Column clamp: -0.2 V (reduced reverse bias on PD)
   - Data: Periodic dummy readout every 5 minutes (optional)

2. Typical dark increase in standby:
   - First 1 hour: ~20 DN drift
   - 1-4 hours: ~40 DN total (sub-linear)
   - Plateaus after 4 hours

Application Note AN-152 (Low-Power Operation) provides more details.
```

#### 도움이 안 될 경우의 대체 방법

1. **Datasheet 상세 읽기**: "Recommended Operating Conditions" 섹션
2. **Technical support**: 직접 기술 유선 또는 이메일
3. **자체 최적화**: 여러 gate/column 조합을 테스트해서 최적값 찾기

---

### 2.4 항목: Shielded/Optical-black Pixel 정보

#### 요청 내용

> **제목**: 광 차폐 픽셀(Optical Black / Shielded Pixel) 존재 여부 및 위치 정보
>
> **내용**:
> - 각 프레임에서 블라인드 픽셀(X-ray 차폐 영역)을 사용해 실시간 dark offset 추정을 하려고 합니다.
> - 다음을 알려 주실 수 있을까요?
>   1. **광 차폐 픽셀이 존재하는가?**
>   2. 있다면:
>      - 위치 (상단/하단/좌측/우측, row/column 번호 범위)
>      - 크기 (수백 개 픽셀 정도면 충분)
>   3. **이 픽셀들이 일반 data channel을 통해 읽혀지는가?**
>      - (특별한 조치 없이 일반 read sequence에 포함되는지)
>
> **이유**: blind pixel 평균값이 frame-level dark offset 추정에 사용되어, idle 이후 dark 보정 정확도 향상 가능

#### 예상 응답 형식

```
Optical Black Pixel Configuration:

Available: Yes

Locations:
- Top margin: Rows 0-1 (entire width)
- Bottom margin: Rows 2046-2047 (entire width)
- Left margin: Columns 0 (entire height)
- Right margin: Columns 2047 (entire height)

Total optical black pixels:
- Top/Bottom: 2 rows × 2048 cols = 4096 pixels
- Left/Right: 2046 rows × 2 cols = 4092 pixels
- Total: ~8000 pixels

Integration:
- These pixels are included in standard full-frame readout
- No special addressing needed
- Recommended averaging method: mean of all optical black pixels
```

#### 도움이 안 될 경우의 대체 방법

1. **Datasheet 그림**: 보통 pixel map이 있음
2. **자체 확인**: 패널의 가장자리 픽셀을 직접 읽어서 X-ray 차폐 여부 확인
3. **제조사 문의**: 기술 지원팀에 이미지 샘플 요청

---

## 3. 중요 참고 항목 (2순위 - 시도 가능)

### 3.1 장시간 Idle 후 Dark Drift (Optional)

#### 요청 내용

> 문의: "실온에서 panel이 power ON, readout 없이 idle 상태일 때, dark level이 시간에 따라 어떻게 변하는지에 대한 내부 평가 결과가 있는지?"
>
> 예시 답변 형식: "2시간 idle에서 평균 dark가 약 2%~3% 증가하고, 이후 plateau"

#### 대체 방법

자체 측정으로 충분히 가능:

```python
# 온도 제어 환경에서 수행
import numpy as np
import time

temperatures = [20, 25, 30, 35]  # °C

for T in temperatures:
    set_temp(T)
    wait(stabilization=30*60)  # 30분 기준 안정화
    
    dark_values = []
    for idle_time in [0, 30, 60, 120, 240]:  # minutes
        capture_dark_frame()
        dark_values.append(mean_pixel_value)
        
        # idle_time이 지날 때까지 기다림
        wait(idle_time * 60)
```

---

### 3.2 Lag Typical 값

#### 요청 내용

> "High/Low 교번 패턴에서 Lag 1, Lag 2의 typical 값(몇 % 정도인지)이 있으면 참고하고 싶습니다."

#### 대체 방법

자체 lag 측정 프로토콜로 충분:

```
H (high exposure) → L (no exposure) → L → L → ...
measure: residual signal in L frames as % of H
```

---

## 4. 현실적으로 어렵거나 불필요한 항목

### 4.1 피해야 할 요청

**← 이 정도는 요청하지 말 것:**

- TFT leakage current vs V_GS, V_DS (상세 I-V 곡선)
- V_TH 분포, 임계 전압 바이어스 의존성
- Gate oxide 두께, 산화막 정보
- TFT 채널 길이/폭, 이동도 µ
- Defect state 에너지, trap 분포
- 공정 라인 yield, 불량률
- 상세한 회로도 또는 트랜지스터 레벨 설계
- "Reverse engineering" 의심 요청

**이유**:
- IP 보호 영역
- NDA 급 정보
- 경쟁 회사와 구분되는 핵심 공정
- 벤더가 절대 줄 수 없음

### 4.2 필요 없는 정보

**← 이건 없어도 우리가 직접 측정 가능:**

- 매우 상세한 dark current 히스토그램 (histogram)
- Pixel-to-pixel 편차 (σ)
- 각 온도에서의 gain shift (우리 시스템에 따라 다름)
- 특정 X-ray 스펙트럼에 대한 효율
- 광신호 대비 dark 비율 (우리 scintillator/설정에 따라 다름)

---

## 5. 요청서 템플릿 (실제 사용)

### 5.1 공식 요청서 예시

```
─────────────────────────────────────────────

To: [Vendor] Technical Support
From: [Your Company] Engineering Team
Subject: Technical Specification Request – FPD Panel Model [XXX-XXXX]
Date: 2026-02-09

Project: Digital Radiography System Optimization
– Idle State Management & Algorithm Development

─────────────────────────────────────────────

Dear Technical Support Team,

We are developing an optimized driving algorithm for our DR system
using your FPD panel [model]. To finalize the design, we would
appreciate the following technical information:

[문항 1-4 (필수) 상세 내용 삽입]

Appendix: Desired Response Format
[각 항목별 예상 형식 제시]

Thank you for your support. We look forward to collaborating.

Best regards,
[Your name]
[Company]
[Contact]

─────────────────────────────────────────────
```

### 5.2 응답 추적 및 기록

요청 후:

```
[요청일] → [응답 기한] (보통 2주 내) → [수신] → [정리]

정리 항목:
- 도착 날짜
- 담당자
- 응답 형식 (이메일/문서/전화)
- 주요 내용 요약
- 추가 질문 필요 여부
- 정보 신뢰도 평가 (★★★★★)
```

---

## 6. 실측 프로토콜 (벤더 응답 부족 시 자체 수행)

### 6.1 Dark Drift Rate 측정 (필수)

**목표**: 온도별 drift rate LUT 수립

**필요 장비**:
- 온도 제어 챔버 또는 항온실 (±2°C 정확도)
- 온습도 센서
- 데이터 로깅 소프트웨어

**절차**:

```
1. Panel power ON
2. 15분 thermal stabilization
3. Temperature set to T ∈ {15, 20, 25, 30, 35, 40}°C
4. For each temperature:
   a. t=0 min: Capture DARK frame (D0)
   b. t=30 min: Capture DARK frame (no readout during idle)
   c. t=60 min: DARK frame
   d. t=120 min: DARK frame
   e. t=240 min: DARK frame
5. Calculate: k(T) = [D(t) - D0] / t
6. Plot k(T) vs T, extract E_A

Result: Drift rate table
T (°C) | k(T) (DN/min) | t_max @50DN
20     | 0.75          | 67 min
25     | 1.0           | 50 min
30     | 1.8           | 28 min
...
```

**소요 시간**: 각 온도 ~5시간 × 6 온도 = 30시간 (분산 가능)

**출력**: Dark LUT 2D 테이블

---

### 6.2 V_TH Drift 모니터링 (선택)

**목표**: Pixel gain 변화 추적

**절차**:

```
1. Capture reference pattern (fixed-brightness image)
   → Signal V_out_1, gain G_1
2. Idle 4 hours (DC bias maintained, no readout)
3. Capture same reference pattern again
   → Signal V_out_2, gain G_2
4. ΔGain = (G_2 - G_1) / G_1 × 100%
5. Record per-pixel gain changes, σ, distribution
```

**소요 시간**: 1회 ~4.5시간

**출력**: V_TH drift 특성 (온도/시간)

---

### 6.3 Warm-up 시퀀스 검증 (필수)

**목표**: Idle 후 첫 프레임 dark 특성 파악

**절차**:

```
1. Panel normal state: Capture baseline dark frame (D_baseline)
2. Idle 30 min (L2 low-bias mode)
3. Immediately after:
   a. RESET → 100 ms integration → READ (Dark_warmup)
   b. RESET → 100 ms integration → READ (Dark_2nd)
4. Compare:
   - Dark_warmup vs D_baseline (상승량)
   - Dark_warmup vs Dark_2nd (수렴)
5. Repeat for idle = 1hr, 2hr, 4hr
```

**소요 시간**: 1회 ~0.5시간, 여러 idle time은 병렬 가능

**출력**: Idle time별 warm-up dark 보정 필요량

---

## 7. 정보 수집 일정 및 체크리스트

### 7.1 타임라인

```
Week 1: 벤더 요청서 작성 및 발송 (금요일)
        → 벤더 기술팀 할당 및 응답 시작

Week 2-3: 벤더 응답 대기 (보통 2주)
         → 부분 응답 수신 시 추가 질문

Week 4: 벤더 응답 부족 항목 자체 측정 시작
        (온도 제어 환경 예약 및 준비)

Week 5-7: 자체 측정 진행 (온도별 dark drift 등)

Week 8: 정보 통합, LUT 생성, 구동 알고리즘 최종화
```

### 7.2 체크리스트

- [ ] 벤더 기술 문의 연락처 확보
- [ ] 필수 4개 항목 요청서 작성 완료
- [ ] 요청서 발송 및 응답 기한 기록
- [ ] 벤더 응답 수신 (1순위 항목)
- [ ] 응답 내용 정리 및 기술팀 공유
- [ ] 부족한 항목 확인 → 자체 측정 계획 수립
- [ ] 온도 챔버 예약 또는 항온실 확보
- [ ] Dark LUT 측정 완료
- [ ] 모든 정보 통합 → 최종 구동 알고리즘 설계

---

## 최종 노트

**벤더와의 협력 성공 요인**:

1. **명확한 요청**: "무엇을" "왜" "어떤 형식으로"를 명시
2. **실현성**: 기술팀이 "할 수 있는" 수준의 정보 요청
3. **상호 이익**: 우리 시스템의 성공이 벤더의 제품 가치 증대
4. **신뢰 구축**: 진정한 기술 문제 해결 의도 표현
5. **후속 피드백**: "이 정보가 도움이 되었다"는 피드백 공유

**정보 부족 대비**:

- 대부분의 항목은 **자체 측정으로 충분히 보완 가능**
- 벤더 정보 ≈ 40% + 자체 측정 ≈ 60%의 혼합이 현실적
- 중요한 것은 **온도별 drift rate LUT** (벤더 정보 또는 자체 측정)

---

**작성**: 2026-02-09  
**버전**: 1.0 (최종)  
**상태**: 즉시 사용 가능

