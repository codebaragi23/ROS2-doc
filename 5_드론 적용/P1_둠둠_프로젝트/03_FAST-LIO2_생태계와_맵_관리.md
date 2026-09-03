# 03. FAST-LIO2 생태계와 맵 관리

> 드론 적용 — 둠둠 프로젝트(P1) 시리즈
> 이전 문서: [[02_빠른_적용_로드맵과_체크리스트|02. 빠른 적용 로드맵과 체크리스트]]

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 4 — 프로젝트 (패키지 선정) |
| 예상 선행 지식 | [[02_빠른_적용_로드맵과_체크리스트\|둠둠 02]] |
| 학습 목표 | FAST-LIO2 생태계의 ROS2 지원 현황을 안다 / 왜 통합 패키지를 택했는지 설명할 수 있다 / 매핑 시 다각도 커버리지가 왜 중요한지 안다 / 맵 저장 포맷과 재위치 입력의 호환을 확인할 수 있다 |
| 기준 환경 | ROS2 Humble, Livox Mid-360, livox_ros_driver2 |

## 1. 개요

[[01_통합_노드_아키텍처|01]]에서 "odom 노드는 FAST-LIO2, 그 위에 루프클로저/맵 저장이 얹힌다"고 그렸는데, **실제로 어떤 패키지를 쓸 것인가**는 별도로 조사가 필요했다 — FAST-LIO2 생태계의 확장 패키지 상당수가 아직 ROS1/구형 Python 기반이기 때문이다. 이 문서는 조사 결과와 최종 선택, 그리고 맵 관리 관련 실무 사항을 정리한다.

## 2. 핵심 개념: 패키지 선택 — 왜 하나로 합쳐지는가

### 2.1 조사 결과 요약

| 후보 | 상태 | 판정 |
|---|---|---|
| `hku-mars/FAST_LIO`(원저장소) | 메인 브랜치는 ROS1 전용. 별도 `ROS2` 브랜치가 존재하지만 README에 안내가 없고 활동이 적음 | 기준선으로만 참고 |
| `Ericsii/FAST_LIO_ROS2` | ROS2(Humble 권장) 포팅, 꾸준히 유지보수됨 | FAST-LIO2 단독이 필요하면 안전한 선택 |
| `gisbi-kim/FAST_LIO_SLAM`(Scan Context 루프클로저) | **ROS1 전용, 사실상 유지보수 중단**. ROS2 포트(`rohrschacht/FAST_LIO_SLAM_ros2`)가 있지만 이것도 정체됨 | 비권장 |
| `HViktorTsoi/FAST_LIO_LOCALIZATION`(재위치) | **ROS1 + Python 2.7 + Open3D 0.9 확정**. Ubuntu 22.04에 올리려면 Python 2.7 자체를 되살려야 함 | **포팅하지 않는다** |
| `deepglint/FAST_LIO_LOCALIZATION_HUMANOID` | ROS2(Humble 브랜치) 지원을 공식 명시, 활발히 유지보수됨. 이름은 "휴머노이드"지만 내용은 범용 LiDAR 재위치 | 대안 후보 |
| **`liangheming/FASTLIO2_ROS2`** | **FAST-LIO2 + PGO(루프클로저) + 온라인 재위치를 한 워크스페이스에 통합**. ROS2 Humble 네이티브, 가장 최근까지 유지보수됨 | **1순위 채택** |

### 2.2 최종 선택 — `liangheming/FASTLIO2_ROS2`

원래 이 프로젝트의 아이디어는 "FAST-LIO2(odometry) + Scan Context 루프클로저(별도 패키지) + FAST-LIO-LOCALIZATION(별도 패키지)"를 각각 붙이는 것이었다. 조사 결과 **그중 둘(루프클로저, 재위치)이 사실상 죽은 ROS1/구형 스택**이라, 대신 **odometry·루프클로저·재위치를 한 번에 제공하는 `liangheming/FASTLIO2_ROS2`를 채택**한다.

이 선택은 [[00_프로젝트_개요와_시나리오_정의|00]]이 원했던 "작은 노드 조합"이라는 방향과 완전히 배치되지는 않는다 — **odometry, 매핑(PGO), 재위치가 여전히 별도 ROS2 노드/서비스로 분리**되어 있고([[01_통합_노드_아키텍처|01]]의 ①/②-매핑/②-재위치 구분과 그대로 대응한다), 다만 "누가 관리하는 패키지인가"가 하나로 합쳐질 뿐이다. RTAB-Map처럼 설정과 내부 동작까지 하나로 뭉친 프레임워크가 아니라, **패키지는 하나지만 기능은 여전히 노드 단위로 나뉘어 있는 구조**다.

> **주의**: README가 중국어로만 제공된다. 빌드·실행 명령은 코드/launch 파일을 직접 읽고 확인해야 하며, Phase 1([[02_빠른_적용_로드맵과_체크리스트|02]])에서 이 부분에 시간이 걸릴 수 있음을 감안한다.

### 2.3 LiDAR 드라이버 — `livox_ros_driver2`

