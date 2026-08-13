# 센서 연동 - LiDAR C1

> ROS2 응용 & 센서 연동 시리즈 · 6편
> 선행 학습: ROS2 기초 7편(TF2 기초), 8편(rqt/RViz2 활용), 이 시리즈 5편(RealSense D435i)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 3 — 센서 연동 실습 |
| 예상 선행 지식 | ROS2 기초 7편(TF2), 8편(RViz2), 이 시리즈 5편 |
| 학습 목표 | RPLiDAR C1을 ROS2에 연동하고 스캔 커버리지를 검증할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | ROS2 기초 7·8·9편에서 예고된 심화 주제. nav_docs A1/B3(매핑·costmap)의 LiDAR 입력이 이 문서의 결과물이다 |

---

## 1. 먼저 알아야 할 핵심

1. RPLiDAR C1은 2D 스캔 데이터를 `sensor_msgs/LaserScan` 타입으로 발행하는 회전형 LiDAR다.
2. 5편(RealSense)과 마찬가지로 이 연동도 노드 실행 + Topic + Parameter + TF + RViz2 확인의 조합이다.
3. 이 프로젝트에서는 실제로 **LiDAR 후방 사각지대 문제**를 겪었고, 각도 파라미터 제한 해제와 물리적 마운트 위치 조정으로 해결한 이력이 있다([[ros2-nav-yahboom]] 참고) — 이 경험이 이 문서의 진단 절차 근거다.

## 2. 이 개념은 무엇인가

`sllidar_ros2`(또는 동급 드라이버) 노드가 LiDAR 하드웨어로부터 거리 데이터를 받아 `/scan` Topic으로 발행한다. 각 스캔 포인트는 각도와 거리로 구성되며, `laser_link` 좌표계를 기준으로 표현된다.

## 3. 왜 필요한가

nav_docs B3(Costmap)의 `obstacle_layer`가 `/scan`을 직접 관측원으로 사용하고, A1(RTAB-Map 매핑)의 ICP Odometry도 이 스캔 데이터에 의존한다. LiDAR 연동이 부실하면(사각지대, 노이즈) 이후 매핑·Nav2 전체 품질에 영향을 준다.

## 4. 주요 Topic과 좌표계

| Topic | 내용 |
|---|---|
| `/scan` | 2D LaserScan 데이터 |

| TF 좌표계 | 의미 |
|---|---|
| `laser_link` (또는 `laser_frame`) | LiDAR 기준 좌표계 — `base_link → laser_link` Static Transform으로 로봇 몸체 기준 장착 위치를 표현 (7편 실습과 동일 패턴) |

## 5. 기본 실습

### 설치

```bash
sudo apt install ros-humble-sllidar-ros2   # 실제 패키지명은 하드웨어 벤더 드라이버에 따라 다를 수 있음
```

### 실행

```bash
ros2 launch sllidar_ros2 sllidar_c1_launch.py \
  frame_id:=laser_link \
  angle_compensate:=true
```

* `frame_id`: 발행되는 `/scan` 메시지의 `header.frame_id`. 7편에서 다룬 대로, 이 값이 TF 트리의 실제 좌표계 이름과 정확히 일치해야 RViz2에서 정상 표시된다.
* `angle_compensate`: 회전 속도 변화로 인한 각도 왜곡을 보정할지 여부.

### 확인

```bash
ros2 topic hz /scan
ros2 run rviz2 rviz2
```

RViz2에서 `LaserScan` Display를 추가하고 `Topic: /scan`을 지정한다(8편 절차). 로봇을 중심으로 360도 점들이 고르게 찍히는지 확인한다.

## 6. 진단 관점

| 증상 | 확인 순서 | 관련 사례 |
|---|---|---|
| 특정 각도 구간에 스캔이 안 찍힘(사각지대) | LiDAR 각도 제한 파라미터(`angle_min`/`angle_max`), 물리적 가림(마운트 위치/케이블) | [[ros2-nav-yahboom]] 후방 사각지대 해결 사례 — 파라미터 해제 + 마운트 높이 조정으로 360도 확보 |
| `/scan`이 RViz2에 안 보임 | `frame_id`와 TF 트리 이름 일치 여부(7편), Fixed Frame 설정 |
| 스캔에 노이즈가 많음 | 반사가 심한 표면(유리, 거울) 근처인지, `range_min`/`range_max` 필터링 |

## 7. 다음 학습 주제

- 이것으로 "ROS2 응용 & 센서 연동" 시리즈가 완결된다.
- 다음은 이 두 센서(RealSense + LiDAR)를 실제로 융합하는 **RTAB-Map & Nav2 심화 시리즈 Part A(A1. RTAB-Map 매핑 원리)**로 자연스럽게 이어진다 — 이미 작성되어 있으므로 바로 이어 읽으면 된다.

## 8. 참고자료

- sllidar_ros2 공식 GitHub — launch 인자와 파라미터 목록
- [[ros2-nav-yahboom]] — LiDAR 후방 사각지대 진단 및 해결 이력
