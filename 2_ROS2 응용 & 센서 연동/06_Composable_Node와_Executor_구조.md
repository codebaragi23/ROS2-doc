# 06. Composable Node와 Executor 구조

> ROS2 응용 & 센서 연동 시리즈 (시리즈 마지막)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 응용 (성능 최적화 — 시스템이 일단 동작한 뒤에 읽는 문서) |
| 예상 선행 지식 | [[02_노드란_무엇인가\|ROS2 기초 02. 노드란 무엇인가]], [[06_Launch_파일_작성법\|06. Launch 파일 작성법]] |
| 학습 목표 | 노드를 한 프로세스에 묶으면 왜 빨라지는지(zero-copy) 설명할 수 있다 / Executor의 종류와 콜백 그룹 개념을 이해한다 / 콜백 안에서 블로킹 대기를 하면 왜 데드락이 나는지 설명할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | 이전: [[05_Launch_시스템_디버깅\|응용 05. Launch 시스템 디버깅]] / 다음: [[00_SLAM이란_무엇이고_왜_쓰는가\|SLAM 공통 기초 00]] |

> **언제 읽는가**: 이 문서는 "센서가 안 붙는다" 단계가 아니라 **"다 동작하는데 CPU가 부족하다"** 단계에서 필요하다. [[02_센서_연동_RealSense_D435i\|응용 02]]의 CPU 부하 실험에서 한계를 느꼈다면 지금이 그 시점이다.

## 1. 개요

ROS2 기초 02에서 "노드 하나 = 프로세스 하나"가 기본이라고 배웠지만, 대용량 데이터(카메라 이미지, 포인트클라우드)를 다루는 노드가 많아지면 이 방식은 비효율적이다. 이 문서는 여러 노드를 하나의 프로세스로 묶어 이 문제를 완화하는 **Composable Node**와, 그 안에서 콜백 실행을 관리하는 **Executor**를 다룬다.

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
        00[노드 A] -->|메모리 참조| 01[노드 B]
        01 -->|메모리 참조| 02[노드 C]
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

`ComposableNodeContainer`는 ROS2 기초 06에서 배운 `Node` 액션과 비슷하지만, 노드를 개별 프로세스가 아니라 이 컨테이너 프로세스 안에 로드하도록 지정한다. `plugin='realsense2_camera::RealSenseNodeFactory'`처럼 일반 실행 파일 이름 대신 컨테이너가 동적으로 로드할 **라이브러리 클래스 이름**을 지정하는 점이 다르다.

## 4. 관련 명령어

| 목적 | 명령어 |
|---|---|
| 컨테이너 직접 실행 | `ros2 run rclcpp_components component_container` |
| 실행 중인 컨테이너에 노드 동적 로드 | `ros2 component load <container> <package> <plugin>` |
| 특정 컨테이너에 로드된 노드 확인 | `ros2 component list` |

## 5. 진단 관점

카메라·LiDAR 데이터 처리에서 CPU 사용량이 예상보다 높다면, 현재 파이프라인이 일반 노드로 나뉘어 있는지 Composable Node 구조인지부터 확인한다. `ros2 node list`에는 Composable Node도 개별 이름으로 나타나므로 겉보기엔 일반 노드와 구분이 안 될 수 있다 — `ros2 component list`로 어떤 컨테이너에 어떤 노드가 로드되어 있는지 확인해야 한다.

## 6. 다음 문서와의 연결

- 이것으로 ROS2 응용 & 센서 연동 시리즈가 끝난다. 01(DDS/QoS)에서 다룬 프로세스 간 통신 비용을, 이 문서의 Composable Node가 프로세스 내부(intra-process)에서 우회하는 구조라는 점을 함께 놓고 보면 시리즈 전체가 이어진다.
- 다음 시리즈: **[[00_SLAM이란_무엇이고_왜_쓰는가|SLAM 공통 기초 — 00. SLAM이란 무엇이고 왜 쓰는가]]**

## 7. 이해도 점검

1. 노드를 하나의 프로세스로 묶으면 성능이 좋아지는 근본 이유는 무엇인가?
2. 모든 노드를 다 묶는 것이 항상 최선인가?
3. `SingleThreadedExecutor`에서 콜백 안에 블로킹 대기를 넣으면 왜 위험한가?
4. 콜백 그룹의 `MutuallyExclusive`와 `Reentrant`는 각각 무엇을 보장하는가?

> [!info]- 정답 및 해설 보기
> 1. **프로세스 간 데이터 복사(직렬화/역직렬화)가 사라지기 때문**이다. 같은 프로세스 안이면 포인터만 전달하는 intra-process(zero-copy) 통신이 가능해, 이미지·포인트클라우드처럼 큰 데이터에서 효과가 크다.
> 2. **아니다.** 한 프로세스에 묶인 노드는 **한 노드가 죽으면 같이 죽는다.** 안정성이 중요한 노드는 분리해두는 편이 낫고, 성능 이득이 큰 대용량 데이터 경로(카메라→처리)에 선택적으로 적용하는 것이 일반적이다.
> 3. `SingleThreadedExecutor`는 콜백을 **하나씩 순차 처리**한다. 콜백 안에서 다른 콜백의 결과를 기다리면, 그 결과를 처리할 콜백이 영원히 실행되지 못해 **데드락**에 빠진다. [[06_Relocalization_감시_로직_구현|지상로봇 적용 - Relocalization 심화 - 06]]의 예제 코드가 `wait_for_server()` 대신 `server_is_ready()`를 쓰는 이유가 바로 이것이다.
> 4. **`MutuallyExclusive`**: 같은 그룹의 콜백들이 **동시에 실행되지 않음**을 보장한다(공유 자원 보호에 안전). **`Reentrant`**: 같은 그룹의 콜백이 **병렬로 실행될 수 있다**(처리량은 늘지만 공유 자원에 락이 필요).

## 8. 참고자료

- [ROS2 — About Composition](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Composition.html) — Composable Node와 intra-process 통신
- [ROS2 — About Executors](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Executors.html) — Executor 종류와 콜백 그룹
