# Step 1: NavMesh 기초 — 따라오는 AI

> **난이도**: ⭐ 초급  
> **목표**: NavMesh 시스템을 이해하고, 플레이어를 따라오는 기본 AI 캐릭터 구현  
> **소요 시간**: 30~40분

---

## 1. 개념 설명

### NavMesh (Navigation Mesh)란?

NavMesh는 3D 공간에서 AI 캐릭터가 **이동 가능한 영역**을 나타내는 데이터 구조입니다.
씬의 지오메트리를 분석하여 **걸어갈 수 있는 표면**을 파란색 오버레이로 표시합니다.

### Unity 6 AI Navigation 2.0 시스템

Unity 6(6000.4)부터는 **AI Navigation 2.0** 패키지가 표준으로 제공됩니다.
이전 버전과의 주요 차이점:

| 항목 | Unity 2022 이전 | Unity 6 (AI Navigation 2.0) |
|------|----------------|-----------------------------|
| NavMesh 생성 | Navigation Window에서 Bake | `NavMesh Surface` 컴포넌트로 Bake |
| Bake 데이터 | 씬에 저장 (NavMesh 데이터) | NavMesh Surface 컴포넌트가 관리 |
| 런타임 Bake | 제한적 지원 | 지원 (`Update()` 메서드) |
| 다중 표면 | 단일 NavMesh만 가능 | 여러 NavMesh Surface 사용 가능 |

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| `NavMesh Surface` | 씬 지오메트리로부터 NavMesh 생성 (Bake) |
| `NavMesh Agent` | AI 캐릭터에 부착하여 NavMesh 위를 이동 |
| `NavMesh Obstacle` | 동적 장애물을 NavMesh에 반영 |
| `NavMesh Link` | 끊긴 NavMesh 사이를 연결 (점프, 계단) |

### NavMesh Agent 주요 프로퍼티

| 프로퍼티 | 설명 | 기본값 |
|----------|------|--------|
| `Speed` | 이동 최대 속도 | 3.5 |
| `Angular Speed` | 회전 속도 (deg/s) | 120 |
| `Acceleration` | 가속도 | 8 |
| `Stopping Distance` | 목표 도착 판정 거리 | 0 |
| `Radius` | 에이전트 반경 (좁은 통로 통과 여부 결정) | 0.5 |
| `Height` | 에이전트 높이 | 2 |
| `Auto Braking` | 목표 지점에서 자동 감속 | true |

---

## 2. 실습: 플레이어를 따라오는 AI

### Step 1.1 — 프로젝트 설정

1. Unity Hub를 열고 **New Project** 클릭
2. 템플릿: **3D (Built-In Render Pipeline)** 선택
3. 프로젝트 이름: `Unity_AI_Step1`
4. 위치: 적절한 위치 선택 후 **Create**
5. **Window > Package Manager** → **Unity Registry** → `AI Navigation` 검색 후 **Install**

### Step 1.2 — 기본 씬 구성

바닥과 장애물을 배치합니다:

```csharp
// Hierarchy 구성:
// - Directional Light (기본)
// - Ground (Plane)      ← 바닥
// - Wall (Cube)         ← 장애물 (여러 개 복제)
// - Player (Capsule)    ← 플레이어 (우리가 조종)
// - Enemy (Capsule)     ← AI (플레이어 추적)
```

**구체적인 설정:**

1. **Ground**: `GameObject > 3D Object > Plane` → 이름을 `Ground`로 변경  
   - Transform Position: (0, 0, 0), Scale: (2, 1, 2)

2. **Wall**: `GameObject > 3D Object > Cube` → 이름을 `Wall`로 변경  
   - Transform Position: (3, 0.5, 0), Scale: (0.5, 1, 3)  
   - 복제하여 여러 장애물 배치

3. **Player**: `GameObject > 3D Object > Capsule` → 이름을 `Player`로 변경  
   - Transform Position: (-5, 0.5, 0), Scale: (1, 1, 1)  
   - 카메라가 Player를 따라다니도록 설정 (선택)

4. **Enemy**: `GameObject > 3D Object > Capsule` → 이름을 `Enemy`로 변경  
   - Transform Position: (5, 0.5, 3), Scale: (1, 1, 1)  
   - 색상 구분을 위해 Material 변경 (빨간색 계열 권장)

### Step 1.3 — NavMesh Surface Bake

1. **Ground** 오브젝트 선택
2. **Inspector > Add Component** → `NavMesh Surface` 검색 후 추가
3. 설정 확인:
   - **Agent Type**: `Humanoid` (기본값)
   - **Default Area**: `Walkable`
   - **Include Layers**: `Everything`
4. **Bake** 버튼 클릭
5. 씬 뷰에 파란색 오버레이가 나타나는지 확인
   - 보이지 않으면 씬 뷰 툴바에서 **AI Navigation** 오버레이 버튼 클릭 후 `Show NavMesh Surface` 활성화
   - Gizmos 토글도 켜져 있는지 확인

> ⚠️ **문제 해결**: 바닥에 NavMesh Surface를 추가했는데 Bake가 안 되면, 바닥의 Mesh Renderer가 활성화되어 있는지 확인하고, Static 플래그가 설정되어 있는지 확인하세요.

### Step 1.4 — Enemy에 NavMesh Agent 추가

1. **Enemy** 오브젝트 선택
2. **Inspector > Add Component** → `NavMesh Agent` 추가
3. Agent 설정:
   - **Speed**: `5.0`
   - **Angular Speed**: `120`
   - **Acceleration**: `10`
   - **Stopping Distance**: `1.5` (Player 근처에서 멈춤)
   - **Radius**: `0.5`
   - **Height**: `2.0`

