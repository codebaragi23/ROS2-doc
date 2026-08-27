# D2-6. RTAB-Map 통합 구조 — 코드 레벨 분석

> SLAM 백엔드 평가 시리즈
> 이전 문서: D2-5. ZUPT 메커니즘과 자세 안정화
> 이 문서는 프로젝트 내부 자료(`OdometryOpenVINS.cpp`/`.h` 실제 소스 분석)를 근거로 작성됐다 — 일반적인 OpenVINS 문서 지식이 아니라 이 repo에 실제로 빌드된 코드 기준이다.

## 1. 개요

D2-1~D2-5가 OpenVINS 자체의 알고리즘을 다뤘다면, 이 문서는 "그 알고리즘이 RTAB-Map이라는 상위 시스템 안에서 실제로 어떻게 호출되고 연결되는지"를 다룬다. A1~A3(RTAB-Map 매핑)에서 배운 RTAB-Map의 odometry 플러그인 구조에 OpenVINS가 어떻게 꽂히는지 확인하는 문서다.

## 2. 핵심 개념: OdometryOpenVINS의 동작 흐름

```text
OdometryOpenVINS::computeTransform()
├── vioManager_가 없으면(최초): 첫 IMU+이미지 프레임에서
│   카메라 intrinsic/extrinsic을 ROS 토픽에서 읽어와
│   ov_msckf::VioManager를 1회 생성
├── vioManager_가 있으면(이후 매 프레임):
│   ├── IMU만 오면: feed_measurement_imu()
│   └── 이미지가 오면: feed_measurement_camera() 후
│       state->_imu->pos()/quat()에서 현재 pose를 읽어와 반환
```

RTAB-Map의 odometry 인터페이스(A2에서 다룬 파이프라인의 "Odometry" 단계)가 매 프레임 `computeTransform()`을 호출하고, 이 함수 내부에서 OpenVINS의 핵심 객체인 `ov_msckf::VioManager`(D2-1의 MSCKF 필터 본체)를 초기화하거나 갱신한다.

## 3. Pose 반환 로직의 핵심 코드

```cpp
if(!p.isNull() && !p.isIdentity()) {
    ...
    t = previousPoseInv_ * p;
}
```

`state`의 pose가 여전히 identity(초기화 전 기본값)면 `t`는 null로 남고 odometry가 안 나온다 — 이것이 **"초기화 전에는 pose가 안 나온다"의 실체**다. D2-3에서 다룬 정적 초기화가 아직 완료되지 않은 구간에서는, RTAB-Map 쪽에서 봤을 때 OpenVINS가 아무 odometry도 발행하지 않는 "조용한 대기 상태"로 보인다.

## 4. `reset()`의 특이한 동작 — RTAB-Map의 리셋 요청이 무시될 수 있다

```cpp
void OdometryOpenVINS::reset(const Transform & initialPose) {
    if(!initGravity_) {
        vioManager_.reset();  // VioManager 자체를 파괴
        ...
    }
    initGravity_ = false;
}
```

이 코드가 이 문서에서 가장 중요한 부분이다. `initGravity_`가 true(즉 한 번이라도 중력 방향이 잡혀서 D2-3의 초기화가 정상적으로 완료된 적이 있으면), **`reset()`을 호출해도 `vioManager_`가 실제로는 파괴되지 않는다.**

즉 RTAB-Map 상위 레이어(Odometry 기반 클래스)가 "재초기화해라"라고 호출해도, **OpenVINS 래퍼는 한 번 초기화된 뒤엔 사실상 리셋 요청을 무시한다.**

- 이 설계는 의도적으로 보인다 — D2-1에서 배웠듯 MSCKF는 리셋하면 그동안 쌓은 슬라이딩 윈도우 상태 추정을 전부 버리는 셈이라 계산 비용상 "비싼" 작업이다. OpenVINS 통합 코드는 이 비용을 피하려는 것으로 해석할 수 있다.
- 하지만 이는 **RTAB-Map 쪽에서 "지금 상태가 이상하니 재초기화해야 한다"고 판단한 상황에서도 실제로는 리셋이 먹히지 않을 수 있다**는 뜻이다. B8(Nav2와 RTAB-Map의 관계)에서 다룬 "Nav2가 relocalization을 요청해도 실제 백엔드가 이를 받아들이지 않을 수 있다"는 문제의식과 같은 계열의 이슈다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- D2-8에서 다룬 ZUPT 오발동 가설이 만약 사실이라면, 이 `reset()` 무시 동작 때문에 **한 번 "조용한 정지" 상태에 빠지면 RTAB-Map이 아무리 재초기화를 요청해도 스스로 빠져나오지 못할 가능성**이 있다 — 이는 아직 명시적으로 검증되지 않았지만, 코드 구조상 논리적으로 따라오는 우려다.
- 라이브 운영 시 이 특성을 반드시 유의해야 한다: "리셋을 호출했으니 괜찮아지겠지"라는 가정이 OpenVINS 통합에서는 통하지 않을 수 있다.

## 6. 진단 관점

- OpenVINS 기반 odometry가 이상 상태에 빠졌는데 RTAB-Map의 재초기화 명령 이후에도 회복되지 않는다면, 이 `reset()` 동작을 원인으로 의심해야 한다.
- 진짜 재초기화가 필요하다면, RTAB-Map 레벨의 `reset()` 호출이 아니라 **노드 자체를 재시작**해야 `vioManager_`가 실제로 새로 생성될 가능성이 높다(코드 흐름상 `initGravity_`가 초기 상태로 돌아가려면 객체가 새로 만들어져야 하기 때문).

## 7. 다음 문서와의 연결

- 다음: **D2-7. 이 프로젝트에서의 실측 비교와 튜닝 계획** — D2-1~D2-6에서 다룬 모든 개념이 실제로 어떤 수치로 나타났는지, 그리고 다음에 무엇을 검증할지 정리하는 D2 시리즈의 실측 정리 문서다.

## 8. 참고자료

- 프로젝트 내부 자료 — `/workspace/ROS/rtabmap_openvins_ws/src/rtabmap/corelib/src/odometry/OdometryOpenVINS.cpp` 실제 소스 분석
- A2(이 시리즈) — RTAB-Map 매핑 파이프라인에서 Odometry 단계의 역할
