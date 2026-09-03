# 05. Launch 시스템 디버깅

> ROS2 응용 & 센서 연동 시리즈

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 응용 (노드가 여러 개가 된 뒤 필요해지는 문서) |
| 예상 선행 지식 | [[06_Launch_파일_작성법\|ROS2 기초 06. Launch 파일 작성법]], [[09_문제_해결_자주_발생하는_오류_모음\|09. 문제 해결]] |
| 학습 목표 | Launch 실패를 세 갈래(미실행/즉시종료/순서문제)로 나눠 진단할 수 있다 / 오류 메시지가 없는데도 노드가 안 뜨는 경우의 원인을 찾을 수 있다 / 이벤트 핸들러로 노드 실행 순서를 강제할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | 이전: [[04_colcon_빌드_오류_모음\|응용 04. colcon 빌드 오류]] / 다음: [[06_Composable_Node와_Executor_구조\|응용 06. Composable Node와 Executor]] |

## 1. 개요

ROS2 기초 06은 "노드 하나가 죽어도 나머지는 계속 실행된다"는 기본 동작만 다뤘다. 이 문서는 **왜 특정 노드만 실패하는지**를 진단하는 방법을 다룬다. 실제 로봇 시스템(LiDAR+카메라+RTAB-Map+Nav2)처럼 노드가 10개 이상인 Launch 파일에서는 "어느 노드가, 왜" 실패했는지 로그에서 찾아내는 것 자체가 일이다.

## 2. 핵심 개념

Launch 디버깅은 크게 세 갈래다: **어떤 노드가 실행조차 안 됐는지**(조건부 실행/인자 문제), **실행은 됐는데 즉시 종료됐는지**(노드 내부 오류), **실행 순서/타이밍 문제**(의존 노드가 준비되기 전에 실행됨). Launch의 조건부 실행(`IfCondition`/`UnlessCondition`)과 이벤트 핸들러가 잘못 구성되면, 오류 메시지 없이 그냥 노드가 실행되지 않는 경우도 있다는 점이 가장 까다롭다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

[[ros2-nav-yahboom]]에서 실제로 겪었던 "Nav2 패널이 unknown"인 사례는, `navigation_rtabmap_launch.py` 안에서 `rtabmap_nav_launch`가 **주석 처리**되어 있었던 것이 원인이었다 — 오류 메시지 없이 그냥 "그 노드들이 애초에 실행되지 않은" 경우다. 이런 문제는 로그를 아무리 봐도 "오류"가 없기 때문에, **Launch 파일 자체를 소스 코드처럼 읽는 것** 외에는 발견할 방법이 없다.

## 4. 진단 도구와 명령어

| 목적 | 명령어/방법 |
|---|---|
| Launch 인자와 조건 확인 | `ros2 launch <pkg> <file>.launch.py --show-args` |
| 실행되는 노드 개수를 기대치와 대조 | `ros2 node list` (실행 전 몇 개가 떠야 정상인지 미리 목록화) |
| 상세 로그로 재실행 | `ros2 launch <pkg> <file>.launch.py --ros-args --log-level debug` |
| 의심 노드만 단독 확인 | Launch 대신 `ros2 run`으로 해당 노드만 따로 실행 |
| 노드 실행 순서 강제 | `RegisterEventHandler` + `OnProcessStart`/`OnProcessExit` |

```python
from launch.actions import RegisterEventHandler
from launch.event_handlers import OnProcessStart

event_handler = RegisterEventHandler(
  OnProcessStart(
    target_action=camera_node,
    on_start=[rgbd_sync_node]
  )
)
```

## 5. 진단 관점

**특정 노드 그룹이 아예 실행되지 않는 경우(주석/조건부 실행)**: Launch 파일을 직접 열어 `IncludeLaunchDescription`이나 개별 `Node` 액션이 주석 처리되어 있지 않은지, `condition=` 인자가 있다면 그 조건이 실제로 만족되는지 확인한다. 오류가 없다고 정상이 아니다.

**노드가 실행됐다가 즉시 종료되는 경우**: 터미널 출력에서 해당 노드 이름이 붙은 로그 줄(`[node_name-N]` 접두어)을 찾는다. 대부분 파라미터 누락이나 필수 Topic 미구독으로 초기화 단계에서 예외가 발생한다.

**노드 실행 순서 문제(A가 B보다 먼저 필요한데 동시 실행됨)**: `RegisterEventHandler`로 특정 노드의 시작/종료를 감지해 다음 노드를 그 이후에 실행하도록 순서를 강제한다.

## 6. 다음 문서와의 연결

- 다음: **[[06_Composable_Node와_Executor_구조|Composable Node와 Executor 구조]]** — 시스템이 동작한 뒤 "CPU가 부족하다"는 단계에서 읽는 성능 최적화 문서다.

## 7. 이해도 점검

1. Launch를 실행했는데 **오류 메시지가 하나도 없이** 특정 노드만 안 떴다. 로그를 아무리 봐도 단서가 없다면 무엇을 해야 하는가?
2. 노드가 떴다가 즉시 죽는다. 로그에서 무엇을 단서로 찾는가?
3. A 노드가 준비된 뒤에 B 노드가 떠야 하는데 동시에 실행된다. 어떻게 순서를 강제하는가?
4. 여러 노드 중 하나만 따로 떼어 확인하고 싶다면?

> [!info]- 정답 및 해설 보기
> 1. **Launch 파일 자체를 소스 코드처럼 직접 읽는다.** 해당 `Node`/`IncludeLaunchDescription`이 주석 처리되어 있거나, `condition=` 인자의 조건이 실제로 거짓이라 실행 대상에서 빠졌을 수 있다. **오류가 없다고 정상인 것이 아니다** — 이 프로젝트의 "Nav2 패널 unknown" 사례가 정확히 이 경우였다([[ros2-nav-yahboom]]).
> 2. 터미널 출력에서 **`[node_name-N]` 접두어가 붙은 로그 줄**을 찾는다. Launch는 여러 노드의 출력을 섞어서 보여주므로 이 접두어로 걸러내야 한다. 대부분 파라미터 누락이나 초기화 단계 예외다.
> 3. **`RegisterEventHandler` + `OnProcessStart`**로 A의 시작을 감지해 그때 B를 실행하도록 구성한다.
> 4. Launch 대신 **`ros2 run <pkg> <node>`로 그 노드만 단독 실행**한다. 다른 노드들의 로그에 묻히지 않아 원인이 훨씬 잘 보인다.

## 8. 참고자료

- [ROS2 — Using event handlers](https://docs.ros.org/en/humble/Tutorials/Intermediate/Launch/Using-Event-Handlers.html) — `OnProcessStart`, `OnProcessExit`
- [ROS2 — Using substitutions](https://docs.ros.org/en/humble/Tutorials/Intermediate/Launch/Using-Substitutions.html) — `IfCondition`, `UnlessCondition` 조건부 실행
