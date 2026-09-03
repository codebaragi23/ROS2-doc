# 03. 센서 연동 - LiDAR C1

> ROS2 응용 & 센서 연동 시리즈

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 응용 (두 번째 하드웨어 연동) |
| 예상 선행 지식 | [[07_TF2_기초\|ROS2 기초 07. TF2 기초]], [[08_rqt_rviz2_활용\|08. rqt/RViz2 활용]], [[02_센서_연동_RealSense_D435i\|응용 02. RealSense D435i]] |
| 학습 목표 | LiDAR 드라이버를 실행해 `/scan`을 확인할 수 있다 / `frame_id`와 TF 좌표계 이름이 일치해야 하는 이유를 설명할 수 있다 / 스캔 사각지대·노이즈의 원인을 구분해 진단할 수 있다 / 카메라와 LiDAR의 TF를 하나의 `base_link` 기준으로 정합할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble, Slamtec RPLiDAR C1 |
| 관련 문서 | 이전: [[02_센서_연동_RealSense_D435i\|응용 02. RealSense D435i]] / 다음: [[04_colcon_빌드_오류_모음\|응용 04. colcon 빌드 오류]] |

## 1. 개요

RPLiDAR C1은 2D 스캔 데이터를 `sensor_msgs/LaserScan` 타입으로 발행하는 회전형 LiDAR다. [[02_센서_연동_RealSense_D435i|응용 02(RealSense)]]와 마찬가지로 노드 실행 + Topic + Parameter + TF + RViz2 확인의 조합이며, 이 결과물은 [[00_RTAB-Map_매핑_원리|지상로봇 적용 - 지도제작 00]](매핑)과 [[02_Costmap|Nav2 내비게이션 02]](Costmap)의 LiDAR 입력으로 이어진다.

## 2. 핵심 개념

`sllidar_ros2`(또는 동급 드라이버) 노드가 LiDAR 하드웨어로부터 거리 데이터를 받아 `/scan` Topic으로 발행한다. 각 스캔 포인트는 각도와 거리로 구성되며, `laser_link` 좌표계를 기준으로 표현된다.

| Topic | 내용 |
|---|---|
| `/scan` | 2D LaserScan 데이터 |

| TF 좌표계 | 의미 |
|---|---|
| `laser_link` | LiDAR 기준 좌표계 — `base_link → laser_link` Static Transform으로 로봇 몸체 기준 장착 위치를 표현 (ROS2 기초 07 실습과 동일 패턴). 드라이버가 발행하는 실제 이름은 `frame_id` 파라미터로 정해지므로, TF에서 쓰는 이름과 반드시 일치시켜야 한다 |

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

```bash
sudo apt install ros-humble-sllidar-ros2   # 실제 패키지명은 하드웨어 벤더 드라이버에 따라 다를 수 있음

ros2 launch sllidar_ros2 sllidar_c1_launch.py \
  frame_id:=laser_link \
  angle_compensate:=true
```

`frame_id`는 발행되는 `/scan` 메시지의 `header.frame_id`다. ROS2 기초 07에서 다룬 대로, 이 값이 TF 트리의 실제 좌표계 이름과 정확히 일치해야 RViz2에서 정상 표시된다. `angle_compensate`는 회전 속도 변화로 인한 각도 왜곡을 보정할지 여부다.

이 프로젝트에서는 실제로 **LiDAR 후방 사각지대 문제**를 겪었고, 각도 파라미터 제한 해제와 물리적 마운트 위치 조정으로 해결한 이력이 있다([[ros2-nav-yahboom]] 참고).

## 4. 실습 — LiDAR 연동과 두 센서 TF 정합

### 실습 목표

`/scan`을 확인하고, **앞서 붙인 카메라와 LiDAR가 같은 `base_link` 기준에서 서로 어긋나지 않는지**까지 검증한다. 센서 하나씩은 잘 되는데 둘을 같이 켜면 안 맞는 경우가 흔하므로, 이 정합 확인이 이 실습의 핵심이다.

### 준비 사항

- RPLiDAR C1 연결, 시리얼 포트 권한 확보
- [[02_센서_연동_RealSense_D435i|응용 02]]의 카메라 연동을 이미 완료했을 것(6단계에서 필요)

### 절차

**1단계 — 시리얼 포트 권한 (가장 흔한 첫 관문)**

```bash
ls -l /dev/ttyUSB*
sudo usermod -aG dialout $USER   # 적용하려면 로그아웃 후 재로그인 필요
```

권한이 없으면 드라이버가 포트를 못 열고 바로 죽는다. `dialout` 그룹 추가 후 **재로그인해야 반영**된다는 점을 놓치기 쉽다.

**2단계 — 드라이버 실행**

```bash
ros2 launch sllidar_ros2 sllidar_c1_launch.py \
  frame_id:=laser_link \
  angle_compensate:=true
```

**3단계 — 스캔이 흐르는지**

```bash
ros2 topic hz /scan
ros2 topic echo /scan --once
```

`echo --once` 출력에서 `header.frame_id`가 **2단계에서 지정한 `laser_link`와 정확히 같은지** 확인한다. 이게 어긋나면 RViz2에서 아무리 설정을 바꿔도 안 보인다.

**4단계 — 로봇 몸체 기준 장착 위치 발행**

```bash
# 예: 로봇 중심에서 전방 5cm, 높이 15cm에 LiDAR를 장착한 경우
ros2 run tf2_ros static_transform_publisher \
  0.05 0 0.15 0 0 0 base_link laser_link
```

**5단계 — RViz2에서 스캔 확인**

`LaserScan` Display 추가, `Topic: /scan`, `Fixed Frame: base_link`. 로봇을 중심으로 점들이 찍히는지 본다.

