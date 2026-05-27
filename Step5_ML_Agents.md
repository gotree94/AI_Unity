# Step 5: ML-Agents 강화학습

> **난이도**: ⭐⭐⭐⭐⭐ 고급  
> **목표**: ML-Agents 4.0을 이용한 강화학습 환경 구축, PPO 학습, 모델 임베딩  
> **소요 시간**: 90~120분 (학습 시간 별도)  
> **선행 조건**: Python 기본 지식, Unity 기초, Step 1~4 권장

---

## 1. 개념 설명

### ML-Agents란?

ML-Agents는 Unity 환경에서 **강화학습(Reinforcement Learning)** 을 구현하는 오픈소스 프레임워크입니다.

| 버전 | 출시일 | 특징 |
|------|--------|------|
| ML-Agents 4.0 | 2025.08 | Unity 6 필수, Inference Engine 2.2.1 탑재 |
| ML-Agents 3.0 | 2024.09 | Sentis 통합 시작 |
| ML-Agents 2.0 | 2023 | 안정화, 확장 패키지 분리 |

### 강화학습 기본 개념

```
                    ┌─────────────┐
                    │   Agent     │ ←── 학습하는 주체 (NPC)
                    └──────┬──────┘
               Action(행동) │ ▲  Reward(보상)
                           │ │
      ┌────────────────────┼─┼────────────────────┐
      │                    ▼ │                    │
      │           ┌──────────┴──────────┐        │
      │           │    Environment      │ ←── 학습 환경
      │           │  (Unity Scene)      │        │
      │           └──────────┬──────────┘        │
      │                      │ State(관찰)       │
      │                      ▼                   │
      │           ┌──────────────────┐           │
      │           │     Policy       │ ←── 정책 (뉴럴넷)  │
      │           │  (Agent Brain)   │           │
      │           └──────────────────┘           │
      └──────────────────────────────────────────┘
```

| 용어 | 설명 | Unity ML-Agents 대응 |
|------|------|---------------------|
| **Agent** | 학습하는 주체 | `Agent` 클래스 |
| **Environment** | 상호작용 공간 | Unity Scene |
| **State (Observation)** | 환경 상태 | `CollectObservations()` |
| **Action** | Agent의 행동 | `OnActionReceived()` |
| **Reward** | 행동에 대한 보상 | `AddReward()` |
| **Policy** | 행동 결정 네트워크 | Neural Network (PPO) |
| **Episode** | 시작~종료까지의 과정 | `EndEpisode()` |

### PPO (Proximal Policy Optimization)

ML-Agents의 기본 학습 알고리즘:

```
PPO 특징:
- 안정적인 정책 업데이트 (클리핑)
- 샘플 효율성 우수
- 연속/이산 행동 공간 모두 지원
- 하이퍼파라미터에 덜 민감
```

---

## 2. 실습: ML-Agents로 공 굴리기 (3DBall)

가장 기본적인 ML-Agents 예제인 **공 위에서 균형 잡기**를 구현합니다.

### Step 5.1 — Python 환경 설정

**Python 3.10~3.12 필요** (가상환경 권장):

```bash
# 가상환경 생성 (선택)
python -m venv mlagents_env
# 활성화 (Windows)
mlagents_env\Scripts\activate

# ML-Agents Python 패키지 설치
pip install mlagents==1.3.0

# 설치 확인
mlagents-learn --help
```

### Step 5.2 — Unity 프로젝트 설정

1. Unity Hub → **New Project** → **3D (Built-In Render Pipeline)**
2. 프로젝트 이름: `Unity_AI_Step5`
3. **Window > Package Manager** → **Unity Registry**
4. `ML-Agents` 검색 후 **Install** (v4.0.3)

### Step 5.3 — 3D Ball 환경 구성

**씬 구성:**

```
Hierarchy:
├── Floor (Plane)          ← 평평한 바닥
├── Target (Sphere)        ← Agent가 균형 잡아야 할 공
│   └── BallAgent (스크립트)
├── Environment (Empty)    ← 환경 그룹
│   ├── Platform_A (Cube)
│   ├── Platform_B (Cube)
│   └── ...
└── Directional Light
```

**Target (Sphere)** 설정:

1. `GameObject > 3D Object > Sphere` 생성
   - 이름: `Target`
   - Position: (0, 0.5, 0), Scale: (2, 2, 2)
2. **Rigidbody** 컴포넌트 추가:
   - Mass: `1`
   - Drag: `0.5`
   - Angular Drag: `0.5`
   - **Is Kinematic**: 🔓 해제
   - **Use Gravity**: ✅
