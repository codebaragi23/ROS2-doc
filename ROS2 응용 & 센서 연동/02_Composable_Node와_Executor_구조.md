# ROS2 성능 - Composable Node와 Executor 구조

> ROS2 응용 & 센서 연동 시리즈 · 2편
> 선행 학습: ROS2 기초 2편(노드란 무엇인가), 6편(Launch 파일 작성법)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 3 — 성능 심화 |
| 예상 선행 지식 | ROS2 기초 2편(노드), 6편(Launch) |
| 학습 목표 | Composable Node가 왜 필요한지 설명할 수 있다 / Executor 종류와 콜백 그룹 개념을 이해한다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | ROS2 기초 2편 12장에서 예고된 심화 주제 |

---

## 1. 먼저 알아야 할 핵심

1. ROS2 기초 2편에서 "노드 하나 = 프로세스 하나"가 기본이라고 배웠지만, 대용량 데이터(카메라 이미지, 포인트클라우드)를 다루는 노드가 많아지면 이 방식은 비효율적이다.
2. **Composable Node**는 여러 노드를 하나의 프로세스 안에 묶어 실행해, 노드 간 데이터 전달 시 메모리 복사를 줄이는 방식이다.
3. **Executor**는 하나의 프로세스 안에서 여러 콜백(구독 콜백, 타이머 콜백 등)을 어떤 순서/스레드로 실행할지 결정하는 내부 메커니즘이다.
4. RealSense D435i처럼 고해상도 이미지를 자주 발행하는 센서에서 이 개념이 실제로 CPU 사용량에 큰 영향을 준다.

## 2. 이 개념은 무엇인가

**일반 노드**는 각각 별도 프로세스로 실행되며, 노드 간 Topic 통신은 DDS를 통해 직렬화(serialize)와 프로세스 간 통신(IPC)을 거친다. 이미지처럼 큰 데이터는 이 직렬화/복사 비용이 상당하다.

**Composable Node**는 여러 노드를 하나의 **컨테이너 프로세스** 안에 라이브러리(.so) 형태로 로드해서, 노드 간 통신이 같은 프로세스 내 메모리 참조로 처리될 수 있게 한다(Intra-process communication). 직렬화 단계를 생략할 수 있어 대용량 데이터 파이프라인에서 지연시간과 CPU 사용량이 크게 줄어든다.

**Executor**는 한 프로세스(또는 컨테이너) 안에서 여러 콜백을 관리하는 스케줄러다.

| Executor 종류 | 동작 |
|---|---|
| `SingleThreadedExecutor` | 하나의 스레드에서 콜백을 순차 실행 (기본값) |
| `MultiThreadedExecutor` | 여러 스레드에서 콜백을 동시에 실행 가능 |
| `StaticSingleThreadedExecutor` | 콜백 목록을 미리 고정해 오버헤드를 줄인 단일 스레드 버전 |

## 3. 왜 필요한가

RealSense D435i가 30fps로 RGB+Depth 이미지를 발행하고, 이를 구독하는 `rgbd_sync`, `rtabmap` 노드가 각각 별도 프로세스라면, 매 프레임마다 이미지 데이터가 직렬화되어 프로세스 경계를 넘나든다. 이 오버헤드가 [[ros2-nav-yahboom]]에서 확인된 "RealSense 드라이버 CPU 47% 점유" 문제의 근본 원인 중 하나로 작용할 수 있다 — Composable Node로 묶으면 이 복사 비용 자체를 줄일 수 있다.

## 4. ROS2 시스템에서의 위치

```mermaid
flowchart TB
    subgraph 일반노드["일반 노드 방식 (별도 프로세스)"]
        N1[카메라 드라이버] -->|직렬화+IPC| N2[rgbd_sync]
        N2 -->|직렬화+IPC| N3[rtabmap]
    end
    subgraph 컴포저블["Composable Node 방식 (같은 컨테이너)"]
        C1[카메라 드라이버] -->|메모리 참조| C2[rgbd_sync]
        C2 -->|메모리 참조| C3[rtabmap]
    end
```

