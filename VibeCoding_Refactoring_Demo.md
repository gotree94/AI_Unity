# 🎬 Unity AI 바이브코딩 & 리팩토링 실전 데모

> **참고 영상**: [Unity AI와 함께하는 리팩토링 생산성 혁신](https://youtu.be/6Ik94spWREI)  
> **대상**: Unity 개발자, 게임 프로그래머, AI 기반 개발 워크플로에 관심 있는 분  
> **핵심 메시지**: 단순 코드 자동 생성이 아니라, **실제 개발 흐름 안에서 AI를 활용해 생산성을 높이고 리팩토링 판단을 돕는 방법**

---

## 데모 개요

이 데모는 Unity AI Assistant의 **Ask → Plan → Agent** 3단계 모드를 활용하여
**실제 프로젝트의 코드를 분석하고, 문제를 파악하고, 더 나은 구조로 리팩토링**하는 과정을 보여줍니다.

### 시나리오

"레거시 RPG 프로젝트의 `PlayerController`가 너무 비대해졌다.
이동, 체력, 인벤토리, UI 업데이트까지 모두 한 스크립트에서 처리하고 있다.
Unity AI Assistant와 함께 리팩토링하여 책임을 분리하자."

---

## 1단계: 문제 진단 (Ask 모드) — 코드 분석

### 🎯 상황

기존 `PlayerController.cs`가 **800줄**에 달합니다.
이 스크립트 하나가 너무 많은 일을 하고 있습니다.

### 🧪 Before 코드 (일부 발췌)

```csharp
using UnityEngine;
using UnityEngine.UI;

public class PlayerController : MonoBehaviour
{
    // ===== Movement =====
    public float moveSpeed = 5f;
    public float jumpForce = 10f;
    private Rigidbody rb;
    private bool isGrounded;

    // ===== Health =====
    public float maxHealth = 100f;
    private float currentHealth;
    public Image healthBar;
    public GameObject deathEffect;

    // ===== Inventory =====
    public int gold;
    public int potions;
    public Text goldText;
    public Text potionsText;

    // ===== Combat =====
    public int attackDamage = 20;
    public float attackRange = 2f;
    public LayerMask enemyLayer;

    void Start()
    {
        rb = GetComponent<Rigidbody>();
        currentHealth = maxHealth;
        UpdateUI();
    }

    void Update()
    {
        // Movement
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");
        Vector3 move = new Vector3(h, 0, v) * moveSpeed * Time.deltaTime;
        transform.Translate(move, Space.World);

        if (Input.GetKeyDown(KeyCode.Space) && isGrounded)
        {
            rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
            isGrounded = false;
        }

        // Combat
        if (Input.GetMouseButtonDown(0))
        {
            Attack();
        }

        // Inventory UI update (매 프레임 갱신 — 불필요)
        UpdateUI();
    }

    void Attack()
    {
        Collider[] hits = Physics.OverlapSphere(transform.position, attackRange, enemyLayer);
        foreach (var hit in hits)
        {
            // 데미지 처리 로직...
            Debug.Log($"Attacked {hit.name} for {attackDamage} damage");
        }
    }

    public void TakeDamage(float damage)
    {
        currentHealth -= damage;
        healthBar.fillAmount = currentHealth / maxHealth;

        if (currentHealth <= 0)
        {
            Die();
        }
    }

    void Die()
    {
        Instantiate(deathEffect, transform.position, Quaternion.identity);
        gameObject.SetActive(false);
    }

    void UpdateUI()
    {
        goldText.text = $"Gold: {gold}";
        potionsText.text = $"Potions: {potions}";
    }

    public void AddGold(int amount)
    {
        gold += amount;
    }

    public void UsePotion()
    {
        if (potions > 0)
        {
            potions--;
            currentHealth = Mathf.Min(maxHealth, currentHealth + 30);
            healthBar.fillAmount = currentHealth / maxHealth;
        }
    }

    void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Ground"))
            isGrounded = true;
    }
}
```

### 🔍 Unity AI Assistant Ask 모드 — 코드 분석 요청

```
[Ask 모드]
User> "이 PlayerController 스크립트를 분석해줘.
      어떤 문제점들이 보이는지 알려줘."
```

#### Assistant 응답 (예시)

```
분석 결과, 다음과 같은 문제점들이 있습니다:

1. 🔴 SRP (단일 책임 원칙) 위반
   - 이동, 체력, 인벤토리, 전투, UI 갱신을 모두 한 클래스가 처리
   - 각 책임을 별도 컴포넌트로 분리 필요

2. 🟡 Update()에서 매 프레임 UpdateUI() 호출
   - UI 텍스트는 값이 변경될 때만 갱신해도 충분
   - GC 부하 없는 간단한 할당이지만, 불필요한 호출

3. 🟡 Public 필드 과다 노출
   - gold, potions 등이 public 필드로 직접 노출
   - 캡슐화를 위해 property나 메서드로 보호 필요

4. 🔴 하드코딩된 입력 처리
   - Input.GetAxis 등이 직접 하드코딩
   - Input System을 사용하면 리매핑/컨트롤러 지원에 유리

5. 🟡 매직 넘버
   - healthBar.fillAmount = currentHealth / maxHealth (0~1 정규화)
   - 점프 힘, 공격 데미지 등이 하드코딩
```

### 💡 핵심 인사이트

Ask 모드는 코드를 **수정하지 않고 분석만** 합니다.
개발자는 AI의 분석을 보고 **리팩토링 방향을 결정**합니다.

> AI가 발견한 문제를 보고 "아, 이걸 고쳐야겠다"라고 판단하는 것은 **개발자의 몫**입니다.
> AI는 도구일 뿐, 최종 결정은 사람이 내립니다.

---

## 2단계: 리팩토링 계획 수립 (Plan 모드)

### 🎯 상황

Ask 모드의 분석을 바탕으로, 본격적인 리팩토링 계획을 수립합니다.

```
[Plan 모드]
User> "PlayerController를 다음 기준으로 리팩토링할 계획을 세워줘.

      Current state:
      - PlayerController 하나에 이동/체력/인벤토리/전투/UI가 모두 있음
      - 800줄의 모노리스 스크립트

      Desired outcome:
      - 책임별로 컴포넌트 분리
      - 각 컴포넌트는 하나의 책임만 가짐
      - 기존 기능은 모두 유지 (회귀 없음)

      Technical details:
      - Unity 6, URP 프로젝트
      - 기존 Player GameObject에 컴포넌트로 추가 가능해야 함
      - MonoBehaviour 상속
      - SerializeField로 Inspector 노출
      - 값 변경 시에만 UI 갱신 (Event 방식)
```

#### Assistant Plan 모드 응답 (예시)

```
📋 리팩토링 계획

Phase 1: 컴포넌트 분리 구조 설계
------------------------------------
Player GameObject 구성 (예정):
├── PlayerController (기존) → 제거
├── PlayerMovement     (이동 담당)
├── PlayerHealth       (체력/사망 담당)
├── PlayerInventory    (인벤토리 담당)
├── PlayerCombat       (전투 담당)
└── PlayerUI           (UI 갱신 담당, 다른 컴포넌트 구독)

Phase 2: 데이터 흐름 설계
------------------------------------
- 컴포넌트 간 통신은 C# 이벤트 사용
- UI 컴포넌트는 다른 컴포넌트의 이벤트를 구독
- Manager/Controller 클래스는 최소화

Phase 3: 파일 구조
------------------------------------
Scripts/
├── Player/
│   ├── PlayerMovement.cs
│   ├── PlayerHealth.cs
│   ├── PlayerInventory.cs
│   ├── PlayerCombat.cs
│   └── PlayerUI.cs
└── Interfaces/
    └── IDamageable.cs (선택)

[이 계획을 승인하시겠습니까?]
→ 승인 시 Agent 모드로 전환하여 자동 구현합니다.
```

### 💡 핵심 인사이트

Plan 모드는 **구조화된 구현 계획**을 생성합니다.
- 현재 상태 → 목표 상태 → 기술적 제약 조건을 명확히 전달
- AI가 제안한 계획을 검토하고 수정 가능
- 승인 시 자동으로 Agent 모드로 전환

---

## 3단계: 리팩토링 실행 (Agent 모드)

### 🎯 상황

Plan 모드에서 수립한 계획을 Agent 모드로 실행합니다.
AI가 직접 파일을 생성하고 코드를 작성합니다.

### 실행 예시

```
[Agent 모드 — 자동 실행]

User> "Phase 1 계획을 실행해줘.
      먼저 PlayerMovement 컴포넌트를 만들어줘."
```

#### Assistant가 생성하는 코드:

**`PlayerMovement.cs`** (이동 책임 분리):

```csharp
using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class PlayerMovement : MonoBehaviour
{
    [Header("Movement Settings")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private float jumpForce = 10f;
    [SerializeField] private LayerMask groundLayer = ~0;

    private Rigidbody rb;
    private bool isGrounded;

    public Vector3 MoveDirection { get; private set; }
    public bool IsJumping { get; private set; }

    void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }

    void Update()
    {
        HandleInput();
    }

    void FixedUpdate()
    {
        ApplyMovement();
    }

    private void HandleInput()
    {
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");
        MoveDirection = new Vector3(h, 0, v).normalized;

        if (Input.GetKeyDown(KeyCode.Space) && isGrounded)
        {
            IsJumping = true;
        }
    }

    private void ApplyMovement()
    {
        // 이동
        Vector3 move = MoveDirection * moveSpeed * Time.fixedDeltaTime;
        rb.MovePosition(rb.position + move);

        // 점프
        if (IsJumping)
        {
            rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
            isGrounded = false;
            IsJumping = false;
        }
    }

    void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.layer == LayerMask.NameToLayer("Ground"))
            isGrounded = true;
    }

    void OnCollisionExit(Collision collision)
    {
        if (collision.gameObject.layer == LayerMask.NameToLayer("Ground"))
            isGrounded = false;
    }
}
```

**`PlayerHealth.cs`** (체력 책임 분리):

```csharp
using UnityEngine;
using System;

public class PlayerHealth : MonoBehaviour
{
    [Header("Health")]
    [SerializeField] private float maxHealth = 100f;
    [SerializeField] private GameObject deathEffect;

    private float currentHealth;

    // 이벤트: UI 등에서 구독
    public event Action<float> OnHealthChanged;  // 인자: (current/max) 비율
    public event Action OnPlayerDied;

    public float CurrentHealth => currentHealth;
    public float MaxHealth => maxHealth;
    public float HealthPercent => currentHealth / maxHealth;

    void Start()
    {
        currentHealth = maxHealth;
        OnHealthChanged?.Invoke(HealthPercent);
    }

    public void TakeDamage(float damage)
    {
        currentHealth = Mathf.Max(0, currentHealth - damage);
        OnHealthChanged?.Invoke(HealthPercent);

        if (currentHealth <= 0)
        {
            Die();
        }
    }

    public void Heal(float amount)
    {
        currentHealth = Mathf.Min(maxHealth, currentHealth + amount);
        OnHealthChanged?.Invoke(HealthPercent);
    }

    private void Die()
    {
        OnPlayerDied?.Invoke();
        if (deathEffect != null)
            Instantiate(deathEffect, transform.position, Quaternion.identity);
        gameObject.SetActive(false);
    }
}
```

**`PlayerInventory.cs`** (인벤토리 책임 분리):

```csharp
using UnityEngine;
using System;

public class PlayerInventory : MonoBehaviour
{
    [Header("Inventory")]
    [SerializeField] private int gold;
    [SerializeField] private int potions;

    // 이벤트
    public event Action<int> OnGoldChanged;
    public event Action<int> OnPotionsChanged;

    public int Gold => gold;
    public int Potions => potions;

    void Start()
    {
        // 초기 UI 갱신
        OnGoldChanged?.Invoke(gold);
        OnPotionsChanged?.Invoke(potions);
    }

    public void AddGold(int amount)
    {
        gold += amount;
        OnGoldChanged?.Invoke(gold);
    }

    public bool SpendGold(int amount)
    {
        if (gold < amount) return false;
        gold -= amount;
        OnGoldChanged?.Invoke(gold);
        return true;
    }

    public bool UsePotion()
    {
        if (potions <= 0) return false;

        potions--;
        OnPotionsChanged?.Invoke(potions);

        // 플레이어 힐은 이벤트로 전달하지 않고,
        // PlayerHealth를 직접 찾거나 별도 처리
        GetComponent<PlayerHealth>()?.Heal(30);
        return true;
    }

    public void AddPotion(int amount)
    {
        potions += amount;
        OnPotionsChanged?.Invoke(potions);
    }
}
```

**`PlayerCombat.cs`** (전투 책임 분리):

```csharp
using UnityEngine;

public class PlayerCombat : MonoBehaviour
{
    [Header("Combat")]
    [SerializeField] private int attackDamage = 20;
    [SerializeField] private float attackRange = 2f;
    [SerializeField] private LayerMask enemyLayer;
    [SerializeField] private float attackCooldown = 0.5f;

    private float lastAttackTime;

    void Update()
    {
        if (Input.GetMouseButtonDown(0) && CanAttack())
        {
            PerformAttack();
        }
    }

    private bool CanAttack()
    {
        return Time.time >= lastAttackTime + attackCooldown;
    }

    private void PerformAttack()
    {
        lastAttackTime = Time.time;

        Collider[] hits = Physics.OverlapSphere(transform.position, attackRange, enemyLayer);
        foreach (var hit in hits)
        {
            if (hit.TryGetComponent<IDamageable>(out var damageable))
            {
                damageable.TakeDamage(attackDamage);
            }
            else if (hit.TryGetComponent<PlayerHealth>(out var health))
            {
                // 임시: 직접 데미지 처리
                Debug.Log($"Hit {hit.name} for {attackDamage}");
            }
        }
    }

    void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, attackRange);
    }
}
```

**`PlayerUI.cs`** (UI 갱신 책임 분리):

```csharp
using UnityEngine;
using UnityEngine.UI;

public class PlayerUI : MonoBehaviour
{
    [Header("UI References")]
    [SerializeField] private Image healthBar;
    [SerializeField] private Text goldText;
    [SerializeField] private Text potionsText;

    [Header("Target")]
    [SerializeField] private PlayerHealth playerHealth;
    [SerializeField] private PlayerInventory playerInventory;

    void Start()
    {
        // 자동 참조 (없을 경우에만)
        if (playerHealth == null)
            playerHealth = GetComponentInParent<PlayerHealth>();
        if (playerInventory == null)
            playerInventory = GetComponentInParent<PlayerInventory>();

        if (playerHealth == null || playerInventory == null)
        {
            Debug.LogError("PlayerUI: Missing required components!");
            return;
        }

        // 이벤트 구독 — 값이 변경될 때만 UI 갱신
        playerHealth.OnHealthChanged += UpdateHealthBar;
        playerInventory.OnGoldChanged += UpdateGoldText;
        playerInventory.OnPotionsChanged += UpdatePotionsText;

        // 초기값 표시
        UpdateHealthBar(playerHealth.HealthPercent);
        UpdateGoldText(playerInventory.Gold);
        UpdatePotionsText(playerInventory.Potions);
    }

    private void UpdateHealthBar(float percent)
    {
        if (healthBar != null)
            healthBar.fillAmount = percent;
    }

    private void UpdateGoldText(int gold)
    {
        if (goldText != null)
            goldText.text = $"Gold: {gold}";
    }

    private void UpdatePotionsText(int potions)
    {
        if (potionsText != null)
            potionsText.text = $"Potions: {potions}";
    }

    void OnDestroy()
    {
        // 이벤트 구독 해제 (메모리 누수 방지)
        if (playerHealth != null)
            playerHealth.OnHealthChanged -= UpdateHealthBar;
        if (playerInventory != null)
        {
            playerInventory.OnGoldChanged -= UpdateGoldText;
            playerInventory.OnPotionsChanged -= UpdatePotionsText;
        }
    }
}
```

---

## 4단계: 결과 검증 (Ask 모드)

### 🎯 상황

Agent 모드로 리팩토링을 완료했습니다.
Ask 모드로 결과를 검증합니다.

```
[Ask 모드]
User> "리팩토링된 코드 구조를 다시 분석해줘.
      이전과 비교해서 어떤 점이 개선됐는지 알려줘."
```

#### Assistant 응답

```
✅ 개선된 점:

1. ✅ 단일 책임 원칙 (SRP) 준수
   - PlayerMovement: 이동만 담당
   - PlayerHealth: 체력/사망만 담당
   - PlayerInventory: 인벤토리만 담당
   - PlayerCombat: 전투만 담당
   - PlayerUI: UI 갱신만 담당

2. ✅ 이벤트 기반 UI 갱신
   - Before: 매 프레임 UpdateUI() 호출
   - After: 값 변경 시에만 UI 갱신 (event 구독)

3. ✅ 캡슐화
   - Before: public gold, public potions (외부에서 직접 수정 가능)
   - After: private serialized + public 메서드로 보호

4. ✅ 유지보수성
   - 각 컴포넌트가 독립적이라 테스트/수정/재사용이 쉬움
   - 새 기능 추가 시 기존 코드 수정 없이 새 컴포넌트 추가

5. ✅ 코드 길이
   - Before: PlayerController 800줄 (하나의 파일)
   - After: 각 파일 40~80줄로 분산

⚠️ 주의사항:
   - PlayerInventory가 직접 PlayerHealth.Heal()을 호출 (약한 결합 위반)
   → 개선 방안: 중재자(Mediator) 패턴이나 EventBus 도입 고려
```

---

## 바이브코딩 워크플로우 요약

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Unity AI 바이브코딩 워크플로우                      │
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  🔍 Ask  │───▶│  📋 Plan │───▶│  🤖 Agent│───▶│  🔍 Ask  │      │
│  │  분석    │    │  계획    │    │  실행    │    │  검증    │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │             │
│       ▼               ▼               ▼               ▼             │
│  코드 읽고      구조화된        AI가 직접       결과를 다시          │
│  문제 식별      구현 계획       파일 생성/       분석하고             │
│  (읽기 전용)    수립           수정/삭제        개선 확인            │
│                                                                      │
│  ◄──────────────────── 반복 (Iterative) ────────────────────────►   │
│     "AI와 함께 반복적으로 코드 품질 높이기"                          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 주요 프롬프트 패턴 (복붙 가이드)

### 패턴 1: 코드 분석 요청 (Ask)

```
[Ask 모드]
"다음 스크립트를 분석해줘: [파일명]
- 어떤 책임들을 가지고 있는지
- SRP 위반 사항
- 성능 문제
- 개선 제안
```

### 패턴 2: 리팩토링 계획 (Plan)

```
[Plan 모드]
"현재 상태: [기존 코드 설명, 예: PlayerController 하나에 모든 기능]
목표 상태: [원하는 구조, 예: 책임별 컴포넌트 분리]
기술 제약: [Unity 버전, 사용 중인 시스템 등]
계획을 세워줘."
```

### 패턴 3: 특정 리팩토링 실행 (Agent)

```
[Agent 모드]
"Plan에서 수립한 Phase 1을 실행해줘.
[컴포넌트명]을 생성하고 기존 PlayerController에서 관련 코드를 분리해줘.
기존 기능이 깨지지 않도록 주의해줘."
```

### 패턴 4: 코드 리뷰 (Ask)

```
[Ask 모드]
"리팩토링된 [컴포넌트명] 코드를 리뷰해줘.
- 잠재적 버그
- 개선 가능한 부분
- 기존 코드와의 일관성"
```

---

## 실제 데모 워크플로우 (영상 기준)

영상에서 시연된 전체 과정을 타임라인으로 정리:

```
타임라인:

00:00 ─ 인트로: "Unity AI로 코드를 읽고, 문제 파악하고, 구조 개선"
       문제: 기존 코드가 너무 비대해짐 (모노리스)

04:26 ─ Unity AI 바이브코딩 데모
       - Ask 모드로 기존 코드 분석
       - AI가 코드 구조와 문제점 식별
       - "이 부분을 이렇게 바꾸면 어떨까" 제안

12:59 ─ Unity AI 리팩토링
       - Plan 모드로 리팩토링 계획 수립
       - "이 컴포넌트들을 이렇게 분리하겠다" 구조 설계
       - Agent 모드로 실제 코드 생성/수정 실행
       - AI가 파일을 만들고 기존 코드 마이그레이션

32:50 ─ 최종 결과
       - Before/After 비교
       - 성능, 유지보수성, 가독성 개선 확인
       - AI가 발견하지 못한 추가 개선점 논의

35:44 ─ 마무리
       - AI는 완벽하지 않다. 최종 결정은 개발자.
       - AI와 협업하는 방법에 대한 인사이트
       - "AI는 훌륭한 페어 프로그래머, 하지만 당신이 Captain"
```

---

## 핵심 인사이트

### ✅ AI가 잘하는 것
- **코드 분석**: 방대한 코드에서 문제점 빠르게 찾기
- **패턴 인식**: SRP 위반, 중복 코드 등 정형화된 문제 발견
- **코드 생성**: 명확한 명세에 따른 구현체 작성
- **리팩토링**: 구조적 개선의 첫 번째 초안 생성
- **반복 작업**: 지루한 마이그레이션 작업 처리

### ❌ AI가 아직 부족한 것
- **비즈니스 로직 이해**: 게임의 "재미"나 "의도" 파악
- **트레이드오프 판단**: "이게 더 좋은 구조인가?"는 사람이 결정
- **컨텍스트 유지**: 프로젝트 전체 맥락을 완벽히 파악하지 못함
- **최종 검증**: AI가 만든 결과는 반드시 사람이 리뷰 필요

### 🎯 결론

> **Unity AI는 "코드를 대신 짜주는 도구"가 아니라
> "개발자와 함께 코드를 고민하는 페어 프로그래머"입니다.**
>
> - 분석은 AI에게, 판단은 사람에게
> - AI가 80%를 만들면, 사람이 20%를 다듬는다
> - Ask → Plan → Agent → Ask 순환으로 품질을 높인다
> - 반복할수록 더 나은 결과

---

## 직접 해보기

```
실습 순서:

1. 자신의 프로젝트에서 가장 비대한 스크립트 찾기
2. Ask 모드로 분석 요청
3. AI의 분석을 읽고 리팩토링 방향 결정
4. Plan 모드로 계획 수립
5. Agent 모드로 실행
6. Ask 모드로 결과 검증
7. 3~6 반복 (Iterative)
```

---

**→ 전체 로드맵으로 돌아가기: [README.md](./README.md)**  
**→ Unity AI 2.0 개념 보기: [Step 0: Unity AI 2.0 Beta](./Step0_UnityAI2_Beta.md)**
