# B1. Nav2 아키텍처 개요

> 지상로봇 적용 — Nav2 내비게이션 (Part B) 시리즈
> 이전 문서: A3. 매핑 품질 진단

## 1. 개요

Part A에서 만든 지도를 가지고 실제로 로봇을 목적지까지 움직이는 것이 Nav2의 역할이다. 이 문서는 Nav2 전체 구조를 큰 그림으로 먼저 잡고, B2~B7에서 각 구성요소를 하나씩 깊게 다룬다.

## 2. 핵심 개념: 전체 흐름

```text
목표 Pose
  → bt_navigator          (행동 흐름 관리)
  → planner_server         전역 경로 생성
  → controller_server      로컬 경로 추종, cmd_vel 생성
  → velocity_smoother       속도 명령 완화
  → 로봇 구동부

센서/지도
  → global_costmap    전역 경로 계획용 비용 지도
  → local_costmap     근거리 장애물 회피용 비용 지도
  → RTAB-Map localization    map/odom 관계 보정
```

Nav2는 하나의 거대한 프로그램이 아니라, 각자 역할이 분리된 여러 **서버(Lifecycle Node)**가 함께 동작하는 구조다. 각 서버는 독립적으로 실행되고, `bt_navigator`가 이들을 Behavior Tree라는 규칙에 따라 호출한다.

| 서버 | 역할 | 이 시리즈에서 다루는 문서 |
|---|---|---|
| `bt_navigator` | 전체 행동 흐름(경로 계획→추종→도착 판정→recovery) 관리 | B6 |
| `planner_server` | 전역 경로 생성 | B4 |
| `controller_server` | 로컬 경로 추종, 실제 속도 명령 생성 | B5 |
| `behavior_server` | recovery 동작(spin/backup/wait 등) 실행 | B6 |
| `waypoint_follower` | 여러 목표점을 순차 이동 | B7 |
| `costmap` (global/local) | 장애물 회피용 비용 지도 | B3 |
| `velocity_smoother` | 최종 속도 명령 완화 | B7 |

이 목록에 **AMCL이 없다는 점**이 중요하다. Nav2 표준 예제(nav2_bringup)는 대부분 AMCL을 로컬라이제이션으로 쓰지만, 이 프로젝트는 **RTAB-Map localization**이 `map→odom` TF를 대신 채워준다. Nav2 입장에서는 이 TF가 어디서 오든 상관없이 소비만 하므로 위 서버 목록에서 로컬라이제이션 자체는 별도로 존재하지 않는다 — 이 구조는 B2에서 자세히 다룬다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

과거 진단에서 `/bt_navigator`, `/controller_server`, `/planner_server`, `/waypoint_follower` 노드가 아예 실행되지 않아 Nav2 패널이 `unknown`으로 표시된 사례가 있었다([[ros2-nav-yahboom]] 참고). 원인은 `navigation_rtabmap_launch.py` 안에서 `rtabmap_nav_launch`가 주석 처리되어 있었던 것 — 즉 **위 표의 서버들이 아예 실행되지 않으면 Nav2는 그냥 아무것도 하지 않는다**는 점을 실제로 보여준 사례다.

```bash
# Nav2가 정상 동작 중인지 가장 먼저 확인할 명령
ros2 node list
# /bt_navigator, /controller_server, /planner_server, /behavior_server, /waypoint_follower
# 가 모두 보여야 함
```

## 4. 관련 파라미터

전체 구조를 결정하는 상위 설정들만 정리한다. 세부 파라미터는 각 담당 문서(B3~B7)에서 다룬다.

| 설정 | 값(이 프로젝트) | 의미 |
|---|---|---|
| `global_frame` (bt_navigator) | `map` | 목표/전역 경로의 기준 좌표계 |
| `robot_base_frame` | `base_footprint` | Nav2가 로봇 중심으로 쓰는 좌표계 |
| `odom_topic` | `odom` | 로컬 속도 추정에 사용하는 오도메트리 토픽 (EKF 출력) |

## 4-1. Lifecycle Node — "떠 있는데 반응이 없는" 이유

2장에서 Nav2 서버들을 **Lifecycle Node**라고 불렀는데, 이것이 Nav2 진단에서 대단히 중요하다.

