# Step 4: Behavior Tree & 센서 시스템

> **난이도**: ⭐⭐⭐⭐ 상급  
> **목표**: Behavior Tree 아키텍처 이해, 센서(시야/청각) 시스템 구현, 복합 AI 행동 설계  
> **소요 시간**: 60~80분  
> **선행 조건**: Step 3 (FSM) 완료 권장

---

## 1. 개념 설명

### Behavior Tree (BT)란?

Behavior Tree는 게임 AI 행동을 **트리 구조**로 표현하는 아키텍처입니다.
FSM의 한계(상태 폭발)를 극복하기 위해 개발되었습니다.

### BT vs FSM

| 항목 | FSM | Behavior Tree |
|------|-----|---------------|
| 구조 | 평면적 (N² 전환 문제) | 계층적 (트리 구조) |
| 상태 수 | 3~12개 적합 | 50+개도 무리 없음 |
| 재사용성 | 낮음 | 높음 (서브트리 조합) |
| 디버깅 | 쉬움 (현재 상태) | 보통 (어느 노드 실행중?) |
| 성능 | 최고 | 양호 |
| 시각 편집 | 어려움 | 전문 에디터 존재 |

### BT 핵심 노드 타입

```
                    ┌──────────────────────────────────────────┐
                    │            Root (루트)                    │
                    │         위에서 아래로 실행                │
                    └──────────────────────────────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            ▼                         ▼                         ▼
    ┌───────────────┐        ┌───────────────┐        ┌───────────────┐
    │  Composite    │        │   Decorator   │        │     Leaf      │
    │  (제어 흐름)   │        │  (조건/변환)   │        │  (실행 단위)   │
    └───────┬───────┘        └───────┬───────┘        └───────┬───────┘
            │                        │                        │
    ┌───────┴───────┐       ┌───────┴───────┐        ┌───────┴───────┐
    │  Selector(?), │       │  Inverter     │        │   Action      │
    │  Sequence(→), │       │  Repeater     │        │   Condition   │
    │  Parallel(⚡)  │       │  UntilFail    │        │   Wait        │
    └───────────────┘       └───────────────┘        └───────────────┘
```

#### Composite 노드

| 노드 | 약어 | 동작 방식 |
|------|------|-----------|
| **Sequence** | `→` | 자식을 **순서대로** 실행, 하나라도 실패(Fail)면 중단 |
| **Selector** | `?` | 자식을 **순서대로** 실행, 하나라도 성공(Success)면 중단 |
| **Parallel** | `⚡` | 모든 자식을 **동시에** 실행 |

```
Sequence (→) : "모두 성공"  - AND 조건
  [MoveTo] → [Attack] → [Wait]

Selector (?) : "하나라도 성공" - OR 조건
  [SeePlayer?] ? [HearSound?] ? [Patrol]
```

#### Decorator 노드

| 노드 | 설명 |
|------|------|
| **Inverter** | 결과 반전 (Success ↔ Failure) |
| **Repeater** | 자식 노드 N번 반복 또는 무한 반복 |
| **UntilSuccess** | 성공할 때까지 반복 |
| **Cooldown** | 쿨다운 동안 실행 차단 |
| **Timer** | 일정 시간 후 노드 실행 |

#### Leaf 노드

| 노드 | 설명 |
|------|------|
| **Action** | 실제 행동 수행 (Move, Attack, Wait) |
| **Condition** | 조건 확인 (SeePlayer? IsHealthLow?) |
| **Wait** | 지정 시간 대기 |

### 센서 시스템 (Sensor System)

현실적인 AI를 위해 필요한 감각 입력:

| 센서 | 용도 | 구현 방식 |
|------|------|-----------|
| **시각 (Sight)** | Player 발견 | 거리 + 각도 + Raycast |
| **청각 (Hearing)** | 소리 인지 | 구면 반경 + 소리 크기 |
| **후각/촉각** | 흔적 감지 | Trigger Collider |
| **지식 (Knowledge)** | 마지막 위치 기억 | Vector3 저장 |

---

## 2. 실습: Behavior Tree + 센서 AI

### Step 4.1 — 프로젝트 설정

