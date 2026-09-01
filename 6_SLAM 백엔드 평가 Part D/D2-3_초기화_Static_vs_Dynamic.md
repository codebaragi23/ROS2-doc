# D2-3. 초기화: Static vs Dynamic

> SLAM 백엔드 평가 시리즈
> 이전 문서: D2-2. OpenVINS의 핵심 설계
> 이 문서는 프로젝트 내부 코드 분석 자료(RTAB-Map `OdomOpenVINS` 파라미터)와 D1-2(ORB-SLAM3 초기화 알고리즘)의 비교를 결합해 작성했다.

## 1. 개요

D1-2에서 ORB-SLAM3가 "충분히 움직였는가"를 게이트로 삼아 초기화를 시작한다고 배웠다. OpenVINS는 **기본값이 정반대**다 — "충분히 가만히 있었는가"를 기준으로 삼는다. 이 문서는 왜 이런 차이가 생기는지, 그리고 이 프로젝트에서 어떤 방식을 쓰고 있는지 정리한다.

## 2. 핵심 개념: Static Initialization

OpenVINS의 기본 초기화 방식은 **정적(static) 초기화**다.

- 로봇이 **가만히 정지한 짧은 구간**의 IMU 가속도 데이터를 관찰한다.
- 정지 상태에서 가속도계가 측정하는 값은 이론적으로 **중력 벡터 하나뿐**이어야 한다(움직임에 의한 가속도가 없으므로).
- 이 원리를 이용해, 정지 구간의 가속도 평균으로 **중력 방향**을 추정하고 시스템을 시작한다.

핵심 판정 기준은 "이 구간이 정말 정지 상태인가"이며, 이는 가속도의 **분산(variance)**으로 판단한다 — 분산이 작을수록(가속도 값이 안정적일수록) "가만히 있다"고 본다.

## 3. RTAB-Map 통합 파라미터 (이 프로젝트 기준)

```yaml
OdomOpenVINS/InitWindowTime:    2.0   # 초기화에 쓸 시간(초)
OdomOpenVINS/InitIMUThresh:     1.0   # 가속도 분산(variance) 임계값 -- 이 값 미만이어야 "정지"로 판정
OdomOpenVINS/InitMaxDisparity: 10.0   # 정지 상태로 볼 최대 이미지 disparity(시차)
OdomOpenVINS/InitDynUse:       false  # 동적(모션 기반) 초기화 사용 여부
```

- `InitIMUThresh`(1.0)는 언뜻 ORB-SLAM3의 `InitMinAcceleration`(D1-2에서 다룬 가속도 게이트, 기본 0.5)과 비슷해 보이지만 **측정 단위 자체가 다르다.**

| 시스템 | 무엇을 보는가 | 판정 기준 |
|---|---|---|
| ORB-SLAM3 (`InitMinAcceleration`) | 가속도의 **크기(magnitude)** | "충분히 세게 움직였는가?" |
| OpenVINS (`InitIMUThresh`, 기본 static) | 가속도의 **분산(variance)** | "충분히 안정적으로 정지해 있는가?" |

즉 두 시스템은 IMU 초기화를 시작하는 **철학 자체가 반대**다. ORB-SLAM3(D1-2)는 "움직임에서 중력·바이어스 정보를 뽑아내자"는 접근이고, OpenVINS(기본값)는 "정지 상태에서 중력만 먼저 확실히 잡고 시작하자"는 접근이다.

## 4. Dynamic Initialization (동적 초기화)

`InitDynUse: true`로 전환하면 OpenVINS도 "움직이는 중에 초기화"하는 방식으로 바뀐다 — 개념적으로 D1-2에서 다룬 ORB-SLAM3의 모션 기반 초기화(FastInit)와 더 가까워진다. 관련 파라미터가 `InitDyn*` 접두어로 12개 존재하며, 슬라이딩 윈도우 내 pose 개수, 방향 변화량, MLE(Maximum Likelihood Estimation) 반복 횟수 등을 설정한다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- **이 repo에서는 `InitDynUse`를 켜본 이력이 없다** — 지금까지 항상 정적 초기화(기본값)로만 동작해왔다.
- 정적 초기화는 "로봇이 켜질 때 가만히 있는 상황"을 전제한다. 그런데 실제 운영에서는 로봇이 이미 켜진 채로 방 안을 돌아다니다가 VIO가 뒤늦게 붙는 경우가 흔할 수 있다 — 이런 시나리오에서는 정적 초기화가 불리하다.
- D1-2/D1-5에서 ORB-SLAM3가 "움직여야 시작한다"는 이유로 겪은 문제와, OpenVINS가 "가만히 있어야 시작한다"는 이유로 겪을 수 있는 문제는 **정반대 성격의 리스크**라는 점이 흥미로운 대비점이다.

## 6. 진단 관점

- OpenVINS 기반 시스템이 켜지자마자 로봇이 이미 움직이고 있는 상황이라면, 정적 초기화 조건(`InitIMUThresh` 미만의 낮은 분산)을 만족하는 구간이 나타나지 않아 **초기화 자체가 계속 지연**될 수 있다. 이런 증상이 보이면 `InitDynUse: true` 전환을 우선 검토 대상으로 삼는다(D2-7의 튜닝 후보 4-2와 연결).
- 반대로 초기화는 잘 되는데 이후 동작이 불안정하다면, 초기화 문제가 아니라 D2-8(ZUPT 오발동 가설)이나 D2-4(feature 관리)의 문제일 가능성이 높다.

## 7. 다음 문서와의 연결

- 다음: **D2-4. 상태 표현 — Sliding Window와 MSCKF/SLAM Feature 이원화** — D2-1에서 소개한 슬라이딩 윈도우가 이 프로젝트에서 실제로 어떤 크기·구성으로 운영되는지 다룬다.

## 8. 참고자료

- Geneva et al., "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 ([DOI: 10.1109/ICRA40945.2020.9196524](https://doi.org/10.1109/ICRA40945.2020.9196524))
- 프로젝트 내부 자료 — RTAB-Map `OdomOpenVINS` 파라미터 정의(`Parameters.h`) 및 코드 분석
- [[D1-2_Visual-Inertial_초기화_알고리즘|D1-2. Visual-Inertial 초기화 알고리즘]] — ORB-SLAM3의 대조되는 초기화 철학
