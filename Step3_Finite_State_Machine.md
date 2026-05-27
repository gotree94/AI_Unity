# Step 3: 유한 상태 머신 — 지능형 NPC 행동

> **난이도**: ⭐⭐⭐ 중상급  
> **목표**: FSM(Finite State Machine) 패턴을 이해하고, 순찰/추적/공격 행동을 하는 NPC 구현  
> **소요 시간**: 50~60분  
> **선행 조건**: Step 1, 2 완료 권장

---

## 1. 개념 설명

### FSM (Finite State Machine)이란?

FSM은 **제한된 수의 상태(State)** 와 **상태 간 전환(Transition)** 으로
시스템의 행동을 모델링하는 패턴입니다.

### 게임 AI에서의 FSM

게임 NPC의 행동은 FSM으로 표현하기에 매우 적합합니다:

```
        ┌──────────┐
        │  Patrol   │ ←── 순찰 구역 배회
        └────┬─────┘
     Player  │         Player
     발견 ──┤         ├── 시야 소실
            ▼          │
        ┌──────────┐   │
        │  Chase    │──┘
        └────┬─────┘  ←── 플레이어 추적
    근접    │
    ──────┤
          ▼
        ┌──────────┐
        │  Attack   │ ←── 공격
        └──────────┘
```

### FSM의 장단점

| 장점 | 단점 |
|------|------|
| ✅ 직관적이고 이해하기 쉬움 | ❌ 상태가 많아지면 관리 어려움 |
| ✅ CPU 사용률 낮음 | ❌ 상태 전환 폭발 (N² 문제) |
| ✅ 디버깅 쉬움 (현재 상태 명확) | ❌ 재사용성 낮음 |
| ✅ 모바일/콘솔에 최적 | ❌ 복잡한 행동 설계에 부적합 |

> **권장 상태 수**: 3~12개. 더 많은 상태가 필요하면 Behavior Tree 고려 (Step 4).

### FSM 디자인 패턴

가장 일반적인 두 가지 구현 방식:

**방식 1: enum + switch (간단, 초보자용)**

```csharp
public enum AIState { Idle, Patrol, Chase, Attack }

void Update()
{
    switch (currentState)
    {
        case AIState.Idle:   UpdateIdle();   break;
        case AIState.Patrol: UpdatePatrol(); break;
        // ...
    }
}
```

**방식 2: 클래스 기반 State 패턴 (확장성 우수, 전문가용)**

각 상태를 별도 클래스로 분리하여 OCP(Open-Closed Principle)를 지킵니다.
이 실습에서는 **방식 2**를 사용합니다.

---

## 2. 실습: FSM 기반 AI NPC

### Step 3.1 — 프로젝트 설정

1. 새 3D 프로젝트 생성 (`Unity_AI_Step3`)
2. AI Navigation 패키지 설치
3. Step 1과 동일하게 Ground, Player, Enemy 구성
4. Enemy 주변에 순찰 지점(Waypoint) 추가

**씬 구성:**

```
Hierarchy:
├── Ground (Plane) ← NavMesh Surface
├── Player (Capsule) ← PlayerMovement 스크립트
├── Enemy (Capsule)
│   ├── AI_Agent (스크립트)
│   └── Empty (StateMachine)
├── Waypoints (Empty)
│   ├── Waypoint_1
│   ├── Waypoint_2
│   ├── Waypoint_3
│   └── Waypoint_4
└── Obstacles (여러 Cube)
```

**Waypoint 배치 예시:**

```
Waypoint_1: (-6, 0, -4)
Waypoint_2: (-6, 0,  4)
Waypoint_3: ( 6, 0,  4)
Waypoint_4: ( 6, 0, -4)
```

**Player**는 Step 1의 `PlayerMovement.cs`를 그대로 사용합니다.
**Tag**를 `Player`로 설정하세요.

### Step 3.2 — 상태 인터페이스 정의

`IState.cs`:

```csharp
using UnityEngine;

public interface IState
{
    void Enter();          // 상태 진입 시 1회 호출
    void Update();         // 매 프레임 업데이트
    void FixedUpdate();    // 물리 업데이트 (선택)
    void Exit();           // 상태 이탈 시 1회 호출
}
```

### Step 3.3 — FSM 시스템 (상태 머신)

`StateMachine.cs`:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

public class StateMachine : MonoBehaviour
{
    private Dictionary<Type, IState> states = new();
    private IState currentState;

    // 현재 상태 이름 (디버깅/UI 표시용)
    public string CurrentStateName => currentState?.GetType().Name ?? "None";

    void Update()
    {
        currentState?.Update();
    }

    void FixedUpdate()
    {
        currentState?.FixedUpdate();
    }