3. **가시성**: Material을 빨간색/파란색 등 눈에 띄는 색으로 설정

**Floor** 설정:
1. `GameObject > 3D Object > Plane`
   - Position: (0, 0, 0)
   - Material: 흰색
2. **Ground** 태그로 설정 (기본)

**벽 설치 (공이 떨어지지 않도록):**

`GameObject > 3D Object > Cube`로 4개의 벽 생성:
- Position: (0, 0.5, -6), Scale: (12, 1, 0.5) — 뒤
- Position: (0, 0.5, 6), Scale: (12, 1, 0.5) — 앞
- Position: (-6, 0.5, 0), Scale: (0.5, 1, 12) — 왼
- Position: (6, 0.5, 0), Scale: (0.5, 1, 12) — 오른

### Step 5.4 — BallAgent 스크립트

`BallAgent.cs`:

```csharp
using UnityEngine;
using Unity.MLAgents;
using Unity.MLAgents.Sensors;
using Unity.MLAgents.Actuators;

public class BallAgent : Agent
{
    [Header("References")]
    [SerializeField] private Rigidbody targetRigidbody;
    [SerializeField] private Transform targetTransform;

    [Header("Settings")]
    [SerializeField] private float forceMultiplier = 10f;
    [SerializeField] private float maxAngularVelocity = 50f;
    [SerializeField] private float maxTorque = 30f;

    [Header("Rewards")]
    [SerializeField] private float rewardPerSecond = 0.01f;
    [SerializeField] private float fallPenalty = -1f;

    // 초기화
    public override void Initialize()
    {
        if (targetRigidbody == null)
            targetRigidbody = GetComponent<Rigidbody>();

        targetRigidbody.maxAngularVelocity = maxAngularVelocity;
    }

    // 에피소드 시작 시마다 호출
    public override void OnEpisodeBegin()
    {
        // 공을 중앙으로 리셋
        targetTransform.localPosition = Vector3.zero;
        targetTransform.localRotation = Quaternion.identity;
        targetRigidbody.linearVelocity = Vector3.zero;
        targetRigidbody.angularVelocity = Vector3.zero;

        // 랜덤 회전 (더 어렵게)
        // targetTransform.rotation = Quaternion.Euler(0, Random.Range(0f, 360f), 0);
    }

    // 관찰 정보 수집 (상태)
    public override void CollectObservations(VectorSensor sensor)
    {
        // 1. 공의 로컬 위치 (x, z) → 2개
        sensor.AddObservation(targetTransform.localPosition.x);
        sensor.AddObservation(targetTransform.localPosition.z);

        // 2. 공의 로컬 회전 → 3개 (x, y, z Euler)
        Vector3 localEuler = targetTransform.localEulerAngles;
        sensor.AddObservation(localEuler.x / 180f);   // -1~1 정규화
        sensor.AddObservation(localEuler.z / 180f);

        // 3. 공의 속도 (velocity) → 3개
        Vector3 velocity = targetRigidbody.linearVelocity;
        sensor.AddObservation(velocity.x / 10f);
        sensor.AddObservation(velocity.y / 10f);
        sensor.AddObservation(velocity.z / 10f);

        // 4. 공의 각속도 (angular velocity) → 3개
        Vector3 angularVel = targetRigidbody.angularVelocity;
        sensor.AddObservation(angularVel.x / 10f);
        sensor.AddObservation(angularVel.y / 10f);
        sensor.AddObservation(angularVel.z / 10f);

        // 총 관찰 수: 11개
    }

    // 행동 수신 및 실행
    public override void OnActionReceived(ActionBuffers actions)
    {
        // 연속 행동 공간 (Continuous Actions): 2개
        float moveX = Mathf.Clamp(actions.ContinuousActions[0], -1f, 1f);
        float moveZ = Mathf.Clamp(actions.ContinuousActions[1], -1f, 1f);

        // 토크(회전력) 적용
        Vector3 torque = new Vector3(moveZ, 0, -moveX) * maxTorque;
        targetRigidbody.AddTorque(torque, ForceMode.Force);

        // 생존 보상 (매 스텝마다 소량)
        AddReward(rewardPerSecond * Time.fixedDeltaTime);

        // 떨어짐 체크
        if (targetTransform.localPosition.y < -1f)
        {
            AddReward(fallPenalty);
            EndEpisode();
        }
    }

    // 사람이 직접 조종할 때 (Heuristic)
    public override void Heuristic(in ActionBuffers actionsOut)
    {
        var continuousActions = actionsOut.ContinuousActions;
        continuousActions[0] = Input.GetAxis("Horizontal");
        continuousActions[1] = Input.GetAxis("Vertical");
    }

    // 디버그 시각화
    void OnDrawGizmos()
    {
        if (Application.isPlaying)
        {
            Vector3 infoPos = targetTransform.position + Vector3.up * 3f;
            UnityEditor.Handles.Label(infoPos,
                $"Reward: {GetCumulativeReward():F2}"
            );
        }
    }
}
```

