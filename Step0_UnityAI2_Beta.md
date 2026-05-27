# Step 0: Unity AI 2.0 (Beta) — AI로 게임 개발 가속화

> **참고 영상**: [Unity AI, 이렇게 쓸 수 있어요](https://youtu.be/72eXC39omHA)  
> **대상 버전**: Unity 6 (6000.0 이상) + `com.unity.ai.assistant` 패키지  
> **난이도**: ⭐ 입문  
> **성격**: 위 Step 1~5(NavMesh/FSM/ML-Agents)와 **별개 트랙**입니다.  
>   이 Step 0은 **AI를 활용한 게임 개발 워크플로우**를 다룹니다.

---

## 1. Unity AI 2.0이란?

Unity AI 2.0(Beta)은 **게임 개발자를 위한 AI 도구 스위트**입니다.
게임 엔진에 특화된 AI 에이전트가 에디터 안에서 직접 작업을 수행합니다.

> **기존 자료(Step 1~5)와의 차이점**
> 
> | 구분 | Step 1~5 (NavMesh/FSM/ML-Agents) | Step 0 (Unity AI 2.0) |
> |------|----------------------------------|----------------------|
> | 목적 | **게임 속 AI** (NPC 행동, 길찾기, 학습) | **개발 도구 AI** (생산성 향상) |
> | 대상 | 게임 플레이어와 상호작용하는 캐릭터 | 게임을 만드는 **개발자** |
> | 기술 | NavMesh, Behavior Tree, 강화학습 | LLM, MCP, Generative AI |
> | 비유 | 게임 속 적 캐릭터의 지능 | 개발자의 **AI 페어 프로그래머** |

Unity AI 2.0의 핵심 기능 4가지:

---

## 2. 4가지 핵심 기능

### 2.1 AI Assistant — 에디터 내장 AI 에이전트

에디터에 직접 통합된 AI 비서입니다. **3가지 모드**로 동작합니다.

```
┌─────────────────────────────────────────────────────┐
│                 AI Assistant                         │
├──────────────┬──────────────┬────────────────────────┤
│   🔍 Ask    │   📋 Plan    │   🤖 Agent             │
│  (질문 모드)  │  (계획 모드)  │  (실행 모드)           │
├──────────────┼──────────────┼────────────────────────┤
│ 읽기 전용    │ 게임 기획 →  │ 실제 에디터 작업 수행   │
│ 설명/조언    │ 구현 계획 생성│ 오브젝트 생성/수정 등   │
│ 성능 분석    │ 태스크 분해   │ 어셋 임포트/정리       │
│ 코드 리뷰    │ 의존성 분석   │ 콘솔 에러 자동 수정    │
└──────────────┴──────────────┴────────────────────────┘
```

**Ask 모드 사용 예:**

```
"이 씬에서 Draw Call을 줄일 수 있는 방법을 알려줘"
→ 배칭 가능한 오브젝트 식별, LOD 제안, 라이트맵 적용 추천

"이 스크립트의 성능 병목을 분석해줘"
→ Update() 내부의 GC 할당 지점, 불필요한 GetComponent 호출 식별
```

**Agent 모드 사용 예:**

```
"이 에셋들로 간단한 3인칭 캐릭터 셋업을 만들어줘"
→ Animator Controller 생성, 캐릭터에 할당, 기본 이동 스크립트 작성

"Hierarchy의 모든 오브젝트를 의미 단위로 폴더링해줘"
→ Environment/Characters/UI 등으로 자동 재구성
```

### 2.2 AI Gateway — 타사 AI 모델 연결

AI Gateway는 Unity Assistant 내에서 **외부 AI 모델을 선택하고 연결**하는 관리 레이어입니다.

```
┌─────────────────────────────────────────────────────────┐
│                   AI Gateway                             │
│  Assistant 창 안에서 모델을 선택/전환                     │
├─────────────────────────────────────────────────────────┤
│  지원 에이전트:                                          │
│  • Claude Code (v2.1.45+)                               │
│  • Gemini CLI                                           │
│  • Codex CLI                                            │
│  • Cursor CLI (2026.02.13+)                             │
├─────────────────────────────────────────────────────────┤
│  설정 방법:                                              │
│  1. Assistant → AI Gateway → Agent Type 선택             │
│  2. API Key 등록 (Anthropic/Gemini/OpenAI)              │
│  3. 저장 → 자동으로 Assistant가 해당 모델 사용            │
└─────────────────────────────────────────────────────────┘
```

**AI Gateway를 쓰는 이유:**

| 이유 | 설명 |
|------|------|
| **모델 선택 자유도** | 상황에 따라 Claude/GPT/Gemini 전환 |
| **API 키 중앙 관리** | Unity 환경설정에서 일괄 관리 |
| **보안** | 토큰/키가 에디터 설정에 안전하게 저장 |
| **확장성** | 새 모델이 나와도 Gateway만 업데이트 |

### 2.3 Unity MCP — 외부 AI와 Unity 연결

**MCP(Model Context Protocol)** 는 오픈 표준으로, 외부 AI 클라이언트가 Unity 에디터를 **직접 제어**할 수 있게 합니다.

```
┌──────────────┐         MCP Protocol         ┌──────────────┐
│  AI Client   │ ◄══════════════════════════►  │  Unity MCP   │
│  (Claude     │    도구 호출 / 결과 반환        │  Server      │
│   Code,      │                               │  (Relay)     │
│   Cursor,    │                               │              │
│   Windsurf,  │                               │  • 씬 관리    │
│   Copilot)   │                               │  • 스크립팅   │
└──────────────┘                               │  • 콘솔 접근  │
                                               │  • 에셋 조작  │
                                               └──────────────┘
```

**설정 방법:**

```json
// MCP 클라이언트 설정 (예: Claude Desktop)
{
  "mcpServers": {
    "unity-mcp": {
      "command": "~/.unity/relay/relay_mac_arm64",
      "args": ["--mcp"]
    }
  }
}
```

```
Unity 측:
Edit > Project Settings > AI > Unity MCP
→ 대기 중인 연결 승인 (Accept)
```

**MCP로 가능한 작업:**

| 작업 | 예시 |
|------|------|
| 씬 조작 | "씬에 빨간 구체 3개를 (0,0,0)에 배치해줘" |
| 스크립트 편집 | "PlayerController.cs에 점프 기능을 추가해줘" |
| 디버깅 | "현재 콘솔 에러를 분석하고 원인을 찾아줘" |
| 에셋 관리 | "Assets/Models 폴더의 중복 FBX를 제거해줘" |

### 2.4 Unity AI Generators — 텍스트로 에셋 생성

텍스트 프롬프트만으로 게임 에셋을 직접 생성합니다.

| 생성 가능 에셋 | 설명 | 프롬프트 예시 |
|--------------|------|--------------|
| **이미지/스프라이트** | 2D 아트, 아이콘, 캐릭터 스프라이트 | "16x16 픽셀 스타일의 검 아이콘" |
| **PBR 머티리얼** | 전체 PBR 텍스처 맵 세트 | "녹슨 금속 질감의 방패 머티리얼" |
| **3D 메시** | 정적 3D 모델 (Prefab) | "낮은 폴리곤의 오크 나무" |
| **사운드 효과** | SFX, 배경 음향 | "동굴에 떨어지는 물방울 소리" |
| **휴머노이드 애니메이션** | AnimationClip | "기쁘게 점프하는 애니메이션" |
| **스카이박스** | 큐브맵/등장방형 이미지 | "일출의 판타지 하늘" |

> Muse Animate (텍스트-투-애니메이션)는 머신러닝으로 휴머노이드 캐릭터의 자연스러운 움직임을 생성합니다.

---

## 3. 영상 기반 실습: 해커톤 워크플로우

YouTube 영상([Unity AI, 이렇게 쓸 수 있어요](https://youtu.be/72eXC39omHA))의
타임라인 기반 학습 경로입니다.

### 타임라인

| 구간 | 시간 | 내용 |
|------|------|------|
| 🏁 해커톤 시작 | 00:00 | Unity AI 2.0을 해커톤에 실전 도입한 사례 |
| 🧩 기능 개요 | 03:28 | Unity AI의 3대 기능 소개 |
| 🌉 AI Gateway | 04:24 | 타사 AI 모델 연결 방법 |
| 🔌 Unity MCP | 04:57 | 외부 AI 클라이언트가 Unity 제어 |
| 🏭 Generators | 06:58 | 텍스트로 게임 에셋 생성 |
| 🤖 Assistant | 10:44 | Ask/Plan/Agent 모드 상세 |
| 🔄 워크플로우 | 15:58 | 실제 게임 개발에 적용한 예제 |
| 💡 활용 팁 | 20:24 | 효과적인 Unity AI 활용 전략 |
| 🎯 마무리 | 22:30 | 요약 및 향후 전망 |

### 추천 실습 순서

```
1. Assistant 설치 (Packages > AI Assistant)
2. Ask 모드로 씬 분석 요청
3. Plan 모드로 미니 게임 기획 → 계획 수립
4. Agent 모드로 실제 오브젝트 생성
5. AI Gateway에 Claude Code 연결
6. Unity MCP로 외부 AI 클라이언트 연결
7. Generators로 텍스트 → 에셋 생성
```

---

## 4. 실제 워크플로우 예제

### 예제 1: 프로토타입 제작 (15분 컷)

```
1. "2D 플랫포머 게임을 만들어줘" (Plan 모드)
   → 계획 수립: 캐릭터, 플랫폼, 적, 점프 물리

2. "캐릭터 스프라이트를 생성해줘" (Generators)
   → 텍스트 → PNG 스프라이트 생성

3. "기본 점프와 이동 스크립트를 추가해줘" (Agent 모드)
   → PlayerController.cs 자동 생성 및 할당

4. "적이 플레이어를 따라오게 해줘" (Agent 모드)
   → NavMesh Agent 설정, 추적 스크립트 추가
```

### 예제 2: 에셋 최적화

```
1. "이 씬의 성능을 분석해줘" (Ask 모드)
   → Draw Call 230개, 15ms 프레임 타임 식별

2. "정적 배치 가능한 오브젝트를 Bake해줘" (Agent 모드)
   → Static 플래그 설정, 라이트맵 Bake

3. "LOD 그룹을 자동 생성해줘" (Agent 모드)
   → LOD 0/1/2 자동 셋업
```

### 예제 3: 버그 수정

```
1. Console에 "NullReferenceException" 발생
   → "이 에러의 원인을 찾아줘" (Ask 모드)

2. "참조가 누락된 필드를 자동으로 연결해줘" (Agent 모드)
   → SerializeField에 누락된 오브젝트 참조 연결
```

---

## 5. Unity AI 2.0 vs 기존 AI (Step 1~5)

이해를 돕기 위한 비교:

```
                    Unity AI 생태계 전체 지도

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   🎮 게임 속 AI (In-Game AI)          🛠️ 개발 도구 AI      │
│   ──────────────────────────          ────────────────     │
│                                                             │
│   Step 1: NavMesh 기초            Step 0: Unity AI 2.0    │
│   Step 2: NavMesh 심화              ├─ AI Assistant        │
│   Step 3: FSM 상태머신             ├─ AI Gateway           │
│   Step 4: Behavior Tree            ├─ Unity MCP            │
│   Step 5: ML-Agents                └─ AI Generators        │
│                                                             │
│   "NPC가 똑똑해지는 법"          "개발자가 빨라지는 법"   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

두 트랙은 **상호 보완적**입니다:
- Unity AI 2.0으로 **빠르게 프로토타입** 제작
- Step 1~5의 기술로 **게임 속 AI** 구현
- Unity AI 2.0 Agent 모드로 Step 1~5의 **반복 작업 자동화**
- MCP로 외부 AI(Claude 등)와 연결하여 **코드 리뷰/리팩토링**

---

## 6. Unity AI 2.0 활용 팁

영상(20:24)에서 소개된 주요 팁:

| 팁 | 설명 |
|----|------|
| **구체적으로 말하라** | "캐릭터 만들어줘" ❌ → "2D 픽셀 스타일, 빨간 망토를 입은, 점프 가능한 플레이어 캐릭터" ✅ |
| **단계적으로 실행** | 한 번에 모든 것을 요청하지 말고, 작은 단위로 나누어 실행 |
| **Plan 모드를 먼저** | Agent 모드 실행 전 Plan 모드로 계획을 검토 |
| **Checkpoint 활용** | AI가 생성한 변경사항은 Checkpoint로 관리하여 롤백 가능 |
| **권한 설정** | Agent 모드의 자동 실행 권한을 단계별로 설정 |
| **Ask로 검증** | Agent가 만든 결과를 Ask 모드로 다시 검토 |
| **AI + 수동 혼합** | AI가 생성한 결과를 수동으로 미세 조정 |

---

## 7. 시작하기

### 설치

```
1. Unity 6 (6000.0+) 설치
2. 프로젝트 열기
3. Window > Package Manager > Unity Registry
4. "AI Assistant" 검색 및 설치 (com.unity.ai.assistant)
5. 에디터 상단 AI 버튼 클릭 → Assistant 창 열기
6. Unity Cloud에 프로젝트 연결 (필수)
```

### 첫 실행 워크플로우

```
1. Ask 모드: "이 프로젝트의 구조를 분석해줘"
2. Plan 모드: "간단한 3인칭 탐험 게임을 기획해줘"
3. Agent 모드: "기본 플레이어 캐릭터와 바닥을 생성해줘"
4. 수동 조정: 생성된 결과를 직접 미세 조정
```

---

## 8. 확장 과제

1. **Ask 모드 탐색**: 현재 작업 중인 프로젝트에서 성능/코드 분석 요청
2. **Plan → Agent 연계**: Plan이 세운 계획을 Agent가 그대로 실행하는 전체 워크플로우 체험
3. **AI Gateway 연결**: Claude Code 또는 Gemini를 Gateway에 연결하고 Assistant에서 전환 사용
4. **MCP 연동**: Claude Desktop에서 Unity MCP로 연결하여 외부에서 씬 제어
5. **Generator 실험**: 텍스트만으로 다양한 에셋(메시, 텍스처, 사운드) 생성 비교

---

## 9. 참고 자료

| 자료 | 링크 |
|------|------|
| Unity AI 공식 페이지 | https://unity.com/features/ai |
| Assistant 문서 | https://docs.unity3d.com/Packages/com.unity.ai.assistant@2.0/manual/index.html |
| AI Gateway 문서 | https://docs.unity3d.com/Packages/com.unity.ai.assistant@2.0/manual/ai-gateway-intro.html |
| Unity MCP 문서 | https://docs.unity3d.com/Packages/com.unity.ai.assistant@2.0/manual/integration/unity-mcp-get-started.html |
| Unity AI Open Beta 소개 영상 | https://www.youtube.com/watch?v=VpfksT0uH_A |
| Unity AI 활용 팁 영상 | https://youtu.be/72eXC39omHA |

---

**→ 전체 로드맵으로 돌아가기: [README.md](./README.md)**
