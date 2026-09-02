# ROS2 성능 - Composable Node와 Executor 구조

> ROS2 응용 & 센서 연동 시리즈 · 6편 (시리즈 마지막)
> 선행 학습: ROS2 기초 2편(노드란 무엇인가), 6편(Launch 파일 작성법)

## 1. 개요

ROS2 기초 2편에서 "노드 하나 = 프로세스 하나"가 기본이라고 배웠지만, 대용량 데이터(카메라 이미지, 포인트클라우드)를 다루는 노드가 많아지면 이 방식은 비효율적이다. 이 문서는 여러 노드를 하나의 프로세스로 묶어 이 문제를 완화하는 **Composable Node**와, 그 안에서 콜백 실행을 관리하는 **Executor**를 다룬다.

## 2. 핵심 개념

**일반 노드**는 각각 별도 프로세스로 실행되며, 노드 간 Topic 통신은 DDS를 통해 직렬화(serialize)와 프로세스 간 통신(IPC)을 거친다. 이미지처럼 큰 데이터는 이 직렬화/복사 비용이 상당하다.

**Composable Node**는 여러 노드를 하나의 **컨테이너 프로세스** 안에 라이브러리(.so) 형태로 로드해서, 노드 간 통신이 같은 프로세스 내 메모리 참조로 처리될 수 있게 한다(Intra-process communication). 직렬화 단계를 생략할 수 있어 대용량 데이터 파이프라인에서 지연시간과 CPU 사용량이 크게 줄어든다.

**Executor**는 한 프로세스(또는 컨테이너) 안에서 여러 콜백을 관리하는 스케줄러다.

| Executor 종류 | 동작 |
|---|---|
| `SingleThreadedExecutor` | 하나의 스레드에서 콜백을 순차 실행 (기본값) |
| `MultiThreadedExecutor` | 여러 스레드에서 콜백을 동시에 실행 가능 |
| `StaticSingleThreadedExecutor` | 콜백 목록을 미리 고정해 오버헤드를 줄인 단일 스레드 버전 |

```mermaid
flowchart TB
    subgraph 일반노드["일반 노드 방식 (별도 프로세스)"]
        N1[노드 A] -->|직렬화+IPC| N2[노드 B]
        N2 -->|직렬화+IPC| N3[노드 C]
    end
    subgraph 컴포저블["Composable Node 방식 (같은 컨테이너)"]
        C1[노드 A] -->|메모리 참조| C2[노드 B]
        C2 -->|메모리 참조| C3[노드 C]
    end
```

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

RealSense D435i가 30fps로 RGB+Depth 이미지를 발행하고, 이를 구독하는 `rgbd_sync`, `rtabmap` 노드가 각각 별도 프로세스라면, 매 프레임마다 이미지 데이터가 직렬화되어 프로세스 경계를 넘나든다. 이 오버헤드가 [[ros2-nav-yahboom]]에서 확인된 "RealSense 드라이버 CPU 47% 점유" 문제의 근본 원인 중 하나로 작용할 수 있다 — Composable Node로 묶으면 이 복사 비용 자체를 줄일 수 있다.

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

`ComposableNodeContainer`는 ROS2 기초 6편에서 배운 `Node` 액션과 비슷하지만, 노드를 개별 프로세스가 아니라 이 컨테이너 프로세스 안에 로드하도록 지정한다. `plugin='realsense2_camera::RealSenseNodeFactory'`처럼 일반 실행 파일 이름 대신 컨테이너가 동적으로 로드할 **라이브러리 클래스 이름**을 지정하는 점이 다르다.

## 4. 관련 명령어

| 목적 | 명령어 |
|---|---|
| 컨테이너 직접 실행 | `ros2 run rclcpp_components component_container` |
| 실행 중인 컨테이너에 노드 동적 로드 | `ros2 component load <container> <package> <plugin>` |
| 특정 컨테이너에 로드된 노드 확인 | `ros2 component list` |

## 5. 진단 관점

카메라·LiDAR 데이터 처리에서 CPU 사용량이 예상보다 높다면, 현재 파이프라인이 일반 노드로 나뉘어 있는지 Composable Node 구조인지부터 확인한다. `ros2 node list`에는 Composable Node도 개별 이름으로 나타나므로 겉보기엔 일반 노드와 구분이 안 될 수 있다 — `ros2 component list`로 어떤 컨테이너에 어떤 노드가 로드되어 있는지 확인해야 한다.

## 6. 다음 문서와의 연결

- 이것으로 ROS2 응용 & 센서 연동 시리즈가 끝난다. 1편(DDS/QoS)에서 다룬 프로세스 간 통신 비용을, 이 문서의 Composable Node가 프로세스 내부(intra-process)에서 우회하는 구조라는 점을 함께 놓고 보면 시리즈 전체가 이어진다.
- 다음 시리즈: **[[00_SLAM이란_무엇이고_왜_쓰는가|SLAM 공통 기초 — 00. SLAM이란 무엇이고 왜 쓰는가]]**

## 7. 참고자료

- [ROS2 — About Composition](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Composition.html) — Composable Node와 intra-process 통신
- [ROS2 — About Executors](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Executors.html) — Executor 종류와 콜백 그룹
