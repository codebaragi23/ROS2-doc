# B5. Controller — 경로 추종 (ROS1의 Local Planner)

> Nav2 내비게이션 (Part B) 시리즈
> 이전 문서: B4. Global Planner

## 0. 용어 정리 — Controller인가 Local Planner인가

**둘 다 같은 것을 가리키지만, ROS2 Nav2의 공식 용어는 "Controller"다.**

| | ROS1 (navigation stack) | ROS2 (Nav2) |
|---|---|---|
| 이름 | **Local Planner** | **Controller** |
| 서버/패키지 | `base_local_planner`, `dwa_local_planner` | `controller_server` |
| 대표 구현 | `base_local_planner`(Trajectory Rollout), DWA | DWB, MPPI, RPP 등 |

Nav2 공식 문서도 "Controllers, also known as **local planners** in ROS 1"이라고 명시한다. 즉 **`local planner`는 틀린 말이 아니라 ROS1 시절의 이름**이며, 지금도 커뮤니티에서 널리 통용된다. 다만 Nav2 설정 파일과 문서에서는 `controller_server`, `controller_plugins`로 나오므로, **설정을 만질 때는 "controller"로 찾아야 한다.**

> ROS1의 DWA를 계승한 것이 Nav2의 **DWB** 컨트롤러다 — 이름이 바뀐 이유 중 하나가 이 계보를 정리하기 위해서였다.

## 1. 개요

`controller_server`는 Global Planner가 만든 경로를 따라 실제 `cmd_vel`(속도 명령)을 만든다. Planner가 "어디로 갈지"를 정한다면, **Controller는 "지금 이 순간 바퀴를 얼마나 돌릴지"를 정한다.**

Nav2는 여러 controller를 **플러그인**으로 제공한다. 이 문서는 선택지 전체를 먼저 정리하고, 그중 이 프로젝트가 MPPI를 쓰는 이유를 3장에서 다룬다.

## 2. 핵심 개념

### 2.1 Controller 플러그인 전체 선택지

| 플러그인 | 방식 | 강점 | 유의점 |
|---|---|---|---|
| **DWB** (Dynamic Window Approach B) | 여러 속도 후보를 시뮬레이션해 critic으로 채점, **최고 점수 하나를 선택** | ROS1 DWA의 계승자라 자료가 많고 검증됨. 구조가 단순해 이해·디버깅이 쉬움 | 동적 장애물 대응이 MPPI보다 덜 매끄러움. 좁은 공간에서 후보가 모두 막히면 멈춤 |
| **MPPI** (Model Predictive Path Integral) | 제어 입력에 노이즈를 뿌려 수천 개 후보를 만들고, **비용에 따라 가중 평균** | 부드럽고 예측적인 주행. 동적 장애물 회피가 우수. critic 조합이 유연 | **연산 부하가 가장 큼**(수천 개 궤적을 매 주기 시뮬레이션) |
| **RPP** (Regulated Pure Pursuit) | 경로 위 전방의 목표점(lookahead)을 향해 조향, 곡률·장애물에 따라 속도 조절 | **경로를 정확히 따라간다.** 매우 가볍고 동작이 예측 가능 | 경로 자체를 벗어나 회피하지는 않음 — 운동학적으로 타당한 경로를 만드는 planner와 짝지어야 함 |
| **Rotation Shim** | 컨트롤러가 아니라 **래퍼(wrapper)**. 새 경로의 방향으로 **제자리 회전을 먼저** 시킨 뒤, 내부에 지정한 실제 컨트롤러에 넘김 | DWB·MPPI·RPP 등을 내부 플러그인으로 감쌀 수 있음. 경로 시작 방향과 로봇 방향이 크게 다를 때 유용 | 단독으로는 쓸 수 없음 |

> **버전 유의**: Humble 기준으로 위 네 가지가 기본 제공된다. 이후 배포판(Iron/Jazzy 등)에서 Graceful Controller, Vector Pursuit 등이 추가됐으므로, 다른 배포판을 쓴다면 해당 버전 문서를 확인해야 한다.

**고르는 기준**

| 상황 | 권장 |
|---|---|
| 계산 자원이 넉넉하고 동적 장애물이 많음 | **MPPI** |
| 자원이 빠듯하거나 단순·안정 우선 | **DWB** 또는 **RPP** |
| 정해진 경로를 정확히 따라가는 것이 최우선(순찰, 라인 추종 성격) | **RPP** |
| 경로 시작 시 제자리 회전이 필요 | **Rotation Shim** + 위 중 하나 |

### 2.2 MPPI의 동작 방식

이 프로젝트가 쓰는 MPPI를 자세히 본다.

```yaml
controller_plugins: ["FollowPath"]
FollowPath:
  plugin: "nav2_mppi_controller::MPPIController"
```

MPPI는 매 제어 주기마다 다음 과정을 반복한다.

