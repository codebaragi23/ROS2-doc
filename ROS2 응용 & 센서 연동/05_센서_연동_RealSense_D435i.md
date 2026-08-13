# 센서 연동 - RealSense D435i

> ROS2 응용 & 센서 연동 시리즈 · 5편
> 선행 학습: ROS2 기초 7편(TF2 기초), 8편(rqt/RViz2 활용)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 3 — 센서 연동 실습 |
| 예상 선행 지식 | ROS2 기초 7편(TF2), 8편(RViz2), 5편(Parameter) |
| 학습 목표 | RealSense D435i를 ROS2에 연동하고 RViz2로 데이터를 검증할 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble, Jetson 계열 보드 |
| 관련 문서 | ROS2 기초 7·8·9편에서 예고된 심화 주제. nav_docs A2(매핑 파이프라인)의 카메라 입력이 이 문서의 결과물이다 |

---

## 1. 먼저 알아야 할 핵심

1. RealSense D435i는 RGB 카메라 + Depth 카메라 + IMU가 결합된 센서로, `realsense2_camera` 패키지가 ROS2 드라이버를 제공한다.
2. 이 센서를 연동한다는 것은 결국 지금까지 배운 개념들의 조합이다 — **노드 실행(2편) + Topic 발행(3편) + Parameter 설정(5편) + TF 발행(7편) + RViz2 확인(8편)**.
3. 해상도/fps 설정이 곧 CPU 부하와 직결된다 — 이 부분은 [[ros2-nav-yahboom]]에서 실제로 문제가 됐던 지점이다.

## 2. 이 개념은 무엇인가

`realsense2_camera` 노드는 D435i 하드웨어로부터 RGB 이미지, Depth 이미지, IMU 데이터를 받아 각각 Topic으로 발행하고, 카메라 각 부품(RGB 렌즈, Depth 렌즈, IMU)의 위치 관계를 TF로 함께 발행한다.

## 3. 왜 필요한가

nav_docs A1~A3(RTAB-Map 매핑)이 실제로 동작하려면 RGB-D 데이터가 안정적으로 들어와야 한다. 이 문서는 그 입력을 만드는 첫 단계다. RViz2로 이미지·포인트클라우드·TF를 직접 확인하는 절차는 8편에서 배운 Display 활용의 실전 적용이다.

## 4. 주요 Topic과 좌표계

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

이 좌표계들은 7편에서 배운 `base_link → laser_link` Static Transform과 같은 원리로, `base_link → camera_link`가 로봇 몸체 기준 카메라 장착 위치를 나타내는 Static Transform이 되어야 한다.

## 5. 기본 실습

### 설치

```bash
sudo apt install ros-humble-realsense2-camera
```

### 실행

```bash
ros2 launch realsense2_camera rs_launch.py \
  depth_module.profile:=640x480x15 \
  rgb_camera.profile:=640x480x15 \
  enable_gyro:=true \
  enable_accel:=true
```

* `depth_module.profile`/`rgb_camera.profile`: 5편에서 배운 Parameter 지정 방식과 동일한 개념이다. [[ros2-nav-yahboom]]에서 1280x720x30이 CPU 47%를 잡아먹어 640x480x15로 낮췄던 사례가 이 값의 실전 기준이다.
* `enable_gyro`/`enable_accel`: IMU 데이터를 함께 발행할지 결정. RTAB-Map에서 IMU 융합 오도메트리를 쓰려면 활성화해야 한다.

### 확인

```bash
ros2 topic hz /camera/color/image_raw
ros2 run rviz2 rviz2
```

RViz2에서 `Image`, `PointCloud2`, `TF` Display를 추가해 실제 영상과 포인트클라우드, 좌표계가 정상 표시되는지 확인한다(8편 실습과 동일한 절차).

## 6. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| CPU 사용량이 지나치게 높음 | 해상도/fps 설정부터 낮춤 (2-2편의 Composable Node도 대안) |
| RViz2에 포인트클라우드가 안 보임 | `Fixed Frame`이 `camera_link`나 그 상위 좌표계로 설정됐는지(7편), Topic이 정확히 지정됐는지(8편) |
| 이미지와 Depth가 어긋나 보임 | `align_depth` 옵션이 켜져 있는지, RGB/Depth 광학 좌표계 TF가 올바른지 |

## 7. 다음 학습 주제

- 다음: **센서 연동 - LiDAR C1** — 이 문서와 같은 절차로 LiDAR를 연동한 뒤, 두 센서의 TF가 `base_link` 기준으로 일관되게 배치됐는지 함께 검증한다.
- 이 문서의 결과물(카메라 데이터+TF)은 nav_docs **A2. 매핑 실행 파이프라인**의 입력으로 그대로 이어진다.

## 8. 참고자료

- realsense2_camera 공식 GitHub — 파라미터 전체 목록, launch 인자
- [[ros2-nav-yahboom]] — 이 프로젝트의 실제 해상도/fps 튜닝 이력
