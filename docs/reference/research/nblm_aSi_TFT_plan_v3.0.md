a-Si TFT 누설 전류 관리 및 Idle 상태 대응 통합 구현 계획서 (v3_final)

1. 문서 개요 및 개정 이력

본 문서는 a-Si(수소화 비정질 규소) TFT 기반 Flat Panel Detector(FPD)의 Idle 상태에서 발생하는 물리적 열화(누설 전류 및 임계 전압 드리프트)를 제어하고, 영상 품질의 일관성을 확보하기 위한 시스템 통합 구동 및 보정 전략을 수립한다. v3.0에서는 v2.1의 기본 모델을 확장하여 3단계 Idle 관리 전략과 온도 기반 동적 보정 알고리즘을 구체화하였다.

1.1 개정 이력 요약

버전	주요 변경 사항	비고
v2.1	Dark/Lag 기본 특성 분석 및 구동 알고리즘 초안 수립	2025년 작성
v3.0	- Idle 모드 3단계(L1/L2/L3) 대응 전략 통합 및 억제 기술 추가<br>- Arrhenius 모델 고도화 (E_A=0.45eV, n \approx 1.5 \sim 2 반영)<br>- FPGA MUX 바이어스 제어 및 MCU t_{max}(T) 실시간 계산 로직 확정<br>- 벤더 협조 4대 필수 항목 정의 및 자체 측정 프로토콜 수립	최종안


--------------------------------------------------------------------------------


2. 물리적 현상 분석 및 모델링 (Phenomenon Analysis)

2.1 a-Si TFT 누설 메커니즘

a-Si:H TFT는 비정질 구조의 높은 결함 밀도로 인해 다음과 같은 비선형 누설 경로를 가진다.

* Subthreshold Leakage: TFT OFF 상태에서 Gate-Drain 전기장에 의해 발생하며, I_{leak} \propto \exp(qV_{GS}/nkT) 모델을 따른다. 이때 Ideality factor(n)는 비정질 실리콘의 특성에 따라 1.5 \sim 2 범위를 가진다.
* Deep State Generation Current: 밴드갭 중앙의 깊은 결함 상태(Deep Trap)에서 Shockley-Read-Hall(SRH) 메커니즘에 의해 발생한다. 이는 T^{3/2} \exp(-E_A/kT) 항을 포함하여 온도에 극도로 민감하게 반응한다.

2.2 온도 의존성 및 드리프트 모델

TFT 누설에 의한 Dark Drift Rate(k(T))는 Arrhenius 모델로 정량화하며, Deep State 특성을 반영한 활성화 에너지(E_A) 약 0.45 \, eV를 적용한다. k(T) = a \exp\left(-\frac{E_A}{k_B T}\right) \quad (\text{단, } a \approx 0.05)

* 지수적 특성: 온도 10^\circ C 상승 시 누설 전류가 2배 증가하며, 이는 약 4 \, DN/^\circ C의 Dark Level 상승으로 직결된다.
* Plateau 현상: 장시간 Idle 시 누설률은 초기 급격한 Decay 후 일정 수준에서 안정화되는 Plateau 거동을 보인다.

2.3 이미지 프롬프트

[IMAGE PROMPT: A detailed technical cross-section diagram of an a-Si:H TFT pixel structure highlighting the leakage paths (Subthreshold and SRH generation) within the amorphous silicon layer, with heat maps indicating temperature dependency.]



--------------------------------------------------------------------------------


3. 통합 Idle 상태 관리 전략 (Integrated Idle Strategy)

3.1 Idle 모드 계층화 (L1/L2/L3)

대기 시간에 따른 열화 억제를 위해 바이어스(V_{PD})와 스캔 주기를 계층화한다.

구분	L1: Normal Idle	L2: Low-bias Idle	L3: Deep Sleep
진입 조건	대기 < 10분	10분 \le 대기 < 100분	대기 \ge 100분
바이어스 설정	Normal (-1.5V)	Quasi Zero-bias (-0.2V)	Power Down
스캔 주기	없음	주기적 Dummy Reset (30~60s)	없음
허용 시간(t_{max})	~50분 (@25^\circ C)	~200분 (@25^\circ C)	거의 무한