### Step 1.5 — Player 이동 스크립트

Player가 키보드로 움직이도록 스크립트를 만듭니다.

`PlayerMovement.cs`:

```csharp
using UnityEngine;

public class PlayerMovement : MonoBehaviour
{
    public float speed = 5.0f;

    void Update()
    {
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");

        Vector3 direction = new Vector3(h, 0, v).normalized;
        if (direction.magnitude > 0.1f)
        {
            transform.Translate(direction * speed * Time.deltaTime, Space.World);
            transform.rotation = Quaternion.LookRotation(direction);
        }
    }
}
```

1. **Player** 오브젝트 선택
2. **Add Component** → `New Script` → 이름: `PlayerMovement`
3. 위 코드 붙여넣기

### Step 1.6 — Enemy AI 스크립트 (플레이어 추적)

`EnemyAI.cs`:

```csharp
using UnityEngine;
using UnityEngine.AI;

public class EnemyAI : MonoBehaviour
{
    [SerializeField] private Transform target;       // 추적할 목표 (Player)
    [SerializeField] private float updateInterval = 0.2f; // 경로 재계산 간격

    private NavMeshAgent agent;
    private float timer;

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();

        // target이 설정되지 않은 경우 Player 태그로 자동 찾기
        if (target == null)
        {
            GameObject player = GameObject.FindGameObjectWithTag("Player");
            if (player != null) target = player.transform;
        }
    }

    void Update()
    {
        timer += Time.deltaTime;
        if (timer >= updateInterval && target != null)
        {
            timer = 0f;
            agent.SetDestination(target.position);
        }
    }

    // Gizmos로 시각화 (에디터에서만 표시)
    void OnDrawGizmosSelected()
    {
        if (agent != null && agent.hasPath)
        {
            Gizmos.color = Color.red;
            for (int i = 0; i < agent.path.corners.Length - 1; i++)
            {
                Gizmos.DrawLine(agent.path.corners[i], agent.path.corners[i + 1]);
            }
        }
    }
}
```

1. **Enemy** 오브젝트에 스크립트 추가
2. **Player** 오브젝트의 Tag를 `Player`로 설정 (Inspector 상단)
3. 실행하여 Player가 움직일 때 Enemy가 따라오는지 확인

---

## 3. 코드 상세 해설

### `NavMeshAgent.SetDestination()`

```csharp
agent.SetDestination(target.position);
```

가장 핵심적인 메서드입니다. 내부적으로:
1. 현재 위치에서 목적지까지의 경로를 NavMesh 데이터 위에서 계산
2. 장애물을 우회하는 최적 경로 찾기
3. 에이전트의 속도/가속도 설정에 따라 부드럽게 이동

### Update 간격 제한 (`updateInterval`)

매 프레임마다 SetDestination을 호출하면 성능에 부담이 될 수 있습니다.
`0.2초` 간격으로 호출하여 최적화했습니다.

### NavMesh 경로 시각화 (`OnDrawGizmosSelected`)

`agent.path.corners` 배열을 통해 계산된 경로의 웨이포인트를 확인할 수 있습니다.
에디터에서 Enemy를 선택하면 빨간색 선으로 경로가 표시됩니다.

---

## 4. 핵심 개념 정리

| 개념 | 설명 |
|------|------|
| **NavMesh Baking** | 씬 지오메트리를 분석해 이동 가능 영역을 미리 계산 |
| **Agent** | NavMesh 위를 이동하는 캐릭터 (물리 충돌 아님, NavMesh 기반 이동) |
| **SetDestination()** | AI 목표 지점 설정, 자동 경로 탐색 및 이동 |
| **Stopping Distance** | 목표 도착 판정 거리 (0에 가까울수록 정확) |
| **Agent Radius** | 좁은 통로 통과 가능 여부 결정 |

### NavMesh vs 물리 충돌

| 구분 | NavMesh Agent | Rigidbody + Physics |
|------|--------------|---------------------|
| 이동 방식 | NavMesh 데이터 기반 경로 탐색 | 물리 시뮬레이션 |
| 장애물 회피 | 자동 (NavMesh에 없는 영역으로 이동 불가) | 수동 구현 필요 |
| 성능 | 매우 가벼움 | 비교적 무거움 |
| 용도 | AI 캐릭터 길찾기 | 플레이어 조종, 물리 효과 |

---

## 5. 확장 과제

1. **여러 Enemy 만들기**: Enemy를 프리팹(Prefab)으로 만들고 3~5개씩 배치
2. **속도 다양화**: Enemy마다 Speed를 다르게 설정하여 다양한 움직임 연출
3. **추적 사운드**: Enemy가 Player 근처(Stopping Distance 이하)에 도착하면 경고음 재생
4. **NavMesh Surface 적용 범위**: 바닥 일부를 `Not Walkable`로 설정하여 이동 불가 영역 만들기

---

## 6. 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| Enemy가 움직이지 않음 | NavMesh Surface가 Bake되지 않음 | Bake 버튼 누르고 파란색 오버레이 확인 |
| Enemy가 벽 통과 | Agent Radius가 너무 작음 | Radius를 0.3~0.5로 설정 |
| Enemy가 Player에 딱 붙음 | Stopping Distance = 0 | 1.0~2.0으로 설정 |
| 경로가 이상함 | 바닥 지오메트리에 문제 | Ground의 Mesh Renderer 및 Static 설정 확인 |

---

**→ 다음 단계: [Step 2: NavMesh 심화 — 장애물과 링크](./Step2_NavMesh_Advanced.md)**
