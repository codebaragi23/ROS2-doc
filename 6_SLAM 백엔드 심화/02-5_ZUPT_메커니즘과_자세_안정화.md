# 02-5. ZUPT 메커니즘과 자세 안정화

> SLAM 백엔드 심화 시리즈
> 이전 문서: [[02-4_상태_표현_Sliding_Window와_Feature_이원화|02-4. 상태 표현 — Sliding Window와 MSCKF]]/SLAM Feature 이원화

## 1. 개요

01 시리즈(ORB-SLAM3)에는 없는 OpenVINS만의 기능이 **ZUPT(Zero velocity UPdaTe)**다. 로봇이 정지했다고 판단되면 "속도=0"이라는 강한 제약을 필터에 인위적으로 주입해 드리프트를 억제하는 기법이다. 이 문서는 이 기능의 일반적인 원리와 파라미터를 다룬다. 이 기법이 이 프로젝트에서 오작동할 수 있다는 미검증 가설은 별도 문서 **02-8**에서 다룬다.

## 2. 핵심 개념: ZUPT란 무엇인가

관성항법(INS) 분야에서 오래전부터 쓰인 기법으로, 보행자 항법(pedestrian dead reckoning) 등에서 "발이 땅에 닿아 정지한 순간"을 감지해 속도 오차를 주기적으로 리셋하는 것이 원조다. VIO에서도 같은 원리를 적용한다:

- 로봇/센서가 실제로 멈춰 있다고 판단되면, "지금 속도는 0이다"라는 매우 확실한(불확실성이 낮은) 관측을 필터에 추가로 넣어준다.
- 정지 중에는 카메라의 시각 정보만으로 미세한 드리프트를 완전히 막기 어려운데, ZUPT는 이 구간의 드리프트를 강하게 억제하는 효과가 있다.

## 3. 관련 파라미터 (RTAB-Map `OdomOpenVINS` 통합 기준)

```yaml
OdomOpenVINS/TryZUPT:             true   # 정지 시 zero-velocity 업데이트 사용
OdomOpenVINS/ZUPTMaxVelodicy:      0.1   # 이 속도 이하일 때만 ZUPT 시도
OdomOpenVINS/ZUPTMaxDisparity:     0.5   # 이 disparity(시차) 이하일 때만 ZUPT 시도
OdomOpenVINS/ZUPTNoiseMultiplier: 10.0
OdomOpenVINS/ZUPTOnlyAtBeginning: false  # 시작 시점뿐 아니라 정지할 때마다 매번 발동
```

* `TryZUPT`: ZUPT 기능 자체를 켜고 끄는 스위치. **기본값이 ON**이다.
* `ZUPTMaxVelocity`: 이 값보다 추정 속도가 낮을 때만 "정지"로 판단해 ZUPT를 시도한다. 값이 클수록 더 쉽게 정지로 판단한다.
* `ZUPTMaxDisparity`: 연속 프레임 간 이미지 시차(disparity)가 이 값 이하일 때만 "정지"로 판단하는 보조 조건이다. 카메라가 실제로 안 움직였다면 시차도 작을 것이라는 가정에 기반한다.
* `ZUPTNoiseMultiplier`: ZUPT로 주입하는 "속도=0" 관측의 불확실성(노이즈) 크기를 조절한다.
* `ZUPTOnlyAtBeginning`: `false`면 정지할 때마다 매번 ZUPT가 발동하고, `true`면 초기화 직후에만 적용된다.

**설계상 주의점**: `ZUPTMaxDisparity`처럼 "이미지 시차가 작다"를 "정지했다"의 근거로 삼는 방식은, 저텍스처 환경(시차 신호 자체가 약한 장면)에서는 **실제로는 움직이고 있어도 정지로 오판할 위험**을 구조적으로 안고 있다. 이는 이 특정 프로젝트만의 문제가 아니라 disparity 기반 ZUPT 트리거 설계 전반의 일반적인 트레이드오프다.

## 4. 이 프로젝트에서의 적용 (Yahboom X3)

이 프로젝트는 RTAB-Map의 `OdomOpenVINS` 통합을 통해 위 파라미터를 기본값(ZUPT ON)으로 사용하고 있다. 저텍스처 근접 장면에서 ZUPT가 오발동해 "조용히 멈춰있는" 현상으로 이어질 수 있다는 가설이 제기되어 현재 검증 대기 중이다 — 관찰된 현상, 가설의 세부 근거, 검증 계획은 **[[02-8_이_프로젝트의_ZUPT_오발동_가설과_검증_계획|SLAM 백엔드 심화 - 02-8. 이 프로젝트의 ZUPT 오발동 가설과 검증 계획]]**에서 다룬다.

## 5. 진단 관점 (일반)

- ZUPT가 관여하는 시스템에서는 `initialization_reset_count` 같은 "실패 신호" 지표만으로는 문제를 잡기 어렵다 — ZUPT 오발동은 명시적 실패가 아니라 "정상처럼 보이는 정지"로 나타나기 때문이다.
- 실제 이동 거리(예: 프레임 간 translation 크기) 지표를 함께 확인해야, ZUPT가 정말 정지 구간에서만 발동했는지 검증할 수 있다.
- ZUPT를 완전히 끄는 것과, 트리거 조건(`ZUPTMaxDisparity` 등)만 더 엄격하게 좁히는 것은 서로 다른 대응이다 — 전자는 정지 구간 드리프트 억제 효과 자체를 포기하는 것이고, 후자는 오발동 가능성만 줄이는 절충안이다.

## 6. 다음 문서와의 연결

- 다음: **02-6. RTAB-Map 통합 구조 — 코드 레벨 분석** — 지금까지 다룬 OpenVINS 개념들이 실제로 `OdometryOpenVINS.cpp`에서 어떻게 RTAB-Map과 연결되는지 다룬다.
- 이 프로젝트에서 이 기능이 실제로 문제를 일으키고 있는지에 대한 사례 연구는 **02-8**에서 이어진다.

## 7. 참고자료

- ZUPT의 관성항법 분야 원류 개념 — pedestrian dead reckoning 관련 문헌
- 프로젝트 내부 자료 — RTAB-Map `OdomOpenVINS` ZUPT 파라미터 정의(`Parameters.h`)
