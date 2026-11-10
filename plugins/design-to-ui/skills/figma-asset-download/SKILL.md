---
name: figma-asset-download
description: Figma REST API로 프레임 내 아이콘·이미지를 SVG·PNG로 다운로드하는 범용 스킬. Figma 에셋 다운로드, 아이콘 추출, 벡터/이미지 내보내기가 필요할 때 사용하세요. output_dir만 플랫폼별로 지정하면 iOS·Web·Android 공통으로 사용 가능합니다.
license: Apache-2.0
metadata:
  author: NAVER
  version: "2.0"
  type: workflow
  platform: agnostic
---

# Figma Asset Download

Codex에서는 각 명령의 `${CLAUDE_SKILL_DIR}`를 이 `SKILL.md`가 설치된 디렉터리의 절대 경로로 바꿔 실행한다.

Figma **REST API**를 사용해 지정한 프레임 내 이미지·벡터 노드를 로컬에 SVG 또는 PNG로 다운로드하는 **범용** 스킬입니다.
Figma MCP와 무관하며, `output_dir` 인자만 바꾸면 어떤 플랫폼에서도 사용할 수 있습니다.

## 역할

- 프레임 노드 ID 기준으로 에셋(아이콘·이미지)을 자동 식별하고 Figma Images API로 다운로드합니다.
- **하이브리드 수집 전략**: 1순위 `exportSettings`(디자이너 지정), 2순위 아이콘 휴리스틱(TEXT 없는 INSTANCE/COMPONENT), 3순위 leaf FRAME 벡터 그래픽(TEXT 없음 + VECTOR 있음 + 자식 FRAME 없음)을 사용합니다.
- componentId 기준 중복 제거로 동일 아이콘 인스턴스를 1회만 다운로드합니다.
- 파일명: 노드의 Figma name 속성을 snake_case로 변환하여 `ic_<name>.<format>`로 저장합니다. 비ASCII 이름은 nodeId를 fallback으로 사용합니다.
- 이미지 Fill 노드(`fills[].type == "IMAGE"`)는 `/files/{key}/images` API로 별도 다운로드합니다.

## Prerequisites

**Figma Personal Access Token**이 필요합니다. 토큰이 없으면 스크립트가 실패합니다.

1. [Figma → Settings → Personal access tokens](https://www.figma.com/settings)에서 토큰 발급 (권한에 **file_content:read** 포함)
2. 환경 변수에 설정:

```bash
echo 'export FIGMA_ACCESS_TOKEN="figd_xxx"' >> ~/.zshrc && source ~/.zshrc
```

## 실행 방법

### 사용법 1 — 개별 node ID 지정

분류된 node ID를 직접 지정하여 다운로드합니다. design-to-ui Step 5에서 주로 사용합니다.

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/download_figma_frame_images.sh" \
  <file_key> <output_dir> <node_id_1> [node_id_2] ...
```

| 인자 | 설명 |
|------|------|
| `file_key` | Figma 파일 키 (`figma.com/design/{file_key}/...`에서 추출) |
| `output_dir` | 저장 경로 (예: `/tmp/figma_assets`, `app/src/main/res/drawable`) |
| `node_id_1 ...` | 다운로드할 노드 ID (1개 이상 필수) |

**예시:**

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/download_figma_frame_images.sh" \
  <FILE_KEY> /tmp/figma_assets 1234:5678 1234:5679 2345:6789
```

### 사용법 2 — 프레임 전체 스캔

프레임 하위 전체를 순회하여 에셋을 자동 발견·다운로드합니다.

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/download_figma_frame_images.sh" \
  --scan <file_key> <frame_node_id> <output_dir>
```

| 인자 | 설명 |
|------|------|
| `--scan` | 스캔 모드 활성화 (필수 플래그) |
| `file_key` | Figma 파일 키 |
| `frame_node_id` | 스캔할 프레임 노드 ID |
| `output_dir` | 저장 경로 |

**예시:**

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/download_figma_frame_images.sh" \
  --scan <FILE_KEY> 1234:5678 app/src/main/res/drawable
```

포맷은 노드별로 자동 결정됩니다 (exportSettings 지정 포맷 또는 SVG 기본).

**출력 예시:**

```
ic_icon_weather_xxl.svg   ← 아이콘 INSTANCE (TEXT 없음 → SVG)
ic_icon_weather_m.svg     ← 아이콘 INSTANCE (TEXT 없음 → SVG)
ic_icon_toolbar.svg       ← exportSettings SVG 지정
ic_photo_bg.png           ← exportSettings PNG 지정
```

## 수집 전략 (하이브리드)

```
walk(node):
  1순위: exportSettings가 있는 노드
    → 디자이너가 Figma Export 패널에서 지정한 포맷(SVG/PNG)으로 수집
    → 자식 탐색 중단 (에셋 단위 확정)

  2순위: INSTANCE/COMPONENT 중 TEXT 없음 + VECTOR 있음
    → 순수 아이콘으로 판단, SVG로 수집
    → 자식 탐색 중단

  3순위: leaf FRAME (VECTOR 있음 + TEXT 없음 + 자식 FRAME 없음)
    → 그래프·게이지 등 벡터 그래픽으로 판단, SVG로 수집
    → 자식 탐색 중단

  제외: TEXT가 섞인 레이아웃 프레임 → 자식만 계속 탐색
  제외: VECTOR 없는 단순 도형 (구분선 등) → 코드로 직접 그리기
  제외: 자식에 FRAME이 있는 레이아웃 컨테이너 → 자식만 계속 탐색

  중복 제거: componentId 기준으로 동일 컴포넌트 인스턴스 1회만 수집
```

## 스크립트 동작 흐름

```
1. GET /v1/files/{fileKey}/nodes?ids={nodeId}&depth=10
   → 프레임 자식 노드 트리 조회

2. 하이브리드 수집 (exportSettings 1순위 + 아이콘 휴리스틱 2순위 + leaf FRAME 3순위)
   → name → snake_case 파일명 생성
   → fills[].imageRef 유무로 벡터/이미지Fill 분류
   → componentId 중복 제거

3a. SVG 노드 → GET /v1/images/{fileKey}?ids=...&format=svg
3b. PNG 노드 → GET /v1/images/{fileKey}?ids=...&format=png&scale=2
3c. 이미지 Fill 노드 → GET /v1/files/{fileKey}/images (imageRef)

4. output_dir에 ic_<name>.<svg|png> 저장
```

## 주의사항

- **Figma 임시 URL을 앱 코드에 직접 사용하지 마세요.** `figma.com/api/mcp/asset/...` 등의 URL은 7일 후 만료되므로, 반드시 이 스크립트로 다운로드한 로컬 파일을 사용해야 합니다.
- **SVG/PNG는 플랫폼별로 추가 변환이 필요할 수 있습니다.** 예를 들어 Android는 SVG를 직접 사용할 수 없으므로 Vector Drawable XML로 변환해야 합니다. 변환은 이 스킬의 범위 밖이며, 플랫폼 스킬(예: design-to-ui)에서 처리합니다.

## Related

- **design-to-ui** — Step 5에서 이 스킬의 `download_figma_frame_images.sh`로 다운로드 후, `convert_assets.sh`로 Android 리소스 변환

<!--
Design-To-UI
Copyright (c) 2026-present NAVER Corp.
Apache-2.0
-->
