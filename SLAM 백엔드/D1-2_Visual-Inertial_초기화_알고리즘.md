# D1-2. Visual-Inertial 초기화 알고리즘

> RTAB-Map & Nav2 심화 시리즈 · Part D. 대안 SLAM 백엔드
> 이전 문서: D1-1. ORB-SLAM3 시스템 개요
> 참고 논문: Campos, Montiel, Tardós, "Inertial-Only Optimization for Visual-Inertial Initialization," ICRA 2020 (arXiv:2003.05766) / ORB-SLAM3 논문 4장(IMU Initialization)

## 1. 개요

D1-1에서 "Local Mapping 스레드가 IMU 초기화를 수행한다"고 짚었다. 이 문서는 그 초기화가 정확히 어떤 3단계 알고리즘으로 이루어지는지, GitHub가 명시적으로 인용하는 IMU-Initialization 논문을 근거로 정리한다. 이 알고리즘 이해가 D1-5(프로젝트 이슈 재해석)의 전제가 된다.

## 2. 핵심 개념: 왜 "초기화"가 별도로 필요한가

IMU를 카메라와 융합하려면 시작 시점에 다음 값들을 알아야 한다: **스케일**(모노큘러의 경우), **중력 방향**, **초기 속도**, **가속도계/자이로 바이어스**. 이 값들 없이 바로 시각-관성 최적화(BA)를 시작하면(예: VI-DSO 방식) 관성 변수 수렴에 최대 30초가 걸릴 정도로 느리다는 것이 논문의 문제의식이다. ORB-SLAM3는 이를 **3단계 MAP(Maximum-a-Posteriori) 추정 문제**로 재정의해 몇 초 안에 초기화를 끝내는 것을 목표로 한다.

## 3. 3단계 알고리즘 상세

### 1단계: Vision-only MAP Estimation

- 순수 모노큘러 SLAM(기존 ORB-SLAM 방식)으로 시작해 **2초 동안, 4Hz 주기로 키프레임을 삽입**한다.
- 이 짧은 기간 동안 만들어진 **k=10개의 카메라 pose와 수백 개의 포인트**로 이루어진, 스케일 미정(up-to-scale)인 지도를 얻고 visual-only BA로 최적화한다.
- 키프레임 삽입 주기를 의도적으로 높이는(4~10Hz) 이유: IMU preintegration 구간이 짧아야 불확실성이 낮아지기 때문이다.
- 이 pose들을 body(IMU) 기준 좌표계로 변환한다.

### 2단계: Inertial-only MAP Estimation

- 1단계에서 얻은 스케일 미정 궤적과, 그 사이의 관성 측정값만을 이용해 **관성 변수들을 MAP 추정 관점에서 최적으로 추정**한다.
- 추정 대상: 자이로 바이어스, 중력 방향(SO(3) 최소 파라미터화), (모노큘러의 경우) 스케일, 가속도계 바이어스, 각 키프레임 시점의 속도.
- 논문의 핵심 novelty: 기존 방법들이 IMU 측정 불확실성을 무시한 채 대수 방정식이나 임시방편적 최소자승법을 풀었던 것과 달리, **최초로 이 문제를 MAP 추정으로 정식화**해 센서 불확실성을 제대로 반영한다.

### 3단계: Visual-Inertial MAP Estimation

- 1·2단계 결과를 시드(seed)로 삼아, visual 잔차와 inertial 잔차를 함께 최소화하는 **완전한 Joint VI-BA**를 수행해 최종적으로 결합 최적해를 구한다.
- 이후 이 초기화 결과를 가지고 ORB-SLAM Visual-Inertial(지역 VI-BA)을 본격 가동한다.

## 4. 성능 특성 (논문 수치)

| 지표 | 값 |
|---|---|
| 초기화 소요 시간 | 약 2~4초 이내 (EuRoC 데이터셋 기준) — 기존 방법 대비 최대 30초 이상 걸리던 것을 크게 단축 |
| 초기 스케일 오차 | 평균 약 5.3% (~5.29%) |
| 2초 시점 스케일 오차 | 5% 미만으로 수렴 |
| 15초 시점 스케일 오차 | 약 1% 수준까지 추가 수렴 (루프가 없는 시퀀스에서도) |

## 5. 이 프로젝트에서의 적용 (Yahboom X3, RGB-D-Inertial)

이 프로젝트는 모노큘러가 아니라 RGB-D-Inertial 모드를 쓰므로, **스케일은 이미 depth로부터 알려져 있어 1단계 결과가 처음부터 metric 스케일**이라는 차이가 있다. 그럼에도 2단계(중력 방향, 바이어스, 속도 추정)는 동일하게 필요하다.

핵심 연결점: [[ros2-nav-yahboom]]에서 확인된 리셋 유형들을 이 3단계와 대응시키면 다음과 같이 재해석할 수 있다.

| 프로젝트에서 관찰된 리셋 유형 | 대응 가능한 알고리즘 단계 |
|---|---|
| "not enough acceleration" 리셋 | 2단계 진입 조건(자극/게이팅) — 논문의 3가지 통찰 중 "센서 불확실성을 무시하면 예측 불가능한 큰 오차가 난다"는 부분과 관련 |
| active-map IMU reset (Local Mapping 재초기화) | 2~3단계 결과가 시스템 내부 수렴/일관성 기준을 통과하지 못해 **1단계부터 다시 시작**하는 것으로 추정됨 |

## 6. 진단 관점

D1의 원래 진단(자극 부족 가설 배제, fastInit 버그 수정 후에도 active-map reset 지속)을 이 알고리즘 구조로 다시 보면: **자극(움직임)과 게이팅 임계값을 모두 통과해도 리셋이 발생했다는 것은, 문제가 "1단계(Vision-only)의 입력 품질"이나 "2단계(Inertial-only)의 최적화 자체가 수렴 기준을 만족하지 못하는 것" 중 하나에 있을 가능성이 높다**는 것을 시사한다. D1-1에서 다룬 "Local Mapping 재초기화"라는 관찰 자체가, 이 3단계 파이프라인이 성공 판정을 내리지 못하고 있다는 신호로 해석된다.

## 7. 다음 문서와의 연결

- 다음: **D1-3. Multi-Map System(Atlas)과 재추적·병합** — 초기화가 반복 실패할 때 Atlas가 어떻게 새 지도를 만들고 나중에 병합하는지 다룬다. "active-map IMU reset" 이후 실제로 무슨 일이 일어나는지의 다음 단계다.

## 8. 참고자료

- Campos, Montiel, Tardós, "Inertial-Only Optimization for Visual-Inertial Initialization," ICRA 2020 (arXiv:2003.05766)
- ORB-SLAM3 논문(Campos et al., 2021) 4장 "IMU Initialization"
- [[ros2-nav-yahboom]] — 이 프로젝트의 IMU.fastInit 진단, 게이팅 격리 테스트 이력
