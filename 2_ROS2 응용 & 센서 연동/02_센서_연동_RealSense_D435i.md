# 센서 연동 - RealSense D435i

> ROS2 응용 & 센서 연동 시리즈 · 2편
> 선행 학습: ROS2 기초 7편(TF2 기초), 8편(rqt/RViz2 활용), 5편(Parameter), [[01_DDS와_QoS_이해하기|응용 1편(DDS와 QoS)]] — 센서가 안 보이는 문제의 1순위 원인

## 1. 개요

RealSense D435i는 RGB 카메라 + Depth 카메라 + IMU가 결합된 센서로, `realsense2_camera` 패키지가 ROS2 드라이버를 제공한다. 이 문서는 이 센서를 실제로 연동하고 검증하는 절차를 다룬다 — 결국 지금까지 배운 개념들의 조합이다: **노드 실행 + Topic 발행 + Parameter 설정 + TF 발행 + RViz2 확인.** 이 결과물은 지상로봇 적용의 A2(매핑 실행 파이프라인)의 카메라 입력으로 그대로 이어진다.

## 2. 핵심 개념

`realsense2_camera` 노드는 D435i 하드웨어로부터 RGB 이미지, Depth 이미지, IMU 데이터를 받아 각각 Topic으로 발행하고, 카메라 각 부품(RGB 렌즈, Depth 렌즈, IMU)의 위치 관계를 TF로 함께 발행한다.

| Topic | 내용 |
|---|---|
| `/camera/color/image_raw` | RGB 이미지 |
| `/camera/depth/image_rect_raw` | 정렬된 Depth 이미지 |
| `/camera/depth/color/points` | RGB-D를 합친 포인트클라우드 |
| `/camera/imu` | 내장 IMU 데이터 |

| TF 좌표계 | 의미 |
|---|---|
| `camera_link` | 카메라 전체의 기준 좌표계 |
| `camera_color_optical_frame` | RGB 렌즈 기준(광학 좌표계, z축이 렌즈 방향) |
| `camera_depth_optical_frame` | Depth 렌즈 기준 |

이 좌표계들은 ROS2 기초 7편에서 배운 `base_link → laser_link` Static Transform과 같은 원리로, `base_link → camera_link`가 로봇 몸체 기준 카메라 장착 위치를 나타내는 Static Transform이 되어야 한다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

```bash
sudo apt install ros-humble-realsense2-camera

ros2 launch realsense2_camera rs_launch.py \
  depth_module.profile:=640x480x15 \
  rgb_camera.profile:=640x480x15 \
  enable_gyro:=true \
  enable_accel:=true
```

`depth_module.profile`/`rgb_camera.profile`은 ROS2 기초 5편에서 배운 Parameter 지정 방식과 동일한 개념이다. [[ros2-nav-yahboom]]에서 1280x720x30이 CPU 47%를 잡아먹어 640x480x15로 낮췄던 사례가 이 값의 실전 기준이다. `enable_gyro`/`enable_accel`은 IMU 데이터를 함께 발행할지 결정하며, RTAB-Map에서 IMU 융합 오도메트리를 쓰려면 활성화해야 한다.

## 4. 확인

```bash
ros2 topic hz /camera/color/image_raw
ros2 run rviz2 rviz2
```

RViz2에서 `Image`, `PointCloud2`, `TF` Display를 추가해 실제 영상과 포인트클라우드, 좌표계가 정상 표시되는지 확인한다(ROS2 기초 8편 실습과 동일한 절차).

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| CPU 사용량이 지나치게 높음 | 해상도/fps 설정부터 낮춤 (이 시리즈 2편의 Composable Node도 대안) |
| RViz2에 포인트클라우드가 안 보임 | `Fixed Frame`이 `camera_link`나 그 상위 좌표계로 설정됐는지, Topic이 정확히 지정됐는지 |
| 이미지와 Depth가 어긋나 보임 | `align_depth` 옵션이 켜져 있는지, RGB/Depth 광학 좌표계 TF가 올바른지 |

## 6. 다음 문서와의 연결

- 다음: **[[03_센서_연동_LiDAR_C1|센서 연동 - LiDAR C1]]** — 이 문서와 같은 절차로 LiDAR를 연동한 뒤, 두 센서의 TF가 `base_link` 기준으로 일관되게 배치됐는지 함께 검증한다.

## 7. 참고자료

- [realsense2_camera 공식 GitHub](https://github.com/IntelRealSense/realsense-ros) — 파라미터 전체 목록, launch 인자
- [[ros2-nav-yahboom]] — 이 프로젝트의 실제 해상도/fps 튜닝 이력
