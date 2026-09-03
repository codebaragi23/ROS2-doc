# 02-2. OpenVINS의 핵심 설계 — FEJ, 온라인 캘리브레이션, Feature 표현

> SLAM 백엔드 심화 시리즈
> 이전 문서: [[02-1_OpenVINS_개요와_MSCKF_알고리즘_원리|02-1. OpenVINS 개요와 MSCKF 알고리즘 원리]]
> 참고 논문: Geneva et al., "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 4 — 심화 |
| 예상 선행 지식 | [[02-1_OpenVINS_개요와_MSCKF_알고리즘_원리\|심화 02-1]] |
| 학습 목표 | FEJ가 왜 필요한지(관측 가능성 일관성) 설명할 수 있다 / 온라인 캘리브레이션의 범위를 안다 / MSCKF feature와 SLAM feature의 이원화를 이해한다 |
| 기준 환경 | 개념 위주 (논문 + 프로젝트 파라미터) |

## 1. 개요

02-1에서 MSCKF의 기본 아이디어를 다뤘다. OpenVINS 공식 논문은 이 기본 MSCKF 위에 **연구·실전 양쪽에 유용한 추가 설계**들을 얹었다고 소개한다. 이 문서는 그 설계 요소들을 정리한다.

## 2. 핵심 개념: 공식 논문이 밝힌 OpenVINS의 기능 목록

논문 초록은 OpenVINS의 핵심 기능을 다음과 같이 나열한다.

1. **On-manifold sliding window Kalman filter**: 회전(rotation)을 수학적으로 올바르게 다루는 슬라이딩 윈도우 필터.

   > **왜 회전은 특별한가**: 회전값은 일반 숫자처럼 더하고 뺄 수 없다. 180° + 180° = 360° = 0°이고, 좌표를 조금씩 더해가다 보면 엉뚱한 값이 나온다. 즉 회전은 **평평한 공간(유클리드 공간)이 아니라 휘어 있는 공간 위에 산다** — 이 휘어 있는 공간을 **매니폴드(manifold)**라 부르고, 그 위에서 계산 규칙을 따로 지키는 것이 "on-manifold로 다룬다"는 말의 뜻이다. 01-2에서 중력 방향을 표현할 때 나온 `SO(3)`가 바로 이 곡면의 이름이다.
2. **온라인 카메라 내부/외부 파라미터 캘리브레이션**: 카메라의 초점거리·왜곡(intrinsic)과 카메라-IMU 간 위치 관계(extrinsic)를 시스템이 동작하는 동안 스스로 보정할 수 있다.
3. **카메라-IMU 시간 오프셋 캘리브레이션**: 두 센서의 타임스탬프가 미세하게 어긋나 있어도(하드웨어 동기화 오차) 이를 온라인으로 추정해 보정한다.
4. **다양한 표현 방식의 SLAM landmark + 일관된 FEJ(First-Estimates Jacobian) 처리**: 아래 3절에서 자세히 다룬다.
5. **모듈형 타입 시스템**: 상태 관리를 위한 확장 가능한 구조.
6. **확장 가능한 시뮬레이터**: 알고리즘 검증용 VIO 시뮬레이터.
7. **평가 툴박스**: 정확도·성능 분석 도구 일체.

## 3. FEJ(First-Estimates Jacobian) — 왜 "일관성(consistency)"이 문제가 되는가

EKF는 매 업데이트마다 현재 추정값 근처에서 비선형 함수를 선형화(linearize)해서 계산한다. 문제는 **같은 변수를 서로 다른 시점에 서로 다른 값으로 선형화**하면, 실제로는 관측 불가능한(unobservable) 방향에 대해서도 필터가 마치 정보를 얻은 것처럼 착각하는 **비일관성(inconsistency)**이 생긴다는 것이다. 이는 필터가 자기 확신을 과도하게 키우는(overconfident) 결과로 이어진다 — 02-1에서 언급한 "순환 의존성" 문제의 또 다른 얼굴이다.

**FEJ(First-Estimates Jacobian)** 기법은 이 문제를 해결하기 위해, 같은 변수에 대한 Jacobian을 계산할 때 **그 변수가 처음 추정된 값(first estimate)을 항상 일관되게 사용**하도록 강제한다. OpenVINS는 이 FEJ 처리를 MSCKF 전반에 일관되게 적용해, 장시간 동작해도 필터의 신뢰도(공분산)가 비현실적으로 작아지는 문제를 억제한다.

