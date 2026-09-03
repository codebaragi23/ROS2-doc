# 02-1. OpenVINS 개요와 MSCKF 알고리즘 원리

> SLAM 백엔드 심화 시리즈
> 참고 논문: Geneva, Eckenhoff, Lee, Yang, Huang, "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 / Mourikis, Roumeliotis, "A Multi-State Constraint Kalman Filter for Vision-Aided Inertial Navigation," ICRA 2007
> 이전 문서: [[01-5_이_프로젝트의_VIO_이슈_재해석|SLAM 백엔드 심화 - 01-5. 이 프로젝트의 VIO 이슈 재해석]] (01 ORB-SLAM3 시리즈 마지막)
> 선행 학습: [[01-1_ORB-SLAM3_시스템_개요|01-1. ORB-SLAM3 시스템 개요]] — 같은 문제(카메라+IMU로 위치 추정)를 완전히 다른 방식으로 푸는 두 시스템을 대비해서 읽으면 이해가 빠르다.

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 4 — 심화 (OpenVINS 시작점) |
| 예상 선행 지식 | [[00-1_심화_시리즈를_위한_최소_수학\|심화 00-1]], [[01-1_ORB-SLAM3_시스템_개요\|심화 01-1]](대조군) |
| 학습 목표 | MSCKF가 특징점을 상태에 넣지 않고도 제약을 쓰는 원리를 안다 / null-space 투영의 역할을 설명할 수 있다 / 필터 기반이 최적화 기반과 무엇이 다른지 대비할 수 있다 |
| 기준 환경 | 개념 위주 (논문 기반) |

## 1. 개요

01 시리즈에서 다룬 ORB-SLAM3는 "지도를 만들고(map), 그 지도에 자신을 맞춰가는" 키프레임 기반 SLAM이었다. OpenVINS(University of Delaware RPNG Lab)는 이것과 **근본적으로 다른 패러다임**인 **MSCKF(Multi-State Constraint Kalman Filter)** 기반 필터형 VIO(Visual-Inertial Odometry)다. 이 문서는 초보자가 "왜 이렇게 다른 두 방식이 존재하는가"부터 이해할 수 있도록 MSCKF의 핵심 아이디어를 설명한다.

## 2. 핵심 개념: EKF가 카메라를 직접 다루기 어려운 이유

**칼만 필터(Kalman Filter)**는 "현재 상태를 예측하고, 새 측정값이 들어오면 그 예측을 보정한다"를 반복하는 알고리즘이다. IMU만 있다면 이 예측-보정이 비교적 단순하다(가속도로 위치를 적분해 예측하고, 새 IMU 측정으로 보정). 그런데 **카메라가 보는 특징점의 3D 위치는 그 자체로 모르는 값**이라는 문제가 있다.

카메라 특징점을 필터의 상태(state)에 그냥 추가하면 다음과 같은 **순환 의존성** 문제가 생긴다:

> 특징점의 3D 위치를 추정하려면 카메라의 위치를 알아야 하고, 카메라의 위치를 더 정확히 추정하려면 특징점 위치를 알아야 한다 — 같은 필터 안에서 이 둘을 동시에 업데이트하면, 필터가 자기 자신의 추정치를 근거로 삼아 스스로에게 과도한 확신을 갖게 되는(artificially high confidence) 문제가 생긴다.

## 3. MSCKF의 핵심 아이디어: "특징점을 상태에 넣지 않는다"

Mourikis와 Roumeliotis(2007)가 제안한 MSCKF는 이 문제를 다음과 같이 우회한다.

1. **상태 확장(State Augmentation)**: 새 카메라 프레임이 들어올 때마다, 그 순간의 카메라 pose를 필터 상태에 "복제(clone)"해서 추가한다. 즉 상태 벡터 안에 **최근 여러 시점의 카메라 pose들이 슬라이딩 윈도우 형태로 쌓인다.**
2. **특징점은 상태에 넣지 않는다**: 하나의 특징점이 이 슬라이딩 윈도우 안의 여러 pose에서 관측되면, 그 특징점의 3D 위치를 **명시적으로 추정하지 않고**, 여러 pose 사이의 기하학적 제약(같은 3D 점을 봤다는 사실)만 필터 업데이트에 활용한다.
3. **Null-space projection(영공간 투영)**: 이 제약을 수학적으로 다루기 위해, 측정 잔차(residual, 예측값과 실제 측정값의 차이)를 특징점 위치의 Jacobian이 만드는 영공간(null space)에 투영한다. 이렇게 하면 "특징점 위치가 정확히 얼마인지"는 몰라도 "여러 pose가 이 기하학적 제약을 만족해야 한다"는 정보만 깔끔하게 필터에 반영할 수 있다.

