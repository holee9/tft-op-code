# 제품 개요 (Product Overview)

## 프로젝트 정보

**프로젝트명**: TFT-Leak-plan
**작성일**: 2026-02-10
**버전**: 3.1 (수정)

---

## 1. 미션 및 비전 (Mission & Vision)

### 미션 (Mission)

a-Si(수소화 비정질 실리콘) TFT 기반 Flat Panel Detector(FPD)에서 발생하는 Idle 상태 누설 전류 문제를 소프트웨어 정의 구동 알고리즘으로 해결하여, 의료 영상의 품질 일관성을 확보하고 시스템 안정성을 향상시킨다.

### 비전 (Vision)

하드웨어 설계 변경 없이 구동 알고리즘 최적화만으로 장시간 대기 후에도 임상적으로 유효한 화질을 제공하는 표준 솔루션을 구축하여, a-Si TFT 기반 디지털 영상 시스템의 성능 한계를 극복한다.

---

## 2. 핵심 사용자 및 사용 시나리오 (Key Users & Use Cases)

### 주요 사용자 (User Personas)

| 사용자 유형 | 역할 | 주요 요구사항 |
|------------|------|-------------|
| **시스템 엔지니어** | FPGA/MCU/소프트웨어 개발 | 구현 가능한 알고리즘 명세, 타이밍 제어 로직 |
| **품질 관리자** | 테스트 및 검증 | 성능 지표(KPI), 검증 프로토콜 |
| **필드 엔지니어** | 현장 기술 지원 | 문제 해결 가이드, 트러블슈팅 |
| **연구원** | 물리적 현상 분석 | 누설 메커니즘 이해, 모델링 데이터 |

### 핵심 사용 시나리오 (Use Cases)

#### 시나리오 1: 장시간 대기 후 촬영
- **상황**: 병원 당직 상황에서 1~2시간 대기 후 응급 촬영
- **문제**: 첫 번째 영상의 Dark Level 상승으로 화질 저하
- **해결**: Warm-up 프레임 처리 및 2D LUT 보정으로 잔류 Dark 100 DN 이하 제어

#### 시나리오 2: 온도 변동 환경 운용
- **상황**: 일교차가 큰 환경(20~35°C)에서 연속 촬영
- **문제**: 온도에 따른 비선형적 Dark 상승
- **해결**: 온도 기반 t_max(T) 실시간 계산 및 동적 모드 전환

#### 시나리오 3: 연구용 고정밀 모드
- **상황**: 250ms integration 기반 연구용 촬영
- **문제**: 정밀 보정 필요
- **해결**: Dark/Lag frame 주기적 캡처로 30%/50% 보정 효율 달성

#### 시나리오 4: 임상용 고속 모드
- **상황**: 5fps 실시간 임상 촬영
- **문제**: 비동기 Dark 캡처 제약
- **해결**: Blind Pixel 활용 프레임 레벨 보정

---

## 3. 핵심 문제 및 해결 방안 (Core Problems & Solutions)

### 문제 1: Dark Level 상승 (Leakage-Induced Dark Signal)

**물리적 원인**:
- TFT OFF 상태에서의 Subthreshold Leakage
- Deep Defect State에서의 SRH Generation Current
- 온도 10°C 상승 시 약 2배 증가 (지수 의존성)

**해결 방안**:
- 3단계 Idle 모드 계층화 (L1/L2/L3)
- Quasi Zero-bias 기술 (V_PD ≈ -0.2V)
- 주기적 Dummy Reset (30~60초 주기)
- **효과**: Dark 상승 20~30% 억제

### 문제 2: V_TH 드리프트 (Gain 불안정)

**물리적 원인**:
- 장시간 DC 바이어스로 인한 Charge Trapping
- 픽셀별 Gain 편차 (7~8%/4hr)

**해결 방안**:
- 주기적 Dummy Scan으로 V_TH 드리프트 방지
- Stretched Exponential Model 기반 예측
- 온도 적응형 Lag 보정

### 문제 3: 온도 의존성

**특성**:
- Dark Level: ±4 DN/°C
- Drift Rate: 2×/10°C (지수)

**해결 방안**:
- Arrhenius 모델 기반 온도 보상 (E_A = 0.45 eV)
- 2차원 Dark LUT (온도 × Idle 시간)
- 실시간 Blind Pixel 오프셋 추적

---

## 4. 주요 기능 및 특징 (Key Features)

### 4.1 3단계 Idle 관리 전략

| 모드 | 진입 조건 | 바이어스 | 스캔 | 허용 시간 (@25°C) |
|------|----------|---------|------|------------------|
| **L1: Normal Idle** | 대기 < 10분 | Normal (-1.5V) | 없음 | ~50분 |
| **L2: Low-bias Idle** | 10~100분 | Quasi Zero (-0.2V) | 주기적 (60s) | ~200분 |
| **L3: Deep Sleep** | > 100분 | Power Down | 없음 | 무한 |

### 4.2 구동 알고리즘 모드

#### 방법론 ①: 연구용 모드 (250ms)
- Dark/Lag frame 주기적 캡처
- Dark 보정 효율: 30%
- Lag 보정 효율: 50%

#### 방법론 ②: 임상용 모드 (5fps)
- 비동기 Dark 캡처
- Dark 보정 효율: 20%
- Lag 보정 효율: 30%
- Warm-up 오버헤드: < 500ms

### 4.3 보정 알고리즘