## 5. 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Component Container | Composable Node들을 로드하는 프로세스 |
| `ros2 component load` | 실행 중인 컨테이너에 새 노드를 동적으로 추가 |
| Callback Group | 콜백들을 그룹으로 묶어 동시 실행 가능 여부를 제어 |
| `MutuallyExclusive` | 같은 그룹의 콜백은 동시에 실행되지 않음(기본값) |
| `Reentrant` | 같은 그룹이라도 콜백이 동시에(재진입) 실행될 수 있음 |

## 6. 기본 동작 과정

1. **컨테이너 실행**: `ros2 run rclcpp_components component_container`로 빈 컨테이너를 띄우거나, Launch 파일에서 `ComposableNodeContainer` 액션으로 정의한다.
2. **노드 로드**: `ComposableNode` 항목을 컨테이너의 `composable_node_descriptions`에 나열하면, 해당 노드들이 같은 프로세스에 라이브러리로 로드된다.
3. **Intra-process 통신 활성화**: 컨테이너 옵션에서 intra-process communication을 켜면, 같은 컨테이너 안의 Publisher-Subscriber 쌍이 메모리 복사 없이 데이터를 주고받는다.
4. **Executor가 콜백 스케줄링**: 컨테이너 안의 여러 노드가 만드는 콜백들을 Executor(기본 SingleThreaded 또는 MultiThreaded)가 실행 순서를 관리한다.

## 7. Launch 파일 예시

```python
from launch_ros.actions import ComposableNodeContainer
from launch_ros.descriptions import ComposableNode

container = ComposableNodeContainer(
  name='perception_container',
  namespace='',
  package='rclcpp_components',
  executable='component_container',
  composable_node_descriptions=[
    ComposableNode(
      package='realsense2_camera',
      plugin='realsense2_camera::RealSenseNodeFactory',
      name='camera'
    ),
    ComposableNode(
      package='rtabmap_sync',
      plugin='rtabmap_sync::RGBDSync',
      name='rgbd_sync'
    ),
  ],
  output='screen'
)
```

* `ComposableNodeContainer`: 6편에서 배운 `Node` 액션과 비슷하지만, 노드를 개별 프로세스가 아니라 이 컨테이너 프로세스 안에 로드하도록 지정한다.
* `plugin='realsense2_camera::RealSenseNodeFactory'`: 일반 실행 파일 이름 대신, 컨테이너가 동적으로 로드할 **라이브러리 클래스 이름**을 지정한다.

## 8. 진단 관점

- 카메라·LiDAR 데이터 처리에서 CPU 사용량이 예상보다 높다면, 현재 파이프라인이 일반 노드로 나뉘어 있는지 Composable Node 구조인지부터 확인한다.
- `ros2 node list`에는 Composable Node도 개별 이름으로 나타나므로 겉보기엔 일반 노드와 구분이 안 될 수 있다 — `ros2 component list`로 어떤 컨테이너에 어떤 노드가 로드되어 있는지 확인한다.

## 9. 핵심 요약

1. Composable Node는 여러 노드를 한 프로세스에 묶어 노드 간 데이터 복사 비용을 줄이는 방식이다.
2. 대용량 데이터(카메라, 포인트클라우드)를 다루는 파이프라인에서 성능 이점이 크다.
3. Executor는 한 프로세스 안의 콜백 실행을 관리하며, SingleThreaded/MultiThreaded 중 상황에 맞게 선택한다.

## 10. 다음 학습 주제

- 다음: **DDS와 QoS 이해하기** — 프로세스 간(inter-process) 통신에서 발생하는 지연/신뢰성 문제를 다룬다. Composable Node는 이 문제를 프로세스 내부(intra-process)에서 우회하는 방법이라는 점에서 서로 대비되는 개념이다.

## 11. 참고자료

- ROS2 공식 문서 — Composable Node 개념과 Intra-process Communication
- ROS2 공식 문서 — Executor 종류와 콜백 그룹(MutuallyExclusive/Reentrant)