3.2 상태 전환도

stateDiagram-v2
    [*] --> L1: Power On / Stabilization
    L1 --> L2: t_idle > 10 min
    L2 --> L3: t_idle > 100 min
    L2 --> L1: Readout Request
    L3 --> L1: Wake-up / Warm-up Frame
    L1 --> [*]: Power Off


3.3 주요 억제 기술 구현 원리

* Quasi Zero-bias (V_{PD\_idle} \approx -0.2V): 저장 노드(Storage Node)와 포토다이오드 양단의 전위차를 최소화하여 Dark 전류 생성을 물리적으로 차단한다. 이를 통해 Dark 상승률을 20~30% 감소시킨다.
* 주기적 Dummy Reset (Standby Scan): T_{dummy} 주기로 저장 노드를 강제 리셋하여 Dark 누적 시간을 제한한다.
  * 모델: D(t) = D_0 + k(T) \cdot \min(t \mod T_{dummy}, T_{dummy})


--------------------------------------------------------------------------------


4. 시스템 아키텍처 및 구동 알고리즘 (Implementation Architecture)

4.1 FPGA 타이밍 제어

* 바이어스 MUX 제어: MCU로부터 2-bit MUX 제어 신호를 수신하여 AFE(Analog Front End)의 V_{PD}를 Normal(-1.5V)과 Idle(-0.2V) 전위로 즉시 전환하는 로직을 구현하라.
* Dummy Scan 엔진: L2 모드 진입 시 모든 Row에 대해 순차적 Reset 펄스를 인가하되, 데이터 Read 과정은 생략하여 전력 소모를 최소화하고 V_{TH} 드리프트를 방지하라.

4.2 MCU 펌웨어 로직

* 온도 폴링 및 계산: 온도 센서를 1 \, Hz 주기로 폴링하고, 현재 온도(T)를 기반으로 허용 Idle 한계 시간 t_{max}(T) = \Delta D_{max} / k(T)를 실시간 갱신하라. (단, \Delta D_{max} = 50 \, DN)
* 상태 머신 관리: 대기 타이머와 온도 임계치를 비교하여 L1/L2/L3 모드 전환 신호를 FPGA에 전달하고, 각 상태 전이 시 타임스탬프를 기록하라.

4.3 호스트 소프트웨어 보정

* 2D Dark LUT 인터폴레이션: 온도(T)와 Idle 시간(\Delta t_{idle})을 축으로 하는 2차원 룩업 테이블을 구축하라. 런타임 시 메타데이터 헤더에서 수신한 T, \Delta t 정보를 바탕으로 양선형 인터폴레이션(Bilinear Interpolation)을 수행하여 Dark 오프셋을 추정하라.
* Blind Pixel 실시간 추적: 패널 주변부의 Shielded Pixels(Optical Black) 데이터를 활용하여 LUT가 포착하지 못한 잔여 프레임 레벨 오프셋 드리프트를 매 프레임 추적 및 제거하라.
* 적응형 Lag 보정: T와 \Delta t_{idle} 이력에 따라 Lag 보정 계수(\alpha_1, \alpha_2)를 동적으로 가변 적용하라.


--------------------------------------------------------------------------------


5. 성능 목표 (KPI) 및 검증 계획

5.1 주요 성능 지표 (KPI)

지표	방법론 ① (250ms)	방법론 ② (5fps)
Dark 보정 효율	30%	20%
Lag 보정 효율	50%	30%
First Frame Residual Dark	< 100 DN	< 100 DN
Warm-up 오버헤드	N/A	< 500 ms (2 frames)

5.2 테스트 시나리오

