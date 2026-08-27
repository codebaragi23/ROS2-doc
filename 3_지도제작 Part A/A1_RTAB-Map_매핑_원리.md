# A1. RTAB-Map 매핑 원리

> 지도제작 (Part A) 시리즈
> 이전 문서: [[A0_SLAM이란_무엇이고_왜_쓰는가|A0. SLAM이란 무엇이고 왜 쓰는가]]
> 선행 학습: `ROS2 기초` 0~9편 — 특히 [[03_Topic과_Message|3편(Topic/Message)]], [[07_TF2_기초|7편(TF2 기초)]]을 읽었다는 전제로 작성됨

## 1. 개요

RTAB-Map(Real-Time Appearance-Based Mapping)은 카메라·LiDAR 데이터를 이용해 **지도를 만들면서 동시에 로봇 자신의 위치를 추정**하는 그래프 기반 SLAM 라이브러리다. [[A0_SLAM이란_무엇이고_왜_쓰는가|A0]]에서 배운 SLAM의 일반 개념(Odometry, Mapping, Loop Closure, 프론트엔드/백엔드 구분)이 RTAB-Map에서 실제로 어떻게 구현되는지가 이 문서의 주제다. 이 문서는 이후 Nav2 문서들과 relocalization 심화 문서를 이해하기 위한 기초 개념을 다룬다. RTAB-Map은 카메라만(Visual), LiDAR만(ICP), 또는 둘을 함께 쓰는 Visual+LiDAR 융합 방식 중 원하는 조합으로 구성할 수 있다.

## 2. 핵심 개념

### 2.1 먼저: 특징점, 기술자, 그리고 inlier

Appearance-Based Loop Closure를 이해하려면 그 재료가 되는 세 가지 개념을 먼저 알아야 한다.

**특징점(feature/keypoint)** — 이미지에서 **조명이나 보는 각도가 조금 달라져도 다시 찾아낼 수 있는 눈에 띄는 지점**이다. 보통 코너(모서리), 얼룩진 무늬처럼 주변과 뚜렷이 구별되는 곳이 선택된다. 평평한 흰 벽은 어디를 찍어도 주변과 똑같아 보이므로 특징점이 거의 안 나오는데, 이것이 이 시리즈에서 "저텍스처 환경이 어렵다"고 반복해 말하는 이유다.

- 특징점을 **찾아내는** 알고리즘(검출기)과, 찾은 지점 주변을 **숫자열로 요약하는** 알고리즘(기술자)은 서로 다른 역할이다. `SIFT`, `ORB`, `GFTT` 같은 이름은 이 둘 중 하나 또는 둘 다를 가리킨다 — 종류별 차이는 A2의 `Vis/FeatureType`과 [[D0-2_특징점_검출_기술자_매칭_비교|D0-2]]에서 다루며, 지금은 "이런 알고리즘들이 특징점을 뽑아준다" 정도면 충분하다.

**bag-of-words(시각 단어 가방)** — 이미지를 **문서처럼 다뤄 검색하는 기법**이다. 비슷하게 생긴 특징점들을 미리 모아 "시각 단어 사전"을 만들어 두고, 이미지 한 장이 들어오면 그 안의 특징점들이 각각 어떤 단어에 해당하는지 세어 **단어 출현 횟수 목록(히스토그램)**으로 바꾼다. 그러면 "이 사진과 저 사진이 닮았는가"라는 질문이 "이 두 문서가 비슷한 단어를 쓰는가"라는 **문서 검색 문제**로 바뀌어 매우 빠르게 비교할 수 있다. RTAB-Map 이름의 "Appearance-Based"가 바로 이 방식을 가리킨다.

**inlier / outlier** — 두 이미지의 특징점을 짝지어 보면 **일부는 반드시 잘못 짝지어진다**(비슷하게 생긴 다른 지점끼리 매칭되는 경우). 이때 "카메라가 이만큼 움직였다"는 **하나의 일관된 위치 변환으로 설명되는 매칭들이 inlier**, 그 변환과 어긋나는 엉뚱한 매칭이 **outlier**다. 잘못된 짝을 걸러내고 inlier만 남기는 대표적인 방법이 RANSAC이며, 남은 inlier 개수가 충분한지를 보는 것이 `Vis/MinInliers`의 역할이다.

> **왜 중요한가**: inlier 개수는 이 시리즈 전체에서 "지금 위치 추정을 믿을 수 있는가"를 판단하는 핵심 지표로 계속 등장한다(C1, C2-1, C7). inlier가 급감했다는 것은 곧 "지금 보는 장면을 지도의 어느 곳과도 제대로 맞추지 못하고 있다"는 뜻이다.

### 2.2 Appearance-Based Loop Closure

RTAB-Map의 핵심 아이디어는 "지금 보는 장면이 예전에 봤던 장면과 같은가?"를 위에서 설명한 bag-of-words로 판단하는 것이다.

