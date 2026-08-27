# B5. Local Controller (MPPI)

> Nav2 내비게이션 (Part B) 시리즈
> 이전 문서: B4. Global Planner

## 1. 개요

`controller_server`는 Global Planner가 만든 경로를 따라 실제 `cmd_vel`(속도 명령)을 만든다. 이 프로젝트는 MPPI(Model Predictive Path Integral) 컨트롤러를 사용한다.

## 2. 핵심 개념: MPPI의 동작 방식

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

- 예측 호라이즌: `time_steps: 56`, `model_dt: 0.05` → 총 `56 × 0.05 = 2.8초`를 앞서 내다보고 궤적을 평가한다. 샘플링은 `batch_size: 2000`개의 후보 궤적을 한 주기에 평가한다.
- `motion_model: "DiffDrive"` — 현재 X3는 메카넘 휠 기반이지만, Nav2 controller는 DiffDrive(횡이동 없음) 모델을 사용한다. 과거 Omni(전방위) 모델을 시도했을 때 `base_node_X3`의 odom과 불일치해 localization jump가 발생한 이력이 있어, 실제 메카넘 odom이 정확히 나오기 전까지는 DiffDrive를 유지하는 것으로 결정되어 있다.
- `vy_max: 0.0` — 위와 같은 이유로 횡이동 속도를 0으로 제한한다.
- `vx_max: 0.34`, `wz_max: 1.0` — 전진/회전 최대 속도. 실내 사무실 환경 안전성을 고려한 값이다.

### MPPI Critics (궤적 평가 함수)

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

- Nav2 공식 문서 — MPPI Controller 설정 가이드, Critic 목록과 파라미터
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 MPPI 설정값
