# B3. Costmap

> Nav2 내비게이션 (Part B) 시리즈
> 이전 문서: B2. 좌표계와 로컬라이제이션 기반

## 1. 개요

Costmap은 "여기는 지나가도 되는지, 얼마나 위험한지"를 격자(cell) 단위 비용으로 표현한 지도다. Nav2는 Global costmap(전역 경로 계획용)과 Local costmap(근거리 회피용) 두 개를 별도로 운영한다.

## 2. 핵심 개념: Global vs Local

| 구분 | Global Costmap | Local Costmap |
|---|---|---|
| 기준 좌표계 | `map` | `odom` |
| 크기 | 지도 전체 | 로봇 주변 일정 크기(rolling window) |
| 목적 | 전역 경로가 지나갈 수 있는지 판단 | 실시간 장애물 회피 |
| 갱신 방식 | 정적 지도 + 저빈도 장애물 반영 | 매 주기 센서 데이터로 즉시 갱신 |

두 costmap 모두 여러 **레이어(layer)**를 겹쳐서 최종 비용을 만든다.

- **static_layer**: map_server가 제공하는 정적 지도(벽, 고정 구조물)를 반영. Global costmap에서만 사용.
- **obstacle_layer**: 실시간 센서(LiDAR, depth camera)로 감지한 장애물을 반영. `clearing`(빈 공간 지우기)과 `marking`(장애물 찍기) 두 동작을 한다.
- **inflation_layer**: 장애물 주변에 "가까이 갈수록 비용이 커지는" 경사를 만들어, 로봇이 장애물에 바짝 붙어 계획하지 않도록 유도.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

Local costmap의 obstacle_layer는 두 개의 관측원(observation source)을 함께 쓴다.

- **`/scan`**: RPLiDAR C1의 2D 스캔
- **`/camera_scan`**: RealSense D435i의 depth 이미지를 LaserScan 형태로 변환한 것

`/camera_scan`을 추가한 이유는 LiDAR가 놓치는 낮은 장애물(의자 다리, 문턱 등)을 depth 카메라로 보완하기 위함이다. 다만 depth 기반 스캔은 바닥을 장애물로 오탐하거나 최소 감지 거리 한계가 있어, 실제 값이 맞는지 RViz로 직접 확인이 필요하다.

현재 로봇 크기 설정:

| 설정 | Local | Global |
|---|---|---|
| `robot_radius` | 0.2 m | 0.22 m |
| `inflation_radius` | 0.2 m | 0.35 m |
| `cost_scaling_factor` | 8.0 | 5.0 |

Global의 robot_radius가 Local보다 약간 크게 잡혀 있는 것은, 전역 경로 계획 단계에서 더 여유 있게(보수적으로) 통로를 고르게 하기 위함으로 보인다.

## 4. 관련 파라미터

| 파라미터 | 역할 | 튜닝 관점 |
|---|---|---|
| `robot_radius` | 로봇 반경으로 취급할 값 | **새 바디로 교체 시 가장 먼저 맞춰야 하는 값** — 틀리면 planner가 통과 가능하다고 판단해도 실제로는 부딪히거나, 반대로 지나갈 수 있는 통로를 못 지나간다고 판단 |
| `inflation_radius` | 장애물 주변 비용 경사 폭 | 로봇이 장애물에 너무 붙으면 키움 |
| `cost_scaling_factor` | 비용 감소 곡선의 급격함 | 높을수록 장애물 근처에서만 비용이 급하게 줄어듦 |
| `raytrace_range` (obstacle_layer) | clearing ray 적용 거리 | 길게 잡으면 먼 공간 clearing에 유리하지만 센서 노이즈 영향도 커짐 |
| `map_subscribe_transient_local` | static_layer | `/map`을 늦게 구독해도 마지막 지도를 받을 수 있게 함(True 권장) |

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| 전역 경로가 아예 안 나옴 | `allow_unknown` 설정, global `robot_radius`/`inflation_radius`, map 품질(A3 참고) |
| 좁은 통로에서 진입을 거부 | local `inflation_radius`, `robot_radius`가 실제보다 과도하게 크지 않은지 |
| 장애물에 너무 붙어서 이동 | `robot_radius`가 실제보다 작지 않은지, footprint가 센서 위치와 맞는지 |
| 낮은 장애물(의자다리 등)에 부딪힘 | `/camera_scan`이 실제로 costmap에 반영되고 있는지 RViz로 확인 |

## 6. 다음 문서와의 연결

- 다음: **B4. Global Planner** — Global costmap 위에서 실제로 경로를 계산하는 SmacPlanner2D를 다룬다.
- Costmap의 `robot_radius`/`inflation_radius`는 B5(MPPI Controller)의 ObstaclesCritic과 항상 함께 봐야 하는 값이다 — costmap이 "안전하다"고 판단해도 controller의 critic이 다르게 판단하면 로봇이 멈추거나 회피가 과도해질 수 있다.

## 7. 참고자료

- Nav2 공식 문서([docs.nav2.org](https://docs.nav2.org/)) — Costmap 2D 레이어 구조, obstacle/inflation layer 파라미터
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 costmap 설정값과 튜닝 우선순위