## 4. SLAM Landmark 표현 방식

02-1에서 "MSCKF feature는 상태에 안 들어간다"고 했지만, OpenVINS는 **일부 특징점(오래 추적되는 것들)은 SLAM feature로 승격시켜 상태 벡터에 직접 포함**시키기도 한다(02-4에서 자세히 다룰 `MaxSLAM` 파라미터가 이것). 이때 3D 위치를 어떤 좌표계·형식으로 표현할지에 따라 여러 옵션이 있다.

| 표현 방식 | 개념 |
|---|---|
| Global XYZ | 전역 좌표계 기준 3D 좌표 그대로 |
| Anchored XYZ | 특정 기준(anchor) 카메라 pose를 원점으로 한 3D 좌표 |
| Global Inverse Depth | 전역 좌표계 기준, 깊이의 역수(1/depth)로 표현 — 먼 거리 특징점의 불확실성을 다루기에 유리 |
| Anchored Inverse Depth | anchor 기준 + inverse depth 조합 |
| Anchored MSCKF Inverse Depth | MSCKF 스타일 처리와 anchored inverse depth를 결합 |

표현 방식에 따라 필터의 수치적 안정성과 계산 비용이 달라지며, OpenVINS는 이를 설정으로 선택할 수 있게 해준다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- 02-6(RTAB-Map 통합 구조)에서 다룰 것처럼, 이 프로젝트는 OpenVINS를 RTAB-Map의 odometry 플러그인으로 통합해 사용한다. 온라인 캘리브레이션(카메라 intrinsic/extrinsic, 시간 오프셋)은 이론적으로 D435i처럼 IMU가 공장 미보정 상태인 센서에서 특히 유용할 수 있는 기능이지만, [[ros2-nav-yahboom]]의 진단 이력을 보면 **이 프로젝트에서 이 온라인 캘리브레이션 기능이 적극적으로 활용됐다는 기록은 없다** — 대부분 고정된 파라미터(02-6, 02-7)로만 실행됐다.
- 이는 01-5(ORB-SLAM3 VIO 이슈)에서 다룬 "`rs-imu-calibration` 미시도"와 같은 계열의 미검증 변수다: OpenVINS의 온라인 캘리브레이션을 켜면 D435i IMU의 공장 미보정 문제를 부분적으로 완화할 수 있을지 아직 확인되지 않았다.

## 6. 진단 관점

- FEJ가 제대로 동작하지 않으면(구현 버그 또는 잘못된 설정), 필터가 장시간 동작 후 실제보다 훨씬 확신에 찬(공분산이 비정상적으로 작은) 상태가 되면서 갑작스러운 큰 오차에 취약해질 수 있다 — 이런 증상이 보이면 FEJ 관련 설정이나 구현을 의심해볼 수 있다.
- SLAM landmark 표현 방식은 보통 기본값을 그대로 쓰는 경우가 많지만, 저텍스처 환경(02-8, 냉장고 근접 사례)에서 특징점 깊이 불확실성이 크다면 inverse depth 계열 표현이 이론적으로 더 안정적일 수 있다.

## 7. 다음 문서와의 연결

- 다음: **[[02-3_초기화_Static_vs_Dynamic|SLAM 백엔드 심화 - 02-3. 초기화: Static vs Dynamic]]** — OpenVINS가 시작 시점에 중력 방향과 초기 상태를 어떻게 잡는지, 그리고 이 프로젝트가 실제로 어떤 방식을 쓰고 있는지 다룬다.

## 8. 참고자료

- Geneva, Eckenhoff, Lee, Yang, Huang, "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 ([DOI: 10.1109/ICRA40945.2020.9196524](https://doi.org/10.1109/ICRA40945.2020.9196524))
- Huang, Mourikis, Roumeliotis, "A First-Estimates Jacobian EKF for Improving SLAM Consistency," ISER 2008 ([PDF](https://people.csail.mit.edu/ghuang/paper/Huang2008ISER.pdf))
- [[ros2-nav-yahboom]] — 이 프로젝트의 OpenVINS 통합 및 파라미터 사용 이력
