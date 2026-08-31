# D3-2. 초기화 — Loosely-Coupled Vision-IMU Alignment

> SLAM 백엔드 평가 시리즈
> 이전 문서: D3-1. VINS-Fusion 개요와 최적화 기반 알고리즘 원리
> 참고: Qin, Li, Shen, "VINS-Mono," IEEE T-RO 2018, IV장(Estimator Initialization)
> ⚠️ 문헌 기반 문서 — 프로젝트 실측 없음(D3-1 안내 참고)

## 1. 개요

D1-2(ORB-SLAM3의 MAP 기반 3단계 초기화)와 D2-3(OpenVINS의 static/dynamic 초기화)에 이어, VINS-Fusion(VINS-Mono 계승)의 초기화 방식을 다룬다. 세 시스템의 초기화 철학을 나란히 놓고 보면 VIO 초기화 문제 전체의 지형이 잘 드러난다.

## 2. 핵심 개념: Loosely-Coupled(느슨한 결합) 초기화

D1-2에서 ORB-SLAM3 논문은 자신들의 MAP 기반 방식을 "최초로 IMU 초기화를 MAP 추정 문제로 정식화했다"고 소개하며, 그 이전의 방식들을 **"disjoint(분리형)" 또는 "loosely-coupled(느슨한 결합)"**로 분류했다. VINS-Mono의 초기화가 바로 이 loosely-coupled 계열의 대표적인 방법이다.

VINS-Mono 논문(IV장)이 설명하는 절차는 크게 4단계다.

### 1단계: Vision-only SfM (Structure from Motion)

- 슬라이딩 윈도우 안에서 **순수 시각 정보만으로** 카메라 간 상대 pose(스케일 미정)와 특징점 3D 위치를 먼저 계산한다.
- 이는 D1-2의 1단계(Vision-only MAP Estimation)와 개념적으로 유사하지만, MAP 추정이 아니라 **고전적인 SfM(기하학적 계산)**이라는 점이 다르다.

### 2단계: 자이로 바이어스 보정(Gyroscope Bias Calibration)

- 연속된 두 프레임의 SfM 결과 회전(rotation)과, 그 사이 IMU를 preintegration해서 얻은 회전 추정치를 비교한다.
- 이 둘의 차이를 최소화하는 방향으로 **자이로 바이어스를 선형 최소자승법으로 보정**한다.
- 보정된 바이어스로 모든 IMU preintegration 항을 다시 계산(re-propagate)한다.

### 3단계: 속도·중력·스케일 추정(Velocity, Gravity Vector and Metric Scale)

- SfM 결과(스케일 미정)와 IMU preintegration을 정렬(align)해서, 각 키프레임의 속도, 중력 벡터, 그리고(모노큘러의 경우) 절대 스케일을 선형 시스템으로 푼다.

### 4단계: 중력 벡터 정제(Gravity Refinement)

- 중력의 크기(magnitude)는 이미 알려진 물리 상수(약 9.81 m/s²)이므로, 이 제약을 활용해 중력 방향 추정을 추가로 정제한다.

## 3. 세 시스템의 초기화 철학 비교

| | ORB-SLAM3(D1-2) | OpenVINS(D2-3, 기본) | VINS-Fusion(D3, 이 문서) |
|---|---|---|---|
| 정식화 방식 | MAP(Maximum-a-Posteriori) 추정 — 센서 불확실성을 명시적으로 반영 | 정지 구간 가속도 분산 기반 | Loosely-coupled 선형 정렬(SfM + IMU 정렬) |
| 필요 조건 | 2초간 움직이며 SfM 성립 | 짧은 정지 구간 필요(기본값) | 슬라이딩 윈도우 내 SfM 성립(움직임 필요) |
| 불확실성 반영 | 명시적으로 반영(MAP) | 필터 자체가 불확실성을 상태로 관리 | 논문 원 버전은 상대적으로 단순한 선형 최소자승 — ORB-SLAM3 논문은 이런 부류를 "센서 불확실성을 무시해 큰 오차를 낼 수 있다"고 비판(D1-2 참고) |
| 초기화 속도 | 2~4초(D1-2) | 정지 구간 길이에 의존(`InitWindowTime`, 기본 2초, D2-3) | 문헌상 수 초 내외(정확한 수치는 구현·환경에 따라 다름) |

D1-2에서 언급했듯, ORB-SLAM3 논문 저자들은 VINS-Mono류의 "분리형" 초기화가 IMU 측정 불확실성을 충분히 반영하지 못한다고 지적하며 자신들의 MAP 방식을 대안으로 제시했다. 이는 학계에서 실제로 있었던 방법론적 논쟁이며, 어느 쪽이 "옳다"기보다 **설계 철학의 차이**로 이해하는 것이 정확하다 — 실제 성능은 환경, 센서 품질, 구현 디테일에 따라 달라진다(D4의 문헌 벤치마크 참고).

## 4. 이 프로젝트와의 관련성 (가상 적용 시나리오)

- D2-3에서 다룬 OpenVINS의 "정적 초기화가 기본이라 이미 움직이는 상황에 불리할 수 있다"는 우려는, VINS-Fusion의 이 방식(애초에 슬라이딩 윈도우 내 SfM이 성립해야 하므로 **움직임이 필요**)에는 해당하지 않는다 — VINS-Fusion을 도입한다면 이 지점에서 OpenVINS의 정적 초기화 문제를 우회할 수 있는 대안이 될 가능성이 있다.
- 다만 D435i의 IMU가 공장 미보정 상태라는 점(D1-5, D2-2에서 반복 언급된 이슈)은 VINS-Fusion에도 동일하게 적용되는 제약이다 — 자이로 바이어스 보정(2단계)의 초기 품질이 낮으면 이후 단계 전체에 영향을 준다.

## 5. 진단 관점 (일반적 특성 기준, 실증 전)

- VINS-Fusion 계열은 초기화 실패 시 보통 SfM 자체가 성립하지 않는 형태(특징점 부족, 시차 부족)로 나타난다고 알려져 있다 — D1-2의 "충분히 움직여야 한다"는 요구와 유사한 성격의 실패 모드를 예상할 수 있다.
- 실제 도입 시에는 D2-7처럼 프로젝트 자체 bag으로 재현 테스트를 거쳐야 이 문서의 일반론이 이 하드웨어(D435i, Jetson)에서도 맞는지 확인할 수 있다.

## 6. 다음 문서와의 연결

- 다음: **D3-3. Sliding Window 최적화와 Marginalization** — 초기화 이후 정상 운영 중 VINS-Fusion이 어떻게 슬라이딩 윈도우를 유지하는지 다룬다.

## 7. 참고자료

- Qin, Li, Shen, "VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator," IEEE T-RO 2018, IV장
- [[D1-2_Visual-Inertial_초기화_알고리즘|D1-2. Visual-Inertial 초기화 알고리즘]] — ORB-SLAM3 논문의 "loosely-coupled 방식 비판" 원문 맥락