1. **2D Dark LUT 보정**: 온도와 Idle 시간을 축으로 하는 양선형 인터폴레이션
2. **Blind Pixel 보정**: 패널 주변 광 차폐 픽셀 활용 실시간 오프셋 추적
3. **적응형 Lag 보정**: 온도/Idle 이력에 따른 계수 동적 적용

---

## 5. 성능 목표 (Performance Goals)

### 주요 성능 지표 (KPI)

| 지표 | 방법론 ① (250ms) | 방법론 ② (5fps) |
|------|-----------------|-----------------|
| Dark 보정 효율 | 30% | 20% |
| Lag 보정 효율 | 50% | 30% |
| 첫 프레임 잔류 Dark | < 100 DN | < 100 DN |
| Warm-up 오버헤드 | N/A | < 500 ms |
| Idle 허용 시간 연장 | 2~4배 | 2~4배 |

### 온도별 허용 Idle 시간

| 온도 (°C) | Drift Rate (DN/min) | t_max @ 50DN |
|----------|---------------------|--------------|
| 20 | 0.75 | 67분 |
| 25 | 1.0 | 50분 |
| 30 | 1.8 | 28분 |
| 35 | 3.2 | 16분 |
| 40 | 5.8 | 9분 |

---

## 6. 차별화 요소 (Differentiation)

### 기존 접근법 vs 제안 방식

| 구분 | 기존 방식 | 제안 방식 |
|------|---------|---------|
| Idle 관리 | 단일 모드 | 3단계 계층화 (L1/L2/L3) |
| Dark 보정 | 1D LUT (온도만) | 2D LUT (온도 × 시간) |
| 바이어스 제어 | 고정 | 동적 MUX 제어 |
| VTH 드리프트 | 방치 | 주기적 Dummy Reset |
| Warm-up | 없음 | 필수 프레임 처리 |

### 기술적 우위

1. **소프트웨어 정의 구동**: 하드웨어 설계 변경 없이 펌웨어만으로 구현 가능
2. **물리 기반 모델링**: Arrhenius 모델과 SRH 메커니즘 기반 정량적 예측
3. **실시간 적응**: 온도 폴링 (1Hz) 및 t_max(T) 동적 계산
4. **벤더 무관성**: 자체 측정 프로토콜로 어떤 패널에도 적용 가능

---

## 7. 비즈니스 목표 (Business Goals)

### 단기 목표 (8.5주)

- [ ] FPGA 바이어스 MUX 제어 로직 구현
- [ ] MCU Arrhenius 모델 및 상태 머신 개발
- [ ] 호스트 SW 2D LUT 및 보정 알고리즘 완성
- [ ] 통합 시스템 테스트 및 KPI 검증

### 중기 목표

- [ ] 온도별 Drift Rate LUT 측정 완료
- [ ] 벤더 협조 필수 4개 항목 확보
- [ ] 8시간 연속 운전 안정성 검증

### 장기 목표

- [ ] a-Si TFT FPD 표준 Idle 관리 솔루션 자리매김
- [ ] 임상/연구 이중 모드 표준 구동 방식론 정립
- [ ] 타 패널 타입으로 기술 확장 가능성 검토

---

## 8. 제약 사항 및 위험 요소 (Constraints & Risks)

### 기술적 제약

- a-Si TFT의 본질적인 물리적 한계 (높은 결함 밀도)
- 온도 제어 시스템 의존성 (권장 ±3°C)
- 벤더 정보 확보 불확실성

### 완화 전략

- 자체 측정 프로토콜 수립 (벤더 정보 부족 시 대비)
- 3단계 계층화로 다양한 운용 환경 지원
- 보정 알고리즘 중심 다중 레이어 접근

---

## 9. 프로젝트 산출물 (Deliverables)

### 기술 문서

1. `aSi_TFT_Leakage_Implementation_Plan_v3_Final.md` - 통합 구현 계획서
2. `aSi_TFT_Drive_Algorithm_ExecutionPlan_v2_2.md` - 구동 알고리즘 실행 계획
3. `aSi_TFT_Idle_Strategy_Report_v3_Final.md` - Idle 상태 관리 전략 보고서
4. `nblm_연구논문_초안.md` - 연구 논문 초안
5. `Vendor_Cooperation_Protocol_Guide.md` - 벤더 협조 가이드

### 구현 산출물

- FPGA RTL 코드 (바이어스 MUX, Dummy Scan 엔진)
- i.MX8 Plus 펌웨어 (.NET 기반 애플리케이션 + Cortex-M4 RTOS)
- 호스트 SW 라이브러리 (2D LUT, 보정 알고리즘) - **향후 별도 프로젝트로 구현**

---

## 10. 참고 문헌 (References)

1. Tredwell, T. J., et al. (2009). "Flat-Panel Imaging Arrays for Digital Radiography." SPIE Proceedings.
2. Starman, J., et al. (2011). "A forward bias method for lag correction of an a-Si flat panel imager." Physics in Medicine & Biology.
3. Chang, J. H., et al. (2007). "Temperature Dependence of Leakage Current in Segmented a-Si:H n-i-p Photodiodes." IEEE TED.

---

**문서 버전**: 3.1
**최종 수정일**: 2026-02-10
**상태**: 수정 완료
**변경 내역**:
- 프로젝트 산출물: i.MX8 Plus 플랫폼 반영
- 호스트 SW: 향후 별도 프로젝트로 구현 예정 표기 추가
