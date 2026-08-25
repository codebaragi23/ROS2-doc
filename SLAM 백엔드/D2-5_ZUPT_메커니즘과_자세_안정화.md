# D2-5. ZUPT 메커니즘과 자세 안정화

> RTAB-Map & Nav2 심화 시리즈 · Part D. 대안 SLAM 백엔드 (SLAM 백엔드 평가)
> 이전 문서: D2-4. 상태 표현 — Sliding Window와 MSCKF/SLAM Feature 이원화
> 이 문서는 [[ros2-nav-yahboom]]에 기록된 **현재 검증 대기 중인 최우선 가설**을 다룬다 — D1(ORB-SLAM3)과 마찬가지로 결론이 확정되지 않은 진행형 내용이다.

## 1. 개요

D1 시리즈(ORB-SLAM3)에는 없는 OpenVINS만의 기능이 **ZUPT(Zero velocity UPdaTe)**다. 로봇이 정지했다고 판단되면 "속도=0"이라는 강한 제약을 필터에 인위적으로 주입해 드리프트를 억제하는 기법이다. 이 문서는 이 기능의 원리와, 이 프로젝트에서 이것이 오작동할 수 있다는 미검증 가설을 정리한다.

## 2. 핵심 개념: ZUPT란 무엇인가

관성항법(INS) 분야에서 오래전부터 쓰인 기법으로, 보행자 항법(pedestrian dead reckoning) 등에서 "발이 땅에 닿아 정지한 순간"을 감지해 속도 오차를 주기적으로 리셋하는 것이 원조다. VIO에서도 같은 원리를 적용한다:

- 로봇/센서가 실제로 멈춰 있다고 판단되면, "지금 속도는 0이다"라는 매우 확실한(불확실성이 낮은) 관측을 필터에 추가로 넣어준다.
- 정지 중에는 카메라의 시각 정보만으로 미세한 드리프트를 완전히 막기 어려운데, ZUPT는 이 구간의 드리프트를 강하게 억제하는 효과가 있다.

## 3. RTAB-Map 통합 파라미터 (이 프로젝트 기준)

```yaml
OdomOpenVINS/TryZUPT:             true   # 정지 시 zero-velocity 업데이트 사용
OdomOpenVINS/ZUPTMaxVelodicy:      0.1   # 이 속도 이하일 때만 ZUPT 시도
OdomOpenVINS/ZUPTMaxDisparity:     0.5   # 이 disparity(시차) 이하일 때만 ZUPT 시도
OdomOpenVINS/ZUPTNoiseMultiplier: 10.0
OdomOpenVINS/ZUPTOnlyAtBeginning: false  # 시작 시점뿐 아니라 정지할 때마다 매번 발동
```

**기본값이 ON**이고, `ZUPTOnlyAtBeginning: false`라서 **로봇이 정지할 때마다 매번** 이 강한 제약이 발동한다는 점이 중요하다.

## 4. 이 프로젝트에서 제기된 가설: ZUPT 오발동

**가설**: `ZUPTMaxDisparity=0.5`가 비교적 낮은 값이라, 저텍스처 근접 장면(예: 냉장고 표면처럼 시차 신호 자체가 약한 상황)에서는 **로봇이 실제로 움직이고 있어도 이미지 disparity가 낮게 측정되어 ZUPT가 오발동**할 수 있다는 것이다.

만약 이 가설이 맞다면:

1. 로봇은 실제로 움직이고 있다.
2. 그런데 저텍스처 장면 때문에 시각적 disparity가 낮게 측정된다.
3. `ZUPTMaxDisparity` 조건을 만족한다고 필터가 착각한다.
4. "정지 상태"로 판단해 속도=0을 강제로 주입한다.
5. 결과적으로 실제 이동 데이터(`step_translation_m`)가 대부분 0에 가깝게 기록된다.

이는 특정 bag(bag_c로 지칭됨)에서 실제로 관찰된 "조용히 멈춰있는" 현상과 일치한다. D2-1에서 짚었듯 OpenVINS는 ORB-SLAM3처럼 "Fail to track"이라는 명시적 오류를 내지 않으므로, 이 현상은 **겉으로는 정상 동작처럼 보이면서 실제로는 멈춰 있는** 까다로운 형태로 나타난다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- 이 가설은 **아직 검증되지 않았다.** 검증 방법은 `TryZUPT: false`로 끄고 문제가 됐던 bag을 재생해, `step_translation_m`의 분포가 정상 범위(0.01m대)로 돌아오는지 확인하는 것이다.
- 가설이 확정되면, ZUPT를 완전히 끄는 대신 `ZUPTMaxDisparity`를 더 낮춰서(예: 0.2) 오발동만 줄이는 절충안을 시도하는 것이 다음 단계로 계획되어 있다(D2-7의 튜닝 후보 4-1 참고).

## 6. D1과의 대비: "리셋이 없다"의 양면성

D2-1에서 "OpenVINS는 명시적 lost 상태가 없다"고 배웠는데, 이 문서의 ZUPT 오발동 가설이 바로 그 양면성을 구체적으로 보여준다. ORB-SLAM3(D1-1)라면 이런 상황에서 "Fail to track"을 로그에 남기고 리셋했을 것이다. OpenVINS는 그런 명시적 신호 없이 조용히 "정지 상태"로 오판한 채 계속 실행되므로, **더 견고해 보이지만 실제로는 진단하기 더 까다로운 실패 모드**일 수 있다.

## 7. 진단 관점

- 로그나 메트릭에서 `initialization_reset_count`, `post_init_coverage.ratio` 같은 지표만으로는 이 문제를 잡을 수 없다(모두 정상으로 보인다) — 반드시 `step_translation_m`의 p50/mean 같은 실제 이동량 지표를 함께 확인해야 한다(D2-7에서 상세 다룸).
- 라이브 로봇 운영에서는 이 현상이 "RViz 상 로봇이 안 움직이는 것처럼 보이는" 형태로 나타날 가능성이 높다.

## 8. 다음 문서와의 연결

- 다음: **D2-6. RTAB-Map 통합 구조 — 코드 레벨 분석** — 지금까지 다룬 OpenVINS 개념들이 실제로 `OdometryOpenVINS.cpp`에서 어떻게 RTAB-Map과 연결되는지 다룬다.

## 9. 참고자료

- ZUPT의 관성항법 분야 원류 개념 — pedestrian dead reckoning 관련 문헌
- 프로젝트 내부 자료 — RTAB-Map `OdomOpenVINS` ZUPT 파라미터 정의 및 이 프로젝트의 실측 이력(bag_c 현상)
- [[ros2-nav-yahboom]] — 최우선 검증 대기 항목으로 기록됨
