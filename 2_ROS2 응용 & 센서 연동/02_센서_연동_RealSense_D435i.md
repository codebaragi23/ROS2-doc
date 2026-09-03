# 02. 센서 연동 - RealSense D435i

> ROS2 응용 & 센서 연동 시리즈

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 응용 (첫 실제 하드웨어 연동) |
| 예상 선행 지식 | [[07_TF2_기초\|ROS2 기초 07. TF2 기초]], [[08_rqt_rviz2_활용\|08. rqt/RViz2 활용]], [[05_Parameter와_실행_설정\|05. Parameter]], [[01_DDS와_QoS_이해하기\|응용 01. DDS와 QoS]] — 센서가 안 보이는 문제의 1순위 원인 |
| 학습 목표 | RealSense 드라이버를 실행해 RGB·Depth·IMU 토픽을 확인할 수 있다 / 카메라가 발행하는 TF 좌표계들의 관계를 설명할 수 있다 / 해상도·fps가 CPU 부하와 어떤 관계인지 판단해 설정할 수 있다 / RViz2에서 포인트클라우드가 안 보일 때 원인을 좁힐 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble, Intel RealSense D435i |
| 관련 문서 | 이전: [[01_DDS와_QoS_이해하기\|응용 01. DDS와 QoS]] / 다음: [[03_센서_연동_LiDAR_C1\|응용 03. LiDAR C1]] |

## 1. 개요

RealSense D435i는 RGB 카메라 + Depth 카메라 + IMU가 결합된 센서로, `realsense2_camera` 패키지가 ROS2 드라이버를 제공한다. 이 문서는 이 센서를 실제로 연동하고 검증하는 절차를 다룬다 — 결국 지금까지 배운 개념들의 조합이다: **노드 실행 + Topic 발행 + Parameter 설정 + TF 발행 + RViz2 확인.** 이 결과물은 지상로봇 적용의 01(매핑 실행 파이프라인)의 카메라 입력으로 그대로 이어진다.

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

이 좌표계들은 ROS2 기초 07에서 배운 `base_link → laser_link` Static Transform과 같은 원리로, `base_link → camera_link`가 로봇 몸체 기준 카메라 장착 위치를 나타내는 Static Transform이 되어야 한다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

```bash
sudo apt install ros-humble-realsense2-camera

ros2 launch realsense2_camera rs_launch.py \
  depth_module.profile:=640x480x15 \
  rgb_camera.profile:=640x480x15 \
  enable_gyro:=true \
  enable_accel:=true
```

`depth_module.profile`/`rgb_camera.profile`은 ROS2 기초 05에서 배운 Parameter 지정 방식과 동일한 개념이다. [[ros2-nav-yahboom]]에서 1280x720x30이 CPU 47%를 잡아먹어 640x480x15로 낮췄던 사례가 이 값의 실전 기준이다. `enable_gyro`/`enable_accel`은 IMU 데이터를 함께 발행할지 결정하며, RTAB-Map에서 IMU 융합 오도메트리를 쓰려면 활성화해야 한다.

## 4. 실습 — 센서를 붙이고 끝까지 확인하기

### 실습 목표

드라이버 실행 → 토픽 확인 → TF 확인 → RViz2 시각화까지, **"센서가 정상 동작한다"고 말할 수 있는 근거를 단계별로 확보**한다. 중간에 끊기면 어느 단계에서 막혔는지가 곧 원인의 위치다.

### 준비 사항

- RealSense D435i가 **USB 3.0 포트**에 연결되어 있을 것(USB 2.0에서는 고해상도 스트림이 실패한다)
- 터미널 2~3개

### 절차

**1단계 — 하드웨어가 인식되는지 (ROS2 이전 문제부터 배제)**

```bash
rs-enumerate-devices --compact
```

여기서 장치가 안 보이면 ROS2 문제가 아니다. 케이블·포트·권한을 먼저 해결한다.

**2단계 — 드라이버 실행**

```bash
ros2 launch realsense2_camera rs_launch.py \
  depth_module.profile:=640x480x15 \
  rgb_camera.profile:=640x480x15 \
  enable_gyro:=true \
  enable_accel:=true
```

**3단계 — 토픽이 실제로 흐르는지**

```bash
ros2 topic list | grep camera
ros2 topic hz /camera/color/image_raw
ros2 topic hz /camera/depth/color/points
```

`hz`가 설정한 fps(여기서는 15)에 근접하게 나와야 한다. 목록에는 있는데 `hz`가 멈춰 있다면 [[01_DDS와_QoS_이해하기|응용 01]]의 QoS 문제를 의심한다.

**4단계 — TF 트리 확인**

```bash
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo camera_link camera_color_optical_frame
```

카메라 내부 좌표계들이 서로 연결되어 있는지 확인한다. 이 단계에서 `base_link → camera_link`가 **없는 것이 정상**이다 — 그건 로봇 몸체에 카메라를 어디에 달았는지의 정보라 사용자가 직접 발행해야 한다(5단계).

**5단계 — 로봇 몸체 기준 장착 위치 발행**

```bash
# 예: 로봇 중심에서 전방 10cm, 높이 20cm에 카메라를 장착한 경우
ros2 run tf2_ros static_transform_publisher \
  0.1 0 0.2 0 0 0 base_link camera_link
```

[[07_TF2_기초|ROS2 기초 07]]에서 배운 Static Transform이 여기서 실제로 쓰인다.