    // 상태 등록
    public void RegisterState(IState state)
    {
        states[state.GetType()] = state;
    }

    // 상태 전환
    public void ChangeState<T>() where T : IState
    {
        Type newStateType = typeof(T);
        ChangeState(newStateType);
    }

    public void ChangeState(Type newStateType)
    {
        if (!states.ContainsKey(newStateType))
        {
            Debug.LogWarning($"State {newStateType.Name} is not registered!");
            return;
        }

        // 현재 상태 종료
        currentState?.Exit();

        // 새 상태 시작
        currentState = states[newStateType];
        currentState.Enter();
    }

    // 특정 상태인지 확인
    public bool IsInState<T>() where T : IState
    {
        return currentState is T;
    }
}
```

### Step 3.4 — 개별 상태 구현

**IdleState — 대기 상태**

```csharp
using UnityEngine;

public class IdleState : IState
{
    private readonly EnemyAI owner;
    private readonly float idleDuration = 3f;
    private float timer;

    public IdleState(EnemyAI owner) => this.owner = owner;

    public void Enter()
    {
        timer = 0f;
        owner.Agent.isStopped = true;
        Debug.Log("[FSM] Entered: Idle");
    }

    public void Update()
    {
        timer += Time.deltaTime;

        // Player 감지 → Chase
        if (owner.CanSeePlayer())
        {
            owner.StateMachine.ChangeState<ChaseState>();
            return;
        }

        // 대기 시간 종료 → Patrol
        if (timer >= idleDuration)
        {
            owner.StateMachine.ChangeState<PatrolState>();
        }
    }

    public void FixedUpdate() { }
    public void Exit()
    {
        owner.Agent.isStopped = false;
        Debug.Log("[FSM] Exited: Idle");
    }
}
```

**PatrolState — 순찰 상태**

```csharp
using UnityEngine;

public class PatrolState : IState
{
    private readonly EnemyAI owner;
    private int currentWaypointIndex;

    public PatrolState(EnemyAI owner) => this.owner = owner;

    public void Enter()
    {
        currentWaypointIndex = 0;
        MoveToNextWaypoint();
        Debug.Log("[FSM] Entered: Patrol");
    }

    public void Update()
    {
        // Player 감지 → Chase
        if (owner.CanSeePlayer())
        {
            owner.StateMachine.ChangeState<ChaseState>();
            return;
        }

        // 현재 Waypoint에 도착 → 다음 Waypoint로
        if (!owner.Agent.pathPending && owner.Agent.remainingDistance <= owner.Agent.stoppingDistance)
        {
            MoveToNextWaypoint();
        }
    }

    private void MoveToNextWaypoint()
    {
        if (owner.Waypoints == null || owner.Waypoints.Length == 0) return;

        owner.Agent.SetDestination(owner.Waypoints[currentWaypointIndex].position);
        currentWaypointIndex = (currentWaypointIndex + 1) % owner.Waypoints.Length;
    }

    public void FixedUpdate() { }
    public void Exit()
    {
        Debug.Log("[FSM] Exited: Patrol");
    }
}
```

**ChaseState — 추적 상태**

```csharp
using UnityEngine;

public class ChaseState : IState
{
    private readonly EnemyAI owner;
    private float losePlayerTimer;

    public ChaseState(EnemyAI owner) => this.owner = owner;

    public void Enter()
    {
        losePlayerTimer = 0f;
        owner.Agent.speed = owner.chaseSpeed;
        Debug.Log("[FSM] Entered: Chase");
    }

    public void Update()
    {
        // Player가 시야에 있음
        if (owner.CanSeePlayer())
        {
            losePlayerTimer = 0f;
            owner.Agent.SetDestination(owner.Player.position);

            // 근접 → Attack
            float dist = Vector3.Distance(owner.transform.position, owner.Player.position);
            if (dist <= owner.attackRange)
            {
                owner.StateMachine.ChangeState<AttackState>();
            }
            return;
        }

        // Player 시야 소실 → 타이머 시작
        losePlayerTimer += Time.deltaTime;
        if (losePlayerTimer >= owner.losePlayerTime)
        {
            owner.StateMachine.ChangeState<PatrolState>();
        }
    }

    public void FixedUpdate() { }
    public void Exit()
    {
        Debug.Log("[FSM] Exited: Chase");
    }
}
```

**AttackState — 공격 상태**

```csharp
using UnityEngine;

public class AttackState : IState
{
    private readonly EnemyAI owner;
    private float attackTimer;
    private bool hasAttacked;

    public AttackState(EnemyAI owner) => this.owner = owner;

    public void Enter()
    {
        attackTimer = 0f;
        hasAttacked = false;
        owner.Agent.isStopped = true;
        // 플레이어를 바라봄
        owner.transform.LookAt(owner.Player);
        Debug.Log("[FSM] Entered: Attack");
    }

