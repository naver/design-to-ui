---
name: generate-project-design-system
description: 현재 프로젝트의 디자인 시스템을 분석하여 project-design-system 스킬의 초안을 생성합니다. Figma MCP의 create_design_system_rules와 get_variable_defs를 활용하고, 코드베이스를 분석하여 Figma 토큰 → 코드 매핑 규칙을 도출합니다. 초안 생성 후 /skill-creator와 사용자 피드백으로 다듬는 과정이 필요합니다. "디자인 시스템 스킬 만들어줘", "project-design-system 생성" 같은 요청에 사용하세요.
license: Apache-2.0
metadata:
  author: NAVER
  version: "2.1"
---

# Generate Project Design System Skill

현재 프로젝트의 디자인 시스템을 분석하여, Figma 디자인을 코드로 변환할 때 참조할 `project-design-system` 스킬의 **초안**을 생성합니다.

이 스킬은 분석과 초안 생성까지를 담당합니다. 자동 분석만으로는 import 경로나 세부 컴포넌트를 놓칠 수 있으므로, 초안 생성 후 반드시 `/skill-creator`와 사용자 피드백으로 다듬어야 합니다.

## 워크플로우

### Step 1: Figma 공식 도구로 분석 가이드 획득

Figma MCP `create_design_system_rules` 호출하여 분석 항목 체크리스트를 받습니다.

```
create_design_system_rules(
  clientLanguages = "<프로젝트 언어>",   // kotlin,xml / typescript / swift 등
  clientFrameworks = "<프레임워크>"       // android,jetpack-compose / react / swiftui 등
)
```

반환된 체크리스트(토큰, 컴포넌트, 스타일링, 에셋 등)를 Step 3의 분석 가이드로 사용합니다.

### Step 2: Figma 토큰 구조 확인

사용자가 Figma URL/nodeId를 제공하면 `get_variable_defs`로 실제 토큰 구조를 가져옵니다.

```
get_variable_defs(nodeId = "...", clientLanguages = "<언어>")
```

Figma URL이 없으면 사용자에게 '토큰 패턴 추론에 필요하니 제공해달라'고 요청해서 참고합니다.

### Step 3: 코드베이스 분석

플랫폼에 따라 아래 항목을 Grep/Glob으로 분석합니다.

#### 3-1. 색상 토큰

어떤 디자인 토큰 시스템을 쓰는지, 코드에서 어떻게 참조하는지 파악합니다.

**Android (XML):** `@color/` 패턴에서 가장 많이 쓰이는 접두사와 토큰명 추출
**Android (Compose):** `colorResource`, `MaterialTheme.colorScheme`, 커스텀 테마 등 색상 참조 패턴
**iOS (SwiftUI):** `Color("...")`, `Asset Catalog`, 커스텀 Color extension
**Web (React):** CSS variables, Tailwind config, styled-components theme 등

**도출할 것:** Figma 토큰 키(kebab-case) → 코드 토큰명 변환 규칙

#### 3-2. 텍스트/타이포

**Android:** 커스텀 TextView 클래스, textSize 단위(dp/sp), Compose TextStyle 패턴
**iOS:** Font system, custom font 사용 방식, Text modifier 패턴
**Web:** CSS font 시스템, Typography 컴포넌트, font token 참조 방식

#### 3-3. 레이아웃/컴포넌트

- 주요 레이아웃 타입과 커스텀 컴포넌트 (상위 5개)
- 테마 래퍼 사용 여부
- 이미지 로딩 방식

#### 3-4. 에셋 최종 형식 (아이콘/이미지)

design-to-ui Step 4·5는 에셋 최종 형식을 PDS에 위임한다(없으면 플랫폼 기본값 — [design-to-ui Step 4 표](./../design-to-ui/SKILL.md) 기준). 코드베이스에서 이 프로젝트가 아이콘/이미지를 **어떤 형식·배율로 두는지** 확인하고(iOS `*.xcassets`, Android `res/drawable*`), **기본값과 다르면** 초안에 간단한 규칙 문구로 명시한다(예: "아이콘·사진 모두 PNG @1x/2x/3x, PDF 미사용"). 같으면 생략. (Android/iOS 공통)

### Step 4: 초안 SKILL.md 생성

Step 1~3 결과로 프로젝트의 `project-design-system/SKILL.md` 초안을 생성합니다. Claude Code의 기본 위치는 `.claude/skills/`, Codex는 `.agents/skills/`입니다. Codex에서 기존 `.claude/skills/` 규칙만 발견하면 그 파일을 재사용합니다.

**필수 섹션:**

1. **색상 — Figma 토큰 → 코드 매핑**: 변환 규칙 + 예시 테이블 (3~5개)
2. **텍스트**: 프로젝트 표준 텍스트 컴포넌트와 스타일 패턴
3. **레이아웃 패턴**: 주요 레이아웃, 커스텀 컴포넌트, 테마 구조

**조건부 섹션:**

4. **에셋 형식** (3-4 분석 결과가 플랫폼 기본값과 다를 때만): 아이콘/이미지 최종 형식·배율 규칙을 한두 줄로. design-to-ui Step 4·5가 이 규칙을 참조한다. 기본값과 같으면 이 섹션은 넣지 않는다.

**작성 원칙:**
- 100줄 이내로 간결하게
- Figma 토큰 → 코드 변환 규칙에 집중
- 프로젝트가 여러 UI 프레임워크를 쓰면 모두 커버 (예: Android XML + Compose)

### Step 5: /skill-creator로 다듬기

초안을 사용자에게 보여준 뒤, `/skill-creator`를 사용하여 개선합니다.

자동 분석은 import 경로, 세부 유틸 함수, 프로젝트 관행 같은 디테일을 놓칠 수 있습니다. 사용자 피드백을 한 번이라도 받으면 품질이 크게 올라갑니다.

`/skill-creator`에게 전달할 내용:
- **스킬 목적:** Figma 디자인을 이 프로젝트의 코드로 변환할 때 참조하는 디자인 시스템 매핑 규칙
- **초안 SKILL.md**
- **사용자 피드백**

### Step 6: 도입 사례 공유 제안 (선택 · 사용자 동의 필수)

초안이 정리되면, 이 결과를 **도입 사례로 공유할지 따뜻하게 권유**한다. 같은 플랫폼을 도입하려는 다른 팀에게 실제 사례는 큰 도움이 되기 때문이다. 강요하지 않으며, 거절하면 그냥 넘어간다.

권유 문구 예시:

> "이번에 만든 `project-design-system` 초안, [도입 사례]로 공유해두면 같은 플랫폼을 도입하는 다른 팀에 정말 큰 도움이 돼요. 괜찮으시면 제가 정리해서 **'플러그인/선디자인검수 워크플로우 도입'** 카테고리에 올려드릴까요? 물론 원치 않으시면 안 올려요 🙂"

동의하면 [`discussion` 스킬](./../discussion/SKILL.md)로 위임한다. 이때:
- 카테고리는 **플러그인/선디자인검수 워크플로우 도입** 고정
- #121 형식(개요 / 생성 과정 / 생성된 `project-design-system` 전문)을 따른다
- **등록 전 본문을 사용자에게 보여주고 최종 확인**한다 (discussion 스킬 Step 4)

등록되면 URL과 함께 감사 인사를 전한다.

<!--
Design-To-UI
Copyright (c) 2026-present NAVER Corp.
Apache-2.0
-->
