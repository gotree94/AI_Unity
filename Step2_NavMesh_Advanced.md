# Step 2: NavMesh 심화 — 장애물과 링크

> **난이도**: ⭐⭐ 중급  
> **목표**: 동적 장애물 처리, NavMesh Link를 이용한 점프, 다중 에이전트 타입 이해  
> **소요 시간**: 40~50분  
> **선행 조건**: Step 1 완료

---

## 1. 개념 설명

### NavMesh Obstacle (동적 장애물)

정적 장애물(벽, 바위)은 **Bake 단계**에서 NavMesh에 반영됩니다.
하지만 문, 떨어지는 상자 등 **움직이거나 생성/제거되는 오브젝트**는
런타임에 NavMesh에 반영해야 합니다.

**Obstacle의 두 가지 모드:**

| 모드 | 동작 방식 | 성능 | 사용 예 |
|------|-----------|------|---------|
| **Carve** | 장애물이 이동하면 NavMesh를 실시간으로 깎아냄 | 중간 | 움직이는 문, 엘리베이터 |
| **Collide** | Agent와 물리 충돌만 처리 | 가벼움 | 임시 장애물 |

> **Carve 모드**는 이동할 때마다 NavMesh를 다시 계산하므로,
> 성능에 주의해야 합니다. 많은 수의 Carve Obstacle은 피하세요.

### NavMesh Link (연결)

NavMesh Link는 **NavMesh가 끊긴 영역 사이를 연결**합니다.
사용 예:
- 참호 위를 점프
- 계단 오르내리기
- 플랫폼 사이 이동
- 문턱 넘기

**Link의 두 가지 타입:**

| 타입 | 사용법 |
|------|--------|
| **수동(Manual)** | 두 개의 Transform을 지정하여 연결 |
| **범위(Scope)** | 시작점과 방향으로 연결 영역 자동 생성 |

### 다중 Agent 타입

프로젝트 설정에서 여러 Agent 타입을 정의할 수 있습니다:

| Agent 타입 | Radius | Height | Step Height | Max Slope | 용도 |
|-----------|--------|--------|-------------|-----------|------|
| Humanoid | 0.5 | 2.0 | 0.4 | 45° | 인간형 캐릭터 |
| Stubby | 0.4 | 0.8 | 1.0 | 60° | 작은 생물체 |
| Vehicle | 1.0 | 1.5 | 0.2 | 30° | 탈것 |

각 타입은 **자신만의 NavMesh Surface**가 필요합니다.

---

## 2. 실습: 동적 장애물과 점프

### Step 2.1 — 프로젝트 준비

1. Step 1 프로젝트를 복제하거나 새 프로젝트 생성
2. AI Navigation 패키지 설치 확인
3. 기본 바닥, 장애물, Player, Enemy 설정 (Step 1과 동일)

### Step 2.2 — 움직이는 장애물 (NavMesh Obstacle)

움직이는 문을 만들어 AI가 기다리게 해봅시다.

**씬 구성:**

```
Hierarchy:
├── Ground
├── MovingDoor (Cube)      ← Carve 모드 Obstacle
├── Player
└── Enemy
```

**MovingDoor 설정:**

1. `GameObject > 3D Object > Cube` → 이름 `MovingDoor`
2. Transform Position: (0, 0.5, 0), Scale: (0.3, 1, 2)
3. `NavMesh Obstacle` 컴포넌트 추가
4. 설정:
   - **Carve**: ✅ 체크
   - **Move Threshold**: `0.1` (이동 임계값)
   - **Carve Only Stationary**: 🔓 해제 (움직이는 동안에도 Carve)
   - **Time To Stationary**: `0.5` (정지 후 Carve 안정화 시간)

**MovingDoor 스크립트:**

`MovingDoor.cs`:

```csharp
using UnityEngine;

public class MovingDoor : MonoBehaviour
{
    [SerializeField] private Vector3 offset = new Vector3(3, 0, 0);
    [SerializeField] private float speed = 1.5f;

    private Vector3 startPos;
    private Vector3 endPos;

    void Start()
    {
        startPos = transform.position;
        endPos = startPos + offset;
    }

    void Update()
    {
        // PingPong으로 왕복 운동
        float t = Mathf.PingPong(Time.time * speed, 1f);
        transform.position = Vector3.Lerp(startPos, endPos, t);
    }

    // Gizmos로 이동 범위 표시
    void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.yellow;
        Vector3 start = Application.isPlaying ? startPos : transform.position;
        Gizmos.DrawWireCube(start + offset, GetComponent<Collider>().bounds.size);
    }
}
```

**동작 확인**:
- 문이 열려 있을 때: Enemy가 통과
- 문이 닫혀 있을 때: Enemy가 앞에서 대기
- 문이 다시 열리면: Enemy가 경로를 재계산하여 통과

### Step 2.3 — NavMesh Link로 점프 구현

두 개의 플랫폼 사이를 연결하는 Link를 만듭니다.

**씬 구성:**

```
Hierarchy:
├── Ground
├── PlatformA (Cube)       ← NavMesh Surface (각각 Bake)
├── PlatformB (Cube)       ← NavMesh Surface (각각 Bake)
├── Link_Jump (NavMesh Link) ← 두 플랫폼 연결
├── Player
└── Enemy
```

**플랫폼 설정:**

1. `PlatformA`: Position (-4, 0.5, 0), Scale (2, 1, 2)
2. `PlatformB`: Position (4, 1.5, 0), Scale (2, 1, 2) ← 높이 다름!