    public void Update()
    {
        attackTimer += Time.deltaTime;

        // 공격 실행
        if (attackTimer >= owner.attackCooldown && !hasAttacked)
        {
            hasAttacked = true;
            owner.Attack();
        }

        // Player가 범위 밖으로 나감 → Chase
        float dist = Vector3.Distance(owner.transform.position, owner.Player.position);
        if (dist > owner.attackRange * 1.2f)
        {
            owner.StateMachine.ChangeState<ChaseState>();
            return;
        }

        // Player 시야 완전 소실 → Patrol
        if (!owner.CanSeePlayer() && dist > owner.attackRange * 2f)
        {
            owner.StateMachine.ChangeState<PatrolState>();
        }
    }

    public void FixedUpdate() { }
    public void Exit()
    {
        owner.Agent.isStopped = false;
        Debug.Log("[FSM] Exited: Attack");
    }
}
```

### Step 3.5 — EnemyAI 컨트롤러 (통합)

`EnemyAI.cs`:

```csharp
using UnityEngine;
using UnityEngine.AI;

public class EnemyAI : MonoBehaviour
{
    [Header("Components")]
    public NavMeshAgent Agent { get; private set; }
    public StateMachine StateMachine { get; private set; }
    public Transform Player { get; private set; }

    [Header("Movement")]
    public float patrolSpeed = 3.0f;
    public float chaseSpeed = 5.5f;

    [Header("Combat")]
    public float attackRange = 2.0f;
    public float attackCooldown = 1.5f;
    public int attackDamage = 10;

    [Header("Detection")]
    public float detectionRange = 10.0f;
    public float detectionAngle = 60f;     // 시야각
    public float losePlayerTime = 5.0f;    // 시야 소실 후 포기 시간

    [Header("Patrol")]
    public Transform[] Waypoints;

    private void Awake()
    {
        Agent = GetComponent<NavMeshAgent>();
        StateMachine = gameObject.AddComponent<StateMachine>();

        GameObject playerObj = GameObject.FindGameObjectWithTag("Player");
        if (playerObj != null) Player = playerObj.transform;
    }

    private void Start()
    {
        Agent.speed = patrolSpeed;

        // 모든 상태 등록
        StateMachine.RegisterState(new IdleState(this));
        StateMachine.RegisterState(new PatrolState(this));
        StateMachine.RegisterState(new ChaseState(this));
        StateMachine.RegisterState(new AttackState(this));

        // 초기 상태: Patrol
        StateMachine.ChangeState<PatrolState>();
    }

    // 플레이어 감지 (시야 + 거리)
    public bool CanSeePlayer()
    {
        if (Player == null) return false;

        Vector3 directionToPlayer = Player.position - transform.position;
        float distance = directionToPlayer.magnitude;

        // 거리 체크
        if (distance > detectionRange) return false;

        // 각도 체크 (시야각)
        float angle = Vector3.Angle(transform.forward, directionToPlayer.normalized);
        if (angle > detectionAngle * 0.5f) return false;

        // 시야 차단 확인 (Raycast)
        if (Physics.Raycast(transform.position + Vector3.up * 0.5f,
            directionToPlayer.normalized, out RaycastHit hit, detectionRange))
        {
            return hit.transform.CompareTag("Player");
        }

        return false;
    }

    // 공격 실행
    public void Attack()
    {
        Debug.Log($"[Combat] Attacked player for {attackDamage} damage!");

        // 실제 데미지 처리는 Player 스크립트와 연동
        if (Player.TryGetComponent(out PlayerHealth health))
        {
            health.TakeDamage(attackDamage);
        }
    }

    // 디버그 시각화
    private void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(transform.position, detectionRange);

        // 시야각 표시
        Vector3 forward = transform.forward * detectionRange;
        Vector3 leftBoundary = Quaternion.Euler(0, -detectionAngle * 0.5f, 0) * forward;
        Vector3 rightBoundary = Quaternion.Euler(0, detectionAngle * 0.5f, 0) * forward;
        Gizmos.color = Color.cyan;
        Gizmos.DrawRay(transform.position + Vector3.up * 0.5f, leftBoundary);
        Gizmos.DrawRay(transform.position + Vector3.up * 0.5f, rightBoundary);
    }
}
```

### Step 3.6 — PlayerHealth 스크립트 (공격 연동)

`PlayerHealth.cs`:

```csharp
using UnityEngine;

public class PlayerHealth : MonoBehaviour
{
    [SerializeField] private int maxHealth = 100;
    private int currentHealth;

    void Start()
    {
        currentHealth = maxHealth;
    }

