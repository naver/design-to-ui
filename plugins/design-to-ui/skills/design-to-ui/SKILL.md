---
name: design-to-ui
description: Figma 디자인을 네이티브 UI(Android Compose/XML, iOS SwiftUI/UIKit)로 변환하는 7-Step 워크플로우. 노드 추출·디자인 컨텍스트·검증(Step 1·2·3·7)은 플랫폼 공통이고, 에셋(Step 4·5)과 코드 생성(Step 6)만 플랫폼별 reference로 분기한다. 에셋 A/B/C 분류, 동적 비주얼, project-design-system 위임, 플랫폼 에셋 파이프라인을 포함한다. Figma URL이나 UI 변환 요청 시 이 스킬을 사용하세요.
license: Apache-2.0
metadata:
  author: NAVER
  version: "10.0"
  requires: [project-design-system, figma-asset-download]
  platform: [android, ios]
---

# Design to UI

Codex에서 아래 명령과 참조의 `${CLAUDE_SKILL_DIR}`는 현재 읽은 이 `SKILL.md`의 상위 디렉터리 절대 경로로 치환한다. 앱 프로젝트를 작업 디렉터리로 유지한다.

Figma 디자인을 네이티브 UI로 1:1 시각 충실도로 변환하는 워크플로우입니다.
Figma MCP 서버와 연동하여 디자인 토큰을 올바르게 사용하고, 에셋을 분류·변환하며, 프로젝트 디자인 시스템을 적용합니다.

**지원 플랫폼:** Android(Jetpack Compose / XML Layout), iOS(SwiftUI / UIKit)

## 공통 spine vs 플랫폼 분기

7-Step 중 **Step 1·2·3·7은 플랫폼 공통**(노드·컨텍스트·검증)이며, **Step 4·5(에셋)·6(코드 생성)만 플랫폼별로 분기**합니다. 공통 절차는 spine에 한 벌만 두고, 플랫폼 delta만 분기합니다 — **Step 6 코드 변환 규칙**은 분량이 커서 `references/<platform>/`에 두고, **Step 4·5 에셋 사양**은 작아서 Step 4의 플랫폼 표로 인라인합니다. 새 플랫폼은 spine을 그대로 두고 에셋 표 1행 + 코드 reference + 변환 스크립트만 추가하면 됩니다.

| 영역 | Android | iOS |
|------|---------|-----|
| Step 4·5 에셋 사양 | Step 4 플랫폼 표 + `scripts/android/convert_assets.sh` | Step 4 플랫폼 표 + `scripts/ios/convert_assets.sh` |
| Step 6 코드 (선택지 1) | `references/android/compose.md` | `references/ios/swiftui.md` |
| Step 6 코드 (선택지 2) | `references/android/xml.md` | `references/ios/uikit.md` |

## Prerequisites

- Figma MCP 서버 연결 확인 (`get_design_context` 도구 사용 가능)
- `FIGMA_ACCESS_TOKEN` 환경 변수 설정
- Figma URL 형식: `https://figma.com/design/:fileKey/:fileName?node-id=1-2`

## 서브에이전트 위임 규칙

**design_context는 코드 변환의 원본이므로, 반드시 메인 컨텍스트에서 직접 읽어야 합니다.**

| 작업 | 서브에이전트 위임 | 이유 |
|------|:-:|------|
| **design_context 읽기** | **금지** | React+Tailwind 코드의 flex 비율, gap, padding, 컴포넌트 계층구조가 코드 변환의 원본. 서브에이전트에 "요약"을 맡기면 정확한 수치와 중첩 구조가 소실되어 추측 기반 코드가 됨 |
| **design_context → 코드 변환 (Step 6)** | **금지** | 원본 React 코드를 직접 보면서 1:1 변환해야 레이아웃 비율, 위치, 간격이 정확함 |
| **에셋 다운로드 스크립트 실행 (Step 5)** | 허용 | 독립적 I/O 작업 |
| **기존 리소스 탐색 (에셋 카탈로그/리소스 디렉터리 검색)** | 허용 | 독립적 검색 작업 |
| **빌드 확인** | 허용 | 독립적 검증 작업 |

