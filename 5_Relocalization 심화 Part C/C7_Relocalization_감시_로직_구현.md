# C7. Relocalization 감시 로직 구현

> Relocalization 심화 (Part C) 시리즈 (C7, 확장 추가분)
> 이전 문서: C6. Nav2 표준 relocalization과의 차이
> 이 문서는 C2-1(이 프로젝트의 트리거 구현)에서 이론으로만 정리했던 감시 로직을 실제 ROS2 노드 설계 수준으로 구체화한 것이다. 코드는 개념 검증용 예시이며, 실제 배포 전 프로젝트 환경에서 검증이 필요하다.

## 1. 개요

C2·C3에서 정리한 5가지 relocalization 트리거는 Nav2나 RTAB-Map이 자동으로 감지해주지 않는다. C6에서 짚었듯, 이 감시 자체를 프로젝트가 직접 구현해야 한다. 이 문서는 그 구현의 최소 골격을 제시하며, C 시리즈(C1~C6) 전체의 이론을 실제 코드로 마무리하는 문서다.

## 2. 핵심 개념: 감시 노드가 구독해야 할 정보

| 정보 | Topic/서비스 | 어떤 트리거(C2)와 관련되나 |
|---|---|---|
| Loop closure inlier 수 | `/rtabmap/info` | ② 긴급 relocalization |
| 포즈 공분산 | `map→odom` TF 또는 커스텀 covariance 토픽 | ②③ |
| 오도메트리 연속성 | `/odom` | ③ 누적 드리프트 |
| AprilTag 검출 결과 | (AprilTag 노드의 detection topic) | ④ 랜드마크 보정 |
| 목표 진입 상태 | Nav2 Action Feedback(B1-1 참고) | ⑤ 정밀 작업 직전 |

`/rtabmap/info`는 RTAB-Map이 표준으로 발행하는 진단 정보 토픽으로, loop closure 성공 여부와 매칭 통계를 담고 있다 — 이 문서의 감시 로직이 새로 만들어야 하는 것은 이 정보를 **해석해서 행동을 트리거하는 판단 계층**이다.

## 3. 이 프로젝트에서의 적용: 감시 노드 설계 골격

```python
import rclpy
from rclpy.node import Node
from rtabmap_msgs.msg import Info
from nav2_msgs.action import Spin
from rclpy.action import ActionClient

class RelocalizationMonitor(Node):
  def __init__(self):
    super().__init__('relocalization_monitor')
    # Subscribe to RTAB-Map diagnostic info (loop closure, inliers, etc.)
    self.create_subscription(Info, '/rtabmap/info', self.info_callback, 10)
    # Action client to trigger Nav2's Spin recovery as a relocalization aid
    self.spin_client = ActionClient(self, Spin, '/spin')
    # Threshold constants — must be tuned per environment (see C3)
    self.MIN_INLIERS = 15
    self.consecutive_low_inlier_count = 0
    self.LOST_THRESHOLD = 5  # consecutive low-inlier frames before triggering
    # Guard flag: prevents sending a new goal while one is already running
    self.spin_in_progress = False

  def info_callback(self, msg: Info):
    # NOTE: exact field names depend on the rtabmap_msgs/Info definition
    # used in this project's ROS2 Humble build — verify against the actual
    # message before relying on this in production.
    inliers = msg.loop_closure_inliers if msg.loop_closure_id > 0 else 0

    if inliers < self.MIN_INLIERS:
      self.consecutive_low_inlier_count += 1
    else:
      self.consecutive_low_inlier_count = 0

    # C2 trigger ②: emergency relocalization when tracking looks lost
    if self.consecutive_low_inlier_count >= self.LOST_THRESHOLD:
      self.trigger_spin_recovery()
      self.consecutive_low_inlier_count = 0

  def trigger_spin_recovery(self):
    # Reuse Nav2's standard Spin behavior (B6) for relocalization purposes (B8/C2)
    if self.spin_in_progress:
      return  # already spinning — do not stack goals
    if not self.spin_client.server_is_ready():
      self.get_logger().warn('Spin action server not available yet')
      return

    self.get_logger().warn('Low localization confidence — triggering spin recovery')
    goal = Spin.Goal()
    goal.target_yaw = 6.28  # full rotation, radians
    self.spin_in_progress = True
    self.spin_client.send_goal_async(goal).add_done_callback(self.goal_response_cb)

  def goal_response_cb(self, future):
    handle = future.result()
    if not handle.accepted:
      self.get_logger().warn('Spin goal rejected')
      self.spin_in_progress = False
      return
    handle.get_result_async().add_done_callback(self.result_cb)

  def result_cb(self, future):
    self.get_logger().info('Spin recovery finished')
    self.spin_in_progress = False  # release the guard

def main(args=None):
  rclpy.init(args=args)
  node = RelocalizationMonitor()
  rclpy.spin(node)
  node.destroy_node()
  rclpy.shutdown()

if __name__ == '__main__':
  main()
```

