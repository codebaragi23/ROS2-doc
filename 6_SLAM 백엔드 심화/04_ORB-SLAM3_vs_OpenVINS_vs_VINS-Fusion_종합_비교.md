# 04. ORB-SLAM3 vs OpenVINS vs VINS-Fusion 종합 비교

> SLAM 백엔드 심화 시리즈 (D 전체 시리즈 마지막)
> 이전 문서: [[03-5_이_프로젝트에_적용한다면|03-5. 이 프로젝트에 적용한다면]]
> 이 문서는 01(ORB-SLAM3, 프로젝트 실측+논문), 02(OpenVINS, 프로젝트 실측+논문), 03(VINS-Fusion, 문헌만)의 내용을 종합한다. **출처의 성격이 서로 다르다는 점**을 표마다 명시한다.

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 4 — 심화 (시리즈 마무리) |
| 예상 선행 지식 | 심화 01·02·03 시리즈 전체 |
| 학습 목표 | 세 백엔드를 구조·정확도·속도·CPU 관점에서 비교할 수 있다 / 출처의 성격(실측 vs 문헌) 차이를 감안해 표를 읽을 수 있다 / 이 프로젝트의 다음 단계 우선순위를 판단할 수 있다 |
| 기준 환경 | 실측(01·02) + 문헌(03) 혼합 |

## 1. 개요

01~03 시리즈 전체를 관통하는 질문 — "이 세 SLAM 백엔드는 근본적으로 뭐가 다르고, 이 프로젝트(Yahboom X3, Jetson, D435i)에는 무엇이 맞는가" — 에 대한 종합 정리다.

## 2. 구조적 차이 (알고리즘 부류)

| 항목 | ORB-SLAM3 | OpenVINS | VINS-Fusion |
|---|---|---|---|
| 알고리즘 부류 | 키프레임 기반 SLAM(Bundle Adjustment) | MSCKF 기반 EKF(필터링) | 최적화 기반 슬라이딩 윈도우 |
| 대표 문서 | 01-1~01-5 | 02-1~02-8 | 03-1~03-5 |
| 특징점 상태 포함 | 명시적으로 포함, 계속 최적화 | 대부분 제외(null-space projection) | 윈도우 안에서만 포함, 밖은 marginalize |
| 지도 성장 | Atlas로 계속 누적(01-1, 01-3) | 고정 크기 윈도우(`MaxClones`, 02-4) | 고정 크기 윈도우 + marginalization(03-3) |
| 추적 실패 시 | 명시적 lost → 새 지도 생성(01-3) | 명시적 lost 없음, 조용히 저신뢰 지속(02-1) | 문헌상 failure detection/recovery 기능 존재(03-1, 미검증) |
| Loop Closure | 통합, 항상 켜짐(01-3, 01-4) | 없음(02-1) | 별도 프로세스, 선택적(03-4) |
| 초기화 철학 | MAP 추정, "충분히 움직였는가"(01-2) | 기본은 "충분히 정지했는가"(02-3) | Loosely-coupled 정렬, "충분히 움직였는가"(03-2) |

## 3. 정확도(Accuracy) — 공개 벤치마크 기준

이 절은 **이 프로젝트의 실측이 아니라 학계 공개 벤치마크(주로 EuRoC 데이터셋)** 결과다. 프로젝트 고유 조건(D435i, 저텍스처 근접 장면, Jetson)에서는 다르게 나타날 수 있음에 유의한다.

- **ORB-SLAM3 논문 자체 보고**: monocular-inertial 설정에서 MSCKF·OKVIS·ROVIO 대비 5~10배, VI-DSO·VINS-Mono 대비 2배 이상 정확하다고 주장(01-1의 원 논문 근거).
- **제3자 종합 비교(Skoltech/Sberbank Robotics Lab)**: ORB-SLAM2/ORB-SLAM3, OpenVSLAM, LDSO, OpenVINS, VINS 계열 등 9개 오픈소스 시스템을 여러 데이터셋에서 비교한 결과, **ORB-SLAM3가 여러 환경에서 일관되게 가장 정확했고, 종종 경쟁 시스템 대비 한 자릿수(order of magnitude) 낮은 오차**를 보였다.
- **항공(aviation) 응용 비교 연구**: ORB-SLAM3가 VINS-Mono 대비 거의 2배 성능(RMSE 기준)을 보였다고 보고.
- **야외(unstructured outdoor) 벤치마크(Journal of Field Robotics, 2025)**: ORB-SLAM3, OpenVINS+VINS-Fusion 조합, HFNet-SLAM 등을 비교 — Garden Medium Perimeter 시나리오에서 ORB-SLAM3, OpenVINS+VINS-Fusion 모두 "정확하지만 약간의 스케일 오차"가 있었다고 보고(3개 시스템이 비슷한 수준으로 나온 사례도 있다는 뜻).