1. 8시간 연속 운전 및 Plateau 검증: 상온 대기 환경에서 Dark Drift가 모델링된 Plateau 지점에 도달하고 안정화되는지 확인하라.
2. 온도 스텝 응답: 환경 온도를 \pm 5^\circ C 급변시킨 후 10초 이내에 MCU가 t_{max}를 재계산하고 모드를 적절히 전환하는지 검증하라.
3. Warm-up 프레임 품질: L2/L3 모드 탈출 후 첫 프레임(Warm-up/Calibration Frame)을 폐기 또는 보정용으로 사용하여, 실제 첫 번째 임상 영상의 잔류 Dark가 100 DN 이하인지 확인하라.


--------------------------------------------------------------------------------


6. 구현 로드맵 및 리소스 할당

6.1 WBS 및 일정 (총 8.5주)

* 1~2주: FPGA 2-bit MUX 바이어스 제어 및 Dummy Scan 로직 개발.
* 3~4주: MCU Arrhenius 모델 펌웨어 및 t_{max} 상태 머신 구현.
* 5~6주: 호스트 SW 2D LUT 인터폴레이션 및 적응형 Lag 알고리즘 개발.
* 7~8주: 통합 시스템 테스트 (모드 전환 시간 < 1s 보장 및 KPI 검증).
* 8.5주: 최종 Calibration 데이터 업로드 및 배포.

6.2 리소스 구성 (총 4.6 FTE)

* FPGA (1.0): RTL 설계 및 타이밍 클로징.
* MCU/FW (1.0): 온도 모델링 및 상태 관리.
* Host SW (1.0): 보정 알고리즘 및 메타데이터 처리.
* QA/Test (0.8): 신뢰성 및 KPI 검증.
* Measurement (0.5): 챔버 기반 기초 데이터 수집.


--------------------------------------------------------------------------------


7. 벤더 협조 프로토콜 및 실측 가이드

7.1 필수 요청 항목 (1순위)

1. 타이밍 스펙: Full Resolution Read 시 최소 Line Time 및 최대 Frame Rate.
2. 온도 특성: 15 \sim 40^\circ C 범위 내 100ms Integration 기준 Typical Dark Level 표.
3. Standby 권장 전위: 누설 최소화를 위한 벤더 권장 Gate OFF 및 Column 바이어스 값.
4. Blind Pixel 상세 정보: 광 차폐 영역의 정확한 Row/Column 좌표 범위 및 데이터 채널 공유 여부.

7.2 자체 측정 프로토콜

벤더 데이터 부족 시 다음 절차를 수행하라.

1. 패널을 온도 챔버에 입고하고 15^\circ C \sim 40^\circ C 구간을 5^\circ C 스텝으로 조성하라.
2. 각 온도 도달 후 15분간 안정화 기간을 거친 뒤 0, 30, 60, 120, 240분 간격으로 Dark 프레임을 획득하라.
3. 획득 데이터를 D(t) = D_0 + k(T) \cdot t 수식에 피팅하여 온도별 Drift Rate(k(T)) LUT를 구축하라.


--------------------------------------------------------------------------------


8. 최종 결론 및 체크리스트

본 v3.0 계획서는 a-Si TFT의 물리적 한계인 누설 전류를 시스템 레벨에서 계층적으로 관리함으로써 의료 영상 표준에 부합하는 안정성을 제공한다.

최종 체크리스트

* [ ] FPGA: 2-bit MUX 제어 신호에 따른 V_{PD} 전환 타이밍 검증 완료
* [ ] MCU: Arrhenius t_{max} 계산 로직의 온도 센서 보정 완료
* [ ] Host SW: 온도/시간 2D LUT 및 Blind Pixel 좌표 동기화 확인
* [ ] System: L2 → L1 전환 시 모드 전이 시간 1초 이내 달성 여부
* [ ] Stability: 8시간 연속 운전 시 Dark 잔차 < 100 DN 유지 확인

최종 제언: 구동 알고리즘만으로는 V_{TH} 드리프트의 픽셀별 편차를 완전히 제거할 수 없으므로, 팬(Fan) 또는 히트싱크를 통한 물리적 온도 안정화(±3°C 이내)를 병행할 것을 권고한다.
