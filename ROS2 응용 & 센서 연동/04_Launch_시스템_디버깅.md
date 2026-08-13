# 문제 해결 - Launch 시스템 디버깅

> ROS2 응용 & 센서 연동 시리즈 · 4편
> 선행 학습: ROS2 기초 6편(Launch 파일 작성법), 9편(문제 해결)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 3 — 실전 진단 |
| 예상 선행 지식 | ROS2 기초 6편(Launch), 9편(일반 오류 모음) |
| 학습 목표 | 여러 노드 중 일부만 실패하는 상황을 Launch 레벨에서 진단할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | ROS2 기초 6편 15장에서 예고된 심화 주제 |

---

## 1. 먼저 알아야 할 핵심

1. ROS2 기초 6편은 "노드 하나가 죽어도 나머지는 계속 실행된다"는 기본 동작만 다뤘다. 이 문서는 **왜 특정 노드만 실패하는지**를 진단하는 방법을 다룬다.
2. 실제 로봇 시스템(LiDAR+카메라+RTAB-Map+Nav2)처럼 노드가 10개 이상인 Launch 파일에서는 "어느 노드가, 왜" 실패했는지 로그에서 찾아내는 것 자체가 일이다.
3. Launch의 조건부 실행(`IfCondition`/`UnlessCondition`)과 이벤트 핸들러가 잘못 구성되면, 오류 메시지 없이 그냥 노드가 실행되지 않는 경우도 있다.

## 2. 이 개념은 무엇인가

Launch 디버깅은 크게 세 갈래다.

1. **어떤 노드가 실행조차 안 됐는지** (조건부 실행/인자 문제)
2. **실행은 됐는데 즉시 종료됐는지** (노드 내부 오류)
3. **실행 순서/타이밍 문제** (의존 노드가 준비되기 전에 실행됨)

## 3. 왜 필요한가

[[ros2-nav-yahboom]]에서 실제로 겪었던 "Nav2 패널이 unknown"인 사례는, `navigation_rtabmap_launch.py` 안에서 `rtabmap_nav_launch`가 **주석 처리**되어 있었던 것이 원인이었다 — 이는 오류 메시지 없이 그냥 "그 노드들이 애초에 실행되지 않은" 경우다. 이런 문제는 로그를 아무리 봐도 "오류"가 없기 때문에, Launch 파일 자체를 읽는 것 외에는 발견할 방법이 없다.

## 4. 진단 도구

### 4.1 Launch 인자와 조건 확인

```bash
ros2 launch <pkg> <file>.launch.py --show-args
```

Launch 파일이 어떤 인자를 받는지, 기본값이 무엇인지 확인한다. 조건부 실행(`IfCondition(LaunchConfiguration('use_nav'))`)이 있다면, 이 인자 값에 따라 특정 노드 그룹 전체가 통째로 빠질 수 있다.

### 4.2 실행되는 노드 목록을 실행 전에 예측하고, 실행 후 대조

```bash
# 실행 후
ros2 node list
```

기대한 노드 개수와 실제로 뜬 노드 개수를 항상 대조하는 습관이 중요하다 — 6편에서 이미 강조한 원칙이지만, 노드가 많아질수록 "몇 개가 떠야 정상인지"를 미리 목록화해두지 않으면 빠진 것을 알아채기 어렵다.

### 4.3 로그 레벨을 높여 실행

```bash
ros2 launch <pkg> <file>.launch.py --ros-args --log-level debug
```

노드가 조용히 실패하는 경우, debug 레벨에서 초기화 단계의 상세 로그가 추가로 드러나는 경우가 있다.

### 4.4 특정 노드만 따로 떼어 실행

Launch 파일 전체가 아니라, 의심되는 노드 하나만 `ros2 run`으로 따로 실행해보면 Launch 관련 문제인지 노드 자체의 문제인지 구분된다.

## 5. 자주 발생하는 문제

### 문제: 특정 노드 그룹이 아예 실행되지 않음(주석/조건부 실행)

**확인**: Launch 파일을 직접 열어 `IncludeLaunchDescription`이나 개별 `Node` 액션이 주석 처리되어 있지 않은지, `condition=` 인자가 있다면 그 조건이 실제로 만족되는지 확인한다.

**교훈**: 오류가 없다고 정상이 아니다 — **Launch 파일 자체를 소스 코드처럼 읽는 습관**이 필요하다.

### 문제: 노드가 실행됐다가 즉시 종료됨

**확인**: 터미널 출력에서 해당 노드 이름이 붙은 로그 줄을 찾는다(6편에서 배운 `[node_name-N]` 접두어). 대부분 파라미터 누락이나 필수 Topic 미구독으로 초기화 단계에서 예외가 발생한다.

### 문제: 노드 실행 순서 문제 (A가 B보다 먼저 필요한데 동시 실행됨)

**해결**: `RegisterEventHandler` + `OnProcessStart`(또는 `OnProcessExit`)로 특정 노드의 시작/종료를 감지해 다음 노드를 그 이후에 실행하도록 순서를 강제할 수 있다.

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

## 6. 개념 간 연결

- 이 문서는 ROS2 기초 6편(Launch 작성)의 반대편, "이미 작성된 Launch 파일이 왜 예상대로 안 되는지"를 다룬다.
- [[ros2-nav-yahboom]]의 실제 사례(주석 처리된 launch 포함 문제)가 이 문서의 4.1 항목의 실전 근거다.

## 7. 핵심 요약

1. Launch 문제는 "실행 안 됨/즉시 종료/순서 문제" 세 갈래로 분류하면 접근이 빨라진다.
2. 오류 메시지가 없어도 조건부 실행이나 주석으로 인해 노드가 통째로 빠질 수 있다 — Launch 파일을 직접 읽는 것이 최종 확인 수단이다.
3. `RegisterEventHandler`로 노드 간 실행 순서를 명시적으로 강제할 수 있다.

## 8. 다음 학습 주제

- 다음: **센서 연동 - RealSense D435i** — 지금까지의 진단 도구들을 실제 센서 연동 과정에서 바로 활용하게 된다.

## 9. 참고자료

- ROS2 공식 문서 — Launch 시스템의 이벤트 핸들러(`OnProcessStart`, `OnProcessExit`)
- ROS2 공식 문서 — 조건부 실행(`IfCondition`, `UnlessCondition`)
