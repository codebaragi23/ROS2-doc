# 05. Nav2 표준 relocalization과의 차이

> 지상로봇 적용 — Relocalization 심화 시리즈
> 이전 문서: [[04_실전_진단_체크리스트|04. 실전 진단 체크리스트]]

## 1. 개요

05·00에서 반복해온 "Nav2 표준 relocalization(AMCL 전용) vs 이 프로젝트의 RTAB-Map 커스텀 전략"의 차이를 한곳에 정리하고, 향후 구조 개선 방향을 짚는다.

## 2. 핵심 개념: 전체 대응표

| 항목 | Nav2 표준 (AMCL 기준) | 이 프로젝트 (RTAB-Map localization) |
|---|---|---|
| relocalization 트리거 방식 | `reinitialize_global_localization` 서비스, BT 액션 `ReinitializeGlobalLocalization`으로 **공식 지원** | 공식 서비스 없음 — 프로젝트가 직접 감시 로직을 만들어 판단(01-1) |
| Kidnapped robot 대응 | 파티클을 지도 전체에 다시 흩뿌려 전역 재탐색(파티클 필터 고유 메커니즘) | Spin Recovery(Nav2 표준 기능을 재활용) + RTAB-Map 자체 loop closure 재탐색에 의존 |
| 초기 위치 지정 | `/initialpose` 토픽, `set_initial_pose` 서비스 | `/rtabmap/initialpose` 토픽 (별도 메커니즘) |
| Pose Jump 특성 | 파티클 필터는 점진적으로 수렴하는 경향이 있어 상대적으로 완만 | [REP105](https://www.ros.org/reps/rep-0105.html)에 따라 loop closure/relocalization 시 이산적 점프 발생 (03) |
| 지도 요구사항 | 2D occupancy grid만 있으면 됨 | RTAB-Map DB(그래프+시각적 특징) 필요 — 지도 제작 단계(지도제작)의 품질이 그대로 relocalization 품질에 영향 |
| 계산 자원 | 상대적으로 가벼움 (2D 파티클 필터) | 시각적 특징 매칭이 포함되어 상대적으로 무거움 — Yahboom X3의 Jetson 자원 제약과 직결([[ros2-nav-yahboom]] CPU 병목 이력 참고) |

## 3. 이 프로젝트에서의 적용: 이 구조를 선택한 이유(추정)와 트레이드오프

이 프로젝트가 AMCL 대신 RTAB-Map localization을 택한 이유는 명시적으로 기록되어 있지는 않지만, 구조상 다음과 같은 트레이드오프가 있다.

- **장점**: RTAB-Map은 매핑과 로컬라이제이션을 같은 파이프라인(같은 DB)으로 처리하므로, 별도로 AMCL용 occupancy grid를 관리하지 않아도 된다. 또한 시각적 특징을 쓰기 때문에 LiDAR만으로는 구분이 안 되는 대칭적인 공간(비슷하게 생긴 복도 여러 개 등)에서 AMCL보다 유리할 수 있다.
- **단점**: Nav2가 공식 제공하는 relocalization 인프라(BT 액션, 서비스)를 그대로 못 쓰기 때문에, 이 시리즈 Relocalization 심화 전체가 다룬 것처럼 **감시·트리거·안정화 로직을 전부 프로젝트가 직접 설계**해야 한다. AMCL이었다면 상당 부분이 `ReinitializeGlobalLocalization` BT 노드 하나로 해결됐을 것이다.

## 4. 관련 파라미터 (시리즈 전체 요약)

| 계층 | 핵심 파라미터 | 관련 문서 |
|---|---|---|
| 매핑 품질 | `Vis/MinInliers`, `Rtabmap/LoopThr`, `Mem/*` | 00~02 |
| Nav2 좌표계/costmap | `robot_radius`, `inflation_radius` | 01~02 |
| Nav2 recovery | `progress_checker`, `goal_checker`, Spin/Wait/BackUp | 05 |
| RTAB-Map relocalization | `Mem/IncrementalMemory`, `RGBD/SavedLocalizationIgnored`, `RGBD/OptimizeMaxError` | 02 |
| Pose Jump 안정화 | `RGBD/OptimizeFromGraphEnd`, `robot_localization` 필터링 | 03 |

## 5. 진단 관점: 향후 고려할 수 있는 방향

- **AMCL 병행**: 만약 시각적 특징이 부족한 환경(어둡거나 텍스처 없는 공간)이 많다면, RTAB-Map 단독보다 LiDAR 기반 AMCL을 별도 또는 보조로 병행하는 구조도 검토할 수 있다. 다만 이 경우 `map→odom`을 누가 최종적으로 발행할지 충돌 문제를 새로 설계해야 한다.
- **감시 로직의 노드화**: 01의 5분류를 실제로 자동 판단하려면, `/rtabmap/info`(inlier 수)와 공분산을 구독해 트리거 조건을 판단하는 전용 ROS2 노드가 필요하다. 현재는 이 로직이 문서(전략)로만 존재하며, 이를 실제 ROS2 노드 설계로 구체화하는 것이 다음 문서 06의 주제다.

## 6. 다음 문서와의 연결

- 다음: **[[06_Relocalization_감시_로직_구현|지상로봇 적용 - Relocalization 심화 - 06. Relocalization 감시 로직 구현]]** — 04에서 언급한 "감시 로직 노드화"를 실제 ROS2 노드 설계로 구체화하며 Relocalization 심화를 마무리한다.
- ORB-SLAM3/OpenVINS/VINS-Fusion처럼 [[ros2-nav-yahboom]]에 기록된 대안 SLAM 백엔드 검토는 별도의 "SLAM 백엔드 심화" 시리즈(01~04)에서 다룬다.

## 7. 참고자료

- [Nav2 — AMCL 설정](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/others/configuring_amcl/) / [`ReinitializeGlobalLocalization`](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/bt_plugins/actions/ReinitializeGlobalLocalization/)
- [RTAB-Map FAQ(공식 Wiki)](https://github.com/introlab/rtabmap/wiki/FAQ) — localization mode와 `Mem/IncrementalMemory`
- [[ros2-nav-yahboom]] — 이 프로젝트의 하드웨어 제약(Jetson) 및 과거 진단 이력

> **문서 버전 주의**: `docs.nav2.org`는 Humble 버전 문서를 더 이상 호스팅하지 않아 위 링크는 Jazzy 기준이다. 개념과 대부분의 파라미터는 동일하지만, 플러그인 목록과 일부 기본값은 Humble과 다를 수 있으므로 실제 설정 시 `ros2 param list`로 대조한다.
