# D3-1. VINS-Fusion 개요와 최적화 기반 알고리즘 원리

> SLAM 백엔드 평가 시리즈
> 참고: Qin, Li, Shen, "VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator," IEEE T-RO 2018 / [GitHub HKUST-Aerial-Robotics/VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion)
> ⚠️ **이 D3 시리즈 전체는 이 프로젝트에서 아직 코드 통합·실측이 이루어지지 않은 상태로, 공개 문헌(논문·GitHub)만을 근거로 작성됐다.** D1(ORB-SLAM3), D2(OpenVINS)처럼 프로젝트 실측 데이터를 포함하지 않는다는 점을 유의해야 한다.

## 1. 개요

D1(ORB-SLAM3, 키프레임 SLAM)과 D2(OpenVINS, MSCKF 필터)에 이어 세 번째 방식인 **최적화 기반(optimization-based) 슬라이딩 윈도우** VIO를 다룬다. VINS-Fusion은 홍콩과기대(HKUST) Aerial Robotics Group의 VINS-Mono를 확장한 시스템으로, 모노큘러뿐 아니라 스테레오·스테레오+IMU 등 다양한 센서 조합을 지원한다.

## 2. 핵심 개념: 세 번째 패러다임 — Tightly-Coupled Optimization

D1-D2에서 다룬 두 방식을 다시 정리하면:

- **ORB-SLAM3(D1)**: 키프레임과 map point로 이루어진 명시적 그래프를 Bundle Adjustment로 최적화. 지도가 계속 누적됨.
- **OpenVINS(D2)**: 슬라이딩 윈도우 안의 pose를 EKF로 순차적으로 업데이트. 특징점은 상태에 거의 넣지 않음(null-space projection).

**VINS-Fusion**은 이 둘의 중간적 성격을 가진다:

- OpenVINS처럼 **슬라이딩 윈도우**를 쓰지만, EKF의 "순차적 필터링" 대신 **매 주기 비선형 최적화(nonlinear optimization, Ceres Solver 기반)를 다시 푸는 방식**이다.
- ORB-SLAM3처럼 **특징점 위치를 최적화 변수에 포함**시키지만, 윈도우 밖으로 나간 오래된 특징점/프레임은 **한꺼번에 최적화하지 않고 주변화(marginalization)**해서 사전(prior) 정보로 압축한다.

| | ORB-SLAM3(D1) | OpenVINS(D2) | VINS-Fusion(D3) |
|---|---|---|---|
| 알고리즘 부류 | 키프레임 SLAM(BA) | MSCKF(EKF) | 최적화 기반 슬라이딩 윈도우 |
| 특징점 처리 | 그래프에 명시적 포함, 계속 최적화 | 대부분 상태에서 제외(null-space) | 윈도우 안에서는 최적화 변수, 윈도우 밖은 marginalize |
| 갱신 방식 | 비선형 최적화(반복적, 배치형) | 순차적 필터링(1-step 업데이트) | 매 주기 비선형 최적화(윈도우 크기만큼 반복) |
| 지도 누적 | Atlas로 계속 누적(D1-3) | 없음(고정 윈도우, D2-4) | 없음(고정 윈도우) + 별도 pose graph로 장기 기억 |

## 3. VINS-Fusion 공식 특징 목록

GitHub 공식 저장소는 VINS-Fusion을 "최적화 기반 멀티센서 상태 추정기"로 소개하며 다음 기능을 나열한다.

- 효율적인 IMU pre-integration(바이어스 보정 포함)
- 자동 추정기 초기화(D3-2에서 상세)
- 온라인 외부 파라미터(extrinsic) 캘리브레이션
- 온라인 시간 캘리브레이션(카메라-IMU 시간 오프셋)
- 실패 감지 및 복구(failure detection and recovery)
- Loop detection(D3-4에서 상세)
- 전역 4-DOF pose graph 최적화
- 맵 병합(map merge), pose graph 재사용
- Rolling shutter 카메라 지원

2019년 1월 기준 KITTI Odometry Benchmark 오픈소스 스테레오 알고리즘 중 최상위 성능을 기록했다고 공식 저장소는 밝히고 있다.

## 4. 이 프로젝트와의 관련성 (Yahboom X3)

- 이 프로젝트는 RealSense D435i(RGB-D)를 주력으로 쓰지만, VINS-Fusion은 "stereo cameras only" 모드도 지원한다는 점에서 D435i의 좌우 IR 스테레오 이미지를 활용하는 경로도 이론적으로 가능하다 — 다만 이는 RGB-D 활용과는 다른 통합 방식이라 검증이 필요하다.
- [[ros2-nav-yahboom]]에서 언급된 "IR 스테레오+IMU를 쓰는 다른 R3 백엔드"라는 다음 단계 후보가 바로 이 VINS-Fusion(또는 유사 시스템)을 가리키는 것으로 보인다.

## 5. 진단 관점 (일반적 특성 기준)

- VINS-Fusion은 "실패 감지 및 복구" 기능을 공식적으로 표방하므로, D1(명시적 리셋)과 D2(리셋 없음) 사이의 중간적 동작을 보일 가능성이 있다 — 실제 통합 전까지는 추정에 불과하다.
- Ceres Solver 기반 최적화는 매 프레임마다 반복 연산이 필요해, D2-4에서 다룬 OpenVINS의 "1-step 필터 업데이트"보다 구조적으로 CPU 비용이 높을 가능성이 있다(D4에서 문헌 기반 비교).

## 6. 다음 문서와의 연결

- 다음: **D3-2. 초기화 — Loosely-Coupled Vision-IMU Alignment** — VINS-Fusion(및 그 기반인 VINS-Mono)의 초기화가 D1-2(ORB-SLAM3 MAP 추정), D2-3(OpenVINS static/dynamic)과 어떻게 다른 철학인지 다룬다.

## 7. 참고자료

- Qin, Li, Shen, "VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator," IEEE T-RO 2018 ([arXiv:1708.03852](https://arxiv.org/abs/1708.03852))
- Qin, Cao, Pan, Shen, "A General Optimization-based Framework for Local Odometry Estimation with Multiple Sensors," [arXiv:1901.03638](https://arxiv.org/abs/1901.03638) (VINS-Fusion의 이론적 기반)
- [GitHub HKUST-Aerial-Robotics/VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion) 공식 저장소
- [[ros2-nav-yahboom]] — 다음 SLAM 백엔드 후보로 언급된 이력
