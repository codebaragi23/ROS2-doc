# C7. Relocalization 감시 로직 구현

> RTAB-Map & Nav2 심화 시리즈 · Part C. Relocalization 심화 (C7, 확장 추가분)
> 이전 문서: C6. Nav2 표준 relocalization과의 차이
> 이 문서는 C2(트리거 시점 5분류)에서 이론으로만 정리했던 감시 로직을 실제 ROS2 노드 설계 수준으로 구체화한 것이다. 코드는 개념 검증용 예시이며, 실제 배포 전 프로젝트 환경에서 검증이 필요하다.

## 1. 개요

C2·C3에서 정리한 5가지 relocalization 트리거는 Nav2나 RTAB-Map이 자동으로 감지해주지 않는다. C6에서 짚었듯, 이 감시 자체를 프로젝트가 직접 구현해야 한다. 이 문서는 그 구현의 최소 골격을 제시하며, C 시리즈(C1~C6) 전체의 이론을 실제 코드로 마무리하는 문서다.

## 2. 핵심 개념: 감시 노드가 구독해야 할 정보

| 정보 | Topic/서비스 | 어떤 트리거(C2)와 관련되나 |
|---|---|---|
| Loop closure inlier 수 | `/rtabmap/info` | ② 긴급 relocalization |
| 포즈 공분산 | `map→odom` TF 또는 커스텀 covariance 토픽 | ②③ |
| 오도메트리 연속성 | `/odom` | ③ 누적 드리프트 |
| AprilTag 검출 결과 | (AprilTag 노드의 detection topic) | ④ 랜드마크 보정 |
| 목표 진입 상태 | Nav2 Action Feedback(B6-1 참고) | ⑤ 정밀 작업 직전 |

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
      self.get_logger().warn('Low localization confidence detected — triggering spin recovery')
      self.trigger_spin_recovery()
      self.consecutive_low_inlier_count = 0

  def trigger_spin_recovery(self):
    # Reuse Nav2's standard Spin behavior (B6) for relocalization purposes (B8/C2)
    goal = Spin.Goal()
    goal.target_yaw = 6.28  # full rotation, radians
    self.spin_client.wait_for_server()
    self.spin_client.send_goal_async(goal)

def main(args=None):
  rclpy.init(args=args)
  node = RelocalizationMonitor()
  rclpy.spin(node)
  node.destroy_node()
  rclpy.shutdown()

if __name__ == '__main__':
  main()
```

* `Info` 메시지 필드는 실제 `rtabmap_msgs` 버전에 따라 이름이 다를 수 있어, 이 코드는 **구조 예시**로만 취급해야 한다 — 실제 구현 전 `ros2 interface show rtabmap_msgs/msg/Info`로 정확한 필드를 확인해야 한다.
* `trigger_spin_recovery()`는 B6-1에서 다룬 `Spin` Action을 그대로 재사용한다. 이것이 B8에서 설명한 "Nav2 표준 기능을 relocalization 목적으로 재활용"하는 구조의 실제 코드 형태다.
* `LOST_THRESHOLD`, `MIN_INLIERS` 값은 C3에서 다룬 `Vis/MinInliers`와 같은 맥락의 임계값이며, 실환경 데이터로 튜닝이 필요하다.

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

- 이것으로 "RTAB-Map & Nav2 심화" 시리즈 Part C(C1~C7)가 완결된다. 이 시리즈는 Part A(지도제작) → Part B(Nav2) → Part C(Relocalization 이론+구현)로 마무리된다.
- 향후 이 감시 노드가 실제로 구현·검증되면, 결과를 C5(실전 진단 체크리스트)에 반영하는 별도 갱신이 필요하다.
- 별도 심화가 필요한 대안 SLAM 백엔드 논의(ORB-SLAM3, OpenVINS, VINS-Fusion)는 독립된 "SLAM 백엔드 평가" 시리즈(D1~D4)에서 다룬다.

## 7. 참고자료

- rtabmap_msgs 공식 정의 — `Info` 메시지 필드 (실제 필드명은 빌드 시점 버전 확인 필요)
- Nav2 공식 문서 — `Spin` Action 클라이언트 사용법(B6-1 참고)
- C2, C3, B8 (이 시리즈 내부 문서) — 이 구현의 이론적 근거