**주의**: ORB-SLAM3가 다수 벤치마크에서 가장 정확하게 나오는 경향이 있지만, 이는 대부분 **loop closure가 켜진 상태**와의 비교다. 02-1에서 다뤘듯 OpenVINS는 기본적으로 loop closure가 없는 순수 odometry이므로, "지도 전체의 전역 정확도"가 아니라 "짧은 구간의 odometry 정확도"만 놓고 비교하면 결과가 달라질 수 있다 — 이 프로젝트처럼 RTAB-Map이 loop closure를 별도로 담당하는 구조(02-1, 02-6)에서는 이 비교가 그대로 적용되지 않는다는 점이 중요하다.

## 4. 속도(Runtime) — 문헌 + 프로젝트 실측

| 항목 | ORB-SLAM3 | OpenVINS | VINS-Fusion |
|---|---|---|---|
| 이 프로젝트 실측 여부 | ✅ 있음(01-5) | ✅ 있음(02-7) | ❌ 없음(03-5) |
| 프레임당 처리 시간 관련 문헌 | — | — | 임베디드(Jetson Orin NX급) 환경에서 VINS-Fusion 총 처리 시간 약 61.65ms(시각 프론트엔드 27.69ms 45% + 시각 업데이트 33.76ms 54%) — FAR-AVIO 논문의 비교 실험 수치 |
| 구조적 예상 | 가장 무거움(지도 계속 누적, 01-1) | 가장 가벼움(고정 윈도우+필터링, 02-1, 02-4) | 중간(고정 윈도우이지만 매 주기 재최적화, 03-3) |

VINS-Fusion의 처리 시간 수치는 이 프로젝트가 아닌 **다른 연구(FAR-AVIO 논문)의 비교 실험**에서 가져온 것으로, Jetson Orin NX급 임베디드 플랫폼 기준이라 이 프로젝트(Yahboom X3의 Jetson)와 유사한 조건이지만 **동일한 실험은 아니다.**

## 5. CPU/메모리 점유율

| 항목 | ORB-SLAM3 (이 프로젝트 실측) | OpenVINS (이 프로젝트 실측) | VINS-Fusion (미실측, 예상) |
|---|---|---|---|
| CPU (공통 191.78초 bag) | **96%** | **38%** | 예상: 두 값 사이, OpenVINS보다 높음(03-3의 매 주기 재최적화 근거) |
| max RSS | **1,204 MiB** | **490 MiB** | 미실측 |
| 구조적 근거 | 지도가 계속 커짐(Atlas, 01-1) | 슬라이딩 윈도우 고정(`MaxClones=11`, 02-4) | 슬라이딩 윈도우는 고정이지만 비선형 최적화를 매 주기 반복(03-3) |

공개 벤치마크 하나를 참고하면: SLAM Hive Benchmarking Suite(대규모 클라우드 벤치마크 연구)는 ORB-SLAM2/3와 VINS-Mono/VINS-Fusion을 EuRoC에서 비교하며 "4가지 모드 모두에서 ORB-SLAM3의 CPU 사용량이 (ORB-SLAM2보다) 작고 메모리 사용량은 크다"고 보고했다 — 이는 ORB-SLAM3 대 ORB-SLAM2 비교이지, 이 프로젝트가 실측한 "ORB-SLAM3 대 OpenVINS" 비교와는 다른 대상이다. 서로 다른 연구의 수치를 그대로 이어붙여 결론 내리지 않도록 주의해야 한다 — 이는 이 문서 전체가 지키려는 원칙이다.

## 6. 추적 지속성(Coverage/Robustness) — 프로젝트 실측 (ORB-SLAM3 vs OpenVINS만)

| 항목 | ORB-SLAM3 | OpenVINS |
|---|---|---|
| coverage(공통 bag) | 99.82% | 99.93% |
| 냉장고 근접 bag1 | fail_track 85회 | init_reset 0회 |
| 냉장고 근접 bag_b | fail_track 2회 | init_reset 0회 |
| 냉장고 근접 bag_c | fail_track 76회 | init_reset 0회(단 02-8의 ZUPT 오발동 가설 주의) |

**중요한 해석 주의점(02-7, 02-8에서 반복 강조)**: coverage/reset 지표만 보면 OpenVINS가 압도적으로 안정적으로 보이지만, 이는 **"조용히 멈춰있어도 실패로 집계되지 않는" 설계 특성** 때문일 수 있다. 실제 이동 거리(`step_translation_m`) 같은 지표를 함께 봐야 진짜 견고함인지 판단할 수 있다 — 이 표만 보고 OpenVINS가 "더 낫다"고 단순 결론 내리는 것은 02-8의 경고를 무시하는 것이다.