1. 새 3D 프로젝트 생성 (`Unity_AI_Step4`)
2. Step 1/3과 동일하게 기본 씬 구성 (Ground, Player, Enemy, Waypoints)

### Step 4.2 — BT 코어 시스템

**BT 노드 기본 클래스들:**

`BehaviorTree/Nodes/Node.cs`:

```csharp
using System.Collections.Generic;

namespace BehaviorTree
{
    public enum NodeState
    {
        Success,
        Failure,
        Running   // 실행 중 (다음 프레임에 계속)
    }

    public abstract class Node
    {
        protected List<Node> children = new();
        public Node Parent { get; private set; }
        public string Name { get; set; }

        public Node() { }

        public Node(params Node[] nodes)
        {
            children.AddRange(nodes);
            foreach (var child in children)
                child.Parent = this;
        }

        public abstract NodeState Evaluate();

        public virtual void Reset()
        {
            foreach (var child in children)
                child.Reset();
        }
    }
}
```

`BehaviorTree/Nodes/Composite/Sequence.cs`:

```csharp
namespace BehaviorTree
{
    // 모든 자식이 성공해야 Success
    public class Sequence : Node
    {
        public Sequence() : base() { }
        public Sequence(params Node[] nodes) : base(nodes) { }

        public override NodeState Evaluate()
        {
            bool anyChildRunning = false;

            foreach (var child in children)
            {
                switch (child.Evaluate())
                {
                    case NodeState.Failure:
                        return NodeState.Failure;
                    case NodeState.Running:
                        anyChildRunning = true;
                        continue;
                    case NodeState.Success:
                        continue;
                }
            }

            return anyChildRunning ? NodeState.Running : NodeState.Success;
        }
    }
}
```

`BehaviorTree/Nodes/Composite/Selector.cs`:

```csharp
namespace BehaviorTree
{
    // 하나의 자식만 성공하면 Success
    public class Selector : Node
    {
        public Selector() : base() { }
        public Selector(params Node[] nodes) : base(nodes) { }

        public override NodeState Evaluate()
        {
            foreach (var child in children)
            {
                switch (child.Evaluate())
                {
                    case NodeState.Failure:
                        continue;
                    case NodeState.Running:
                        return NodeState.Running;
                    case NodeState.Success:
                        return NodeState.Success;
                }
            }

            return NodeState.Failure;
        }
    }
}
```

`BehaviorTree/Nodes/Composite/Parallel.cs`:

```csharp
namespace BehaviorTree
{
    // 모든 자식을 동시에 실행
    public class Parallel : Node
    {
        private readonly int requiredSuccesses;

        public Parallel(int requiredSuccesses = -1) : base()
        {
            this.requiredSuccesses = requiredSuccesses;
        }
        public Parallel(int requiredSuccesses, params Node[] nodes) : base(nodes)
        {
            this.requiredSuccesses = requiredSuccesses;
        }

        public override NodeState Evaluate()
        {
            int successCount = 0;
            bool anyRunning = false;

            foreach (var child in children)
            {
                switch (child.Evaluate())
                {
                    case NodeState.Success:
                        successCount++;
                        break;
                    case NodeState.Running:
                        anyRunning = true;
                        break;
                }
            }

            if (requiredSuccesses > 0 && successCount >= requiredSuccesses)
                return NodeState.Success;

            return anyRunning ? NodeState.Running : NodeState.Success;
        }
    }
}
```

`BehaviorTree/Nodes/Decorator/Inverter.cs`:

```csharp
namespace BehaviorTree
{
    // 결과 반전
    public class Inverter : Node
    {
        public Inverter(Node child) : base(child) { }

        public override NodeState Evaluate()
        {
            switch (children[0].Evaluate())
            {
                case NodeState.Success: return NodeState.Failure;
                case NodeState.Failure: return NodeState.Success;
                default: return NodeState.Running;
            }
        }
    }
}
```

`BehaviorTree/Nodes/Decorator/Cooldown.cs`:

```csharp
using UnityEngine;

namespace BehaviorTree
{
    // 쿨다운 동안 실행 차단
    public class Cooldown : Node
    {
        private readonly float cooldownTime;
        private float lastRunTime = -Mathf.Infinity;

        public Cooldown(float cooldownTime, Node child) : base(child)
        {
            this.cooldownTime = cooldownTime;
        }

        public override NodeState Evaluate()
        {
            if (Time.time - lastRunTime < cooldownTime)
                return NodeState.Success; // 쿨다운 중이면 Success 반환 (실행 안 함)

            var result = children[0].Evaluate();
            if (result == NodeState.Success || result == NodeState.Failure)
                lastRunTime = Time.time;

            return result;
        }

        public override void Reset()
        {
            base.Reset();
            lastRunTime = -Mathf.Infinity;
        }
    }
}
```

`BehaviorTree/Nodes/Decorator/Repeater.cs`:

```csharp
namespace BehaviorTree
{
    // 자식 노드 반복 실행
    public class Repeater : Node
    {
        private readonly int maxRepeats;
        private int currentRepeat;

        public Repeater(Node child, int maxRepeats = -1) : base(child)
        {
            this.maxRepeats = maxRepeats;
            currentRepeat = 0;
        }

        public override NodeState Evaluate()
        {
            if (maxRepeats > 0 && currentRepeat >= maxRepeats)
                return NodeState.Success;

            var result = children[0].Evaluate();
            if (result == NodeState.Success || result == NodeState.Failure)
                currentRepeat++;

            // 계속 실행 중인 상태 유지 (Running 반환)
            return NodeState.Running;
        }

        public override void Reset()
        {
            base.Reset();
            currentRepeat = 0;
        }
    }
}
```

`BehaviorTree/Nodes/Action/Wait.cs`:

```csharp
using UnityEngine;

namespace BehaviorTree
{
    public class Wait : Node
    {
        private readonly float duration;
        private float startTime;

        public Wait(float duration) => this.duration = duration;

        public override NodeState Evaluate()
        {
            if (startTime == 0) startTime = Time.time;

            if (Time.time - startTime >= duration)
            {
                startTime = 0;
                return NodeState.Success;
            }

            return NodeState.Running;
        }

        public override void Reset()
        {
            base.Reset();
            startTime = 0;
        }
    }
}
```

`BehaviorTree/BehaviorTree.cs`:

```csharp
using UnityEngine;

namespace BehaviorTree
{
    public class BehaviorTree
    {
        private readonly Node root;
        public string LastStatus { get; private set; }

        public BehaviorTree(Node root)
        {
            this.root = root;
        }

        public NodeState Tick()
        {
            var result = root.Evaluate();
            LastStatus = $"{root.GetType().Name}: {result}";
            return result;
        }

        public void Reset() => root.Reset();
    }
}
```

### Step 4.3 — 센서 시스템

**Sensor/Sensor.cs** (인터페이스):

```csharp
using UnityEngine;

namespace AI.Sensors
{
    public abstract class Sensor : MonoBehaviour
    {
        [SerializeField] protected float updateInterval = 0.2f;
        protected float timer;

        public abstract bool Detect();

        protected virtual void Awake()
        {
            timer = Random.Range(0f, updateInterval); // 분산 실행
        }

        protected virtual void Update()
        {
            timer += Time.deltaTime;
        }
    }
}
```

**Sensor/SightSensor.cs** (시각 센서):