일반 ROS2 노드는 실행되면 곧바로 일한다. 반면 Lifecycle Node는 **상태를 가지며, 활성화되기 전에는 토픽이나 액션 요청을 처리하지 않는다.**

```text
unconfigured ──configure──▶ inactive ──activate──▶ active   ← 여기서만 실제로 동작
```

**핵심**: `ros2 node list`에 노드가 보여도 **`active` 상태가 아니면 아무 일도 하지 않는다.** 목표를 보내도 조용히 무시되며, 에러 메시지도 딱히 나오지 않는다. 이것이 실무에서 "Nav2가 무반응"인 가장 흔한 원인이다.

Nav2에서는 `lifecycle_manager`라는 노드가 이 전환을 자동으로 관리하는데, **이 매니저가 죽었거나 일부 서버 전환에 실패하면 나머지가 통째로 `inactive`에 머무른다.**

```bash
# 각 서버의 현재 lifecycle 상태 확인
ros2 lifecycle get /planner_server
ros2 lifecycle get /controller_server
ros2 lifecycle get /bt_navigator
# → "active [3]" 이어야 정상. "inactive [2]"면 활성화가 안 된 것
```

## 4-2. 실습: 빈 지도에서 목표 하나 보내보기

Part B 전체가 "목표를 주면 로봇이 간다"는 흐름을 다루므로, **가장 단순한 형태로 한 번 성공시켜 보는 것**이 이후 B3~B7을 읽는 기준점이 된다.

### 준비 사항

* A2 실습에서 만든 지도, B2 실습의 로컬라이제이션 모드 실행 상태
* Nav2 실행 중

### 실행 및 확인

```bash
# 1) 먼저 모든 서버가 active인지 확인 (4-1절)
ros2 lifecycle get /bt_navigator

# 2) RViz2에서 "2D Goal Pose" 버튼으로 목표 찍기
#    또는 명령줄로 직접:
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}}" \
  --feedback
```

* `--feedback`을 붙이면 남은 거리 등 진행 상황이 실시간으로 출력된다 — B1-1에서 다룰 Action 피드백이 실제로 흐르는 것을 볼 수 있다.

### 예상 결과

| 결과 | 의미 | 다음에 볼 문서 |
|---|---|---|
| 로봇이 목표까지 이동 | 파이프라인 전체가 정상 | B3부터 순서대로 |
| `Goal was rejected` | 서버가 `active`가 아니거나 목표가 지도 밖 | 4-1절 |
| 경로는 나오는데 안 움직임 | 컨트롤러 또는 costmap 문제 | B3, B5 |
| 경로 자체가 안 나옴(`plan=0`) | 목표가 장애물/미탐색 영역에 있음 | B3, B4 |

## 5. 진단 관점

Nav2가 "unknown" 또는 무반응일 때 확인 순서:

1. `ros2 node list`로 핵심 서버들이 다 떠 있는지 확인 (없으면 Launch 파일부터 점검)
2. 노드는 떠 있는데 반응이 없다면 **`ros2 lifecycle get`으로 `active` 상태인지 확인**(4-1절) — 가장 흔한 원인이다
3. 상태도 정상이라면 TF 트리(`map→odom→base_footprint`)가 끊겨 있지 않은지 확인
4. TF도 정상인데 목표를 줘도 안 움직인다면 costmap에 로봇 위치 전체가 장애물로 덮여 있지 않은지 RViz로 확인 (footprint/robot_radius 설정 오류의 전형적 증상)

## 6. 다음 문서와의 연결

- 다음: **B2. 좌표계와 로컬라이제이션 기반** — 이 문서에서 "RTAB-Map localization이 별도 서버 없이 TF만 채워준다"고 언급한 부분을 자세히 다룬다.
- Part C(Relocalization 심화)는 이 B1의 흐름도 중 "RTAB-Map localization" 박스 안에서 일어나는 일을 깊게 파고드는 것이라고 생각하면 된다.

## 7. 참고자료

- Nav2 공식 문서([docs.nav2.org](https://docs.nav2.org/)) — Nav2 시스템 아키텍처, 각 서버의 역할 정의
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 파라미터 값과 튜닝 관점