**각 플랫폼에 NavMesh Surface 추가:**

1. `PlatformA` 선택 → `NavMesh Surface` 추가
2. `PlatformB` 선택 → `NavMesh Surface` 추가
3. 각각 **Bake** 버튼 클릭

> ⚠️ 플랫폼이 Ground와 분리되어 있으므로 각각 Bake가 필요합니다.

**NavMesh Link 생성:**

1. `GameObject > AI > NavMesh Link` 생성
2. 이름을 `JumpLink`로 변경
3. NavMesh Link 컴포넌트 설정:
   - **Agent Type**: `Humanoid`
   - **Start Transform**: `PlatformA` 할당
   - **End Transform**: `PlatformB` 할당
   - **Cost Override**: `-1` (기본 비용 사용)
   - **Bidirectional**: ✅ (양방향)
   - **Activated**: ✅
   - **Auto Update Position**: ✅

**Agent 설정 (점프 가능하도록):**

Enemy의 NavMesh Agent에서:
- **Auto Traverse Off Mesh Link**: ✅ (기본 활성화)

이제 Enemy가 PlatformA에서 PlatformB로 자동으로 점프/이동합니다!

### Step 2.4 — 링크 커스터마이징

NavMesh Link 통과 방식을 제어하는 스크립트:

`LinkAnimator.cs`:

```csharp
using UnityEngine;
using UnityEngine.AI;

public class LinkAnimator : MonoBehaviour
{
    private NavMeshAgent agent;
    private Animator animator;

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();
        animator = GetComponent<Animator>();
    }

    void Update()
    {
        // OffMeshLink 위에 있을 때
        if (agent.isOnOffMeshLink)
        {
            if (animator != null)
                animator.SetBool("isJumping", true);
        }
        else
        {
            if (animator != null)
                animator.SetBool("isJumping", false);
        }
    }
}
```

---

## 3. 코드 상세 해설

### NavMesh Obstacle 동작 원리

1. `Carve` 모드가 활성화되면 Obstacle은 자신의 형상만큼 NavMesh에서 영역을 제거
2. `Move Threshold` 이상 이동 시 NavMesh 재계산 트리거
3. `Carve Only Stationary`가 켜져 있으면 이동 중에는 Carve하지 않음 (성능 최적화)
4. Agent는 자동으로 Carve된 영역을 회피하여 경로 탐색

### NavMesh Link 동작 흐름

1. Agent가 Link 시작점에 도달
2. `Auto Traverse Off Mesh Link`가 활성화되어 있으면 자동으로 Link 사용
3. Agent가 Link 위를 이동하는 동안 `agent.isOnOffMeshLink`가 true
4. Link 종료점에 도달하면 다시 일반 NavMesh 이동으로 전환

### Cost Override 이해하기

- `-1`: 유형 기본 비용 사용
- `0`: 무조건 이 Link를 선호
- 높은 값: 가능한 다른 경로 우선

---

## 4. Agent Type 커스터마이징

**프로젝트 설정 > Agents 탭:**

1. **Window > AI > Navigation** (또는 **Project Settings > AI > Agents**)
2. 기존 `Humanoid` 외에 새 Agent Type 추가 가능
3. 커스텀 Agent Type을 만들면 각 Type별로 NavMesh Surface를 별도 Bake 필요

```csharp
// 스크립트에서 Agent Type 변경
using UnityEngine.AI;

public class AgentSwitcher : MonoBehaviour
{
    public void SwitchAgentType(string agentTypeName)
    {
        NavMeshAgent agent = GetComponent<NavMeshAgent>();
        agent.agentTypeID = NavMesh.GetSettingsByIndex(
            NavMesh.GetSettingsCount() - 1
        ).agentTypeID;
    }
}
```

---

## 5. 성능 고려사항

| 요소 | 영향 | 최적화 방법 |
|------|------|------------|
| Carving Obstacle 수 | 많을수록 CPU 부하↑ | 10개 이하 유지, 가능하면 Collider 모드 사용 |
| Link 수 | 많아도 무방 (미리 계산) | 제한 없음 |
| 런타임 NavMesh 변경 | 자주 변경 시 성능 저하 | 변경 간격 최소화 |
| 다중 Agent Type | 타입마다 별도 NavMesh | 꼭 필요한 타입만 정의 |

---

## 6. 확장 과제

1. **떨어지는 장애물**: 일정 시간 후 낙하하는 Obstacle 만들기 (Carve 모드)
2. **지능형 문**: AI가 접근하면 자동으로 열리는 문 (Trigger + Obstacle 비활성화)
3. **스파이더 Agent**: Radius 0.2, Step Height 2.0의 작은 Agent 타입 만들어보기
4. **순간이동 링크**: NavMesh Link 대신 코루틴으로 순간이동 효과 구현

---

## 7. 문제 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| Obstacle이 NavMesh를 못 깎음 | Carve 체크 안 됨 | Carve 모드 활성화 확인 |
| Link가 작동 안 함 | 시작/끝 지점이 NavMesh 위에 없음 | 두 지점이 모두 NavMesh 위에 있는지 확인 |
| 점프가 너무 느림 | Agent 속도가 느림 | Speed 또는 Acceleration 증가 |
| Agent가 Link를 무시함 | Cost Override가 너무 높음 | -1 또는 0으로 설정 |

---

**→ 다음 단계: [Step 3: 유한 상태 머신 — 지능형 NPC 행동](./Step3_Finite_State_Machine.md)**