### Step 5.5 — BallAgent Inspector 설정

1. **Target** Sphere 선택
2. `BallAgent` 스크립트가 추가되어 있어야 함
3. **Inspector → BallAgent** 설정:

```
[BallAgent]
├── Target Rigidbody: Target (Rigidbody)
├── Target Transform: Target
├── Force Multiplier: 10
├── Max Angular Velocity: 50
├── Max Torque: 30
├── Reward Per Second: 0.01
└── Fall Penalty: -1

[Behavior Parameters]
├── Behavior Name: "BallLearning"
├── Vector Observation → Space Size: 11
├── Actions → Continuous Actions: 2
└── Model: None (학습 완료 후 지정)

[Decision Requester]
├── Decision Period: 5        ← 5프레임마다 결정
├── Take Actions Between: ✅
└── Take Actions During: ✅
```

> **Decision Period**: 매 프레임마다 결정하지 않고 일정 간격으로 결정하여 학습 안정성 향상

### Step 5.6 — 학습 실행

**학습 설정 파일 생성:**

`config/ppo/BallLearning.yaml`:

```yaml
behaviors:
  BallLearning:
    trainer_type: ppo
    hyperparameters:
      batch_size: 64
      buffer_size: 12000
      learning_rate: 0.0003
      beta: 0.001
      epsilon: 0.2
      lambd: 0.99
      num_epoch: 3
      learning_rate_schedule: linear
    network_settings:
      normalize: true
      hidden_units: 128
      num_layers: 2
      vis_encode_type: simple
    reward_signals:
      extrinsic:
        gamma: 0.995
        strength: 1.0
    max_steps: 500000
    time_horizon: 1000
    summary_freq: 10000
    keep_checkpoints: 5
    checkpoint_interval: 50000
    threaded: true
```

**학습 실행:**

```bash
# 프로젝트 폴더로 이동 후
cd path/to/Unity_AI_Step5

# ML-Agents 학습 시작
mlagents-learn config/ppo/BallLearning.yaml --run-id=BallRun1

# 위 명령어 실행 후 "Start training by pressing the Play button in the Unity Editor" 메시지 확인
# Unity Editor에서 Play 버튼 누르기
```

> 학습 중 Console 출력:
> ```
> BallLearning: Step: 10000. Mean Reward: 1.242. Std of Reward: 0.746. Training.
> BallLearning: Step: 20000. Mean Reward: 3.319. Std of Reward: 0.693. Training.
> BallLearning: Step: 30000. Mean Reward: 5.804. Std of Reward: 0.521. Training.
> ...
> BallLearning: Step: 500000. Mean Reward: 9.872. Std of Reward: 0.123. Saved Model.
> ```

### Step 5.7 — 학습된 모델 Unity에 임베딩

1. 학습 완료 후 `results/BallRun1/BallLearning.onnx` 파일 확인
2. Unity 프로젝트의 `Assets/Models/` 폴더 생성
3. `.onnx` 파일을 `Assets/Models/`로 드래그
4. **Target** Sphere의 **Behavior Parameters** 컴포넌트:
   - **Model**: `BallLearning` 선택
   - **Inference Device**: `GPU` (또는 CPU)
5. **Play** 버튼 → 학습된 AI가 공을 스스로 균형 잡는지 확인!

---

## 3. 심화: 커스텀 환경 (푸시 블록)

### Step 5.8 — PushBlock 환경

Agent가 블록을 목표 지점까지 밀어내는 환경

**씬 구성:**

```
Hierarchy:
├── Floor (Plane)
├── Goal (Cube, 파란색)    ← 목표 지점
├── Block (Cube, 빨간색)   ← 밀어야 할 블록 (Rigidbody)
├── Pusher (Capsule)       ← Agent (밀어내는 캐릭터)
│   └── PushAgent (스크립트)
└── Walls (Empty)
    ├── Wall_N, Wall_S, Wall_E, Wall_W
```

`PushAgent.cs`:

```csharp
using UnityEngine;
using Unity.MLAgents;
using Unity.MLAgents.Sensors;
using Unity.MLAgents.Actuators;

public class PushAgent : Agent
{
    [Header("References")]
    [SerializeField] private Rigidbody agentRigidbody;
    [SerializeField] private Transform blockTransform;
    [SerializeField] private Rigidbody blockRigidbody;
    [SerializeField] private Transform goalTransform;

    [Header("Settings")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private float rotationSpeed = 120f;

    private Vector3 blockStartPos;
    private Vector3 agentStartPos;

    public override void Initialize()
    {
        blockStartPos = blockTransform.position;
        agentStartPos = transform.position;
    }

    public override void OnEpisodeBegin()
    {
        // Agent 리셋
        transform.position = agentStartPos;
        transform.rotation = Quaternion.identity;
        agentRigidbody.linearVelocity = Vector3.zero;
        agentRigidbody.angularVelocity = Vector3.zero;

        // 블록 리셋 (랜덤 위치)
        blockTransform.position = blockStartPos + new Vector3(Random.Range(-2f, 2f), 0, Random.Range(-2f, 2f));
        blockTransform.rotation = Quaternion.identity;
        blockRigidbody.linearVelocity = Vector3.zero;
        blockRigidbody.angularVelocity = Vector3.zero;

        // 목표 리셋 (랜덤 위치)
        goalTransform.position = new Vector3(Random.Range(-5f, 5f), 0.25f, Random.Range(-5f, 5f));
    }

    public override void CollectObservations(VectorSensor sensor)
    {
        // Agent 위치
        sensor.AddObservation(transform.localPosition.x / 10f);
        sensor.AddObservation(transform.localPosition.z / 10f);

        // Agent 속도
        sensor.AddObservation(agentRigidbody.linearVelocity.x / 5f);
        sensor.AddObservation(agentRigidbody.linearVelocity.z / 5f);

        // 블록 위치 (Agent 기준 상대)
        Vector3 relativeBlock = blockTransform.position - transform.position;
        sensor.AddObservation(relativeBlock.x / 10f);
        sensor.AddObservation(relativeBlock.z / 10f);

        // 목표 위치 (Agent 기준 상대)
        Vector3 relativeGoal = goalTransform.position - transform.position;
        sensor.AddObservation(relativeGoal.x / 10f);
        sensor.AddObservation(relativeGoal.z / 10f);

        // 블록-목표 거리
        float blockToGoalDist = Vector3.Distance(blockTransform.position, goalTransform.position);
        sensor.AddObservation(blockToGoalDist / 15f);

        // 총 9개 관찰
    }

    public override void OnActionReceived(ActionBuffers actions)
    {
        float moveX = actions.ContinuousActions[0];
        float moveZ = actions.ContinuousActions[1];

        // 이동
        Vector3 move = new Vector3(moveX, 0, moveZ) * moveSpeed;
        agentRigidbody.AddForce(move - agentRigidbody.linearVelocity, ForceMode.VelocityChange);

        // 블록과의 거리 체크
        float distToBlock = Vector3.Distance(transform.position, blockTransform.position);
        if (distToBlock < 1.2f) AddReward(0.01f); // 블록 근접 시 소량 보상

        // 블록이 목표 지점에 도착?
        float blockToGoalDist = Vector3.Distance(blockTransform.position, goalTransform.position);
        if (blockToGoalDist < 1.0f)
        {
            AddReward(5.0f);
            EndEpisode();
        }

        // 시간 패널티 (빨리 해결하도록)
        AddReward(-0.001f);

        // 떨어짐 체크
        if (transform.localPosition.y < -1f)
        {
            AddReward(-1f);
            EndEpisode();
        }
    }

    public override void Heuristic(in ActionBuffers actionsOut)
    {
        actionsOut.ContinuousActions[0] = Input.GetAxis("Horizontal");
        actionsOut.ContinuousActions[1] = Input.GetAxis("Vertical");
    }
}
```

---

## 4. 보상 설계 원칙

### 좋은 보상 설계 vs 나쁜 보상 설계

| 원칙 | 나쁜 예 | 좋은 예 |
|------|---------|---------|
| **희소성** | 매 프레임 +0.01 (너무 잦음) | 목표 달성 시 +5.0 (도전적) |
| **단계적** | 최종 목표만 보상 | 중간 단계마다 소량 보상 |
| **명확성** | "잘하면" +1 (모호) | "목표 도달" +5, "충돌" -1 (명확) |
| **균형** | 움직이기만 해도 보상 | 움직임은 자체는 보상 없음 |