**design_context가 크면** (50KB 이상): `Read` 도구로 offset/limit를 나눠 여러 번 직접 읽는다. 어떤 유형의 서브에이전트에도 위임하지 않는다.

---

## Why each step matters

에이전트가 각 Step을 건너뛰지 않도록, 왜 필요한지 이해하는 것이 중요합니다.

**Step 0 (플랫폼 판별):** 잘못된 플랫폼으로 판별하면 Step 4·5·6 전체가 엉뚱한 reference를 읽어 처음부터 다시 해야 합니다.

**Step 1 (Node ID 추출):** file key와 node ID가 없으면 이후 모든 MCP 호출이 불가능합니다. URL 파싱 실수(예: `node-id`의 `-`를 `:`로 변환 누락)는 잘못된 노드를 가져오는 원인이 됩니다.

**Step 2 (디자인 컨텍스트):** `get_design_context`가 반환하는 React+Tailwind 코드는 **Step 6 코드 변환의 원본**입니다. 이 코드에 정확한 레이아웃 비율, gap, padding, 컴포넌트 계층구조가 모두 들어있습니다. 또한 이미지가 Figma의 임시 URL(`localhost:3845/assets/...`)로 포함되어 있어 세션 종료 시 만료되므로 Step 4~5의 에셋 파이프라인이 필요합니다.

**Step 3 (스크린샷):** 시각 참조 없이는 에셋 분류(Step 4)에서 동적/정적 판단이 불가능하고, 최종 검증(Step 7)에서 비교 기준이 없습니다.

**Step 4 (에셋 분류):** design_context의 시각 요소를 정적 에셋(A) / 동적 비주얼(B) / 기존 에셋(C)로 분류합니다. 여기서 "시각 요소"란 `<img>` 태그뿐 아니라, **React 컴포넌트로 래핑된 아이콘**(IconToolbar → IconEtc → img 같은 중첩 호출)까지 포함합니다. `<img>` 태그만 추적하면 컴포넌트로 감싸진 아이콘이 누락되어 최종 화면에서 빠지게 됩니다. **분류의 destination과 검증 절차는 플랫폼마다 다르므로** Step 4의 플랫폼 에셋 사양 표를 따릅니다.

**Step 5 (에셋 다운로드 및 처리):** 유형 A는 Figma REST API로 받아 플랫폼 리소스 형식으로 변환·배치하고, **유형 B는 SVG를 다운로드한 뒤 `Read`로 열어 동적 비주얼 파라미터(stroke-width, 색상, opacity, 도형 유형)를 확인**합니다. design_context에서 "추론 가능"하다고 판단하여 이 단계를 생략하면, 전체 원 vs 270° arc, stroke-width 18 vs 16 같은 차이를 잡을 수 없어 코드를 재작성하게 됩니다. 플랫폼별 변환·배치 규칙은 Step 4의 플랫폼 에셋 사양 표를 따릅니다.

**Step 6 (코드 변환):** Step 2의 React 코드를 네이티브 UI로 **1:1 변환**합니다. 스크린샷을 보고 새로 짜는 것이 아니라, React의 컴포넌트 계층·Tailwind 수치를 그대로 옮기는 것입니다. Step 4~5 산출물이 없으면 에셋 참조가 불가능합니다. 플랫폼·프레임워크별 변환 규칙은 `references/<platform>/` 하위 문서를 참조합니다.

**Step 7 (검증):** Step 3의 스크린샷과 Step 5의 SVG 파라미터를 기준으로, 구현 결과의 시각적 일치와 디자인 시스템 규칙 준수를 검증합니다. 앞 단계를 건너뛰면 검증 기준 자체가 없습니다.

---

## Required Workflow

**아래 Step을 순서대로 실행하세요. 절대 건너뛰지 마세요.**

---

### Step 0: 플랫폼·프레임워크 판별

다음 **우선순위**로 변환 대상 플랫폼·프레임워크를 정합니다. 이 결과로 Step 4·5·6 분기가 결정됩니다.

