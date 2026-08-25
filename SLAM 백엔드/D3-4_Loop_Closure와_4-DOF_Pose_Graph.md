# D3-4. Loop Closure와 4-DOF Pose Graph

> SLAM 백엔드 평가 시리즈
> 이전 문서: D3-3. Sliding Window 최적화와 Marginalization
> ⚠️ 문헌 기반 문서 — 프로젝트 실측 없음(D3-1 안내 참고)

## 1. 개요

D2-1에서 "OpenVINS는 기본적으로 loop closure가 없다"고 배웠다. VINS-Fusion은 이와 달리 **별도의 노드로 분리된 loop closure 기능**을 제공한다. 이 문서는 그 구조와, D1(ORB-SLAM3)의 통합형 loop closure(D1-3, D1-4)와 어떻게 다른지 다룬다.

## 2. 핵심 개념: 별도 프로세스로 분리된 Loop Fusion

VINS-Fusion 공식 저장소의 실행 예시를 보면 구조가 드러난다.

```bash
rosrun vins vins_node <config.yaml>            # 핵심 추정기(estimator)
rosrun loop_fusion loop_fusion_node <config.yaml>   # (선택) Loop closure 전용 노드
```

D1-1(ORB-SLAM3)에서 배운 구조와 결정적으로 다른 지점: ORB-SLAM3는 Tracking·Local Mapping·Loop Closing이 **하나의 프로세스 안 세 스레드**로 통합되어 있었다. VINS-Fusion은 핵심 상태 추정(`vins_node`, D3-1~D3-3의 슬라이딩 윈도우 최적화)과 **loop closure(`loop_fusion_node`)가 아예 별도의 프로세스로 분리**되어 있고, 이 기능은 선택적으로만 켤 수 있다.

## 3. Loop Closure 동작 방식

문헌에 따르면 VINS-Fusion의 loop closure는 다음과 같이 동작한다(D1-4에서 다룬 ORB-SLAM3의 DBoW2 방식과 같은 계열이다).

- **DBoW2(BRIEF descriptor 기반) bag-of-words**로 과거 키프레임과의 유사도를 검색해 재방문(place recognition)을 탐지한다. (D1-4에서 다룬 ORB-SLAM3의 DBoW2도 같은 라이브러리 계열이지만, VINS-Fusion은 BRIEF descriptor를, ORB-SLAM3는 ORB descriptor를 사용한다는 차이가 있다.)
- 매칭이 확인되면 **4-DOF pose graph 최적화**를 수행해 전역 일관성을 확보한다.

## 4. 왜 "4-DOF"인가 — 6-DOF와의 차이

D1(ORB-SLAM3)의 pose graph 최적화나 A1(RTAB-Map)의 그래프 최적화는 일반적으로 **6자유도(6-DOF: x, y, z, roll, pitch, yaw)** 전체를 최적화 대상으로 삼는다. VINS-Fusion의 pose graph는 **4자유도(x, y, z, yaw)**만 최적화한다.

- IMU가 중력 방향(roll, pitch에 해당하는 정보)을 항상 관측 가능(observable)하게 만들어주기 때문에, **roll과 pitch는 이미 충분히 정확하다고 보고 최적화 대상에서 제외**한다는 것이 이 설계의 논리다.
- 이렇게 하면 최적화해야 할 변수가 줄어들어 pose graph 최적화가 더 가볍고 빨라진다 — VINS-Mono 논문은 이를 "loop detection과 결합된 tightly-coupled 구조 덕분에 최소한의 계산 비용으로 relocalization이 가능하다"고 설명한다.

## 5. Map Merge와 Pose Graph 재사용

D3-1에서 언급했듯 VINS-Fusion은 맵 병합과 pose graph 저장·재사용(reuse)을 지원한다 — 이는 D1-3(ORB-SLAM3의 Atlas 병합)과 목적은 유사하지만, VINS-Fusion에서는 이 기능이 loop closure와 마찬가지로 **별도 노드가 담당하는 선택적 기능**이라는 점이 구조적 차이다.

## 6. 세 시스템의 Loop Closure 구조 비교

| | ORB-SLAM3(D1) | OpenVINS(D2) | VINS-Fusion(D3) |
|---|---|---|---|
| Loop Closure 존재 여부 | 있음, 시스템에 통합 | 기본적으로 없음(순수 odometry) | 있음, 별도 프로세스로 분리(선택적) |
| Place Recognition | 개선된 DBoW2(D1-4) | 해당 없음 | DBoW2(BRIEF) |
| Pose Graph 자유도 | 6-DOF(전체) | 해당 없음 | 4-DOF(x,y,z,yaw만) |
| 다중 지도 관리 | Atlas(D1-1, D1-3) | 없음 | Map merge/reuse(별도 기능) |

## 7. 이 프로젝트와의 관련성 (가상 적용 시나리오)

- D2-1에서 "OpenVINS는 loop closure가 없어 순수 odometry로만 쓰면 드리프트가 누적된다"고 지적했다. 이 프로젝트는 현재 RTAB-Map(A1)이 loop closure와 지도 관리를 전담하고 OpenVINS는 odometry 소스로만 통합되어 있으므로(D2-6), 이 구조에서는 OpenVINS 자체에 loop closure가 없어도 RTAB-Map이 이를 보완한다.
- 만약 VINS-Fusion을 도입한다면, `loop_fusion_node`를 켤지 끌지에 따라 "RTAB-Map의 loop closure와 VINS-Fusion 자체의 loop closure가 중복되는" 상황이 생길 수 있다 — OpenVINS와 달리 이 중복/충돌 가능성을 설계 단계에서 고려해야 한다는 점이 VINS-Fusion 도입 시의 특이점이다.
- C6(RTAB-Map & Nav2 심화 시리즈)에서 다룬 "relocalization을 누가 담당하는가"라는 질문이, VINS-Fusion을 쓸 경우 RTAB-Map과 VINS-Fusion(`loop_fusion_node`) 중 **어느 쪽의 loop closure를 최종 신뢰할지**라는 새로운 형태의 질문으로 바뀐다.

## 8. 다음 문서와의 연결

- 다음: **D3-5. 이 프로젝트에 적용한다면** — D3 시리즈 전체를 마무리하며, 실제 통합 시 예상되는 고려사항과 미해결 질문을 정리한다.

## 9. 참고자료

- Qin, Li, Shen, "VINS-Mono," IEEE T-RO 2018 — 4-DOF pose graph 최적화 설계 근거
- [GitHub HKUST-Aerial-Robotics/VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion) — `loop_fusion_node` 실행 구조
- D1-4(이 시리즈) — ORB-SLAM3의 통합형 Place Recognition과의 대비