## 7. 이 프로젝트 맥락에서의 요약 판단

| 기준 | 판단 |
|---|---|
| 현재 채택 상태 | RTAB-Map이 loop closure/지도 관리를 전담하고, odometry 소스로 OpenVINS를 통합해 사용 중(02-6). ORB-SLAM3 VIO는 미채택 결정(01-5). VINS-Fusion은 미도입. |
| ORB-SLAM3를 안 쓰는 이유 | CPU 부하가 가장 높고(96%), RGB-D-Inertial 초기화가 반복 실패(active-map IMU reset, 01-5). **[정정]** 초기 가설과 달리 원인은 "RGB-D가 스케일을 몰라서"가 아니다 — RGB-D는 depth로 스케일을 이미 안다. 실제로는 **ORB-SLAM3 공식 저장소가 지원을 명시하는 조합이 monocular/monocular-inertial/stereo/stereo-inertial/RGB-D뿐이고 RGB-D-Inertial은 원저자 미검증 커뮤니티 비공식 확장**이라는 점이 밝혀졌다(01-2 12장, 01-5 8.4절). D435i를 RGB-D+IMU로 쓴 제3자 보고도 같은 성능 저하를 관찰해 이 프로젝트의 경험과 일치한다. |
| OpenVINS를 임시로 쓰는 이유 | CPU 부하가 가장 낮고(38%) 리셋이 없어 안정적으로 보이지만, ZUPT 오발동 가설(02-8)이 아직 미검증이라 "조용한 실패"의 가능성이 열려 있음 |
| VINS-Fusion을 아직 안 쓰는 이유 | 문헌상 이론적 장점(loop closure 통합, 움직임 기반 초기화)이 있지만, RTAB-Map과의 통합 방식(03-5)과 D435i RGB-D 활용 가능 여부가 확인되지 않음 |

## 8. 다음 단계 제안 (우선순위, 01-5 8.4절 정정 반영)

1. **02-7/02-8의 ZUPT 오발동 가설 검증** — 현재 채택된 OpenVINS의 신뢰성을 좌우하는 가장 시급한 항목.
2. **[정정] ORB-SLAM3 stereo-inertial 재검토** — 01-5 8.4절에서 확인했듯, RGB-D-Inertial은 원저자 미검증 비공식 확장인 반면 stereo-inertial은 공식 지원·검증 조합이며 과거 실측에서도 리셋 문제가 더 적었다. ORB-SLAM3를 계속 후보에 둔다면 RGB-D-Inertial보다 이 방향이 우선한다.
3. **D435i IMU 캘리브레이션(`rs-imu-calibration`)** — 01-5, 02-2, 02-7에서 세 문서 모두 공통으로 지목한 근본 원인 후보. 다만 RGB-D-Inertial 조합 자체의 미검증 상태는 해결해주지 못하므로 2번보다 우선순위를 낮춘다.
4. **VINS-Fusion의 RTAB-Map 통합 가능 여부 확인** — 03-5의 1번 질문. 이것이 확인돼야 03 시리즈도 01·02와 같은 수준의 실측 근거를 가질 수 있다.

## 9. 참고자료

- 00-1~00-2(공통 기초), 01-1~01-5, 02-1~02-8, 03-1~03-5 (이 시리즈 전체) 및 그 안의 1차 출처
- [GitHub UZ-SLAMLab/ORB_SLAM3](https://github.com/UZ-SLAMLab/ORB_SLAM3) README — 공식 지원 조합에 RGB-D-Inertial이 없다는 근거(01-2 12장, 01-5 8.4절)
- "Comparison of modern open-source visual SLAM approaches" (Skoltech/Sberbank Robotics Lab, [arXiv:2108.01654](https://arxiv.org/abs/2108.01654))
- "Benchmarking SLAM Algorithms in the Cloud: The SLAM Hive Benchmarking Suite" ([arXiv:2406.17586](https://arxiv.org/abs/2406.17586))
- "Visual-Inertial SLAM for Unstructured Outdoor Environments: Benchmarking the Benefits and Computational Costs of Loop Closing" (Schmidt et al., Journal of Field Robotics, 2025, [arXiv:2408.01716](https://arxiv.org/abs/2408.01716))
- "FAR-AVIO: Fast and Robust Schur-Complement Based Acoustic-Visual-Inertial Fusion Odometry with Sensor Calibration" ([arXiv:2512.20355](https://arxiv.org/abs/2512.20355)) — Jetson 임베디드 플랫폼에서의 VINS-Fusion 런타임 비교 수치
- 프로젝트 내부 자료 — [[ros2-nav-yahboom]], 2026-08-21 OpenVINS 코드 분석 문서
