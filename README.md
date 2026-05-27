# Unity AI 교육 학습 가이드

> **대상 버전**: Unity 6 (6000.4.x)  
> **AI 패키지**: AI Navigation 2.0.12, ML-Agents 4.0.3, Sentis 2.4.1, AI Assistant 2.0  
> **난이도**: ⭐ 초급 ~ ⭐⭐⭐⭐⭐ 고급

---

## 개요

이 교육 자료는 **두 가지 트랙**으로 구성되어 있습니다.

| 트랙 | 대상 | 내용 | Step |
|------|------|------|------|
| **🛠️ 개발 도구 AI** | 게임 개발자 생산성 | Unity AI 2.0 (Assistant, MCP, Gateway, Generators) | **Step 0** |
| | | 바이브코딩 & 리팩토링 실전 데모 | **VibeCoding_Demo** |
| **🎮 게임 속 AI** | NPC 행동 구현 | NavMesh, FSM, Behavior Tree, ML-Agents | **Step 1~5** |

두 트랙은 **상호 보완적**입니다. Step 0으로 빠르게 프로토타입을 만들고, Step 1~5로 게임 AI를 구현하세요.

---

## 학습 로드맵

### 🛠️ 개발 도구 AI 트랙

| 단계 | 주제 | 난이도 | 핵심 개념 |
|------|------|--------|-----------|
| **Step 0** | [Unity AI 2.0 Beta](./Step0_UnityAI2_Beta.md) | ⭐ | Assistant (Ask/Plan/Agent), AI Gateway, Unity MCP, Generators |
| **🎬 Demo** | [바이브코딩 & 리팩토링 실전](./VibeCoding_Refactoring_Demo.md) | ⭐⭐ | Ask→Plan→Agent 워크플로우, SRP 리팩토링, 이벤트 기반 구조 개선 |

### 🎮 게임 속 AI 트랙

| 단계 | 주제 | 난이도 | 핵심 개념 | 비고 |
|------|------|--------|-----------|------|
| **Step 1** | NavMesh 기초 | ⭐ | NavMesh Surface, NavMesh Agent, 목표 추적 | AI 입문 |
| **Step 2** | NavMesh 심화 | ⭐⭐ | NavMesh Obstacle, NavMesh Link, 다중 에이전트 | 장애물 회피 |
| **Step 3** | 유한 상태 머신 | ⭐⭐⭐ | FSM 패턴, Idle/Patrol/Chase/Attack | 행동 설계 |
| **Step 4** | Behavior Tree & 센서 | ⭐⭐⭐⭐ | BT 구조, 시야/청각 센서, 복합 행동 | 지능형 NPC |
| **Step 5** | ML-Agents 강화학습 | ⭐⭐⭐⭐⭐ | PPO, 보상 설계, 학습/임베딩 | 머신러닝 기반 AI |

---

## 전체 구조도

```
                    Unity AI 생태계

┌─────────────────────────────────────────────────────┐
│  🛠️ Step 0: Unity AI 2.0 (Beta)                     │
│  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌───────┐ │
│  │Assistant │  │ Gateway  │  │  MCP   │  │Gen.   │ │
│  │Ask/Plan/ │  │Claude/   │  │외부 AI  │  │텍스처 │ │
│  │Agent     │  │Gemini/   │  │→Unity   │  │메시/  │ │
│  │          │  │Codex     │  │제어     │  │사운드 │ │
│  └──────────┘  └──────────┘  └────────┘  └───────┘ │
│  "개발 도구 AI" — 생산성 향상                        │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  🎮 Step 1~5: 게임 속 AI                            │
│                                                     │
│  Step 1 (NavMesh 기초)                              │
│    ↓                                                │
│  Step 2 (NavMesh 심화)                              │
│    ↓                                                │
│  Step 3 (FSM 상태머신)           Step 5             │
│    ↓                            (ML-Agents          │
│  Step 4 (Behavior Tree)          강화학습)          │
│    ↓                            ↑                  │
│  복합 행동 NPC ◄────────── 학습된 모델              │
│                                                     │
│  "게임 속 AI" — NPC 두뇌 구현                       │
└─────────────────────────────────────────────────────┘
```

## 사전 준비

### 필수 설치
1. **Unity Hub** 및 **Unity 6 (6000.4.x)** 설치
2. 각 Step에서 필요한 패키지 설치 (Package Manager → Unity Registry)
   - **Step 0**: `AI Assistant` (com.unity.ai.assistant) + Unity Cloud 연결
   - **Step 1-2**: `AI Navigation` (v2.0.12)
   - **Step 3-4**: 추가 패키지 불필요 (C# 스크립트만 사용)
   - **Step 5**: `ML-Agents` (v4.0.3) + Python (mlagents Python 패키지)

### 권장 사양
- OS: Windows 10/11, macOS 12+, Ubuntu 20.04+
- GPU: DirectX 12 / Vulkan 지원 (ML-Agents 학습 시 CUDA 권장)
- RAM: 8GB 이상 (ML-Agents 학습 시 16GB 권장)

---

## 목차

### 🛠️ 개발 도구 AI
- [Step 0: Unity AI 2.0 (Beta) — AI로 게임 개발 가속화](./Step0_UnityAI2_Beta.md)
  - AI 개요 영상: [Unity AI, 이렇게 쓸 수 있어요](https://youtu.be/72eXC39omHA)
- [🎬 바이브코딩 & 리팩토링 실전 데모](./VibeCoding_Refactoring_Demo.md)
  - 리팩토링 데모 영상: [Unity AI와 함께하는 리팩토링 생산성 혁신](https://youtu.be/6Ik94spWREI)

### 🎮 게임 속 AI
- [Step 1: NavMesh 기초 - 따라오는 AI](./Step1_Basic_NavMesh.md)
- [Step 2: NavMesh 심화 - 장애물과 링크](./Step2_NavMesh_Advanced.md)
- [Step 3: 유한 상태 머신 - 지능형 NPC 행동](./Step3_Finite_State_Machine.md)
- [Step 4: Behavior Tree & 센서 시스템](./Step4_BehaviorTree_Sensors.md)
- [Step 5: ML-Agents 강화학습](./Step5_ML_Agents.md)

---

## 학습 팁

- 두 트랙은 **독립적**입니다. 원하는 순서로 학습 가능합니다.
- **Step 0 (Unity AI 2.0)** → **Step 1~5** 순서를 권장합니다 (AI 도구로 빠르게 실습 환경 구성).
- 각 Step의 **실습 코드**는 직접 타이핑하며 이해하는 것을 권장합니다.
- **확장 과제**는 필수는 아니지만, 개념을 완전히 이해하는 데 도움이 됩니다.
- 에러가 발생하면 Unity Console 창의 스택 트레이스를 먼저 확인하세요.