1. **제어 입력에 노이즈를 뿌려 후보를 만든다.** 궤적을 직접 그리는 것이 아니라, 현재 제어열(속도 명령 시퀀스)에 무작위 노이즈를 더해 `batch_size`개의 서로 다른 제어 후보를 만든다.
2. **각 후보를 모델로 굴려본다.** 로봇 운동 모델로 각 후보를 `time_steps × model_dt`초 동안 시뮬레이션해 궤적을 얻는다.
3. **각 궤적을 critic들이 채점한다.** 여러 **critic(비용 평가 함수)**이 장애물 근접도, 경로 이탈, 목표 접근 등을 각각 평가하고, 최종 비용은 `Σ(critic별 비용 × cost_weight)`로 합산된다.
4. **점수에 따라 가중 평균한 제어 명령을 낸다.** — 여기가 MPPI의 핵심이다. **가장 좋은 궤적 하나를 고르는 것이 아니라**, 비용이 낮은 궤적일수록 큰 가중치를 주어 **모든 후보를 가중 평균한** 제어 명령을 출력한다.

- **예측 호라이즌**: `time_steps × model_dt`로 계산되는, 몇 초 앞을 내다볼지 정하는 값. 길수록 더 먼 미래까지 고려하지만 연산량이 늘어난다.
- **샘플링 폭(`vx_std`, `wz_std`)**: 1단계에서 더하는 노이즈의 크기다. 크면 다양한 회피 궤적을 탐색하지만 움직임이 거칠어지고, 작으면 부드럽지만 좁은 통로에서 빠져나갈 궤적을 못 찾을 수 있다.
- **온도(`temperature`)**: 4단계 가중 평균이 얼마나 **날카로운지**를 정한다. 온도가 **낮으면** 최저 비용 궤적에 가중치가 거의 몰려 "최선 하나만 고르는" 것에 가까워지고, **높으면** 여러 궤적을 골고루 섞어 부드럽지만 덜 공격적인 명령이 나온다.
- **`gamma`**: 제어 비용(속도를 크게 쓰는 것 자체에 매기는 페널티)의 가중치로, temperature와는 별개 파라미터다. 값이 크면 더 소극적으로 움직인다.

> **왜 "최저 비용 선택"이 아니라 "가중 평균"인가**: 최저 비용 궤적 하나만 고르면 매 주기마다 선택이 튀어 명령이 덜컥거린다. 가중 평균은 비슷하게 좋은 여러 후보를 부드럽게 섞어주므로 제어가 훨씬 안정적이다 — `temperature`라는 파라미터가 존재하는 이유 자체가 이 평균의 날카로움을 조절하기 위해서다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

### 3.1 왜 MPPI인가

2.1절의 선택 기준을 X3에 대입하면 이렇게 된다.

| 판단 항목 | X3의 경우 | 결론 |
|---|---|---|
| 동적 장애물(사람 등)이 있는가 | 실내 사무실 — **있음** | 예측적 회피가 강한 MPPI 유리 |
| 계산 자원이 충분한가 | Jetson — 여유롭지 않음 | MPPI가 부담이지만 현재는 감당 중 |
| 정해진 경로를 정확히 따라가야 하는가 | 아니오(회피가 더 중요) | RPP는 부적합 |
| 선택 | **MPPI** | 부드러운 주행과 동적 회피를 우선 |

**바꿔 쓴다면**: Jetson의 CPU가 빠듯해지면 **DWB**가 현실적인 대안이다(더 가볍고 자료도 많다). 반대로 순찰처럼 정해진 경로를 정확히 따라가는 용도라면 **RPP**가 더 적합하다. 세 컨트롤러 모두 X3의 운동학에 문제없이 맞는다.

> **참고**: 이 프로젝트는 `Rotation Shim`을 쓰지 않는다. MPPI는 자체적으로 회전을 포함한 궤적을 생성하므로 별도 선회 래퍼가 없어도 대체로 동작한다. 다만 "경로 시작 시 엉뚱한 방향으로 먼저 움직인다"는 증상이 반복된다면 Rotation Shim 도입을 검토할 수 있다.

### 3.2 파라미터 설정

- 예측 호라이즌: `time_steps: 56`, `model_dt: 0.05` → 총 `56 × 0.05 = 2.8초`를 앞서 내다보고 궤적을 평가한다. 샘플링은 `batch_size: 2000`개의 후보 궤적을 한 주기에 평가한다.
- `motion_model: "DiffDrive"` — 현재 X3는 메카넘 휠 기반이지만, Nav2 controller는 DiffDrive(횡이동 없음) 모델을 사용한다. 과거 Omni(전방위) 모델을 시도했을 때 `base_node_X3`의 odom과 불일치해 localization jump가 발생한 이력이 있어, 실제 메카넘 odom이 정확히 나오기 전까지는 DiffDrive를 유지하는 것으로 결정되어 있다.
- `vy_max: 0.0` — 위와 같은 이유로 횡이동 속도를 0으로 제한한다.
- `vx_max: 0.34`, `wz_max: 1.0` — 전진/회전 최대 속도. 실내 사무실 환경 안전성을 고려한 값이다.

