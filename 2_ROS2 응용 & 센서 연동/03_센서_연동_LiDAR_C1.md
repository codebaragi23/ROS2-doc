# 센서 연동 - LiDAR 00

> ROS2 응용 & 센서 연동 시리즈 · 3편
> 선행 학습: ROS2 기초 7편(TF2 기초), 8편(rqt/RViz2 활용), 이 시리즈 5편(RealSense D435i)

## 1. 개요

RPLiDAR C1은 2D 스캔 데이터를 `sensor_msgs/LaserScan` 타입으로 발행하는 회전형 LiDAR다. 2편(RealSense)과 마찬가지로 노드 실행 + Topic + Parameter + TF + RViz2 확인의 조합이며, 이 결과물은 지도제작 00(매핑)과 Nav2 내비게이션 02(Costmap)의 LiDAR 입력으로 이어진다.

## 2. 핵심 개념

`sllidar_ros2`(또는 동급 드라이버) 노드가 LiDAR 하드웨어로부터 거리 데이터를 받아 `/scan` Topic으로 발행한다. 각 스캔 포인트는 각도와 거리로 구성되며, `laser_link` 좌표계를 기준으로 표현된다.

| Topic | 내용 |
|---|---|
| `/scan` | 2D LaserScan 데이터 |

| TF 좌표계 | 의미 |
|---|---|
| `laser_link` | LiDAR 기준 좌표계 — `base_link → laser_link` Static Transform으로 로봇 몸체 기준 장착 위치를 표현 (ROS2 기초 7편 실습과 동일 패턴). 드라이버가 발행하는 실제 이름은 `frame_id` 파라미터로 정해지므로, TF에서 쓰는 이름과 반드시 일치시켜야 한다 |

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

```bash
sudo apt install ros-humble-sllidar-ros2   # 실제 패키지명은 하드웨어 벤더 드라이버에 따라 다를 수 있음

ros2 launch sllidar_ros2 sllidar_c1_launch.py \
  frame_id:=laser_link \
  angle_compensate:=true
```

`frame_id`는 발행되는 `/scan` 메시지의 `header.frame_id`다. ROS2 기초 7편에서 다룬 대로, 이 값이 TF 트리의 실제 좌표계 이름과 정확히 일치해야 RViz2에서 정상 표시된다. `angle_compensate`는 회전 속도 변화로 인한 각도 왜곡을 보정할지 여부다.

이 프로젝트에서는 실제로 **LiDAR 후방 사각지대 문제**를 겪었고, 각도 파라미터 제한 해제와 물리적 마운트 위치 조정으로 해결한 이력이 있다([[ros2-nav-yahboom]] 참고).

## 4. 확인

```bash
ros2 topic hz /scan
ros2 run rviz2 rviz2
```

RViz2에서 `LaserScan` Display를 추가하고 `Topic: /scan`을 지정한다(ROS2 기초 8편 절차). 로봇을 중심으로 360도 점들이 고르게 찍히는지 확인한다.

## 5. 진단 관점

| 증상 | 확인 순서 | 관련 사례 |
|---|---|---|
| 특정 각도 구간에 스캔이 안 찍힘(사각지대) | LiDAR 각도 제한 파라미터(`angle_min`/`angle_max`), 물리적 가림(마운트 위치/케이블) | [[ros2-nav-yahboom]] 후방 사각지대 해결 사례 — 파라미터 해제 + 마운트 높이 조정으로 360도 확보 |
| `/scan`이 RViz2에 안 보임 | `frame_id`와 TF 트리 이름 일치 여부, Fixed Frame 설정 |
| 스캔에 노이즈가 많음 | 반사가 심한 표면(유리, 거울) 근처인지, `range_min`/`range_max` 필터링 |

## 6. 다음 문서와의 연결

- 다음: **[[04_colcon_빌드_오류_모음|colcon 빌드 오류 모음]]** — 센서 드라이버를 소스로 빌드하다 막혔을 때 펼쳐보는 참조 문서다. 지금까지 apt로만 설치했다면 건너뛰고 5편으로 가도 된다.
- 두 센서가 모두 붙었다면, 이들을 실제로 융합하는 **[[00_SLAM이란_무엇이고_왜_쓰는가|SLAM 공통 기초]]**로 넘어갈 준비가 된 것이다 — 다만 응용 시리즈의 나머지(4~6편)를 먼저 훑어두면 이후 디버깅이 수월하다.

## 7. 참고자료

- [sllidar_ros2 공식 GitHub](https://github.com/Slamtec/sllidar_ros2) — launch 인자와 파라미터 목록
- [[ros2-nav-yahboom]] — LiDAR 후방 사각지대 진단 및 해결 이력