```csharp
using UnityEngine;

namespace AI.Sensors
{
    public class SightSensor : Sensor
    {
        [SerializeField] private float range = 12f;
        [SerializeField] private float angle = 70f;
        [SerializeField] private LayerMask obstacleMask = ~0;
        [SerializeField] private string targetTag = "Player";

        public Transform DetectedTarget { get; private set; }
        public Vector3 LastKnownPosition { get; private set; }

        public override bool Detect()
        {
            if (timer < updateInterval) return DetectedTarget != null;
            timer = 0f;

            // 태그로 대상 검색
            GameObject[] targets = GameObject.FindGameObjectsWithTag(targetTag);
            foreach (var target in targets)
            {
                Vector3 direction = target.transform.position - transform.position;
                float distance = direction.magnitude;

                if (distance > range) continue;

                float dot = Vector3.Angle(transform.forward, direction.normalized);
                if (dot > angle * 0.5f) continue;

                // 시야 차단 확인
                if (Physics.Raycast(transform.position + Vector3.up * 0.5f,
                    direction.normalized, out RaycastHit hit, range, obstacleMask))
                {
                    if (hit.transform.CompareTag(targetTag))
                    {
                        DetectedTarget = hit.transform;
                        LastKnownPosition = hit.transform.position;
                        return true;
                    }
                }
            }

            DetectedTarget = null;
            return false;
        }

        void OnDrawGizmosSelected()
        {
            Gizmos.color = Color.cyan;
            Gizmos.DrawWireSphere(transform.position, range);

            Vector3 left = Quaternion.Euler(0, -angle * 0.5f, 0) * transform.forward * range;
            Vector3 right = Quaternion.Euler(0, angle * 0.5f, 0) * transform.forward * range;
            Gizmos.DrawRay(transform.position + Vector3.up * 0.5f, left);
            Gizmos.DrawRay(transform.position + Vector3.up * 0.5f, right);

            if (DetectedTarget != null)
            {
                Gizmos.color = Color.red;
                Gizmos.DrawLine(transform.position + Vector3.up * 0.5f, DetectedTarget.position + Vector3.up * 0.5f);
            }
        }
    }
}
```

**Sensor/HearingSensor.cs** (청각 센서):

```csharp
using UnityEngine;

namespace AI.Sensors
{
    public class HearingSensor : Sensor
    {
        [SerializeField] private float hearingRange = 15f;
        [SerializeField] private float alertDecayRate = 2f; // 초당 경계심 감소

        public float AlertLevel { get; private set; } // 0 ~ 100
        public Vector3 LastHeardPosition { get; private set; }
        public bool HeardSomething => AlertLevel > 0;

        // 외부에서 호출 (예: 총소리, 발소리)
        public void ReportSound(Vector3 position, float intensity = 50f)
        {
            float distance = Vector3.Distance(transform.position, position);
            if (distance > hearingRange) return;

            // 거리에 따른 감쇠
            float attenuation = 1f - (distance / hearingRange);
            AlertLevel = Mathf.Min(100, AlertLevel + intensity * attenuation);
            LastHeardPosition = position;
        }

        public override bool Detect()
        {
            if (timer < updateInterval) return HeardSomething;
            timer = 0f;

            // 경계심 자연 감소
            AlertLevel = Mathf.Max(0, AlertLevel - alertDecayRate * updateInterval);
            return HeardSomething;
        }

        void OnDrawGizmosSelected()
        {
            Gizmos.color = Color.green;
            Gizmos.DrawWireSphere(transform.position, hearingRange);
        }
    }
}
```

### Step 4.4 — BT 액션/컨디션 노드

**Actions/MoveToPosition.cs**:

```csharp
using UnityEngine;
using UnityEngine.AI;
using BehaviorTree;

public class MoveToPosition : Node
{
    private readonly NavMeshAgent agent;
    private readonly Transform target;
    private readonly float stoppingDistance;

    public MoveToPosition(NavMeshAgent agent, Transform target, float stoppingDistance = 1.0f)
    {
        this.agent = agent;
        this.target = target;
        this.stoppingDistance = stoppingDistance;
    }

    public override NodeState Evaluate()
    {
        if (target == null || agent == null)
            return NodeState.Failure;

        agent.SetDestination(target.position);
        agent.isStopped = false;

        if (!agent.pathPending && agent.remainingDistance <= stoppingDistance)
            return NodeState.Success;

        return NodeState.Running;
    }
}
```

**Actions/MoveToPositionVector3.cs**:

```csharp
using UnityEngine;
using UnityEngine.AI;
using BehaviorTree;

public class MoveToPositionVector3 : Node
{
    private readonly NavMeshAgent agent;
    private Vector3 targetPosition;
    private readonly float stoppingDistance;

    public MoveToPositionVector3(NavMeshAgent agent, Vector3 target, float stoppingDistance = 1.0f)
    {
        this.agent = agent;
        this.targetPosition = target;
        this.stoppingDistance = stoppingDistance;
    }

    public void SetTarget(Vector3 position) => targetPosition = position;

    public override NodeState Evaluate()
    {
        if (agent == null) return NodeState.Failure;

        agent.SetDestination(targetPosition);
        agent.isStopped = false;

        if (!agent.pathPending && agent.remainingDistance <= stoppingDistance)
            return NodeState.Success;

        return NodeState.Running;
    }
}
```