### 3.3 MPPI Critics (궤적 평가 함수)

| Critic | 역할 | 튜닝 관점 |
|---|---|---|
| `ConstraintCritic` | 속도/가속/운동모델 제약 위반 궤적 배제 | `cost_weight`↑ = 제약 위반에 더 엄격 |
| `ObstaclesCritic` | 장애물 회피 비용 | `repulsion_weight`↑ = 장애물을 더 멀리 피함(좁은 통로에서 모든 궤적이 비싸져 멈출 수 있음), `critical_weight`는 실제 충돌 방지 직결이라 낮출 때 주의 |
| `GoalCritic` / `GoalAngleCritic` | 목표 위치/방향으로 수렴 | 정밀 도착에서 중요, waypoint에서는 너무 강하면 불필요한 회전 유발 |
| `PathAlignCritic` | 전역 경로와 로컬 궤적 정렬 | 너무 높으면 동적 장애물 회피가 둔해질 수 있음 |
| `PathFollowCritic` | 경로를 따라 전진하도록 유도 | 너무 높으면 장애물 회피보다 경로 진행을 과하게 선호 |
| `PathAngleCritic` | 경로 방향과 로봇 진행 방향 정렬 | 너무 높으면 직선 구간에서도 속도 저하 |
| `PreferForwardCritic` | 전진 주행 선호 | DiffDrive에서 후진보다 전진 우선 (Omni에서는 불필요) |

## 4. 관련 파라미터

| 파라미터 | 값 | 튜닝 관점 |
|---|---|---|
| `controller_frequency` | 20.0 Hz | `model_dt`(0.05)와 맞아야 함. 낮으면 반응 둔화, 높으면 CPU 부하↑ |
| `vx_max` / `vx_min` | 0.34 / -0.2 | 전진/후진 최대 속도 |
| `wz_max` | 1.0 | 회전 최대 속도. 낮추면 좁은 복도 회피 궤적 부족, 높이면 급회전·localization 흔들림 위험 |
| `ax_max`/`ax_min`/`az_max` | — | 가속 한계. 급출발/급정지가 있으면 낮춤 |
| `vx_std`/`vy_std`/`wz_std` | — | 후보 궤적 탐색 폭. 키우면 다양한 회피 궤적 탐색, 너무 크면 움직임이 거칠어짐 |
| `prune_distance` | — | 지나간 경로를 잘라내는 거리 |
| `transform_tolerance` | — | TF 시간 차이 허용 범위 |

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| 좁은 통로에서 멈춤 | local/global costmap이 실제로 열려 있는지 → global planner 경로 생성 여부 → MPPI 후보가 장애물 비용으로 모두 막히는지 → `ObstaclesCritic.repulsion_weight`/`cost_scaling_factor`/`collision_margin_distance` → local/global `inflation_radius`(B3) → `progress_checker.movement_time_allowance`(B6) |
| 장애물에 너무 붙음 | robot radius(B3)가 실제보다 작지 않은지 → footprint/센서 위치 정합 → local inflation radius → `repulsion_weight` |
| 움직임이 거침 | MPPI 탐색 노이즈(`*_std`), 가속 한계, velocity smoother(B7) |
| 너무 느림 | `vx_max`/`wz_max`, `PathFollowCritic`, velocity smoother 속도 한계 |

## 6. 다음 문서와의 연결

- 다음: **B6. Behavior Tree와 표준 Recovery** — MPPI가 멈춘 상황(progress_checker 실패)을 Nav2가 어떻게 회복 시도하는지 다룬다.
- costmap(B3)의 `robot_radius`/`inflation_radius`와 이 문서의 critic weight는 항상 짝을 지어 튜닝해야 한다.

## 7. 참고자료

- [Nav2 — Controller Server](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/controller_server/) — controller 플러그인 목록과 공통 설정
- [Nav2 — Navigation Servers](https://docs.nav2.org/jazzy/getting_started/navigation_concepts/navigation_servers/) — "Controllers, also known as local planners in ROS 1" (0장 용어 근거)
- [Nav2 — Rotation Shim Controller](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/controller_plugins/configuring_rotation_shim_controller/) — 선회 래퍼 사용법
- [Nav2 — MPPI Controller](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/controller_plugins/mppi_controller/configuring_mppic/) — Critic 목록과 파라미터 전체
- [Nav2 — DWB Controller](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/controller_plugins/dwb_controller/) / [Regulated Pure Pursuit](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/controller_plugins/configuring_regulated_pp/) — 2.1절의 나머지 후보
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 MPPI 설정값

> **문서 버전 주의**: `docs.nav2.org`는 Humble 버전 문서를 더 이상 호스팅하지 않아 위 링크는 Jazzy 기준이다. 개념과 대부분의 파라미터는 동일하지만, 플러그인 목록과 일부 기본값은 Humble과 다를 수 있으므로 실제 설정 시 `ros2 param list`로 대조한다.
