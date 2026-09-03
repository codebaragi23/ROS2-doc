# ros2-nav-yahboom (프로젝트 진단 노트 — 색인)

> **이 문서는 색인(index)이다.** 실제 진단 노트 원본은 이 학습 vault 바깥에서 관리되는 프로젝트 작업 기록이며, 여기에는 그 내용이 복제되어 있지 않다.
> 이 vault의 여러 문서가 `[[ros2-nav-yahboom]]`을 근거로 인용하고 있으므로, **무엇을 근거로 인용하고 있는지**를 이 색인에서 확인할 수 있게 정리했다.

## 1. 이것은 무엇인가

Yahboom ROSMASTER X3(Jetson + RealSense D435i + RPLiDAR C1) 프로젝트를 진행하며 쌓인 **실측·진단 이력의 원본 기록**이다. 이 vault의 문서들이 "실제 로봇 관점", "이 프로젝트에서의 적용" 섹션에서 드는 사례들의 출처가 대부분 이 노트다.

이 vault(교육용 문서)와 성격이 다르다.

| | 이 vault | 진단 노트(ros2-nav-yahboom) |
|---|---|---|
| 목적 | 개념을 순서대로 학습 | 무슨 일이 있었는지 시간순 기록 |
| 내용 | 일반 이론 + 적용 사례 | 실측값, 오류 로그, 시도와 결과 |
| 갱신 | 개념이 정리되면 | 새 진단이 나올 때마다 |

## 2. 이 vault가 이 노트를 인용하는 항목들

아래는 각 문서가 이 노트의 어떤 기록을 근거로 삼고 있는지 정리한 것이다. 원본 노트를 찾아볼 때의 목차 역할을 한다.

### 센서·성능 관련

| 인용 내용 | 인용하는 문서 |
|---|---|
| RealSense 드라이버 CPU 47% 점유 → 1280x720x30에서 640x480x15로 하향 해결 | [[01_매핑_실행_파이프라인과_핵심_파라미터\|지도제작 01]], [[06_Composable_Node와_Executor_구조\|응용 06]], [[02_센서_연동_RealSense_D435i\|응용 02]] |
| LiDAR 후방 사각지대 진단 → 파라미터 해제 + 마운트 높이 조정으로 360도 확보 | [[03_센서_연동_LiDAR_C1\|응용 03]], [[04_실전_진단_체크리스트\|Relocalization 심화 04]] |
| Nav2 패널이 `unknown` → `navigation_rtabmap_launch.py`에서 `rtabmap_nav_launch`가 주석 처리되어 있었음 | [[00_Nav2_아키텍처_개요\|Nav2 내비게이션 00]], [[05_Launch_시스템_디버깅\|응용 05]] |
| Jetson 하드웨어 자원 제약 일반 | [[05_Nav2_표준_relocalization과의_차이\|Relocalization 심화 05]] |

### ORB-SLAM3 (SLAM 백엔드 심화 01 시리즈)

| 인용 내용 | 인용하는 문서 |
|---|---|
| "active-map IMU reset" 반복 관찰 원본 로그 | [[01-1_ORB-SLAM3_시스템_개요\|01-1]], [[01-3_Multi-Map_System_Atlas와_재추적_병합\|01-3]] |
| `IMU.fastInit` 진단, 게이팅 격리 테스트, stereo-inertial 대비 실측 | [[01-2_Visual-Inertial_초기화_알고리즘\|01-2]], [[01-5_이_프로젝트의_VIO_이슈_재해석\|01-5]] |
| `FAST/MinThreshold` 등 튜닝 이력(85회 → 0회) | [[02-7_이_프로젝트에서의_실측_비교와_튜닝_계획\|02-7(OpenVINS 시리즈)]] |
| 재방문 시 병합 재현 여부 실험 | [[01-4_Loop_Closing과_Place_Recognition\|01-4]] |

### OpenVINS / VINS-Fusion (SLAM 백엔드 심화 02·03 시리즈)

| 인용 내용 | 인용하는 문서 |
|---|---|
| OpenVINS 통합 및 파라미터 사용 이력 | [[02-2_OpenVINS의_핵심_설계\|02-2]] |
| 2026-08-21 OpenVINS 코드 분석 + 3개 bag 실측(`fridge_proximity_*`) | [[02-7_이_프로젝트에서의_실측_비교와_튜닝_계획\|02-7]], [[04_ORB-SLAM3_vs_OpenVINS_vs_VINS-Fusion_종합_비교\|SLAM 백엔드 심화 04(종합 비교)]] |
| ZUPT 오발동 — 최우선 검증 대기 항목으로 기록됨 | [[02-8_이_프로젝트의_ZUPT_오발동_가설과_검증_계획\|02-8]] |
| "IR 스테레오+IMU 백엔드 검토"가 다음 단계 후보로 언급됨 | [[03-1_VINS-Fusion_개요와_최적화_기반_알고리즘_원리\|03-1]], [[03-5_이_프로젝트에_적용한다면\|03-5]] |

## 3. 이 vault 바깥에 실존하는 관련 자료

이 노트와 별개로, 아래 두 문서는 프로젝트 워크스페이스에 실제로 존재하며 이 vault의 여러 문서가 근거로 인용한다.

| 파일 | 이 vault에서 인용하는 곳 | 내용 |
|---|---|---|
| `docs/guides/rtabmap_nav_params_tuning_guide.md` | [[00_Nav2_아키텍처_개요\|Nav2 00]], [[02_Costmap\|Nav2 02]], [[03_Global_Planner\|Nav2 03]], [[04_Controller_경로_추종\|Nav2 04]], [[06_Waypoint_Follower_Velocity_Smoother\|Nav2 06]], [[04_실전_진단_체크리스트\|Relocalization 심화 04]] | Nav2/RTAB-Map 실제 설정값과 튜닝 관점 |
| `docs/guides/relocalization_strategy_guide.md` | [[00_개요와_학술적_정의\|Relocalization 심화 00~05]], [[01-1_이_프로젝트의_트리거_구현\|01-1]] | relocalization 트리거 5분류의 원본 |

경로는 프로젝트 워크스페이스(`amr_nav_lab_ws/`) 기준이다.

## 4. 사용 방법

- 이 vault의 문서에서 `[[ros2-nav-yahboom]]` 링크를 만났다면, **"이 주장의 근거는 실측 기록에 있다"는 표시**로 이해하면 된다.
- 구체적인 수치나 로그를 직접 확인해야 한다면 2장의 표에서 해당 항목을 찾아 원본 노트를 참조한다.
- 원본 노트를 이 vault에 통합하고 싶다면, **교육 문서와 섞지 말고 별도 폴더로 두는 것**을 권한다 — 이 vault는 "일반 개념 → 적용 사례" 분리를 원칙으로 삼고 있고, 진단 노트는 성격상 후자에 속하기 때문이다.