1. **사용자 프롬프트 힌트 우선:** 프롬프트에 프레임워크가 명시되면(예: "SwiftUI로", "Compose로 변환", "XML 레이아웃으로") 그대로 따른다.
2. **변환 대상 화면 파일 기준 자동 판별:** 힌트가 없으면 프로젝트 코드를 검색한다(아래 신호 표). **판별 단위는 모듈 전역이 아니라 "변환 결과를 넣을 대상 화면/파일"이다.** 대상 화면이 단일 프레임워크면 — 프로젝트 전체가 Compose+XML 또는 SwiftUI+UIKit으로 혼재하더라도 — 대상 화면의 프레임워크로 확정한다. (전역 grep으로 먼저 잡힌 프레임워크로 임의 확정하지 않는다.)
3. **모호 시 질문 (임의 선택 금지):** 대상 화면이 불명확하거나(어디에 넣을지 미지정) 단일 프레임워크로 좁혀지지 않으면 — **사용자에게 어느 플랫폼·프레임워크로 변환할지 묻는다.**

| 플랫폼 | 판별 신호 | 프레임워크 분기 |
|--------|-----------|------------------|
| **Android** | `build.gradle(.kts)`, `AndroidManifest.xml`, `*.kt` | Compose(`@Composable`/`setContent`) → `references/android/compose.md` · XML(`*.xml` 레이아웃/`findViewById`) → `references/android/xml.md` |
| **iOS** | `*.xcodeproj`/`*.xcworkspace`, `Package.swift`, `*.swift` | SwiftUI(`View`/`some View`/`@main App`) → `references/ios/swiftui.md` · UIKit(`UIViewController`/storyboard) → `references/ios/uikit.md` |

---

### Step 1: Node ID 추출

Figma URL에서 file key와 node ID를 추출합니다.

**URL 형식:** `https://figma.com/design/:fileKey/:fileName?node-id=1-2`

**추출:**
- **File key:** `:fileKey` — `/design/` 뒤의 세그먼트
- **Node ID:** `1-2` — `node-id` 쿼리 파라미터 값

**예시:**
- URL: `https://figma.com/design/<FILE_KEY>/<FILE_NAME>?node-id=1234-5678`
- File key: `<FILE_KEY>`
- Node ID: `1234-5678`

> **표기 주의:** MCP 도구(`get_design_context` 등)는 **하이픈** 형식(`1234-5678`)을, 에셋 다운로드 스크립트는 **콜론** 형식(`1234:5678`)을 받는다. Step 5 다운로드 시 하이픈을 콜론으로 바꿔 전달한다.

---

### Step 2: 디자인 컨텍스트 확보

MCP 도구로 디자인 정보를 수집합니다. **아래 호출들은 병렬 실행 가능합니다.**

#### 병렬 호출: get_design_context + get_variable_defs

```javascript
// 전체 UI 코드 (React+Tailwind 참고용)
get_design_context(nodeId="1234-5678")

// 색상·타이포 디자인 토큰 (project-design-system 매핑에 필수)
get_variable_defs(nodeId="1234-5678")
```

#### 선택: get_code_connect_map

```javascript
// 기존 컴포넌트 재사용 확인 (Code Connect 설정된 프로젝트에서만 유효)
get_code_connect_map(nodeId="1234-5678")
```

**확보 목표:**
- `design_context` → Step 4에서 에셋 분류, Step 6에서 코드 변환 원본
- `variable_defs` → Step 6에서 색상 토큰 매핑에 사용
- `code_connect_map` → Step 6에서 기존 컴포넌트 재사용 판단

**truncated 처리:** design_context가 잘리면:
1. `get_metadata(nodeId="1234-5678")`로 자식 노드 목록 파악
2. 주요 섹션별로 `get_design_context(nodeId=":childNodeId")` 재호출

---

### Step 3: 시각 참조 캐처

디자인의 스크린샷을 캐처하여 시각적 참고 자료로 사용합니다. **Step 2와 병렬 호출 가능합니다.**

```javascript
get_screenshot(nodeId="1234-5678")
```