* **콜백 안에서 절대 블로킹 대기를 하면 안 된다.** 위 코드가 `wait_for_server()` 대신 `server_is_ready()`로 확인만 하고 넘어가는 이유가 이것이다. 기본 실행기(`SingleThreadedExecutor`)는 콜백 하나를 처리하는 동안 다른 콜백을 처리하지 못하므로, 구독 콜백 안에서 다른 결과를 기다리면 **그 결과를 받아야 할 콜백 자체가 실행되지 못해 노드가 영구히 멈춘다(데드락)**. 이는 `ROS2 응용 & 센서 연동 - 02. Composable Node와 Executor 구조`에서 다룬 실행기 구조와 직결되는 문제다.
* **`spin_in_progress` 플래그가 필수다.** 이것이 없으면 Spin이 도는 중에도 `/rtabmap/info`가 계속 들어오면서 5프레임마다 새 goal을 계속 쏘게 되어, 로봇이 회전을 끝내지 못하고 같은 명령을 반복해서 받는다. goal 수락 응답과 최종 결과를 콜백으로 받아 플래그를 해제하는 구조가 함께 있어야 한다.
* `goal.target_yaw = 6.28`(한 바퀴)인 이유: A1에서 다룬 appearance-based loop closure는 **저장된 장면과 지금 보는 장면의 시각적 유사도**로 재매칭을 시도한다. 한 바퀴를 돌면 카메라가 사방의 장면을 모두 훑게 되어, 지도에 저장된 어느 방향의 키프레임과든 매칭될 기회가 최대가 된다.
* `Info` 메시지 필드는 실제 `rtabmap_msgs` 버전에 따라 이름이 다를 수 있어, 이 코드는 **구조 예시**로만 취급해야 한다 — 실제 구현 전 `ros2 interface show rtabmap_msgs/msg/Info`로 정확한 필드를 확인해야 한다.
* `LOST_THRESHOLD`, `MIN_INLIERS` 값은 C3에서 다룬 `Vis/MinInliers`와 같은 맥락의 임계값이며, 실환경 데이터로 튜닝이 필요하다.

> **이 골격이 다루는 범위**: 위 코드는 C2의 5가지 트리거 중 **② 위치 추정 유실**만 구현한다. ①(초기 기동)은 AprilTag 검출 노드 입력이, ③(누적 드리프트)은 주행 거리 누적 추적이, ④(랜드마크 진입)는 마커 pose 구독이, ⑤(정밀 작업 직전)는 Nav2 목표 진입 상태 감시가 각각 추가로 필요하다.

## 4. 관련 파라미터/설계 고려사항

| 항목 | 고려사항 |
|---|---|
| 임계값(`MIN_INLIERS`, `LOST_THRESHOLD`) | 너무 민감하면 정상 흔들림에도 오발동(C4의 가속도 게이팅과 함께 검토) |
| Spin 발동 중복 방지 | 이미 Spin recovery가 진행 중일 때 중복 트리거되지 않도록 상태 플래그 필요 |
| C2 ⑤(정밀 작업 직전) 로직 | 이 감시 노드와 별개로, 도킹 시작 직전 별도의 "정지-검증" 상태 머신이 필요(원문 5번 항목) |

## 5. 진단 관점

- 이 감시 노드가 실제로 동작하는지 확인하려면 `ros2 node list`에 `/relocalization_monitor`가 떠 있는지, `/rtabmap/info`를 실제로 구독하고 있는지(`ros2 node info`)부터 확인한다.
- Spin recovery가 예상보다 자주/드물게 발동한다면 `MIN_INLIERS`/`LOST_THRESHOLD`를 C5(실전 진단 체크리스트)의 기록 항목과 함께 실험적으로 조정한다.

## 6. 다음 문서와의 연결

- 이것으로 Relocalization 심화(Part C, C1~C7)가 완결된다. 지도제작(Part A) → Nav2 내비게이션(Part B) → Relocalization 심화(Part C)로 이어지는 흐름이 여기서 마무리되며, 다음은 대안 SLAM 백엔드를 평가하는 **[[D0-1_D_시리즈를_위한_최소_수학|SLAM 백엔드 평가 (Part D)]]**로 이어진다.
- 향후 이 감시 노드가 실제로 구현·검증되면, 결과를 C5(실전 진단 체크리스트)에 반영하는 별도 갱신이 필요하다.
- 별도 심화가 필요한 대안 SLAM 백엔드 논의(ORB-SLAM3, OpenVINS, VINS-Fusion)는 독립된 "SLAM 백엔드 평가" 시리즈(D1~D4)에서 다룬다.

## 7. 참고자료

- rtabmap_msgs 공식 정의 — `Info` 메시지 필드 (실제 필드명은 빌드 시점 버전 확인 필요)
- Nav2 공식 문서 — `Spin` Action 클라이언트 사용법(B1-1 참고)
- C2, C2-1, C3, B8 (이 시리즈 내부 문서) — 이 구현의 이론적 근거