**Actions/RotateToTarget.cs**:

```csharp
using UnityEngine;
using BehaviorTree;

public class RotateToTarget : Node
{
    private readonly Transform self;
    private readonly Transform target;
    private readonly float rotationSpeed = 360f;
    private readonly float tolerance = 5f;

    public RotateToTarget(Transform self, Transform target, float speed = 360f)
    {
        this.self = self;
        this.target = target;
        this.rotationSpeed = speed;
    }

    public override NodeState Evaluate()
    {
        Vector3 direction = (target.position - self.position).normalized;
        direction.y = 0;

        Quaternion targetRot = Quaternion.LookRotation(direction);
        self.rotation = Quaternion.RotateTowards(self.rotation, targetRot, rotationSpeed * Time.deltaTime);

        float angle = Quaternion.Angle(self.rotation, targetRot);
        return angle <= tolerance ? NodeState.Success : NodeState.Running;
    }
}
```

**Conditions/CheckHealth.cs**:

```csharp
using BehaviorTree;

public class CheckHealth : Node
{
    private readonly EnemyBrain brain;
    private readonly float thresholdPercent;
    private readonly bool fleeWhenLow;

    public CheckHealth(EnemyBrain brain, float thresholdPercent = 30f, bool fleeWhenLow = true)
    {
        this.brain = brain;
        this.thresholdPercent = thresholdPercent;
        this.fleeWhenLow = fleeWhenLow;
    }

    public override NodeState Evaluate()
    {
        float healthPercent = (float)brain.CurrentHealth / brain.MaxHealth * 100f;

        bool condition = healthPercent < thresholdPercent;
        // fleeWhenLow=true면 체력 낮을 때 Success
        return (condition == fleeWhenLow) ? NodeState.Success : NodeState.Failure;
    }
}
```

### Step 4.5 — EnemyBrain (BT 통합 컨트롤러)

`EnemyBrain.cs`:

