# 문제 해결 - colcon 빌드 오류 모음

> ROS2 응용 & 센서 연동 시리즈 · 1편
> 선행 학습: ROS2 기초 1편(개발 환경과 워크스페이스 구조)

## 문서 정보

| 항목 | 내용 |
|---|---|
| 학습 단계 | Level 2 — 실전 진단 |
| 예상 선행 지식 | ROS2 기초 1편(워크스페이스), 9편(자주 발생하는 오류 모음) |
| 학습 목표 | colcon 빌드 실패의 대표 유형을 구분하고 원인을 스스로 좁혀갈 수 있다 |
| 기준 환경 | Ubuntu 22.04, ROS2 Humble |
| 관련 문서 | 9편(일반 오류 모음)의 colcon 특화 확장판 |

---

## 1. 먼저 알아야 할 핵심

1. ROS2 기초 9편은 "빌드/source 누락"을 가장 흔한 원인으로만 다뤘다. 이 문서는 **빌드가 실행은 되는데 실패하는** 더 구체적인 유형들을 다룬다.
2. colcon 빌드 실패는 크게 **의존성 문제**, **빌드 타입 혼용 문제**, **캐시/증분 빌드 문제** 세 갈래로 나뉜다.
3. 실제 로봇 프로젝트(RealSense, LiDAR, RTAB-Map 소스 빌드)에서는 이 문서의 유형들이 1편의 단순 "Package not found"보다 훨씬 자주 등장한다.

## 2. 이 개념은 무엇인가

colcon은 워크스페이스 안의 여러 패키지를 **의존성 순서에 맞춰** 순차/병렬로 빌드하는 도구다. 이 순서 결정과 빌드 도구 선택(ament_python/ament_cmake) 과정에서 생기는 실패가 이 문서의 주제다.

## 3. 왜 필요한가

RealSense나 LiDAR 드라이버, RTAB-Map처럼 apt로 바로 설치되지 않는 패키지를 소스로 빌드하다 보면 의존성 트리가 깊어지고, C++/Python 빌드 타입이 섞인 워크스페이스에서 예상치 못한 실패가 잦아진다. 1편/9편의 "source 확인" 수준으로는 해결되지 않는 경우가 이 문서의 대상이다.

## 4. 대표 오류 유형

### 유형 A: 의존성 패키지가 시스템에 없음

```
CMake Error: Could not find a package configuration file provided by "realsense2"
```

- **원인**: `package.xml`에 선언된 의존 패키지가 실제로 설치되지 않음.
- **확인**: `rosdep check --from-paths src --ignore-src`
- **해결**: `rosdep install --from-paths src --ignore-src -r -y`로 워크스페이스가 필요로 하는 시스템 패키지를 한 번에 설치한다.

### 유형 A: 워크스페이스 내부 패키지 간 의존성 순서 문제

```
Package 'my_pkg_b' not found (dependency of 'my_pkg_a')
```

- **원인**: `my_pkg_a`가 같은 워크스페이스의 `my_pkg_b`를 의존하는데, `package.xml`에 `<depend>my_pkg_b</depend>`를 선언하지 않아 colcon이 빌드 순서를 잘못 판단함.
- **해결**: 의존 관계를 `package.xml`에 명시적으로 선언한다. colcon은 이 선언을 보고 그래프를 만들어 순서를 결정하므로, 선언이 빠지면 순서가 랜덤해질 수 있다.

### 유형 B: ament_python과 ament_cmake 혼용 워크스페이스에서의 충돌

- **증상**: 특정 패키지만 계속 재빌드되거나, `--symlink-install`을 썼는데 일부 패키지에서 에러.
- **원인**: `--symlink-install`은 Python 패키지에는 잘 동작하지만 C++(`ament_cmake`) 패키지 빌드 산출물과 혼재하면서 캐시가 꼬이는 경우가 있다.
- **해결**: 문제가 되는 패키지만 `--packages-select`로 지정해 `--symlink-install` 없이 재빌드해본다.

### 유형 C: 캐시 오염으로 인한 반복 실패

```
CMake Error: The current CMakeCache.txt directory ... is different from the directory ... where CMakeCache.txt was created.
```

- **원인**: 워크스페이스를 다른 경로로 옮기거나 복사한 뒤 이전 `build/` 캐시가 그대로 남아 경로가 꼬임.
- **해결**: `rm -rf build install log` 후 완전 재빌드. (1편에서 이미 언급되었지만, 이 유형에서는 **반드시** 필요한 유일한 해결책이다.)

### 유형 D: 특정 패키지만 빌드 실패해도 전체가 멈추는 문제

- **증상**: 여러 패키지 중 하나가 실패하면 나머지 패키지들도 `Skipped`로 표시됨.
- **원인**: colcon의 기본 동작은 실패한 패키지에 의존하는 패키지들의 빌드를 건너뛰는 것.
- **해결**: 의존 관계가 없는 패키지는 `--packages-skip-build-finished`나 `--continue-on-error` 옵션으로 나머지를 계속 빌드하게 할 수 있다.

## 5. 자주 사용하는 명령어

| 목적 | 명령어 |
|---|---|
| 워크스페이스 의존성 일괄 설치 | `rosdep install --from-paths src --ignore-src -r -y` |
| 의존성만 체크(설치 안 함) | `rosdep check --from-paths src --ignore-src` |
| 특정 패키지 + 그 의존성만 빌드 | `colcon build --packages-up-to <pkg>` |
| 상세 오류 로그 그대로 출력 | `colcon build --event-handlers console_direct+` |
| 완전 초기화 후 재빌드 | `rm -rf build install log && colcon build` |

## 6. 개념 간 연결

- 이 문서의 유형 A는 ROS2 기초 1편에서 배운 `package.xml`의 `<depend>` 개념이 실전에서 더 복잡하게 얽힌 경우다.
- RealSense D435i, LiDAR C1 연동 문서(이 시리즈 5편, 6편)에서 소스 빌드가 필요할 때 이 문서의 유형들을 다시 참고하게 된다.

## 7. 핵심 요약

1. colcon 빌드 실패는 의존성/빌드타입 혼용/캐시 오염 세 갈래로 분류하면 원인 추적이 빨라진다.
2. `rosdep install`은 시스템 의존성 문제의 표준 해결책이다.
3. 캐시 오염이 의심되면 부분 삭제보다 `rm -rf build install log` 완전 초기화가 가장 확실하다.

## 8. 다음 학습 주제

- 다음: **Composable Node와 Executor 구조** — 빌드가 끝난 뒤, 그 노드들을 어떻게 더 효율적으로 실행할지 다룬다.

## 9. 참고자료

- colcon 공식 문서 — 빌드 옵션 전체 목록
- rosdep 공식 문서 — 의존성 해석 방식
