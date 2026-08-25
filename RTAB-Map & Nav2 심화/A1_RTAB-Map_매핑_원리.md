# A1. RTAB-Map 매핑 원리

> RTAB-Map & Nav2 심화 시리즈 · Part A. 지도제작
> 선행 학습: [ROS2 기초 0~9편](#) — 특히 3편(Topic/Message), 7편(TF2 기초)을 읽었다는 전제로 작성됨

## 1. 개요

RTAB-Map(Real-Time Appearance-Based Mapping)은 카메라·LiDAR 데이터를 이용해 **지도를 만들면서 동시에 로봇 자신의 위치를 추정**하는 그래프 기반 SLAM 라이브러리다. 이 문서는 이후 Nav2 문서들과 relocalization 심화 문서를 이해하기 위한 기초 개념을 다룬다.

Yahboom X3(RealSense D435i + RPLiDAR C1) 구성에서는 카메라의 시각 정보(Visual)와 LiDAR의 거리 정보(ICP)를 함께 사용하는 **Visual+LiDAR 융합 SLAM**으로 동작한다.

## 2. 핵심 개념

### 2.1 Appearance-Based Loop Closure

RTAB-Map의 핵심 아이디어는 "지금 보는 장면이 예전에 봤던 장면과 같은가?"를 이미지의 시각적 특징(bag-of-words)으로 판단하는 것이다.

- 카메라가 새 이미지를 받을 때마다, 이미지에서 특징점(SIFT, ORB, GFTT 등)을 추출해 "단어(word)" 사전에 등록한다.
- 새 이미지의 단어 집합과 과거에 저장해둔 위치들의 단어 집합을 비교해 유사도가 임계값(`Rtabmap/LoopThr`) 이상이면 **loop closure 후보**로 판단한다.
- 후보가 실제로 기하학적으로 정합되는지(`Vis/MinInliers`) 검증한 뒤 통과하면, 그래프에 새로운 제약(constraint)을 추가한다.
- 제약이 추가되면 그래프 최적화(g2o/GTSAM)가 실행되어 지금까지 쌓인 누적 오차를 전체적으로 재분배한다.

### 2.2 메모리 관리 (WM / STM / LTM)

RTAB-Map은 대규모 환경에서도 실시간 성능을 유지하기 위해 메모리를 3단계로 관리한다.

| 단계 | 이름 | 역할 |
|---|---|---|
| STM | Short-Term Memory | 가장 최근 프레임들을 순서대로 보관. 인접 프레임 간 오도메트리 제약을 만드는 데 사용 |
| WM | Working Memory | Loop Closure 탐색의 실제 대상이 되는 활성 프레임 집합. 크기가 커지면 오래되고 덜 특징적인 프레임이 LTM으로 이동 |
| LTM | Long-Term Memory | WM에서 밀려난 프레임들의 압축 저장소. 평소에는 loop closure 탐색 대상이 아니지만, WM으로 다시 불려올 수 있음 |

이 구조 덕분에 지도가 아무리 커져도 매 프레임의 처리 시간이 거의 일정하게 유지된다 — 이것이 RTAB-Map이 "Real-Time"을 이름에 걸 수 있는 이유다.

### 2.3 Odometry + Loop Closure = 그래프 SLAM

RTAB-Map의 지도는 결국 **노드(로봇이 지나간 위치들)와 엣지(위치 간 상대 변환)로 이루어진 그래프**다.

- 인접한 노드 사이의 엣지는 오도메트리(Visual Odometry 또는 ICP Odometry)로 만들어진다 — 이건 누적 드리프트가 생길 수밖에 없다.
- Loop closure로 만들어진 엣지는 "멀리 떨어진 두 시점이 사실은 같은 곳"이라는 강한 제약이다 — 이 제약이 그래프 최적화를 통해 누적 드리프트를 전체적으로 펴준다.

즉 **Loop Closure가 잘 안 되면 지도가 어긋난다**는 것은, 이 그래프에 드리프트를 바로잡아줄 제약이 부족하다는 뜻과 같다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

- **Visual Odometry**: RealSense D435i의 RGB-D 이미지로 프레임 간 카메라 이동을 추정한다.
- **ICP Odometry / Scan Matching**: RPLiDAR C1의 스캔으로 시각적 특징이 부족한 구간(복도, 흰 벽)을 보완한다.
- **2D Occupancy Grid**: LiDAR 스캔을 기반으로 Nav2가 쓰는 2D 점유 격자 지도를 함께 생성한다. RTAB-Map은 3D 포인트클라우드뿐 아니라 이 2D 지도도 동시에 만들어 `/map` 토픽으로 발행한다.
- **DetectionRate 조정**: D435i가 30fps로 데이터를 쏟아내지만, RTAB-Map은 매 프레임을 다 처리할 필요가 없어 `Rtabmap/DetectionRate`로 처리 주기를 낮춰 CPU 부하를 조절한다 (관련 진단 사례는 [[ros2-nav-yahboom]]의 카메라 CPU 병목 이슈 참고).

## 4. 관련 파라미터

| 파라미터 | 역할 | 참고 |
|---|---|---|
| `Rtabmap/LoopThr` | Loop closure 후보로 인정하는 유사도 임계값 | 기본 0.11 |
| `Vis/MinInliers` | 기하학적 검증 통과에 필요한 최소 특징점 매칭 수 | 실내 저텍스처 환경에서는 낮추는 경우가 많음 |
| `Mem/STMSize` | Short-Term Memory 크기 | 크면 최근 맥락을 더 오래 유지 |
| `Mem/RehearsalSimilarity` | 새 프레임이 기존 프레임과 얼마나 비슷하면 WM에 새로 추가하지 않을지 | 낮추면 더 다양한(distinct) 노드를 보존 |
| `Rtabmap/DetectionRate` | RTAB-Map이 프레임을 처리하는 주기 | 카메라 fps보다 낮게 설정해 CPU 부하 절감 |
| `Optimizer/Strategy` | 그래프 최적화 알고리즘 (0=TORO, 1=g2o, 2=GTSAM) | |

## 5. 진단 관점

- Loop closure가 잘 안 된다면: 조명 변화, 회전 속도 과다(모션 블러), `Vis/MinInliers`가 환경 대비 너무 높게 설정된 것을 먼저 의심한다.
- 지도가 이중으로 겹쳐 보인다면: 같은 장소를 다르게 인식해 별도 노드로 저장했다는 뜻 — loop closure 임계값을 완화하거나 수동으로 `detect_more_loop_closures` 서비스를 호출해 후처리할 수 있다.
- 실시간 처리가 느려진다면: WM 크기가 과도하게 커졌거나 `DetectionRate`가 카메라 fps 대비 너무 높게 설정된 것일 수 있다.

## 6. 다음 문서와의 연결

- 다음: **A2. 매핑 실행 파이프라인과 핵심 파라미터** — 여기서 설명한 개념들이 실제 `map_rtabmap_launch.py` 어느 부분에 대응하는지 다룬다.
- 이후 **C1~C6 Relocalization 심화**에서, "매핑 모드"와 "로컬라이제이션 모드"의 메모리 동작 차이(`Mem/IncrementalMemory`)가 relocalization의 핵심 메커니즘으로 다시 등장한다.

## 7. 참고자료

- RTAB-Map 공식 사이트: [introlab.github.io/rtabmap](https://introlab.github.io/rtabmap/) — appearance-based loop closure, 메모리 관리 개념 설명
- M. Labbé and F. Michaud, "Appearance-Based Loop Closure Detection for Online Large-Scale and Long-Term Operation" — WM/STM/LTM 구조와 실시간 성능 근거
- [rtabmap_ros GitHub 이슈 아카이브](https://github.com/introlab/rtabmap_ros/issues) — 실전 파라미터 동작 확인 (`Rtabmap/LoopThr`, `Vis/MinInliers` 등)
