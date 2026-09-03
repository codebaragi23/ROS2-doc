# 06. Waypoint Follower & Velocity Smoother

> 지상로봇 적용 — Nav2 내비게이션 시리즈
> 이전 문서: [[05_Behavior_Tree와_표준_Recovery|05. Behavior Tree와 표준 Recovery]]

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 적용 |
| 예상 선행 지식 | [[05_Behavior_Tree와_표준_Recovery\|Nav2 내비게이션 05]] |
| 학습 목표 | 여러 목표를 순차 이동시키는 방법을 안다 / velocity smoother의 역할과 controller 속도 한계와의 관계를 설명할 수 있다 / 덜컥거림·반응 지연을 어느 파라미터로 조정할지 안다 |
| 기준 환경 | Yahboom ROSMASTER X3, Nav2 |

## 1. 개요

이 문서는 Nav2 파이프라인의 마지막 두 구성요소를 다룬다. `waypoint_follower`는 여러 목표점을 순차 이동할 때, `velocity_smoother`는 controller가 만든 속도 명령을 로봇에 전달하기 직전 마지막으로 다듬는 역할을 한다.

## 2. 핵심 개념

### 2.1 Waypoint Follower

단일 목표가 아니라 "A → B → C" 순서로 이동해야 할 때 사용한다. 각 waypoint에 도착할 때마다 정해진 시간 대기하거나, 실패 시 계속 진행할지 멈출지를 결정한다.

### 2.2 Velocity Smoother

```text
controller_server → cmd_vel_nav → velocity_smoother → cmd_vel → 로봇 구동부
```

MPPI(04)가 만든 속도 명령이 프레임마다 급격히 바뀌면 로봇이 덜컥거릴 수 있다. Velocity Smoother는 이 변화를 완화해 최종 `cmd_vel`을 만든다.

## 3. 이 프로젝트에서의 적용 (Yahboom X3)

- `stop_on_failure: false` — 한 waypoint 실패 시 전체를 멈추지 않고 계속 진행한다. 단, 위험한 임무(정밀 도킹 등)에서는 이 설정을 다시 검토해야 한다.
- `feedback: OPEN_LOOP` — odom 피드백 없이 명령 기반으로만 smoothing한다. MPPI 자체가 이미 가속 제한을 갖고 있어 현재는 open loop로 충분하다고 판단된 상태다.
- `max_velocity`, `max_accel`, `max_decel`은 반드시 04의 MPPI `vx_max`/`vy_max`/`wz_max`와 일치해야 한다 — 두 값이 다르면 "왜 이렇게 느리지/빠르지"를 진단할 때 원인 추적이 어려워진다.

## 4. 관련 파라미터

| 구성요소 | 파라미터 | 역할 |
|---|---|---|
| `waypoint_follower` | `loop_rate` | 실행 주기 |
| `waypoint_follower` | `stop_on_failure` | 한 waypoint 실패 시 전체 중단 여부 (현재 false) |
| `waypoint_follower` | `waypoint_pause_duration` | 도착 후 대기 시간 |
| `velocity_smoother` | `smoothing_frequency` | controller_frequency(04, 20.0Hz)와 맞춤 |
| `velocity_smoother` | `feedback` | OPEN_LOOP / CLOSED_LOOP |
| `velocity_smoother` | `max_velocity`/`min_velocity` | MPPI 속도 한계와 반드시 일치 |
| `velocity_smoother` | `max_accel`/`max_decel` | 최종 가속/감속 제한 |
| `velocity_smoother` | `velocity_timeout` | 일정 시간 새 명령이 없으면 속도를 0으로 |

## 5. 진단 관점

| 증상 | 확인 순서 |
|---|---|
| waypoint에서 너무 오래 머묾 | `waypoint_pause_duration` 확인 |
| 실패 waypoint를 무시하고 넘어감 | `stop_on_failure` 설정이 임무 성격에 맞는지 재검토 |
| 로봇이 덜컥거림 | smoother의 `max_accel`/`max_decel`을 낮춤 |
| controller가 회피 명령을 내는데 반응이 느림 | smoother `max_accel`/`max_decel`을 높임 |
| 속도 한계가 이상하게 다르게 느껴짐 | MPPI 속도 한계(04)와 smoother 속도 한계가 일치하는지 대조 |

## 6. 다음 문서와의 연결

- 다음: **[[07_Relocalization이_일어나는_위치|지상로봇 적용 - Nav2 내비게이션 - 07. Relocalization은 Nav2 파이프라인의 어디에 위치하는가]]** — Nav2 내비게이션에서 다룬 Nav2 파이프라인 전체가 정상 동작하려면 그 바탕의 `map→odom`이 항상 맞아야 한다는 전제를 정리하며 Relocalization 심화로 넘어간다.

## 7. 참고자료

- [Nav2 — Waypoint Follower 설정](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/waypoint_follower/) — 플러그인(wait/photo/input at waypoint) 포함
- [Nav2 — Velocity Smoother 설정](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/configuring_velocity_smoother/) — 파라미터 전체
- `rtabmap_nav_params_tuning_guide.md` (프로젝트 내부 자료) — 이 프로젝트의 실제 설정값

> **문서 버전 주의**: `docs.nav2.org`는 Humble 버전 문서를 더 이상 호스팅하지 않아 위 링크는 Jazzy 기준이다. 개념과 대부분의 파라미터는 동일하지만, 플러그인 목록과 일부 기본값은 Humble과 다를 수 있으므로 실제 설정 시 `ros2 param list`로 대조한다.