> **비유로 이해하기**: 세 사람이 같은 물체를 바라보고 있다. 그 물체가 정확히 어디 있는지는 아무도 모른다. 하지만 **"세 사람의 시선이 어딘가 한 점에서 만나야 한다"는 조건만으로도 세 사람의 상대적 위치 관계를 상당히 좁힐 수 있다.** 물체의 위치를 방정식에서 소거하고 사람들의 위치 관계만 남기는 것 — 이것이 null-space projection이 하는 일이다.
>
> 여기서 **Jacobian**은 "추정값을 조금 바꾸면 예측 측정값이 얼마나 바뀌는가"의 비율을 모아둔 기울기표이고, **영공간(null space)**은 "그 값을 바꿔도 결과에 아무 영향이 없는 방향들"의 모음이다. 특징점 위치 방향의 영공간에 투영한다는 것은 곧 **특징점 위치가 얼마든 상관없어지도록 만든다**는 뜻이다. (더 자세한 배경은 [[00-1_심화_시리즈를_위한_최소_수학|SLAM 백엔드 심화 - 00-1. SLAM 백엔드 심화를 위한 최소 수학]] 4장 참고)
4. **계산량이 특징점 개수에 선형(linear)**: 이 방식 덕분에 특징점 3D 위치를 상태에 포함하지 않고도 대규모 환경에서 실시간 동작이 가능하다.

```mermaid
flowchart LR
    subgraph MSCKF["MSCKF 방식"]
        지도제작 00[새 카메라 프레임] -->|pose를 상태에 clone| SW[Sliding Window
과거 pose들의 집합]
        F1[특징점 여러 프레임에서 관측] -->|Null-space projection| SW
        SW -->|오래된 pose는 marginalize로 압축| SW
    end
```

## 4. ORB-SLAM3(01)와의 근본적 차이

| 항목 | ORB-SLAM3 (01) | OpenVINS(MSCKF) |
|---|---|---|
| 알고리즘 부류 | 키프레임 기반 SLAM (Bundle Adjustment) | MSCKF 기반 EKF(확장 칼만 필터) |
| 상태 표현 | 명시적 지도(키프레임 + map point 그래프) | 슬라이딩 윈도우의 IMU pose clone들 + 소수의 SLAM feature |
| 특징점 위치 | 그래프에 명시적으로 포함, 최적화 대상 | (MSCKF feature는) 상태에 없음, null-space projection으로만 활용 |
| 추적 실패 시 | "Fail to track" → 명시적 lost 상태 → 새 맵 생성(01-1, 01-3) | **명시적 lost 상태 자체가 없음** — 항상 확률적 추정치를 유지, 불확실성만 커짐 |
| 계산량 성장 | 장면이 넓어질수록 맵/키프레임 증가 | 슬라이딩 윈도우 크기로 고정 — 장면 크기와 무관 |
| Loop Closure | 있음(01-3, 01-4) | 기본적으로 없음(순수 odometry, drift 누적) |

이 표의 마지막 두 행이 실제로 관찰되는 현상을 설명한다: OpenVINS는 "맵을 버리고 새로 만든다"는 개념 자체가 없어서 01-3에서 다룬 것과 같은 리셋 이벤트가 구조적으로 발생하지 않는다. 다만 이것이 "더 안전하다"는 뜻은 아니다 — 위치가 실제로 틀렸을 때도 시스템이 "실패했다"고 명시적으로 알려주지 않고 낮은 신뢰도로 조용히 계속 돌 수 있다는 뜻이기도 하다(02-5·02-8에서 다룰 ZUPT 이슈와 연결).

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- Loop closure가 없다는 것은, RTAB-Map(지도제작 00)이나 ORB-SLAM3(01-3)처럼 "이미 지나간 곳을 재방문해서 드리프트를 청소"하는 능력이 OpenVINS 자체에는 없다는 뜻이다. 순수 odometry로만 쓰면 시간이 지날수록 누적 오차가 쌓인다.
- 이 프로젝트에서 RTAB-Map은 OpenVINS를 **odometry 소스 중 하나**로만 사용하고, loop closure와 지도 관리는 여전히 RTAB-Map 자체(지도제작 00)가 담당하는 구조로 통합되어 있다(02-6에서 코드 레벨로 다룸). 즉 "OpenVINS의 정확한 odometry" + "RTAB-Map의 loop closure/지도 관리"를 조합하는 설계다.

## 6. 진단 관점

- OpenVINS 기반 시스템에서 위치가 이상해 보이는데 로그에 명확한 오류가 없다면, "리셋이 없다"는 이 문서의 설계 특성 때문일 수 있다 — ORB-SLAM3처럼 "Fail to track"을 기대하고 로그를 보면 놓치기 쉽다. 대신 신뢰도/공분산이나 실제 이동 거리(step_translation) 같은 정량 지표를 봐야 한다(02-7에서 자세히).

## 7. 다음 문서와의 연결

- 다음: **[[02-2_OpenVINS의_핵심_설계|SLAM 백엔드 심화 - 02-2. OpenVINS의 핵심 설계 — FEJ, 온라인 캘리브레이션, Feature 표현]]** — 공식 논문이 강조하는 OpenVINS만의 설계 요소들을 다룬다.

## 8. 참고자료

- Mourikis, Roumeliotis, "A Multi-State Constraint Kalman Filter for Vision-Aided Inertial Navigation," ICRA 2007 ([PDF](https://www-users.cse.umn.edu/~stergios/papers/ICRA07-MSCKF.pdf))
- Geneva, Eckenhoff, Lee, Yang, Huang, "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 ([DOI: 10.1109/ICRA40945.2020.9196524](https://doi.org/10.1109/ICRA40945.2020.9196524))
- [GitHub rpng/open_vins](https://github.com/rpng/open_vins) — 공식 저장소
