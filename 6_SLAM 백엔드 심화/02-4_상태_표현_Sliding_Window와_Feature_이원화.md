# 02-4. 상태 표현 — Sliding Window와 MSCKF/SLAM Feature 이원화

> SLAM 백엔드 심화 시리즈
> 이전 문서: [[02-3_초기화_Static_vs_Dynamic|02-3. 초기화: Static vs Dynamic]]

## 1. 개요

02-1에서 "특징점을 상태에 넣지 않는다"는 MSCKF의 원칙을 배웠다. 그런데 실제로는 OpenVINS가 **일부 특징점만 예외적으로 상태에 포함**시킨다. 이 문서는 그 이원화 구조와, 이것이 01-1(ORB-SLAM3의 "모든 map point를 동등하게 취급")과 어떻게 다른지 다룬다.

## 2. 핵심 개념: MSCKF Feature vs SLAM Feature

| 구분 | MSCKF Feature | SLAM Feature |
|---|---|---|
| 특징 | 짧게 보이고 사라지는 특징점 | 오래 추적되는 특징점 |
| 상태 벡터 포함 여부 | 포함 안 됨 | **포함됨**(02-2의 표현 방식 중 하나로) |
| 처리 시점 | Marginalize(윈도우에서 빠질 때) 직전에 한 번만 제약으로 사용 | 매 업데이트마다 지속적으로 상태 개선에 기여 |
| 장점 | 계산 비용이 낮음(02-1의 null-space projection) | 오래 추적되므로 정보량이 많음 — 정밀도 향상 |

이 이원화는 02-1에서 배운 "순수 MSCKF"의 한계(오래 추적되는 특징점의 정보를 충분히 활용하지 못함)를 보완하는 설계다.

## 3. RTAB-Map 통합 파라미터 (이 프로젝트 기준)

```yaml
OdomOpenVINS/MaxClones:        11   # 슬라이딩 윈도우 크기(과거 pose clone 개수)
OdomOpenVINS/MaxSLAM:          50   # SLAM feature 최대 개수
OdomOpenVINS/MaxSLAMInUpdate:  25   # 한 번의 업데이트에서 사용할 SLAM feature 최대 개수
OdomOpenVINS/MaxMSCKFInUpdate: 50   # 한 번의 업데이트에서 사용할 MSCKF feature 최대 개수
OdomOpenVINS/FeatRepMSCKF:      0   # MSCKF feature 표현 방식(02-2의 5가지 중 선택)
OdomOpenVINS/FeatRepSLAM:       4   # SLAM feature 표현 방식
```

`MaxClones=11`이 이 문서의 핵심이다 — **최근 11개의 pose만 슬라이딩 윈도우에 유지**하고, 그보다 오래된 pose는 marginalize(제거)된다. 01-1(ORB-SLAM3)의 Atlas가 장면 전체를 계속 누적해서 키프레임과 map point가 계속 늘어나는 것과 정반대로, OpenVINS는 **연산량이 장면 크기와 무관하게 고정**된다.

## 4. ORB-SLAM3와의 대비: 왜 CPU가 구조적으로 낮은가

01-1에서 ORB-SLAM3는 Atlas 전체에 걸쳐 지도가 계속 커진다고 배웠다. 반면 OpenVINS는:

- 슬라이딩 윈도우 크기(`MaxClones`)가 고정되어 있어, 아무리 오래 돌아도 필터 상태 벡터의 크기가 어느 수준 이상으로 커지지 않는다.
- SLAM feature도 `MaxSLAM`으로 상한이 정해져 있다.
- 이것이 02-7에서 다룰 실측 비교(ORB-SLAM3 CPU 96% vs OpenVINS 38%)의 **구조적 원인** 중 하나다 — 단순히 구현이 가볍기 때문이 아니라, 알고리즘 설계 자체가 "계산량을 고정 상한으로 묶어두는" 철학이기 때문이다.

## 5. 이 프로젝트에서의 적용 (Yahboom X3)

- `MaxClones=11`, `MaxSLAM=50` 모두 **RTAB-Map 기본값 그대로**이며(02-7에서 확인), 이 프로젝트 환경(냉장고 근접 저텍스처 장면 등)에 맞춰 조정된 적이 없다.
- 02-7에서 다룰 튜닝 후보 중 하나가 바로 이 값들을 늘리는 것이다 — 저텍스처 구간에서 특징점을 더 오래·많이 확보하려면 `MaxClones`나 `NumPts`(추출 목표 특징점 수)를 늘려야 하지만, 그만큼 CPU 비용이 올라가 ORB-SLAM3 수준(96%)에 가까워질 수 있다는 트레이드오프가 있다.

## 6. 진단 관점

- 저텍스처 환경에서 정확도가 떨어진다면, 먼저 `MaxSLAM`/`MaxClones`가 기본값 그대로인지 확인한다 — 특징점을 충분히 오래 추적하지 못해 SLAM feature로 승격될 기회 자체가 적을 수 있다.
- CPU가 예상보다 높다면 반대로 이 값들이 과도하게 늘어나 있지 않은지 확인한다.

## 7. 다음 문서와의 연결

- 다음: **02-5. ZUPT 메커니즘과 자세 안정화** — ZUPT의 일반 원리를 다룬다. 이 프로젝트에서 가장 우선순위 높은 미해결 가설은 이어지는 **02-8**에서 다룬다.

## 8. 참고자료

- Geneva et al., "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 ([DOI: 10.1109/ICRA40945.2020.9196524](https://doi.org/10.1109/ICRA40945.2020.9196524))
- 프로젝트 내부 자료 — RTAB-Map `OdomOpenVINS` 파라미터 정의 및 코드 분석
