# D1-4. Loop Closing과 Place Recognition

> RTAB-Map & Nav2 심화 시리즈 · Part D. 대안 SLAM 백엔드
> 이전 문서: D1-3. Multi-Map System(Atlas)과 재추적·병합
> 참고 논문: ORB-SLAM3 논문(Campos et al., 2021) 6장 A절(Place Recognition)

## 1. 개요

D1-3에서 "새 키프레임마다 Atlas 전체에서 매칭을 탐색한다"고 했다. 이 문서는 그 탐색이 정확히 어떤 알고리즘(DBoW2 + 기하학적 검증)으로 이루어지는지, 그리고 ORB-SLAM3가 기존 방식(ORB-SLAM2 포함) 대비 recall을 어떻게 개선했는지 정리한다. 이는 A1(RTAB-Map의 appearance-based loop closure)과 원리는 유사하지만 구현이 다른 지점이라 대비해서 이해하면 좋다.

## 2. 핵심 개념: 기존 DBoW2 방식의 한계

논문은 먼저 고전적인 DBoW2 기반 place recognition의 한계를 짚는다.

- DBoW2는 키프레임들의 bag-of-words 벡터로 데이터베이스를 구축하고, 쿼리 이미지가 들어오면 가장 유사한 키프레임들을 효율적으로 찾아준다.
- **가장 유사한 후보 1개만** 사용하는 raw DBoW2 쿼리는 precision·recall이 **50~80%** 수준에 그친다.
- 잘못된 매칭(false positive)이 지도를 오염시키는 것을 막기 위해 DBoW2는 **시간적(temporal) + 기하학적(geometric) 일관성 검사**를 추가로 거치는데, 이렇게 하면 precision은 100%까지 올라가지만 **recall은 30~40%로 떨어진다.**
- 결정적으로, 이 시간적 일관성 검사는 **최소 3개 키프레임에 걸쳐 지연**된다 — 즉 매칭이 실제로 맞더라도 인정받기까지 시간이 걸린다.
- 논문 저자들은 이 지연과 낮은 recall이 **Atlas(다중 지도) 시스템에서 같은/다른 지도 안에 중복된 영역**을 너무 자주 만든다는 것을 발견했다.

## 3. ORB-SLAM3의 개선된 Place Recognition

### 3.1 더 많은 후보를 보는 것으로 Recall 개선

- 새 키프레임이 생성될 때마다, DBoW2 데이터베이스에서 **여러 개의 유사 후보**를 한 번에 조회한다(1개가 아니라).
- 각 후보는 **여러 단계의 기하학적 검증**을 거쳐 100% precision을 유지한다.
- 검증의 기본 연산: 이미지 윈도우 안에서 ORB keypoint의 descriptor가 map point의 ORB descriptor와 매칭되는지, **Hamming distance 임계값**으로 확인한다.
- 검색 윈도우 안에 후보가 여럿이면 모호한 매칭을 걸러내기 위해 **2순위 후보와의 거리 비율(distance ratio)**을 함께 검사한다.

### 3.2 연속 키프레임 요구 제거 + 즉시 3D 정합

- ORB-SLAM3의 place recognition은 **기하학적 검증에 연속된 키프레임 간 합의가 필요하지 않다** — 이것이 기존 DBoW2의 "3키프레임 지연" 문제를 없앤 지점이다.
- 대신 매칭 후보가 나오면 **Sim(3)(모노큘러) 또는 SE(3)(스테레오/RGB-D) 직접 정합**을 즉시 시도한다.
- 정합에 성공하면 D1-3에서 다룬 **welding window**로 이어져 주변 covisible 키프레임까지 중기 데이터 연관을 확장한다.

## 4. RTAB-Map(A1)과의 대비

| 항목 | RTAB-Map (A1) | ORB-SLAM3 |
|---|---|---|
| 검색 기반 | 자체 appearance-based bag-of-words, `Rtabmap/LoopThr` 유사도 임계값 | DBoW2, 다중 후보 조회 |
| 검증 방식 | `Vis/MinInliers` 기하학적 검증 | Hamming distance + distance ratio 기반 다단계 검증 |
| 지연 여부 | WM/STM 관리 안에서 즉시 판단 | 기존 DBoW2의 3키프레임 지연을 제거 — 즉시 Sim(3)/SE(3) 시도 |
| 매칭 후 확장 | 그래프 최적화로 전체 반영 | welding window로 국소 확장 후 pose-graph 최적화 |

두 시스템 모두 "정밀도(precision)를 유지하면서 recall을 얼마나 높이는가"가 설계의 핵심 긴장 관계라는 점은 동일하다 — C3에서 다룬 RTAB-Map의 `RGBD/OptimizeMaxError`(오탐 거부)도 이 긴장 관계를 다루는 RTAB-Map식 해법이다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- RGB-D 모드에서는 SE(3) 직접 정합이 쓰이며, 모노큘러의 스케일 모호성 문제가 없다는 점이 D1-2에서 다룬 초기화의 스케일 추정 부담을 덜어준다.
- 만약 이 프로젝트가 ORB-SLAM3의 loop closing 성능을 RTAB-Map과 비교 평가하려 한다면, 두 시스템의 "후보 수"와 "검증 임계값" 철학이 다르므로 단순히 같은 파라미터명으로 비교할 수 없다는 점을 유의해야 한다.

## 6. 진단 관점

- 같은 장소를 여러 번 지나갔는데도 새로운 지도로 계속 나뉜다면(D1-3), 이 문서의 검증 임계값(Hamming distance, distance ratio)이 텍스처가 부족한 이 프로젝트 환경에 비해 너무 엄격하게 작동하고 있을 가능성을 고려한다.

## 7. 다음 문서와의 연결

- 다음: **D1-5. 이 프로젝트의 VIO 이슈 재해석** — D1-1~D1-4에서 정리한 논문 근거를 [[ros2-nav-yahboom]]의 실제 진단 이력과 종합해, 현재 상태를 다시 정리한다.

## 8. 참고자료

- ORB-SLAM3 논문(Campos et al., 2021) 6장 A절 "Place Recognition"
- Gálvez-López, Tardós, "Bags of Binary Words for Fast Place Recognition in Image Sequences" (DBoW2 원 논문)
