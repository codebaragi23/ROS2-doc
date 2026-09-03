# 03. Pose Jump 제어와 안정화

> 지상로봇 적용 — Relocalization 심화 시리즈
> 이전 문서: [[02_RTAB-Map_핵심_파라미터_대응표|02. RTAB-Map 핵심 파라미터 대응표]]
> 이 문서는 업로드된 원문의 "3. relocalization 시 주의사항 및 안정화 방안"을 공식 근거로 보강한 버전이다.

## 1. 개요

Relocalization이 정확하게 일어나더라도, 그 순간 `map→odom` TF가 갑자기 크게 바뀌는 "Pose Jump" 현상 자체는 구조적으로 피할 수 없다. 이 문서는 왜 이런 점프가 생기는지, 그리고 이를 완화하는 공식적으로 확인된 방법들을 정리한다.

## 2. 핵심 개념: 왜 점프가 생기는가 ([REP105](https://www.ros.org/reps/rep-0105.html))

ROS 좌표계 표준 [REP105](https://www.ros.org/reps/rep-0105.html)는 `map` 프레임을 "불연속적으로 점프할 수 있는(discontinuous) 전역 기준", `odom` 프레임을 "항상 연속적인(continuous) 로컬 기준"으로 정의한다. RTAB-Map 개발자도 공식적으로 이렇게 설명한다.

> "RTAB-Map follows [REP105](https://www.ros.org/reps/rep-0105.html) for map frame, thus will produce discrete jumps when a loop closure or relocalization happens."

즉 **점프 자체는 버그가 아니라 [REP105](https://www.ros.org/reps/rep-0105.html) 표준을 따르는 정상 동작**이다. 문제는 이 점프가 로봇 제어(Nav2 controller, costmap)에 그대로 전달되면 급제동이나 경로 이탈을 유발한다는 점이다.

## 3. 이 프로젝트에서의 적용: 두 가지 공식 대응 방향

RTAB-Map 공식 답변은 이 문제에 대해 두 가지 대안을 제시한다.

### 방법 A: `RGBD/OptimizeFromGraphEnd: true`

```yaml
RGBD/OptimizeFromGraphEnd: true
```

기본값(`false`)에서는 relocalization이 일어날 때 **로봇의 현재 pose가 점프**한다. 이 값을 `true`로 바꾸면 반대로 **지도(map) 쪽이 점프**하고 로봇의 현재 pose는 부드럽게 유지된다.

- 장점: 로봇이 "순간이동"하는 것처럼 보이지 않아 controller/costmap이 급격한 반응을 하지 않는다.
- 트레이드오프: 대신 이미 그려진 지도의 오래된 부분이 재정렬되면서 살짝 움직일 수 있다 — 정적 지도를 유지해야 하는 상황(Nav2가 이미 `/map`을 캐싱해 쓰는 경우 등)에서는 이 부작용을 고려해야 한다.

### 방법 B: `robot_localization` 패키지로 필터링

공식 답변은 relocalization을 "GPS처럼 취급"할 것을 제안한다. 즉 RTAB-Map의 `map→odom` 결과를 EKF(`robot_localization`)의 한 입력 소스로 넣고, 휠 오도메트리/IMU와 함께 융합해 급격한 점프를 부드럽게 흡수시키는 방식이다. 01에서 다룬 `odom→base_footprint`를 만드는 EKF와 같은 계열의 기법을 `map` 레벨 보정에도 적용하는 셈이다.

## 4. 관련 파라미터

| 파라미터/설정 | 역할 | 트레이드오프 |
|---|---|---|
| `RGBD/OptimizeFromGraphEnd` | true 시 지도가 점프, false 시 로봇이 점프 | 지도 안정성 vs 로봇 pose 연속성 |
| Nav2 `speed_limit` (Costmap Filter) | relocalization 중 속도 제한 강화 | 원문에서 언급한 "relocalization 중 속도 제한" 실현 수단 |
| 가속도 기반 게이팅 (프로젝트 커스텀) | 모션 속도가 일정 수준 이하일 때만 relocalization 승인 | False Loop Closure(모션 블러로 인한 오매칭) 방지 |

## 5. 진단 관점

- relocalization 직후 로봇이 급정지하거나 튄다면: `RGBD/OptimizeFromGraphEnd` 설정을 먼저 확인한다. false(기본값) 상태라면 A 방법 적용을 검토한다.
- relocalization 직후 costmap의 장애물이 잘못된 위치에 남아있다면: 점프 직후 costmap을 클리어하는 로직(05의 `ClearEntireCostmap` recovery 액션)을 relocalization 이벤트와 연동할 수 있는지 검토한다.
- 흔들리는 상태에서 자주 relocalization이 발동해 로봇이 계속 덜컥거린다면: 4장에서 언급한 "가속도 기반 게이팅"처럼, 로봇이 정지/저속 상태일 때만 relocalization을 승인하는 조건을 추가하는 것이 원인 완화에 직접적이다.

## 6. 다음 문서와의 연결

- 다음: **04. 실전 진단 체크리스트** — 이 문서와 01, 02에서 다룬 개념들을 실제 현장에서 어떤 순서로 확인할지 체크리스트로 정리한다.

## 7. 참고자료

- [REP105](https://www.ros.org/reps/rep-0105.html) — ROS 좌표계 표준, `map` 프레임의 불연속성(discontinuity) 정의
- RTAB-Map 공식 Q&A(answers.ros.org) — `OptimizeFromGraphEnd` 대안과 `robot_localization` 필터링 제안 원문
- [Nav2 — Speed Filter](https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_filters/speed_filter/) — 속도 제한 필터 설정

> **문서 버전 주의**: `docs.nav2.org`는 Humble 버전 문서를 더 이상 호스팅하지 않아 위 링크는 Jazzy 기준이다. 개념과 대부분의 파라미터는 동일하지만, 플러그인 목록과 일부 기본값은 Humble과 다를 수 있으므로 실제 설정 시 `ros2 param list`로 대조한다.
