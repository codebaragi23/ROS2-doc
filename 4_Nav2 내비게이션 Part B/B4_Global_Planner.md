# B4. Global Planner

> Nav2 내비게이션 (Part B) 시리즈
> 이전 문서: B3. Costmap

## 1. 개요

`planner_server`는 Global costmap 위에서 "지금 위치에서 목표까지 어떻게 가야 하는지"를 계산하는 서버다. 이 프로젝트는 `SmacPlanner2D`를 사용한다.

## 2. 핵심 개념

```yaml
planner_plugins: ["GridBased"]
GridBased:
  plugin: "nav2_smac_planner/SmacPlanner2D"
```

SmacPlanner2D는 격자 기반(grid-based) 경로 탐색 알고리즘으로, Global costmap의 각 셀 비용을 고려해 시작점에서 목표점까지의 경로를 찾는다.

### 격자 위에서 경로를 찾는다는 것 (A* 탐색)

Nav2의 격자 기반 플래너는 대부분 **A\*(A-star)** 계열 탐색을 쓴다. 파라미터를 이해하려면 이 탐색이 어떻게 도는지 알아야 한다.

1. **시작 셀에서 출발**해, 인접한 셀들을 "가볼 후보" 목록에 넣는다.
2. 후보 중에서 **점수가 가장 좋은 셀 하나를 골라 확장**한다(그 셀의 이웃들을 다시 후보에 넣는다). 이 한 번의 확장이 **1 iteration**이다.
3. 목표 셀에 도달할 때까지 2번을 반복하고, 도달하면 **왔던 길을 거슬러 올라가** 경로를 만든다.

여기서 "점수"는 두 값의 합이다.

| 항목 | 의미 |
|---|---|
| **지금까지 온 비용 (g)** | 시작점부터 이 셀까지 실제로 이동한 비용. **여기에 costmap의 셀 비용이 더해진다** — 그래서 inflation이 높은 셀을 지나는 경로는 점수가 나빠져 자연히 회피된다(B3와 직결). |
| **남은 거리 추정 (h, 휴리스틱)** | 이 셀에서 목표까지 대략 얼마나 남았는지의 추정치(보통 직선거리). 이 값이 있어서 목표 반대 방향을 헛되이 탐색하지 않는다. |

**이 구조를 알면 4장의 파라미터가 자명해진다**

- `max_iterations`: 위 2번 확장을 최대 몇 번까지 할지의 상한. 이 횟수를 넘으면 경로를 못 찾은 것으로 처리한다 — 넓은 지도에서 너무 작으면 도달 가능한 목표인데도 실패한다.
- `max_planning_time`: 같은 것을 시간으로 제한한 값.
- `tolerance`: 목표 셀에 정확히 도달하지 못해도, 이 거리 안까지만 가면 성공으로 인정한다.
- `allow_unknown`: 아직 관측 안 된(unknown) 셀을 지나갈 수 있는 후보로 볼지 여부.

> **참고 — 다른 Smac 플래너들**: `SmacPlanner2D`는 로봇을 **원형으로 근사**하고 어느 방향으로든 회전할 수 있다고 가정한다. 차량처럼 제자리 회전이 불가능한 로봇은 `SmacPlannerHybrid`(Hybrid-A*), 임의의 운동학 제약이 있으면 `SmacPlannerLattice`를 쓴다. **경로 계획 단계에서는 운동학을 단순하게 보고, 실제 운동학 제약은 B5의 컨트롤러가 맡는다**는 역할 분담을 기억해두면 좋다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

실내 사무실 환경(좁은 통로, 좌표계 흔들림 가능성)을 고려해 다음과 같은 방향으로 설정되어 있다.

- `tolerance: 0.5` — 목표 cell까지 정확히 도달하는 경로를 요구하지 않고, 경로 생성 자체의 안정성을 우선했다. 값이 너무 작으면 장애물 근처 목표나 localization이 흔들리는 환경에서 `plan=0`(경로 실패)이 잦아질 수 있다.
- `allow_unknown: true` — 미탐색 영역도 경로 후보로 허용한다. 운영상 unknown 영역으로 들어가면 안 되는 경우 `false`로 바꿀 수 있지만, map 가장자리의 미세한 unknown 틈 때문에 경로가 아예 안 나올 위험이 있어 static layer 품질과 함께 검토해야 한다.

## 4. 관련 파라미터

| 파라미터 | 역할 | 튜닝 관점 |
|---|---|---|
| `expected_planner_frequency` | planner가 기대되는 재계획 주기 | 실제 계산 속도보다 높게 잡으면 경고/부하만 늘어남 |
| `tolerance` | 목표 지점 주변 허용 오차 | 작을수록 엄격, 크면 안정성 우선 (현재 0.5) |
| `allow_unknown` | unknown 셀을 경로 후보로 허용할지 | true=탐색 유리, false=안전 운영 우선 |
| `max_iterations` | 경로 탐색 최대 반복 | 높으면 복잡한 맵에서도 찾을 가능성↑, 실패 시 계산시간↑ |
| `max_planning_time` | 1회 planning 최대 허용 시간 | 좁고 복잡한 맵은 너무 짧으면 실패, 너무 길면 응답성↓ |
| `use_final_approach_orientation` | 경로 마지막 구간 목표 방향 반영 여부 | true=방향 정렬 강화 |
| `smoother.w_smooth` / `w_data` | 경로를 부드럽게 vs 원래 경로 유지 | 강한 smoothing은 costmap inflation과 함께 확인 필요(장애물 쪽으로 말릴 수 있음) |

## 5. 진단 관점

경로가 아예 안 나올 때 확인 순서:

1. Global costmap에서 목표 지점 주변에 실제로 열린 공간이 있는지 RViz로 확인
2. `allow_unknown` 설정과 map 경계/틈 상태 확인
3. `tolerance`가 현재 localization 흔들림 수준에 비해 너무 작지 않은지 확인
4. `max_planning_time`이 실제 계산 시간보다 짧지 않은지 로그 확인

경로가 벽에 너무 붙거나 꺾임이 심하면 `smoother` 설정과 global `inflation_radius`(B3)를 함께 조정한다.

## 6. 다음 문서와의 연결

- 다음: **B5. Local Controller (MPPI)** — Global Planner가 만든 경로를 실제 속도 명령으로 바꾸는 단계.
- `tolerance`가 localization 흔들림과 관련된다는 점은 Part C(Relocalization 심화)에서 "목표 도착 판정이 안 될 때"의 원인 중 하나로 다시 등장한다.

## 7. 참고자료

- Nav2 공식 문서 — SmacPlanner2D 설정 가이드
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 planner 설정값과 튜닝 우선순위