- 카메라가 새 이미지를 받을 때마다, 이미지에서 특징점을 추출해 시각 단어 사전에 대응시킨다.
- 새 이미지의 단어 집합과 과거에 저장해둔 위치들의 단어 집합을 비교해 유사도가 임계값(`Rtabmap/LoopThr`) 이상이면 **loop closure 후보**로 판단한다.
- 후보가 실제로 기하학적으로 정합되는지(inlier가 `Vis/MinInliers` 이상인지) 검증한 뒤 통과하면, 그래프에 새로운 제약(constraint)을 추가한다.
- 제약이 추가되면 그래프 최적화(`Optimizer/Strategy`로 고르는 g2o/GTSAM 등)가 실행되어 지금까지 쌓인 누적 오차를 전체적으로 재분배한다. 셋 다 같은 그래프 최적화 문제를 푸는 서로 다른 구현이므로, 특별한 이유가 없으면 기본값을 그대로 쓰면 된다.

### 2.3 ICP — LiDAR로 위치를 맞추는 방법

카메라가 이미지의 특징점으로 위치를 맞춘다면, LiDAR는 **점군(point cloud)끼리 직접 겹쳐 맞추는** 방식을 쓴다. 이것이 **ICP(Iterative Closest Point, 반복 최근접점)**다.

1. 두 스캔에서 서로 **가장 가까운 점끼리** 임시로 짝을 짓는다.
2. 그 짝들의 거리 합이 최소가 되는 회전·이동 변환을 계산해 한쪽 스캔을 옮긴다.
3. 옮긴 상태에서 다시 1번으로 돌아가 짝을 새로 짓는다 — 이 과정을 수렴할 때까지 **반복(Iterative)**한다.

ICP는 시각적 무늬가 전혀 없는 흰 벽 복도에서도 벽면의 기하학적 형태만으로 동작하므로, 카메라가 취약한 구간을 보완해준다. 점을 점에 맞출지(point-to-point) 점을 면에 맞출지(point-to-plane)의 선택은 A2의 `Icp/PointToPlane`에서 다룬다.

### 2.4 메모리 관리 (WM / STM / LTM)

RTAB-Map은 대규모 환경에서도 실시간 성능을 유지하기 위해 메모리를 3단계로 관리한다.

| 단계 | 이름 | 역할 |
|---|---|---|
| STM | Short-Term Memory | 가장 최근 프레임들을 순서대로 보관. 인접 프레임 간 오도메트리 제약을 만드는 데 사용 |
| WM | Working Memory | Loop Closure 탐색의 실제 대상이 되는 활성 프레임 집합. 크기가 커지면 오래되고 덜 특징적인 프레임이 LTM으로 이동 |
| LTM | Long-Term Memory | WM에서 밀려난 프레임들의 압축 저장소. 평소에는 loop closure 탐색 대상이 아니지만, WM으로 다시 불려올 수 있음 |

이 구조 덕분에 지도가 아무리 커져도 매 프레임의 처리 시간이 거의 일정하게 유지된다 — 이것이 RTAB-Map이 "Real-Time"을 이름에 걸 수 있는 이유다.

### 2.5 Odometry + Loop Closure = 그래프 SLAM

RTAB-Map의 지도는 결국 **노드(로봇이 지나간 위치들)와 엣지(위치 간 상대 변환)로 이루어진 그래프**다.

- 인접한 노드 사이의 엣지는 오도메트리(Visual Odometry 또는 ICP Odometry)로 만들어진다 — 이건 누적 드리프트가 생길 수밖에 없다.
- Loop closure로 만들어진 엣지는 "멀리 떨어진 두 시점이 사실은 같은 곳"이라는 강한 제약이다 — 이 제약이 그래프 최적화를 통해 누적 드리프트를 전체적으로 펴준다.

즉 **Loop Closure가 잘 안 되면 지도가 어긋난다**는 것은, 이 그래프에 드리프트를 바로잡아줄 제약이 부족하다는 뜻과 같다.

**"그래프 최적화"는 구체적으로 무엇을 하는가**

이 표현이 이 시리즈 전체에서 계속 등장하므로 한 번 정확히 짚고 간다. 각 엣지에는 두 가지 값이 있다.

1. **관측된 상대 변환** — 오도메트리나 loop closure가 "이 두 위치는 이만큼 떨어져 있다"고 measured한 값
2. **현재 노드 위치로 계산한 상대 변환** — 지금 그래프에 찍혀 있는 두 노드 좌표를 빼서 나오는 값

이 둘이 완벽히 일치하면 오차가 0이지만, 실제로는 항상 어긋난다. **그래프 최적화란 모든 엣지의 이 어긋남(오차)을 제곱해 더한 총합이 가장 작아지도록, 노드들의 위치를 전부 조금씩 옮기는 계산**이다.

여기서 두 가지 중요한 결과가 따라온다.

- Loop closure 엣지가 하나 추가되면 **그 엣지 하나 때문에 지도 전체의 노드 위치가 한꺼번에 조금씩 이동한다** — 로봇의 현재 위치도 그중 하나이므로, 이것이 C4에서 다룰 **Pose Jump(위치가 순간적으로 튀는 현상)**의 물리적 원인이다.
- 반대로 **잘못된 loop closure 엣지가 하나라도 섞이면**, 최적화는 그 틀린 제약까지 만족시키려 애쓰다가 지도 전체를 비틀어버린다 — 이것이 A3에서 다룰 false positive loop closure가 치명적인 이유이자, `RGBD/OptimizeMaxError`가 오차가 과도한 엣지를 아예 거부하는 이유다.

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