**6단계 — 카메라와 함께 켜서 정합 확인 (이 실습의 핵심)**

카메라 드라이버와 그 Static Transform도 함께 실행한 뒤, RViz2에 `LaserScan`과 `PointCloud2`를 **동시에** 띄운다.

```bash
# 카메라 TF (응용 02의 5단계)
ros2 run tf2_ros static_transform_publisher 0.1 0 0.2 0 0 0 base_link camera_link
```

벽을 향해 두고 봤을 때, **LiDAR가 그린 벽 선과 카메라 포인트클라우드의 벽면이 같은 위치에 겹쳐야 한다.**

### 예상 결과

- `/scan`이 설정한 주기로 안정적으로 발행된다.
- RViz2에서 방의 윤곽이 위에서 내려다본 평면도처럼 그려진다.
- 6단계에서 두 센서의 벽이 겹친다.

### 어긋난다면

두 센서의 벽이 어긋나 보인다면 **센서가 고장난 게 아니라 4단계·5단계에서 넣은 장착 위치 값(x, y, z)이 실제와 다른 것**이다. 자를 대고 실측한 값으로 고쳐 넣는다 — [[08_rqt_rviz2_활용|ROS2 기초 08]]에서 "RViz2로 TF 오류를 즉시 발견할 수 있다"고 한 상황이 바로 이것이다.

## 5. 진단 관점

| 증상 | 확인 순서 | 관련 사례 |
|---|---|---|
| 특정 각도 구간에 스캔이 안 찍힘(사각지대) | LiDAR 각도 제한 파라미터(`angle_min`/`angle_max`), 물리적 가림(마운트 위치/케이블) | [[ros2-nav-yahboom]] 후방 사각지대 해결 사례 — 파라미터 해제 + 마운트 높이 조정으로 360도 확보 |
| `/scan`이 RViz2에 안 보임 | `frame_id`와 TF 트리 이름 일치 여부, Fixed Frame 설정 | 실습 3·5단계 |
| 스캔에 노이즈가 많음 | 반사가 심한 표면(유리, 거울) 근처인지, `range_min`/`range_max` 필터링 | — |
| 드라이버가 실행 직후 죽음 | 시리얼 포트 권한(`dialout` 그룹), 포트 번호(`/dev/ttyUSB*`)가 실제와 맞는지 | 실습 1단계 |
| LiDAR와 카메라의 벽 위치가 어긋남 | 두 센서의 Static Transform 값이 실제 장착 위치와 다른 것 | 실습 6단계 |

## 6. 다음 문서와의 연결

- 다음: **[[04_colcon_빌드_오류_모음|응용 04. colcon 빌드 오류 모음]]** — 센서 드라이버를 소스로 빌드하다 막혔을 때 펼쳐보는 참조 문서다. 지금까지 apt로만 설치했다면 건너뛰어도 된다.
- 두 센서가 모두 붙었다면, 이들을 실제로 융합하는 **[[00_SLAM이란_무엇이고_왜_쓰는가|SLAM 공통 기초]]**로 넘어갈 준비가 된 것이다 — 다만 응용 시리즈의 나머지(04~06)를 먼저 훑어두면 이후 디버깅이 수월하다.

## 7. 이해도 점검

1. 드라이버가 실행되자마자 죽는다. 가장 먼저 확인할 것은?
2. `/scan`은 발행되는데 RViz2에 안 보인다. `frame_id`와 관련해 무엇을 확인하는가?
3. LiDAR 스캔과 카메라 포인트클라우드에서 같은 벽이 서로 다른 위치에 그려진다. 원인은 센서 고장인가?
4. 스캔 특정 구간이 항상 비어 있다. 소프트웨어 원인과 하드웨어 원인을 각각 하나씩 들어보라.
5. 유리벽 근처에서 스캔이 이상하게 찍히는 이유는 무엇인가?

> [!info]- 정답 및 해설 보기
> 1. **시리얼 포트 권한**이다. `ls -l /dev/ttyUSB*`로 장치를 확인하고 `dialout` 그룹에 사용자가 속해 있는지 본다. 그룹 추가 후 **재로그인해야 반영**된다는 점을 놓치기 쉽다.
> 2. `ros2 topic echo /scan --once`로 **메시지의 `header.frame_id`를 확인**하고, 그 이름이 TF 트리에 실제로 존재하는 좌표계 이름과 **정확히 일치**하는지 본다. 드라이버 파라미터로 `laser_link`를 줬는데 TF는 `laser`로 발행하고 있다면 RViz2는 아무것도 그리지 못한다.
> 3. **아니다.** 두 센서 각각은 정상이고, `base_link` 기준 **장착 위치(Static Transform) 값이 실제와 다른 것**이다. 실측값으로 x·y·z를 고쳐 넣으면 겹친다.
> 4. **소프트웨어**: `angle_min`/`angle_max` 각도 제한 파라미터가 그 구간을 잘라내고 있음. **하드웨어**: 마운트 구조물이나 케이블이 그 방향을 물리적으로 가리고 있음. 이 프로젝트에서는 **둘 다** 원인이었고, 파라미터 해제 + 마운트 높이 조정으로 해결했다([[ros2-nav-yahboom]]).
> 5. LiDAR는 레이저를 쏘고 반사를 받아 거리를 재는데, **유리는 레이저를 통과시키거나 엉뚱한 방향으로 반사**시킨다. 그래서 유리벽이 없는 것처럼 보이거나, 실제와 다른 거리로 찍힌다. 거울도 같은 이유로 문제가 된다.

## 8. 참고자료

- [sllidar_ros2 공식 GitHub](https://github.com/Slamtec/sllidar_ros2) — launch 인자와 파라미터 목록
- [[ros2-nav-yahboom]] — LiDAR 후방 사각지대 진단 및 해결 이력
