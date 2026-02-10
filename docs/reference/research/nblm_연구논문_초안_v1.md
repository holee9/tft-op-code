a-Si TFT 누설 전류 및 Idle 상태 관리 연구개발 계획서 (v3.0)

1. 서론: 기술적 배경 및 문제 정의

a-Si TFT 픽셀 구조와 누설 경로 정의

비정질 실리콘(a-Si) TFT 기반 픽셀은 'aSi_tft_cross-section.jpg'에 명시된 바와 같이 Storage Capacitor, Storage Node, 그리고 제어 소자인 TFT Switch로 구성됩니다. 이미지 센서의 다크 신호 안정성을 저해하는 주요 요인은 a-Si 채널 내에서 발생하는 두 가지 물리적 누설 경로입니다.

* Subthreshold Leakage (아임계 누설): TFT가 'OFF' 상태일 때 게이트-드레인 사이의 전기장에 의해 소수 캐리어가 발생하는 현상입니다. 이는 a-Si 특유의 **Ideality factor (n \approx 1.5 \sim 2)**에 의해 지수적으로 거동하며, 아임계 영역의 전도 특성을 결정합니다.
* SRH Generation (Deep State 생성 전류): a-Si:H 밴드갭 중앙의 깊은 결함 상태(Deep Trap)에서 Shockley-Read-Hall 메커니즘에 의해 발생하는 열적 생성 전류입니다.

물리적 메커니즘 상세

이러한 누설 메커니즘은 외부 환경, 특히 온도에 극도로 민감합니다. Deep State Generation Current는 Arrhenius 모델(k(T) = a \exp(-E_A/k_B T))로 정량화되며, 활성화 에너지 E_A = 0.45\,eV를 가집니다. 이는 온도가 10^\circ C 상승할 때마다 누설 전류가 약 2배씩 증가하는 지수적 특성을 보이며, 결과적으로 다크 레벨의 급격한 상승과 이미지 아티팩트를 유발합니다.

온도별 Dark Signal Drift 위험성 분석

'dark_signal_drift.png' 데이터 분석 결과, 20^\circ C \sim 30^\circ C 구간에서는 다크 드리프트가 비교적 선형적으로 유지되나, 고온 구간에서는 위험성이 급증합니다. 특히 40^\circ C 환경에서는 대기 시간(Idle Time) 30분을 기점으로 다크 레벨이 기하급수적으로 상승하는 "High Leakage Risk" 구간이 나타납니다. 이는 장시간 대기 후 첫 프레임 획득 시 심각한 품질 저하를 야기하므로 시스템 레벨의 관리가 필수적입니다.


--------------------------------------------------------------------------------


2. 시스템 아키텍처 및 Idle 상태 관리 전략

3계층 제어 시스템 구성

'stucture.jpg'를 기반으로 물리적 현상을 억제하고 보정하기 위해 다음과 같이 역할을 분담합니다.

1. MCU (Temperature Monitor & Idle Timer): 1 Hz 주기로 온도를 폴링하여 t_{max}(T)를 실시간 계산하고 시스템의 Idle 레벨을 결정합니다.
2. FPGA (Timing & Sequencer): MCU 명령에 따라 2-bit MUX를 통해 바이어스를 전환하고, Dummy Scan 시퀀스를 생성합니다.
3. Host PC (2D LUT Correction & Image Processing): MCU로부터 전달받은 온도/시간 메타데이터를 기반으로 2D LUT 보정 및 적응형 알고리즘을 수행합니다.

Idle 상태 계층화 전략 (L1/L2/L3)

대기 시간에 따라 열화를 단계적으로 차단하는 3단계 전략을 적용합니다.

구분	L1 (Normal Idle)	L2 (Low-Bias Idle)	L3 (Deep Sleep)
진입 조건	대기 시간 10분 미만	10분 \le 대기 < 100분	대기 시간 100분 이상
바이어스 (V_{PD})	Normal Bias (-1.5V)	Quasi Zero-bias (-0.2V)	Power Down
주요 액션	스캔 없음	30~60s 주기 Dummy Scan (T_{dummy})	전력 차단
허용 시간 (t_{max})	~50분 (@25^\circ C)	~200분 (@25^\circ C)	거의 무한

참고: 위 표의 바이어스는 포토다이오드 바이어스(V_{PD})이며, Gate OFF 전압(V_{G\_OFF})과는 별개로 제어됨.

구현 원리 및 효과

* Quasi Zero-bias 효과: 저장 노드와 포토다이오드 양단의 전위차를 최소화하여 다크 전류 생성을 물리적으로 억제합니다. 이를 통해 다크 상승률을 20~30% 감소시킬 수 있습니다.
* 주기적 Dummy Reset: 고정된 주기(T_{dummy})로 저장 노드의 전하를 비워줌으로써 다크 전하 누적을 선형적으로 제한하고 시스템을 Plateau 상태로 유지합니다.


--------------------------------------------------------------------------------


3. 구동 방법론 비교 분석: 연구 모드 vs 임상 모드

Methodology 1: High-Quality Research Mode (연구 모드)

'timining_1.jpg' 기반의 정밀 분석 모드입니다.