### PushBlock 보상 구조 분석

```
행동                   보상            이유
────────────────────────────────────────────────────────
블록 근접             +0.01     블록을 찾도록 유도
블록을 목표까지 밈     +5.0      에피소드 성공 (큰 보상)
스텝당               -0.001     빠른 해결 유도 (시간 패널티)
떨어짐               -1.0       실패 패널티
```

---

## 5. 성능 최적화 팁

### 병렬 환경 (학습 속도 향상)

같은 씬에 Agent를 여러 개 배치하여 병렬 학습:

1. Target Sphere를 프리팹(Prefab)으로 생성
2. 씬에 8~12개 배치
3. 각 Agent의 Behavior Parameters가 모두 같은 이름인지 확인

```yaml
# 학습 설정에서 병렬 환경 수 지정 (선택)
environment_parameters:
  num_parallel_envs: 8  # Unity 측에서 여러 Agent 사용 시
```

### 학습 모니터링 (TensorBoard)

```bash
# 학습 중 메트릭 확인
tensorboard --logdir results

# 브라우저에서 http://localhost:6006 열기
```

주요 지표:
- **Mean Reward**: 평균 보상 (100↑ = 성공)
- **Policy Loss**: 정책 손실 (안정적이어야 함)
- **Value Loss**: 가치 손실 (수렴 필요)
- **Entropy**: 탐험도 (적절히 감소해야 함)

---

## 6. 모델 내보내기 및 배포

### ONNX 모델 파일

학습 완료 시 `results/<run-id>/<behavior-name>.onnx` 파일 생성

### Unity에 적용

1. `.onnx` 파일을 `Assets/` 내 임의 폴더에 임포트
2. Agent의 **Behavior Parameters → Model**에 할당
3. **Play** → 학습 완료된 AI 동작 확인 (Python 불필요)

---

## 7. Sentis와의 관계

ML-Agents 4.0부터는 **학습된 모델이 Sentis를 통해 실행**됩니다:

```
ML-Agents 학습 (Python) → .onnx 모델 → Unity Sentis (런타임 추론)
```

Sentis를 직접 사용하면:
- 사전 학습된 이미지 분류 모델 사용
- LLM (Chat) Unity에서 실행
- 커스텀 신경망 아키텍처 구현

---

## 8. 확장 과제

1. **보상 조정 실험**: 보상 값을 변경하며 학습 속도 차이 관찰
2. **관찰 차원 추가**: 공에 RayCast 센서를 추가하여 더 넓은 환경 인지
3. **장애물 추가**: 환경에 장애물을 배치하고 학습 결과 비교
4. **Self-Play**: 두 Agent가 공을 두고 경쟁하도록 설계
5. **Curriculum Learning**: 난이도를 점진적으로 증가시키는 학습

---

## 9. 문제 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| `mlagents-learn` 실행 안 됨 | Python 버전 불일치 | Python 3.10~3.12 확인 |
| Unity에서 Agent가 움직이지 않음 | Behavior Name 불일치 | 학습 설정과 Unity Behavior Name이 같은지 확인 |
| 학습이 수렴하지 않음 | 보상 설계 문제 | 보상 값을 조정하거나 정규화 |
| 모델 로딩 실패 | ONNX 파일 손상 | 재학습 또는 `.nn` 파일 확인 |
| GPU 메모리 부족 | 배치 크기 너무 큼 | `batch_size` 감소 |

---

## 부록: 전체 학습 워크플로우

```
1. Unity 씬 설계
   ├── Agent, Environment, Target 설정
   └── Observation / Action / Reward 정의

2. 학습 환경 점검
   ├── Heuristic 모드로 수동 조종 테스트
   └── Console 로그 확인

3. Python 학습 실행
   ├── mlagents-learn <config> --run-id=<name>
   └── Unity Play 버튼 클릭

4. 학습 모니터링
   ├── TensorBoard로 Mean Reward 확인
   └── 필요 시 조기 중단 후 파라미터 조정

5. 모델 확인 및 배포
   ├── .onnx 파일 Unity에 임포트
   └── Inference 모드로 최종 테스트
```

---

**축하합니다! 🎉 5단계 학습 과정을 모두 완료하셨습니다.**

이제 Unity AI의 기초부터 고급 강화학습까지 모두 경험하셨습니다.
각 Step의 확장 과제를 직접 도전해보며 실력을 더 키워보세요!

[처음으로 돌아가기](./README.md)