이 스크린샷은 구현 전체에 걸쳐 시각 검증의 기준(source of truth)이 됩니다. 구현 완료까지 참조하세요.

---

### Step 4: 에셋 분류 ★ (공통 골격 + 플랫폼 분기)

Figma MCP가 반환하는 localhost 이미지 URL은 세션 종료 시 만료되므로, 에셋을 분류하여 적절히 처리합니다.

design_context의 모든 시각 요소를 확인하여 세 유형으로 분류합니다 — 이 **A/B/C 분류 개념은 플랫폼 공통**입니다.

**주의: 시각 요소는 두 계층에 존재한다.** `const img*` 변수(직접 `<img>` 참조)만 보면 안 된다. React 컴포넌트(`IconToolbar`, `IconEtc` 등)가 내부적으로 `<img>`를 렌더하는 경우, 메인 컴포넌트에서 해당 컴포넌트를 호출하는 것 자체가 시각 요소다. 컴포넌트 호출을 놓치면 최종 화면에서 아이콘이 통째로 빠진다.

| 유형 | 판단 기준 | 예시 | 처리(개념) |
|------|-----------|------|-----------|
| **A. 정적 에셋** | 아이콘·일러스트 — 형태가 데이터와 무관하게 고정 | 날씨 아이콘, 로고, 배경 이미지 | 다운로드 → 플랫폼 리소스로 변환·배치 |
| **B. 동적 비주얼** | 그래프·게이지·차트 — 데이터에 따라 모양이 변화 | 온도 그래프 선, 미세먼지 게이지 arc, 진행률 바 | SVG 다운로드 → 파라미터 추출 → 코드로 직접 그림 |
| **C. 기존 에셋** | 프로젝트 리소스에 동일 용도가 이미 존재, **또는 iOS SF Symbols 시스템 심볼로 대체 가능** | 공통 아이콘, `xmark` 같은 표준 심볼 | Skip (재사용, 내용 검증 필수) |

**공통 분류 절차:**
1. **기존 리소스 목록 확보 (1회):** 플랫폼 리소스 디렉터리의 전체 파일명 목록을 먼저 수집한다. (디렉터리는 아래 플랫폼 에셋 사양 표 참조)
2. **두 계층의 시각 요소를 모두 나열한다:**
   - **계층 1 — `const img*` 변수:** design_context 상단의 이미지 URL 변수 목록
   - **계층 2 — 컴포넌트 호출 인벤토리:** 메인 export 함수에서 호출하는 모든 자식 컴포넌트를 재귀적으로 추적한다. 각 컴포넌트의 props에서 어떤 `<img>`가 렌더되는지 design_context의 컴포넌트 정의를 따라가며 확인한다. **계층 1과 매칭되지 않는 컴포넌트 호출이 있다면, 그 컴포넌트가 시각적으로 무엇을 렌더하는지 반드시 파악한다.**
3. 각 시각 요소의 `data-name`과 Step 3의 screenshot를 대조한다.
4. screenshot에서 해당 영역이 "데이터에 따라 형태가 바뀌는가?"를 판단한다.
   - 예 → **유형 B**
   - 아니오 → 기존 리소스(프로젝트 리소스 + **iOS는 SF Symbols 시스템 심볼 포함**)에 동일 용도의 것이 있는가? → **Type C 검증 절차**(아래) 통과 시 유형 C, 실패 시 유형 A
     - **iOS 단색 아이콘**은 먼저 SF Symbols(`xmark` 등) 대체 가능 여부를 확인한다 — 적합한 시스템 심볼이 있으면 다운로드 없이 유형 C로 처리(Glob `*.xcassets`에는 안 잡히므로 이 분기를 명시적으로 태운다).

**Type C 검증 절차 (필수, 공통 원칙):**
파일명이 비슷하다고 Type C로 판정하면 안 된다. 반드시 후보 리소스의 **실제 내용**(형태·색상)을 열어 Step 3 스크린샷과 대조하고, 불일치하면 **유형 A로 재분류**한다. 리소스 형식별 확인 방법은 아래 표를 따른다.

**플랫폼 에셋 사양 (Step 4·5 공용):**

