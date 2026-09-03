# 04. colcon 빌드 오류 모음

> ROS2 응용 & 센서 연동 시리즈

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 응용 (참조용 — 막혔을 때 펼쳐보는 문서) |
| 예상 선행 지식 | [[01_개발_환경과_워크스페이스_구조\|ROS2 기초 01. 개발 환경과 워크스페이스 구조]], [[09_문제_해결_자주_발생하는_오류_모음\|09. 문제 해결]] |
| 학습 목표 | 빌드 실패를 유형별로 분류해 접근할 수 있다 / 빌드 타입 혼용·캐시 오염 문제를 인식하고 해결할 수 있다 / `rosdep`으로 의존성을 먼저 해결하는 습관을 갖는다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble, colcon |
| 관련 문서 | 이전: [[03_센서_연동_LiDAR_C1\|응용 03. LiDAR C1]] / 다음: [[05_Launch_시스템_디버깅\|응용 05. Launch 시스템 디버깅]] |

> **읽는 방법**: 이 문서는 처음부터 끝까지 정독하는 문서가 아니다. **소스 빌드에서 막혔을 때 증상으로 찾아보는 참조 문서**다. apt 설치만으로 진행 중이라면 지금은 건너뛰고, 나중에 RTAB-Map이나 SLAM 백엔드를 소스로 빌드할 때 돌아오면 된다.

## 1. 개요

ROS2 기초 09는 "빌드/source 누락"을 가장 흔한 원인으로만 다뤘다. 이 문서는 **빌드가 실행은 되는데 실패하는** 더 구체적인 유형들 — 실제 로봇 프로젝트(RealSense, LiDAR, RTAB-Map 소스 빌드)에서 ROS2 기초 09의 단순 "Package not found"보다 훨씬 자주 등장하는 문제들 — 을 다룬다.

## 2. 핵심 개념: 실패의 세 갈래

colcon 빌드 실패는 크게 **의존성 문제**, **빌드 타입 혼용 문제**, **캐시/증분 빌드 문제** 세 갈래로 나뉜다. colcon은 워크스페이스 안의 여러 패키지를 의존성 순서에 맞춰 순차/병렬로 빌드하는 도구이며, 이 순서 결정과 빌드 도구 선택(`ament_python`/`ament_cmake`) 과정에서 생기는 실패가 이 문서의 주제다.

## 3. 대표 오류 유형

### 유형 A-1: 의존성 패키지가 시스템에 없음

```
CMake Error: Could not find a package configuration file provided by "<dep_pkg>"
```

`package.xml`에 선언된 의존 패키지가 실제로 설치되지 않은 경우다. `rosdep check --from-paths src --ignore-src`로 확인하고, `rosdep install --from-paths src --ignore-src -r -y`로 워크스페이스가 필요로 하는 시스템 패키지를 한 번에 설치한다.

### 유형 A-2: 워크스페이스 내부 패키지 간 의존성 순서 문제

```
Package 'my_pkg_b' not found (dependency of 'my_pkg_a')
```

`my_pkg_a`가 같은 워크스페이스의 `my_pkg_b`를 의존하는데 `package.xml`에 `<depend>my_pkg_b</depend>`를 선언하지 않아 colcon이 빌드 순서를 잘못 판단한 경우다. 의존 관계를 명시적으로 선언해야 colcon이 이를 보고 그래프를 만들어 순서를 결정한다.

### 유형 B: ament_python과 ament_cmake 혼용 워크스페이스에서의 충돌

특정 패키지만 계속 재빌드되거나, `--symlink-install`을 썼는데 일부 패키지에서만 에러가 나는 증상이다. `--symlink-install`은 Python 패키지에는 잘 동작하지만 C++(`ament_cmake`) 패키지 빌드 산출물과 혼재하면서 캐시가 꼬이는 경우가 있다. 문제가 되는 패키지만 `--packages-select`로 지정해 `--symlink-install` 없이 재빌드해본다.

### 유형 C: 캐시 오염으로 인한 반복 실패

```
CMake Error: The current CMakeCache.txt directory ... is different from the directory ... where CMakeCache.txt was created.
```

워크스페이스를 다른 경로로 옮기거나 복사한 뒤 이전 `build/` 캐시가 그대로 남아 경로가 꼬인 경우다. `rm -rf build install log` 후 완전 재빌드가 이 유형에서는 **반드시** 필요한 유일한 해결책이다.

### 유형 D: 특정 패키지만 빌드 실패해도 전체가 멈추는 문제