    public void TakeDamage(int damage)
    {
        currentHealth -= damage;
        Debug.Log($"Player HP: {currentHealth}/{maxHealth}");

        if (currentHealth <= 0)
        {
            Die();
        }
    }

    private void Die()
    {
        Debug.Log("Player Died!");
        // 게임 오버 처리 (리스폰, 씬 로드 등)
        gameObject.SetActive(false);
    }
}
```

### Step 3.7 — Enemy 프리팹 설정

1. Enemy 오브젝트에 다음 컴포넌트 추가:
   - `NavMesh Agent` (Step 1 설정 참고)
   - `StateMachine` (자동 추가됨)
   - `EnemyAI` 스크립트
2. EnemyAI Inspector에서 세부 설정:
   - **Patrol Speed**: `3.0`
   - **Chase Speed**: `5.5`
   - **Detection Range**: `10.0`
   - **Detection Angle**: `60`
   - **Attack Range**: `2.0`
   - **Attack Cooldown**: `1.5`
   - **Lose Player Time**: `5.0`
   - **Waypoints**: 배열 크기 4로 설정 → 각 Waypoint 할당
3. Enemy 오브젝트를 **Project 창**으로 드래그하여 Prefab 생성

---

## 3. FSM 동작 흐름

### 전체 상태 전환도

```
[시작]
   │
   ▼
┌───────────┐
│  Patrol   │ ◄────────────────────────────┐
└─────┬─────┘                              │
      │ Player 발견 (CanSeePlayer)          │
      ▼                                     │
┌───────────┐     범위 이탈 (1.5배)     ┌────┴─────┐
│  Chase    │ ──────────────────────────► │  Attack  │
└─────┬─────┘                             └──────────┘
      │ Player 시야 소실 (5초)
      ▼
┌───────────┐
│   Idle    │ (3초 후)
└───────────┘
      │
      ▼
┌───────────┐
│  Patrol   │
└───────────┘
```

### 각 상태의 책임

| 상태 | 진입 조건 | 주요 행동 | 이탈 조건 |
|------|----------|-----------|-----------|
| **Idle** | Chase 실패 후 | 정지, 주시 | Player 발견 or 타이머 만료 |
| **Patrol** | Idle 종료 or Attack 종료 | Waypoint 이동 | Player 발견 |
| **Chase** | Player 발견 | Player 추적 | Player 근접 or 시야 소실 |
| **Attack** | Chase 중 근접 | 공격, 회전 | Player 이탈 |

---

## 4. 확장 설계: 상태에 데이터 추가

각 상태에 추가 데이터를 전달해야 한다면:

```csharp
// 상태 진입 시 데이터 전달
public interface IState
{
    void Enter(Dictionary<string, object> parameters = null);
    // ...
}

// 사용 예
var parameters = new Dictionary<string, object>
{
    { "alertLevel", 3 },
    { "lastKnownPosition", playerPosition }
};
stateMachine.ChangeState<AlertState>(parameters);
```

---

## 5. 디버깅 팁

### 현재 상태 UI 표시

```csharp
using UnityEngine;

public class StateDebugUI : MonoBehaviour
{
    private EnemyAI enemy;

    void Start() => enemy = GetComponent<EnemyAI>();

    void OnGUI()
    {
        Vector3 screenPos = Camera.main.WorldToScreenPoint(
            transform.position + Vector3.up * 3f
        );

        GUI.color = Color.white;
        GUI.Label(
            new Rect(screenPos.x - 50, Screen.height - screenPos.y - 15, 100, 30),
            $"State: {enemy.StateMachine.CurrentStateName}"
        );
    }
}
```

---

## 6. 확장 과제

1. **회피 상태 (FleeState) 추가**: Player가 강하면 도망가는 상태 구현
2. **호출 상태 (AlertState) 추가**: Player 발견 시 주변 Enemy를 호출
3. **의심 상태 (SuspiciousState) 추가**: 미확인 소리/움직임에 반응
4. **계층적 FSM (HFSM)**: Patrol 내부에 Idle → Move → Wait 서브 상태 추가

---

## 7. 문제 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| 상태가 전환되지 않음 | StateMachine에 상태 미등록 | `RegisterState` 호출 확인 |
| CanSeePlayer가 항상 false | Player 태그 미설정 | Player Tag를 "Player"로 설정 |
| Agent가 Waypoint로 이동 안 함 | Waypoints 배열이 비어 있음 | Inspector에서 Waypoint 할당 확인 |
| 공격이 연타됨 | Attack 쿨다운 미적용 | `hasAttacked` 플래그 확인 |

---

**→ 다음 단계: [Step 4: Behavior Tree & 센서 시스템](./Step4_BehaviorTree_Sensors.md)**