> **에셋 최종 형식은 `project-design-system`을 우선 따른다 (Android·iOS 공통).** 프로젝트의 PDS(Step 6)에 에셋 형식 규칙(예: iOS `PNG @1x/2x/3x`)이 명시돼 있으면 그대로 따르고, 없으면 아래 표의 기본 변환 스크립트를 쓴다. 기본과 다른 배율·형식이 필요하면 로컬 업스케일 대신 Figma에서 해당 배율로 재추출한다(REST `format=png&scale=1|2|3`).

| 항목 | Android | iOS |
|------|---------|-----|
| 기존 리소스 목록 (Glob) | `**/res/drawable*/*.xml`, `**/res/drawable*/*.webp` (qualifier 폴더 `drawable-*` 포함) | `**/*.xcassets/**` (+ 단색 아이콘은 SF Symbols 대체 검토) |
| 유형 A 변환·배치 (기본값) | `scripts/android/convert_assets.sh` → `res/drawable/ic_*.xml`(VectorDrawable)·`img_*.webp` | `scripts/ios/convert_assets.sh` → `Assets.xcassets/<name>.imageset`(PDF) |
| 유형 C 실제내용 확인 | drawable XML의 pathData·fillColor·stroke | imageset 이미지 / SF Symbol 형태 |
| 설치 의존성 | vd-tool · webp · JAVA_HOME | librsvg / cairosvg / inkscape (택1) |

> 유형 B(동적 비주얼)를 코드로 그리는 방법은 플랫폼·프레임워크마다 다르므로 Step 6 코드 reference(compose/xml/swiftui/uikit)를 따른다.

**산출물:** 분류 테이블 (변수명 | 유형 | 대상 node_id | 비고)

---

### Step 5: 에셋 다운로드 및 처리 ★ (플랫폼 분기)

> **위임:** [`figma-asset-download` 스킬](./../figma-asset-download/SKILL.md) — 다운로드 자체는 플랫폼 무관.

Step 4에서 **유형 A와 B로 분류된 node ID만** 다운로드한다. 유형 C는 기존 리소스를 재사용하므로 다운로드하지 않는다.

**중요: 유형 A와 B는 반드시 별도 디렉터리에 다운로드한다.** 같은 디렉터리에 섞으면 정리 시 유형 A가 유형 B와 함께 삭제되는 사고가 발생한다.

```bash
# 유형 A: 정적 에셋
bash "${CLAUDE_SKILL_DIR}/../figma-asset-download/scripts/download_figma_frame_images.sh" \
  <file_key> /tmp/figma_type_a <type_a_node_id_1> <type_a_node_id_2> ...

# 유형 B: 동적 비주얼 (파라미터 참조용)
bash "${CLAUDE_SKILL_DIR}/../figma-asset-download/scripts/download_figma_frame_images.sh" \
  <file_key> /tmp/figma_type_b <type_b_node_id_1> <type_b_node_id_2> ...
```

다운로드 후 처리:
- **유형 A** → 플랫폼 리소스 형식으로 변환·배치. 변환 스크립트·배치 경로·의존성은 **Step 4의 플랫폼 에셋 사양 표**를 따른다. **변환 도구(vd-tool, rsvg 등)를 직접 호출하지 말고 표가 지정한 스크립트를 사용할 것.** 변환·배치 완료 후 `/tmp/figma_type_a`는 삭제한다.
- **유형 B** → 다운로드 직후 **각 SVG를 `Read`로 열어 파라미터(도형 유형, stroke-width, 색상, opacity, path data)를 확인한다.** design_context나 스크린샷에서 추측하지 않는다 — SVG 원본이 유일한 파라미터 소스이다. 이 단계에서 삭제하지 않는다(코드로 그린 뒤 Step 6에서 삭제).

**유형 A 변환 스크립트 호출** (다운로드 포맷은 자동: 벡터→SVG, 래스터→PNG. 두 스크립트 모두 두 포맷을 처리):