여러 패키지 중 하나가 실패하면 나머지도 `Skipped`로 표시되는 것이 colcon의 기본 동작(실패한 패키지에 의존하는 패키지들의 빌드를 건너뜀)이다. 의존 관계가 없는 패키지는 `--packages-skip-build-finished`나 `--continue-on-error` 옵션으로 나머지를 계속 빌드하게 할 수 있다.

## 4. 이 프로젝트에서의 적용 (Yahboom X3)

이 프로젝트에서 유형 A-1(의존성 패키지 없음)이 실제로 발생한 대표 사례는 RealSense 드라이버 소스 빌드였다.

```
CMake Error: Could not find a package configuration file provided by "realsense2"
```

`realsense2` SDK가 시스템에 먼저 설치되어 있어야 `realsense2_camera` 패키지가 빌드된다 — `rosdep install`로 잡히지 않는 경우 [[02_센서_연동_RealSense_D435i|응용 02. 센서 연동 - RealSense D435i]] 문서의 설치 절차를 먼저 따른다. RTAB-Map, LiDAR C1 드라이버 소스 빌드에서도 유형 B(빌드 타입 혼용)와 유형 C(캐시 오염)가 실제로 여러 차례 발생했다([[ros2-nav-yahboom]] 참고).

## 5. 관련 명령어

| 목적 | 명령어 |
|---|---|
| 워크스페이스 의존성 일괄 설치 | `rosdep install --from-paths src --ignore-src -r -y` |
| 의존성만 체크(설치 안 함) | `rosdep check --from-paths src --ignore-src` |
| 특정 패키지 + 그 의존성만 빌드 | `colcon build --packages-up-to <pkg>` |
| 상세 오류 로그 그대로 출력 | `colcon build --event-handlers console_direct+` |
| 완전 초기화 후 재빌드 | `rm -rf build install log && colcon build` |

## 6. 진단 관점

빌드 실패 로그를 마지막 줄(`Failed`)만 보지 말고, `--event-handlers console_direct+`로 중간의 `error:`나 `ModuleNotFoundError` 부분을 직접 확인하는 습관이 중요하다. 캐시 오염이 의심되면 부분 삭제보다 `rm -rf build install log` 완전 초기화가 가장 확실하다.

## 7. 다음 문서와의 연결

- 다음: **[[05_Launch_시스템_디버깅|Launch 시스템 디버깅]]** — 노드가 여러 개로 늘어난 뒤 "떴는데 동작하지 않는" 상황을 Launch 레벨에서 진단한다.

## 8. 이해도 점검

1. 빌드 실패를 만났을 때, 로그를 자세히 읽기 **전에** 먼저 해볼 만한 조치는 무엇인가?
2. `rosdep install`이 하는 일은 무엇이며, 왜 소스 빌드 전에 먼저 실행하는가?
3. 어제까지 잘 되던 빌드가 코드를 안 바꿨는데 갑자기 깨졌다. 무엇을 의심하는가?
4. 패키지 하나만 다시 빌드하고 싶을 때 쓰는 옵션은?

> [!info]- 정답 및 해설 보기
> 1. **`build/`·`install/` 디렉터리를 지우고 다시 빌드**해본다(캐시 오염 배제). 상당수의 "이해할 수 없는" 빌드 오류가 이전 빌드 잔여물 때문이며, 이걸 먼저 배제해야 로그가 진짜 원인을 가리킨다.
> 2. `package.xml`에 선언된 의존 패키지들을 찾아 **시스템에 자동 설치**해준다. 소스 빌드는 의존성이 갖춰져 있다고 가정하므로, 이걸 건너뛰면 "헤더를 못 찾겠다"류의 오류가 줄줄이 나온다.
> 3. **캐시 오염 또는 빌드 타입 혼용**을 의심한다. 특히 같은 워크스페이스를 서로 다른 빌드 타입/컴파일러 설정으로 빌드한 적이 있다면 잔여물이 충돌한다. 이 프로젝트에서도 실제로 겪은 유형이다([[ros2-nav-yahboom]]).
> 4. `colcon build --packages-select <패키지명>`. 전체 재빌드는 시간이 오래 걸리므로, 문제가 되는 패키지만 반복 시도할 때 필수적이다.

## 9. 참고자료

- [colcon 공식 문서](https://colcon.readthedocs.io/) — 빌드 옵션 전체 목록
- [rosdep 공식 문서](https://docs.ros.org/en/independent/api/rosdep/html/) — 의존성 해석 방식
