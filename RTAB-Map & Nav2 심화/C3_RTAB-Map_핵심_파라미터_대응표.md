# C3. RTAB-Map 핵심 파라미터 대응표

> RTAB-Map & Nav2 심화 시리즈 · Part C. Relocalization 심화
> 이전 문서: [[C2-1_이_프로젝트의_트리거_구현|C2-1. 이 프로젝트의 트리거 구현]]

## 1. 개요

[[C2-1_이_프로젝트의_트리거_구현|C2-1]]에서 트리거별로 흩어져 언급한 RTAB-Map 파라미터들을 relocalization 관점에서 한곳에 모아 정리한다. A2(매핑 파라미터)와 겹치는 이름이 있지만, 여기서는 **로컬라이제이션 모드에서의 역할**에 집중한다.

## 2. 핵심 개념: 모드 전환이 모든 것의 시작점

```yaml
Mem/IncrementalMemory: false   # 로컬라이제이션 모드 진입
```

이 파라미터가 false가 아니면 애초에 "relocalization"이라는 개념 자체가 성립하지 않는다 — RTAB-Map이 여전히 지도를 계속 확장하는 매핑 모드라면, "위치를 잃었다"는 것은 곧 "그 자리를 새로운 장소로 잘못 추가한다"는 뜻이 되어버리기 때문이다.

## 3. 이 프로젝트에서의 적용: 파라미터 대응표

| 파라미터 | 기본값/예시 | 역할 | C2-1의 어느 트리거와 관련되나 |
|---|---|---|---|
| `Mem/IncrementalMemory` | `false` (로컬라이제이션 모드) | 매핑 확장을 멈추고 기존 지도 안에서만 위치를 찾도록 전환 | 전체의 전제 조건 |
| `RGBD/SavedLocalizationIgnored` | `false`(기본) / `true`(강제 relocalization 시) | true로 설정 시, loop closure가 감지되면 이전에 저장된 localization 값을 무시하고 새로 강하게 relocalization | ④ 랜드마크 보정 |
| `RGBD/OptimizeMaxError` | 10.0 (표준 편차 배수) | 그래프 최적화 후 오차 비율이 이 값을 넘는 loop closure는 거부 — 잘못된(false positive) relocalization 방지 | ③ 누적 드리프트, ② 긴급 relocalization(오탐 방지) |
| `RGBD/OptimizeFromGraphEnd` | `false`(기본) | true로 설정 시 로봇 pose 대신 지도(map) 쪽이 점프하도록 전환 — C4에서 자세히 | 안정화(C4) |
| `Vis/MinInliers` | 환경별 조정 | 재매칭 시 기하학적 검증 최소 기준 — 너무 높으면 ②(긴급 relocalization) 자체가 잘 안 됨 | ② 긴급 relocalization |
| `Rtabmap/LoopThr` | 0.11 | Loop closure 후보 인정 임계값 — 낮추면 재매칭이 더 잘 되지만 오탐 위험도 증가 | ②③④ 전반 |

## 4. 파라미터 간 상충 관계 (튜닝 시 주의)

- `Vis/MinInliers`를 낮춰서 재매칭이 잘 되게 하면, 동시에 `RGBD/OptimizeMaxError`가 걸러내야 할 오탐 후보도 늘어난다. 두 파라미터는 **항상 짝으로 튜닝**해야 하며, 하나만 완화하면 "재매칭은 잘 되는데 가끔 엉뚱한 곳으로 튄다"는 증상이 나타날 수 있다.
- `RGBD/SavedLocalizationIgnored: true`는 랜드마크 재진입 시 강한 보정을 만들지만, 저장된 localization을 매번 무시하므로 **평소 안정적인 추적 상태에서도 불필요하게 자주 relocalization이 발동**할 수 있다. 이 값은 상시 true로 두기보다, C2-1 ④번 트리거 조건(랜드마크 진입)이 성립할 때만 일시적으로 적용하는 방식이 더 안전하다.

## 5. 진단 관점

- "relocalization이 아예 안 일어난다" → `Mem/IncrementalMemory`가 실제로 false인지, `Vis/MinInliers`가 과도하게 높은지 확인.
- "relocalization이 엉뚱한 곳으로 튄다(오탐)" → `RGBD/OptimizeMaxError`가 너무 높게(관대하게) 설정된 것은 아닌지, `Rtabmap/LoopThr`이 너무 낮은지 확인.
- "relocalization 자체는 맞는데 로봇이 덜컥거린다" → 이건 파라미터 문제가 아니라 C4(Pose Jump 제어)의 영역이다.

## 6. 다음 문서와의 연결

- 다음: **C4. Pose Jump 제어와 안정화** — 위 표의 `RGBD/OptimizeFromGraphEnd`를 포함해, relocalization 자체는 맞게 일어나더라도 그 순간의 "점프"를 어떻게 완화할지 다룬다.

## 7. 참고자료

- [rtabmap_ros GitHub 이슈 #1371](https://github.com/introlab/rtabmap_ros/issues/1371) — `RGBD/OptimizeMaxError` 실제 로그와 동작 확인
- [rtabmap_ros GitHub 이슈 #1155](https://github.com/introlab/rtabmap_ros/issues/1155) — `RGBD/SavedLocalizationIgnored` 파라미터 정의 및 용례
- RTAB-Map 공식 파라미터 목록 — [introlab.github.io/rtabmap](https://introlab.github.io/rtabmap/)