**6단계 — RViz2 시각화**

```bash
ros2 run rviz2 rviz2
```

`Image`, `PointCloud2`, `TF` Display를 추가한다. **`Fixed Frame`을 `camera_link`(또는 `base_link`)로 반드시 바꾼다** — 기본값 `map`은 아직 존재하지 않는 좌표계라 아무것도 안 보인다.

### 예상 결과

- `hz`가 15 근처로 안정적으로 나온다.
- RViz2에 RGB 영상과 3D 포인트클라우드가 함께 표시되고, 손을 카메라 앞에서 움직이면 포인트클라우드도 따라 움직인다.
- TF Display에 카메라 좌표계들이 `base_link` 아래에 매달려 표시된다.

### 확인해볼 것 — CPU 부하 실험

이 프로젝트에서 실제로 문제가 됐던 지점이다. 해상도를 바꿔가며 `htop`으로 CPU를 관찰해본다.

```bash
# 고해상도로 재실행 후 CPU 관찰
ros2 launch realsense2_camera rs_launch.py \
  depth_module.profile:=1280x720x30 rgb_camera.profile:=1280x720x30
```

Jetson급 보드에서는 이 설정이 CPU를 크게 잡아먹는다([[ros2-nav-yahboom]]에 47% 점유 기록). **"센서는 잘 붙었는데 나중에 SLAM이 느리다"의 원인이 여기서 이미 만들어진다**는 것을 미리 체감해두는 것이 이 실험의 목적이다.

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| CPU 사용량이 지나치게 높음 | 해상도/fps 설정부터 낮춤 ([[06_Composable_Node와_Executor_구조\|응용 06]]의 Composable Node도 대안) |
| RViz2에 포인트클라우드가 안 보임 | `Fixed Frame`이 `camera_link`나 그 상위 좌표계로 설정됐는지, Topic이 정확히 지정됐는지 |
| 이미지와 Depth가 어긋나 보임 | `align_depth` 옵션이 켜져 있는지, RGB/Depth 광학 좌표계 TF가 올바른지 |

## 6. 다음 문서와의 연결

- 다음: **[[03_센서_연동_LiDAR_C1|응용 03. 센서 연동 - LiDAR C1]]** — 이 문서와 같은 절차로 LiDAR를 연동한 뒤, 두 센서의 TF가 `base_link` 기준으로 일관되게 배치됐는지 함께 검증한다.
- 이 문서의 결과물(RGB-D + IMU)은 [[01_매핑_실행_파이프라인과_핵심_파라미터|지상로봇 적용 - 지도제작 - 01. 매핑 실행 파이프라인]]의 카메라 입력으로 그대로 이어진다.

## 7. 이해도 점검

1. `rs-enumerate-devices`로 먼저 확인하는 이유는 무엇인가?
2. 드라이버를 실행했는데 TF 트리에 `base_link → camera_link`가 없다. 이것은 오류인가?
3. RViz2에 포인트클라우드가 전혀 안 보인다. `Fixed Frame` 외에 확인할 것은?
4. `enable_gyro`/`enable_accel`을 켜야 하는 경우는 언제인가?
5. 해상도를 1280x720x30에서 640x480x15로 낮추면 무엇이 좋아지고 무엇을 잃는가?

> [!info]- 정답 및 해설 보기
> 1. **ROS2 이전 계층(하드웨어·USB·권한) 문제를 먼저 배제하기 위해서**다. 여기서 장치가 안 보이는데 ROS2 드라이버를 아무리 고쳐도 소용없다. 문제를 아래 계층부터 잘라내는 것이 진단의 기본이다.
> 2. **오류가 아니다. 정상이다.** 드라이버는 카메라 *내부* 좌표계(`camera_link → camera_color_optical_frame` 등)만 발행한다. "로봇 몸체 어디에 카메라를 달았는지"는 하드웨어 조립 정보라 드라이버가 알 수 없으므로, 사용자가 `static_transform_publisher`로 직접 발행해야 한다.
> 3. **토픽이 실제로 흐르는지(`ros2 topic hz`)**를 확인한다. `hz`가 멈춰 있다면 QoS 불일치([[01_DDS와_QoS_이해하기|응용 01]])일 가능성이 높다. 그 외에 Display의 Topic 이름이 정확한지, `Reliability` 설정이 Publisher와 맞는지도 본다.
> 4. **IMU 데이터가 필요할 때** — 구체적으로는 RTAB-Map에서 IMU 융합 오도메트리를 쓰거나, 나중에 VIO 계열 백엔드(OpenVINS 등)를 붙일 때다. 안 쓸 거면 꺼두는 편이 부하가 줄어든다.
> 5. **좋아지는 것**: CPU 부하가 크게 줄어 SLAM·Nav2 등 뒤따르는 노드에 연산 여유가 생긴다. **잃는 것**: 이미지 해상도와 프레임레이트 — 즉 세밀한 특징점과 빠른 움직임 대응력이 떨어진다. 이 트레이드오프의 판단 기준은 "SLAM이 실시간을 유지하는가"이지 "화질이 좋은가"가 아니다.

## 8. 참고자료

- [realsense2_camera 공식 GitHub](https://github.com/IntelRealSense/realsense-ros) — 파라미터 전체 목록, launch 인자
- [[ros2-nav-yahboom]] — 이 프로젝트의 실제 해상도/fps 튜닝 이력