* 타이밍 구성: Reset(100ms) \rightarrow Exposure(50ms) \rightarrow Readout(250ms), 총 400ms(2.5 fps) 주기.
* 특징: 충분한 시간 예산을 활용하여 여러 장의 Dark/Lag 프레임을 획득하고 정밀한 물리 모델을 추출하는 데 최적화되어 있습니다.

Methodology 2: 5 fps Clinical Mode (임상 모드)

'2.5fps_clinical_mode.jpg'의 5 fps 목표를 달성하기 위한 고속 구동 모드입니다.

* 타이밍 구성: Fast Reset(10ms) \rightarrow Exposure(40ms) \rightarrow Fast Readout(150ms), 총 200ms(5 fps) 주기.
* 특징: Fast Readout(150ms)은 기존 250ms 대비 약 40%의 시간 단축을 통해 5 fps를 가능하게 하는 핵심 요소입니다.
* 실시간 보정 전략: 200ms의 짧은 주기 내에서는 추가 Dark 프레임 획득이 불가능하므로, Host PC는 MCU가 전송한 실시간 온도 및 타이머 메타데이터를 사용하여 2D LUT에서 최적의 보정값을 선택하는 전략을 취합니다.

모드별 성능 목표 비교

KPI 지표	연구 모드 (400ms)	임상 모드 (5 fps)
Dark 보정 효율	30%	20% (개선 목표)
Lag 보정 효율	50%	30%
Warm-up 오버헤드	N/A	< 500 ms (2 frames)
잔류 Dark 목표	< 100 DN	< 100 DN


--------------------------------------------------------------------------------


4. 실행 계획 및 성능 목표(KPI)

주요 성능 지표(KPI)

임상 모드 기준 Dark 보정 효율 20%는 기존 15% 대비 상향된 개선 목표이며, 모든 모드에서 보정 후 잔류 다크는 100 DN 미만을 유지하는 것을 최종 목표로 합니다.

구현 로드맵 (WBS) - 총 8.5주

* 1~2주 (FPGA): 2-bit MUX 기반 V_{PD} 제어 및 Dummy Scan 로직 구현.
* 3~4주 (MCU): Arrhenius 모델 기반 t_{max} 실시간 계산 및 L1/L2/L3 상태 머신 구축.
* 5~6주 (Host SW): 온도/시간 축 2D LUT 양선형 인터폴레이션 및 적응형 Lag 알고리즘 개발.
* 7~8.5주 (통합 테스트): 환경 온도 급변(\pm 5^\circ C) 대응 및 최종 KPI 검증.

리소스 할당 (Total 4.6 FTE)

* FPGA Engineer (1.0): RTL 설계 및 타이밍 클로징.
* MCU/FW Engineer (1.0): 온도 모델링 및 상태 관리 로직.
* Host SW Engineer (1.0): 2D LUT 보정 및 메타데이터 처리.
* QA/Measurement (1.3):
  * 신뢰성 검증 (0.8): 통합 테스트 및 KPI 확인.
  * 챔버 데이터 수집 (0.5): 온도별 기초 데이터 측정.
* Project Lead (0.3): 기술 조정 및 보고.


--------------------------------------------------------------------------------


5. 벤더 협조 요청 및 자체 측정 프로토콜

필수 기술 정보 요청 항목 (1순위)

패널 제조사에 최적 구동을 위한 다음 4가지 항목을 공식 요청합니다.

1. 타이밍 및 모드 스펙
  * 내용: Full Resolution Read 모드의 최대 fps 및 최소 라인 타이밍(Line Time).
  * 목적: 250ms에서 150ms로의 리드 타임 단축 가능성 확인.
2. Dark Signal vs Temperature 데이터
  * 내용: 15^\circ C \sim 40^\circ C 구간, 100ms Integration 기준 Typical Dark Level 데이터.
  * 목적: 시스템 보정 알고리즘용 기초 모델 수립.
3. Standby/Idle 운용 권장사항
  * 내용: Idle 상태에서 누설 및 열화를 최소화하기 위한 권장 Gate/Column 바이어스 설정.
  * 목적: L2 모드 바이어스 최적화.
4. Shielded/Optical-black Pixel 정보
  * 내용: 광 차폐 픽셀의 정확한 위치(Row/Column 범위) 및 데이터 채널 정보.
  * 목적: 실시간 다크 오프셋 추적 및 제거.

자체 측정 프로토콜 (Fallback Plan)

벤더 응답이 부족할 경우, 온도 챔버를 활용하여 15^\circ C \sim 40^\circ C 구간을 5^\circ C 스텝으로 측정합니다. 각 온도에서 15분 안정화 후 0 \sim 240분 간격으로 다크 프레임을 획득하여 k(T) LUT를 직접 구축합니다.

최종 체크리스트

* [ ] FPGA: Idle 진입 시 V_{PD} 전환 및 Dummy Scan 타이밍 정확성 확인
* [ ] MCU: 온도 센서 폴링 기반 t_{max} 계산 및 Arrhenius 모델 정밀도 검증
* [ ] Host SW: 2D LUT 인터폴레이션 및 Blind Pixel 보정 동기화 확인
* [ ] V_{TH} Drift 모니터링 프로토콜 정의 및 Gain 변화 추적 수립
* [ ] System: L2/L3 탈출 후 2프레임 이내 Warm-up 처리 및 첫 영상 품질 확보 확인
