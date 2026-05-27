# Unity AI 교육 학습 가이드

> **대상 버전**: Unity 6 (6000.4.x)  
> **AI 패키지**: AI Navigation 2.0.12, ML-Agents 4.0.3, Sentis 2.4.1  
> **난이도**: ⭐ 초급 ~ ⭐⭐⭐⭐⭐ 고급

---

## 개요

이 교육 자료는 Unity의 최신 AI 시스템을 **5단계 학습 과정**으로 구성했습니다.
각 단계는 이전 단계를 기반으로 점진적으로 난이도가 상승합니다.

## 학습 로드맵

| 단계 | 주제 | 난이도 | 핵심 개념 | 비고 |
|------|------|--------|-----------|------|
| **Step 1** | NavMesh 기초 | ⭐ | NavMesh Surface, NavMesh Agent, 목표 추적 | AI 입문 |
| **Step 2** | NavMesh 심화 | ⭐⭐ | NavMesh Obstacle, NavMesh Link, 다중 에이전트 | 장애물 회피 |
| **Step 3** | 유한 상태 머신 | ⭐⭐⭐ | FSM 패턴, Idle/Patrol/Chase/Attack | 행동 설계 |
| **Step 4** | Behavior Tree & 센서 | ⭐⭐⭐⭐ | BT 구조, 시야/청각 센서, 복합 행동 | 지능형 NPC |
| **Step 5** | ML-Agents 강화학습 | ⭐⭐⭐⭐⭐ | PPO, 보상 설계, 학습/임베딩 | 머신러닝 기반 AI |

## 사전 준비

### 필수 설치
1. **Unity Hub** 및 **Unity 6 (6000.4.x)** 설치
2. 각 Step에서 필요한 패키지 설치 (Package Manager → Unity Registry)
   - **Step 1-2**: `AI Navigation` (v2.0.12)
   - **Step 3-4**: 추가 패키지 불필요 (C# 스크립트만 사용)
   - **Step 5**: `ML-Agents` (v4.0.3) + Python (mlagents Python 패키지)

### 권장 사양
- OS: Windows 10/11, macOS 12+, Ubuntu 20.04+
- GPU: DirectX 12 / Vulkan 지원 (ML-Agents 학습 시 CUDA 권장)
- RAM: 8GB 이상 (ML-Agents 학습 시 16GB 권장)

---

## 목차

- [Step 1: NavMesh 기초 - 따라오는 AI](./Step1_Basic_NavMesh.md)
- [Step 2: NavMesh 심화 - 장애물과 링크](./Step2_NavMesh_Advanced.md)
- [Step 3: 유한 상태 머신 - 지능형 NPC 행동](./Step3_Finite_State_Machine.md)
- [Step 4: Behavior Tree & 센서 시스템](./Step4_BehaviorTree_Sensors.md)
- [Step 5: ML-Agents 강화학습](./Step5_ML_Agents.md)

---

## 학습 팁

- 각 Step의 **실습 코드**는 직접 타이핑하며 이해하는 것을 권장합니다.
- **확장 과제**는 필수는 아니지만, 개념을 완전히 이해하는 데 도움이 됩니다.
- 에러가 발생하면 Unity Console 창의 스택 트레이스를 먼저 확인하세요.
- 각 Step은 독립적이지만, 순서대로 학습하는 것을 권장합니다.