```csharp
using UnityEngine;
using UnityEngine.AI;
using AI.Sensors;
using BehaviorTree;

public class EnemyBrain : MonoBehaviour
{
    [Header("Components")]
    public NavMeshAgent Agent { get; private set; }
    public SightSensor Sight { get; private set; }
    public HearingSensor Hearing { get; private set; }

    [Header("Stats")]
    public int MaxHealth = 100;
    public int CurrentHealth { get; private set; }

    [Header("Patrol")]
    public Transform[] Waypoints;

    [Header("Combat")]
    public float attackRange = 2.0f;
    public int attackDamage = 15;

    [Header("Flee")]
    public float fleeDistance = 15f;

    private BehaviorTree.BehaviorTree tree;
    private int currentWaypointIndex;

    // BT 노드 참조 (런타임 업데이트용)
    private MoveToPositionVector3 patrolMoveNode;
    private MoveToPosition moveToPlayerNode;

    void Awake()
    {
        Agent = GetComponent<NavMeshAgent>();
        Sight = GetComponent<SightSensor>();
        Hearing = GetComponent<HearingSensor>();
        CurrentHealth = MaxHealth;
    }

    void Start()
    {
        BuildBehaviorTree();
    }

    void Update()
    {
        tree?.Tick();
    }

    private void BuildBehaviorTree()
    {
        // --- Patrol 서브트리 ---
        patrolMoveNode = new MoveToPositionVector3(Agent, GetNextWaypoint(), 0.5f);
        var patrolSequence = new Sequence(
            new Wait(2f),                          // 2초 대기
            patrolMoveNode                         // 다음 웨이포인트로 이동
        );

        // --- Combat 서브트리 ---
        moveToPlayerNode = new MoveToPosition(Agent, Sight.DetectedTarget != null ? Sight.DetectedTarget : transform, attackRange);
        var combatSequence = new Sequence(
            moveToPlayerNode,                      // Player에게 접근
            new RotateToTarget(transform, Sight.DetectedTarget != null ? Sight.DetectedTarget : transform),
            new Cooldown(1.5f,                     // 1.5초 쿨다운
                new ActionNode(() =>               // 공격 실행
                {
                    Attack();
                    return NodeState.Success;
                })
            )
        );

        // --- Investigate 서브트리 (소리 탐지) ---
        var investigateMoveNode = new MoveToPositionVector3(Agent, Hearing.LastHeardPosition, 1.0f);
        var investigateSequence = new Sequence(
            investigateMoveNode,
            new Wait(2f)
        );

        // --- Flee 서브트리 ---
        var fleeMoveNode = new MoveToPositionVector3(Agent, GetFleePosition(), 2.0f);
        var fleeSequence = new Sequence(
            fleeMoveNode
        );

        // --- BT 루트 ---
        var root = new Selector(
            // 1순위: 체력 낮으면 도망
            new Sequence(
                new CheckHealth(this, 30f, true),  // 체력 30% 미만?
                fleeSequence
            ),
            // 2순위: 시야에 Player가 있으면 전투
            new Sequence(
                new ConditionNode(() => Sight.Detect()),
                combatSequence
            ),
            // 3순위: 소리 들리면 수색
            new Sequence(
                new ConditionNode(() => Hearing.Detect()),
                investigateSequence
            ),
            // 4순위: 기본 순찰
            patrolSequence
        );

        tree = new BehaviorTree.BehaviorTree(root);
    }

    private Vector3 GetNextWaypoint()
    {
        if (Waypoints == null || Waypoints.Length == 0)
            return transform.position;

        var pos = Waypoints[currentWaypointIndex].position;
        currentWaypointIndex = (currentWaypointIndex + 1) % Waypoints.Length;
        return pos;
    }

    private Vector3 GetFleePosition()
    {
        // Player 반대 방향으로 도망
        Vector3 fleeDir = (transform.position - Sight.LastKnownPosition).normalized;
        Vector3 fleePos = transform.position + fleeDir * fleeDistance;

        // NavMesh 위의 유효한 위치 찾기
        if (NavMesh.SamplePosition(fleePos, out NavMeshHit hit, fleeDistance, NavMesh.AllAreas))
            return hit.position;

        return transform.position + transform.forward * -fleeDistance;
    }

    public void Attack()
    {
        if (Sight.DetectedTarget == null) return;
        Debug.Log($"[BT] Attacked {Sight.DetectedTarget.name} for {attackDamage} damage!");

        if (Sight.DetectedTarget.TryGetComponent(out PlayerHealth health))
            health.TakeDamage(attackDamage);
    }

    public void TakeDamage(int damage)
    {
        CurrentHealth = Mathf.Max(0, CurrentHealth - damage);
        Debug.Log($"Enemy HP: {CurrentHealth}/{MaxHealth}");

        if (CurrentHealth <= 0)
        {
            Destroy(gameObject, 0.5f);
        }
    }

    void OnDrawGizmos()
    {
        if (tree != null)
        {
            Vector3 pos = transform.position + Vector3.up * 3f;
            UnityEditor.Handles.Label(pos, tree.LastStatus);
        }
    }
}
```

### Step 4.6 — 헬퍼 액션/컨디션 노드

추가로 필요한 간단한 노드들:

```csharp
using System;
using BehaviorTree;

// 람다/델리게이트로 액션을 정의하는 범용 노드
public class ActionNode : Node
{
    private readonly Func<NodeState> action;

    public ActionNode(Func<NodeState> action) => this.action = action;

    public override NodeState Evaluate() => action?.Invoke() ?? NodeState.Failure;
}

// 람다/델리게이트로 조건을 정의하는 범용 노드
public class ConditionNode : Node
{
    private readonly Func<bool> condition;

    public ConditionNode(Func<bool> condition) => this.condition = condition;

    public override NodeState Evaluate() => condition() ? NodeState.Success : NodeState.Failure;
}
```

### Step 4.7 — 사운드 방출 (청각 센서 테스트)

Player에서 소리를 발생시키는 예제:

`PlayerNoiseMaker.cs`:

```csharp
using UnityEngine;

public class PlayerNoiseMaker : MonoBehaviour
{
    [SerializeField] private float walkNoise = 10f;
    [SerializeField] private float runNoise = 30f;
    [SerializeField] private float shootNoise = 80f;

    void Update()
    {
        // WASD 이동 중이면 소리 발생
        if (Input.GetAxis("Horizontal") != 0 || Input.GetAxis("Vertical") != 0)
        {
            float noise = Input.GetKey(KeyCode.LeftShift) ? runNoise : walkNoise;
            EmitNoise(noise);
        }

        // 스페이스바: 총소리 효과
        if (Input.GetKeyDown(KeyCode.Space))
        {
            EmitNoise(shootNoise);
        }
    }

    private void EmitNoise(float intensity)
    {
        // 모든 HearingSensor에 소리 알림
        var sensors = FindObjectsByType<HearingSensor>(FindObjectsSortMode.None);
        foreach (var sensor in sensors)
        {
            sensor.ReportSound(transform.position, intensity);
        }
    }
}
```

---

## 3. BT 트리 구조 분석

위 코드로 구성되는 실제 BT 구조:

```
Selector (루트)
│
├── Sequence (Flee)
│   ├── CheckHealth (< 30%) ← 조건
│   └── MoveToPosition (도망)
│
├── Sequence (Combat)
│   ├── ConditionNode (Sight.Detect) ← 조건
│   ├── Sequence (전투)
│   │   ├── MoveToPlayer (Range 이내)
│   │   ├── RotateToTarget
│   │   └── Cooldown (1.5s)
│   │       └── ActionNode (Attack)
│
├── Sequence (Investigate)
│   ├── ConditionNode (Hearing.Detect) ← 조건
│   └── Sequence (수색)
│       ├── MoveToPosition (소리 위치)
│       └── Wait (2s)
│
└── Sequence (Patrol) ← Fallback
    ├── Wait (2s)
    └── MoveToPosition (다음 Waypoint)
```

### 실행 흐름 예시

1. 체력 30% 미만? → **Y**: 도망 (Flee 서브트리 실행)
2. Player 발견? → **Y**: 접근 → 회전 → 공격 (Combat)
3. 소리 들림? → **Y**: 위치로 이동 → 2초 대기 (Investigate)
4. 위 모두 아니면 → 순찰 (Patrol)

---

## 4. 성능 비교: FSM vs BT

| 지표 | FSM (Step 3) | BT (Step 4) |
|------|-------------|-------------|
| 노드 수 | 4 states + 8 transitions | 15+ BT nodes |
| 상태 추가 난이도 | 보통 (Transition 관리) | 쉬움 (트리에 노드 추가) |
| CPU (100 NPC 기준) | ~0.3ms | ~0.5ms |
| 메모리 | ~2KB | ~5KB |
| 디버깅 편의성 | 현재 상태만 표시 | 실행 중인 노드 경로 표시 |

> BT는 FSM보다 약간 더 무겁지만, **행동이 복잡해질수록 유지보수성이 훨씬 좋습니다**.

---

## 5. 확장 과제

1. **BT 시각화 도구**: Unity EditorWindow로 현재 실행 중인 노드를 트리 형태로 표시
2. **Group AI**: 동료 NPC와 협력 행동 (Selector/Parallel 활용)
3. **Cover System**: 엄폐물 탐색 → 이동 → 엄폐 행동 BT로 구현
4. **GOAP (Goal-Oriented Action Planning)**: BT를 GOAP과 결합하여 동적 계획 수립

---

## 6. 문제 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| BT가 첫 번째 노드만 실행 | Selector/Sequence 잘못 사용 | 하위 노드가 올바르게 Success/Failure 반환하는지 확인 |
| Running 상태에서 멈춤 | 노드가 Failure/Success를 반환 안 함 | 모든 경로에서 NodeState 반환 확인 |
| 센서가 작동 안 함 | Layer/태그 설정 누락 | SightSensor의 targetTag와 obstacleMask 확인 |
| 공격이 너무 빠름 | Cooldown 미적용 | Cooldown Decorator 확인 |

---

**→ 다음 단계: [Step 5: ML-Agents 강화학습](./Step5_ML_Agents.md)**