```bash
# Android: SVG→VectorDrawable XML, PNG→WebP
bash "${CLAUDE_SKILL_DIR}/scripts/android/convert_assets.sh" /tmp/figma_type_a app/src/main/res/drawable

# iOS: SVG→PDF, PNG→imageset (둘 다 Asset Catalog .imageset 생성)
bash "${CLAUDE_SKILL_DIR}/scripts/ios/convert_assets.sh" /tmp/figma_type_a <App>/Assets.xcassets
```

유형 A 리소스가 모두 배치되고, 유형 B 파라미터를 확인한 뒤 Step 6으로 진행합니다.

---

### Step 6: 플랫폼별 코드 변환 ★ (플랫폼 분기)

프로젝트의 `project-design-system/SKILL.md`를 읽고 적용한다. Claude Code는 `.claude/skills/`에서 찾고, Codex는 `.agents/skills/`에서 먼저 찾은 뒤 없으면 기존 `.claude/skills/` 규칙을 재사용한다.

Step 0에서 판별한 플랫폼·프레임워크의 reference를 `Read`로 읽고 따른다:

| 플랫폼 | 프레임워크 | reference |
|--------|-----------|-----------|
| Android | Jetpack Compose | `${CLAUDE_SKILL_DIR}/references/android/compose.md` |
| Android | XML Layout | `${CLAUDE_SKILL_DIR}/references/android/xml.md` |
| iOS | SwiftUI | `${CLAUDE_SKILL_DIR}/references/ios/swiftui.md` |
| iOS | UIKit | `${CLAUDE_SKILL_DIR}/references/ios/uikit.md` |

Step 2의 React 컴포넌트 계층·Tailwind 수치를 그대로 네이티브 코드로 1:1 변환한다.

---

### Step 7: Figma 대조 검증

구현 완료 전, Step 3의 스크린샷(이미 컨텍스트에 있음)과 작성한 코드를 **요소별로 직접 대조**하여 검증합니다.

**체크리스트를 나열만 하고 끝내면 안 된다.** 각 항목에 대해 스크린샷의 어떤 부분과 코드의 어떤 부분을 비교했는지, 일치/불일치 여부를 구체적으로 기술해야 한다.

**체크리스트 (플랫폼 공통):**
- [ ] **스크린샷 시각 대조**: Step 3 스크린샷을 직접 보면서, 화면의 각 영역(헤더, 카드, 아이콘, 텍스트 등)이 구현 코드의 UI 요소와 1:1로 대응되는지 확인. 누락된 요소나 잘못 매핑된 에셋이 있으면 기록
- [ ] **컴포넌트 호출 전수 대조**: design_context의 메인 export 함수에서 호출하는 모든 자식 컴포넌트(래퍼 포함)가 구현 코드에 대응하는 UI 요소가 있는지 확인
- [ ] 레이아웃 일치 (간격, 정렬, 크기 — React 코드의 수치와 비교)
- [ ] 타이포그래피 일치 (폰트, 크기, 굵기, line height)
- [ ] **스크롤/overflow 동작:** 고정 높이를 넘기는 리스트가 실제로 스크롤되는가 — 스크롤 축에 bounded 제약 확보(Compose `verticalScroll`/`LazyColumn`, XML `ScrollView`/`RecyclerView`, SwiftUI `ScrollView`, UIKit `UIScrollView`/`UICollectionView`). 잘리거나 스크롤 안 되는 영역 없음
- [ ] 유형 A 에셋 파일 존재 (플랫폼 리소스 경로에 배치됨)
- [ ] 유형 B 동적 비주얼 파라미터가 SVG 원본과 일치 (stroke-width, arc 각도, gradient)
- [ ] 유형 C 매핑 정확성: 기존 리소스의 **실제 형태**가 스크린샷의 해당 아이콘과 일치하는지 재확인
- [ ] 동적·정적 에셋 오판 없음 (데이터 기반 시각 요소는 반드시 코드로 직접 그림)
- [ ] project-design-system 규칙 준수 (hex 하드코딩 없음)

플랫폼 reference에 추가 검증 항목이 있으면 함께 수행한다.

---

<!--
Design-To-UI
Copyright (c) 2026-present NAVER Corp.
Apache-2.0
-->