이 프로젝트가 쓰는 **Livox Mid-360**(00번 문서 하드웨어 표 참고)의 드라이버는 **`Livox-SDK/livox_ros_driver2`**다. ROS2 Humble을 공식 지원하며 최근까지 활발히 유지보수되고 있고, Mid-360을 명시적으로 지원 목록에 포함한다. 구형 `livox_ros_driver`(v1)는 구형 SDK/기기용이라 혼동하지 않는다.

## 3. 핵심 개념: 매핑 시 고려사항

[[02_빠른_적용_로드맵과_체크리스트|02]] Phase 1~2에서 실제로 확인해야 할 것들이다.

- **루프클로저가 사무실 규모에서도 필요한가**: 필요하다. 아무리 작은 공간이라도 "한 바퀴 돌아 시작점 복귀" 패턴이면 드리프트 보정 없이는 지도 끝단이 어긋난다.
- **다각도 커버리지**: 재위치([[04_재위치_레이어와_map_odom_보정|04]])는 "매핑 시와 비슷한 각도로 지나갈 때" 매칭이 잘 된다. 사무실 구석구석을 다양한 각도로 스캔해둬야 2차 비행(재위치)이 자유로운 시작 위치를 가질 수 있다 — **편도 1회만 지나가면 재위치 강건성이 떨어진다.**
- **맵 저장 포맷**: `liangheming/FASTLIO2_ROS2`는 PGO 결과를 `.pcd`로 저장하는 서비스(`/pgo/save_maps`)를 제공한다. 재위치 노드(`/localizer/relocalize`)도 같은 `.pcd` 경로를 입력으로 받으므로, **포맷 변환 없이 그대로 연결**된다 — 원래 우려했던 "저장 포맷을 재위치 노드 입력에 맞춰 변환해야 하는가" 문제가 이 패키지 선택으로 해소된다.
- **고정 고도의 일관성**: 매핑 비행과 재위치 비행의 고도가 다르면 재위치용 스캔 형상 자체가 달라져 매칭이 실패할 수 있다 — 두 비행의 고도를 동일하게 맞춘다([[00_프로젝트_개요와_시나리오_정의|00]] 2.2절 전제와 직결).

## 4. 이 프로젝트에서의 적용

- Phase 1([[02_빠른_적용_로드맵과_체크리스트|02]])에서 우선 `Ericsii/FAST_LIO_ROS2`(단순 구조)로 FAST-LIO2 odometry 자체의 안정성만 먼저 확인해보는 것도 유효한 전략이다 — 문제가 생겼을 때 "odometry 자체 문제"와 "PGO/재위치 통합 문제"를 분리해서 진단할 수 있다.
- Phase 2부터는 `liangheming/FASTLIO2_ROS2`로 전환해 루프클로저·재위치까지 통합 검증한다.
- `deepglint/FAST_LIO_LOCALIZATION_HUMANOID`(Humble 브랜치)는 `liangheming/FASTLIO2_ROS2`의 재위치 정확도/강건성이 기대에 못 미칠 경우의 **대안 재위치 노드**로 남겨둔다 — odometry(FAST-LIO2)는 그대로 두고 재위치 레이어만 교체 가능한 구조라는 것이 [[01_통합_노드_아키텍처|01]]에서 계층을 나눠둔 실질적 이점이다.

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| 빌드가 안 됨 | README가 중국어라 의존성을 놓쳤을 가능성 — `package.xml`/`CMakeLists.txt`를 직접 읽고 `rosdep`으로 해결 |
| 루프클로저가 전혀 안 잡힘 | PGO 노드가 실제로 실행 중인지, odometry 토픽을 구독하고 있는지 `ros2 node info`로 확인 |
| 저장된 맵을 열었더니 벽이 이중으로 보임 | 루프클로저 오탐 또는 실패 — [[00_RTAB-Map_매핑_원리|지상로봇 적용 - 지도제작 - 00]]에서 다룬 것과 같은 원인(그래프 최적화에 잘못된 제약이 섞임) |
| 재위치 노드가 `.pcd`를 못 읽음 | 저장 시점과 로드 시점의 경로/권한 확인, PGO 저장이 실제로 완료된 뒤 재위치를 시작했는지 확인 |

## 6. 다음 문서와의 연결

- 다음: [[04_재위치_레이어와_map_odom_보정|04. 재위치 레이어와 map→odom 보정]] — 이 문서에서 고른 재위치 노드가 실제로 어떻게 동작해야 하는지, FC 연동 시 주의점까지 다룬다.

## 7. 참고자료

- [liangheming/FASTLIO2_ROS2](https://github.com/liangheming/FASTLIO2_ROS2) — 채택한 통합 패키지(odometry+PGO+재위치)
- [Ericsii/FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2) — FAST-LIO2 단독 ROS2 포팅(Phase 1 대안)
- [deepglint/FAST_LIO_LOCALIZATION_HUMANOID](https://github.com/deepglint/FAST_LIO_LOCALIZATION_HUMANOID) — 대안 재위치 노드(Humble 브랜치)
- [Livox-SDK/livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2) — Livox LiDAR ROS2 드라이버
- [hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO) — 원 저장소(ROS1 메인, ROS2 브랜치는 비공식)
