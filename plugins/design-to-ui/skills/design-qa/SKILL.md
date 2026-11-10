---
name: design-qa
description: >-
  빌드된 앱 화면을 실기기/시뮬레이터에서 캡처해 Figma 원본과 오버레이(50% 블렌드 + diff 히트맵 + figma|real 나란히 크롭 + 픽셀색 비교)로 대조하고, 코드 오차로 확정된 항목을 Figma 선언값으로 되짚어 스스로 보정하는 검증 루프. 비교·측정 엔진은 플랫폼 중립이고 캡처 계층만 플랫폼별이다(Android·iOS·Web 지원). 두 진입: ① codegen Step 7c 위임 호출, ② 직접 호출(/design-qa) — 프롬프트로 검증/보정 모드를 판별하고 Figma 링크 해소·현재 브랜치 빌드·캡처까지 스스로 오케스트레이션. "오버레이 검증", "실기기 대조", "figma랑 겹쳐봐", "design-qa", "어디가 틀렸는지 보정", "Step 7c" 같은 요청에 사용.
license: Apache-2.0
metadata:
  author: NAVER
  version: "3.4"
  platform: [android, ios, web]
---

# design-qa — 오버레이 검증 루프

Codex에서는 본문과 플랫폼 문서의 `scripts/`를 이 `SKILL.md`가 있는 디렉터리 기준으로 해석한다. 명령의 `${CLAUDE_SKILL_DIR}`에는 그 절대 경로를 사용한다.

codegen의 "LLM 눈대중 대조"를 **실기기/시뮬레이터 런타임에서 수치로** 수행하는 보완 트랙이다. JVM
(layoutlib) 정량 트랙과 달리 **실 렌더엔진·다크모드·디바이스 density**가 그대로 반영된다 — 둘은 경쟁이
아니라 병행이다.

**비교·측정 엔진(overlay·align_probe·glyph_probe·edge_probe·enumerate_regions)과 crop 의 `auto`/`box`/
`anchor`/정합 게이트는 플랫폼 중립이다. 플랫폼에 종속되는 건 뷰포트 정규화(0단계)와 캡처, dumpsys `frame`
crop, "고치는 법(코드 어휘)"뿐**이다. 그래서 이 문서는 루프·판정·종료 계약을 중립으로 기술하고,
플랫폼별 명령·보정 어휘는 `references/<platform>.md`로 분리한다(`android.md`, `ios.md`, `web.md`).

## 두 진입 — 빌드·진입·캡처는 공유

목표는 **인간 개입 최소화** 자율 검증이다. 빌드·화면 진입·캡처는 진입과 무관하게 design-qa가 스스로 한다.
두 진입은 *앞단 입력을 어떻게 얻느냐*만 다르다.

- **위임 진입** (codegen Step 7c): `file_key`·`node_id`·crop 인자 + 빌드/실행 정보가 넘어옴 →
  0단계 건너뛰고 "빌드·화면 진입"부터 자율 수행.
- **직접 진입** (`/design-qa <프롬프트>`): 입력이 자연어뿐 → 0단계(입력 해소)로 모드·링크를 먼저 정한 뒤 합류.

## 0단계 — 입력 해소 (직접 진입 전용)

**0a. 모드 판별. 기본값은 보정 모드(자율 루프).**
- 인자 없는 `/design-qa` 또는 애매한 프롬프트 → **보정 모드**: 검증→보정→재빌드→최종 오버레이까지 묻지 않고 자동.
- 오차 서술("틀렸/밀렸/색이/폰트/위치/간격/크기가 …")이 있으면 → **보정 모드**, 그 영역 우선 조준.
- "오버레이만/겹쳐봐/확인만" 신호가 명시되면 → **검증 모드**(blend-only 표시, 코드 미수정).

**0b. Figma 링크 해소.** ① 프롬프트의 URL → ② 없으면 **세션 직전 링크 재사용** → ③ 둘 다 없을 때만 요청.
확보한 URL의 `file_key`·`node_id`를 추출하고 **세션에 유지**해 후속 호출이 ②로 재사용하게 한다.

## 빌드·화면 진입 (양 진입 공유, 자율)

- **빌드:** 타겟 앱을 **지금 체크아웃된 브랜치 그대로** 빌드한다(보정 루프의 코드 수정도 현재 트리에 적용 후 재빌드).
- **화면 진입 + 캡처 fallback (인간 개입은 최후수단):** 대상 화면 포그라운드 진입을 자동 시도한다. 자율 진입이
  끝까지 막힐 때(프로젝트 의존 로그인 등)**에만** 안내 후 사용자 신호를 받아 캡처한다 — 성공하면 신호 없이 진행.

**산출물 위치(outdir) — 기본은 Desktop.** 기본 outdir는 `$HOME/Desktop/design-qa/{검수화면}/`(예:
`$HOME/Desktop/design-qa/날씨-엔드/`)이고, 캡처·크롭 중간물도 같은 디렉터리에 둔다. `{검수화면}`은 node id가
아니라 **사람이 알아보는 화면 이름**(figma 프레임명 등, 공백은 `-`)으로 쓴다 — Desktop에 폴더가 쌓였을 때 어느
검수 결과인지 이름만으로 구분돼야 한다. 이 루프의 마지막 동작이 "사람이 이미지를 눈으로 확인"이라서 산출물은
Finder에서 바로 열리는 자리에 있어야 한다 — `/tmp`는 사용자가 찾아가기 어렵고 정리·재부팅으로 사라진다.
사용자가 경로를 명시하면 그 경로를 따른다.

**마무리는 호출 종류로 갈린다:**
- **직접/자율 호출:** 수렴 후 outdir를 **`blend50.png` 한 장만 남기고 정리** → `open`으로 띄우고 종료. 사람은 이미지 한 장만 본다.
- **위임 보정:** full rubric 산출물 + `OVERLAY-REPORT.md`를 **유지**(위임자가 코드 오차 리스트·리포트 경로를 소비). `open` 불필요.

## 플랫폼 캡처 계층 (유일한 플랫폼 종속)

캡처와 exact-crop만 플랫폼별 구현이다(측정·비교는 중립). 플랫폼은 0단계/위임 인자로 판별한다.

- **Android** → `references/android.md` (adb 캡처·dumpsys frame crop·Compose 보정 어휘)
- **iOS** → `references/ios.md` (simctl 캡처·시뮬레이터 bounds crop·SwiftUI/UIKit 보정 어휘)
- **Web** → `references/web.md` (Playwright Storybook 요소 캡처·`fonts.ready`·trim·CSS/React 보정 어휘·`run.mjs` node-map 오케스트레이션)

**exact-crop 원칙 (중립 — 이 루프의 핵심).** 풀스크린을 좌표 추측으로 자르면 ~15px 오프셋 artifact로 "틀어짐"
오판이 난다. 반드시 **대상 창의 실제 경계로 정확히 crop**한다. 창이 신뢰할 스크린 좌표를 안 주는 환경
(임베디드/멀티윈도 호스트 — dumpsys가 로컬좌표만 줌)에선 **수동 좌표 추측 금지**, figma 기준 래스터를 템플릿으로
위치를 자동 매칭하는 **content 기반 crop**(플랫폼 crop의 `auto`)을 쓴다. crop 후 두 게이트로 검산한다:
`overlay.py`의 `resize_ratio` — **양축이 같은 배율(예 `[3.0,3.0]`, iOS @3x 등)이면 density 정규화(정상, 왜곡 아님)**,
**두 축이 서로 다르면(예 `[1.0,0.94]`) 그 축에 stretch(환경 아티팩트)** → 판정에서 분리하고,
**전역 오프셋(dx,dy)이 임계(±3px) 초과면 crop 박스가 어긋난 것**(요소별 오차 아님) — crop을 다시 맞춘다.

⚠️ **이 두 게이트는 픽셀만 본다 — 0단계(dp 정규화)를 대체하지 못한다.** dp 가 달라도 density 가 상쇄해
**둘 다 초록불이 난다**(실측 360dp vs 411dp: `정합 게이트 OK` + `resize_ratio [1.0, 1.0256]`). 게이트 통과를
"정합됨"의 근거로 쓰기 전에 `overlay.py --figma-dp/--real-dp` 로 dp 를 검산한다 — 이 플래그는 opt-in 이라
**넘기지 않으면 검사 자체가 없다.** 게이트 초록불 + dp 불일치가 이 루프에서 가장 비싼 오판이다.

## 워크플로우

0. **뷰포트 dp 정규화 (플랫폼 공통 · 캡처보다 먼저 · 생략 불가)** — Figma 프레임의 **dp 논리 크기**를
   `get_metadata`/`absoluteBoundingBox`로 읽고, 캡처 대상의 dp 를 **거기에 맞춘다.**
   dp 가 다르면 레이아웃이 **다르게 계산**되므로(줄바꿈·wrap·분포) 어떤 배율로도 되돌릴 수 없는데,
   **정합 게이트들은 통과해버린다** — dp 차이를 density 차이가 상쇄하기 때문(실측 360dp 프레임 vs 411dp
   기기: `정합 게이트 OK` + `resize_ratio [1.0, 1.0256]` + 유령 drift 7개 + `mean_diff 8.55`. dp 를 맞추면
   같은 화면이 `[1.0, 1.0]` / drift 0개 / `1.45`).

   **결정 순서(3단 사다리).** ① 프레임 dp 와 같은 논리 크기의 기기/AVD 가 있으면 그것을 쓴다 → ② 없으면
   size 와 density 를 **함께** 바꾼다(둘 중 하나만 바꾸면 못 맞춘다) → ③ 둘 다 불가면 **중단하고 사용자에게
   알린다.** ⚠️ ③ 을 crop/배율로 흡수하려 하지 말 것 — 그게 억지 정합이고, 오버레이 자체가 성립하지 않는다.
   - **Android:** `scripts/viewport.py apply <W>x<H> --freeze --theme <dark|light>` 가 ①→② 사다리·배율 계산·
     적용 후 검증·프레임버퍼 대기를 모두 하고, 맞출 수 없으면 ③ 으로 비정상 종료한다(계산만 `plan`, 현재값
     `verify`). 루프 종료 시 `viewport.py reset` **필수**. 접히는 기기·비동기 리사이즈 등 세부는
     `references/android.md`. 정규화 후에는 **앱을 재시작**한다 — 살아 있는 액티비티는 이전 설정으로 이미
     측정·배치돼 있다.
   - **iOS:** 프레임과 같은 pt 크기의 시뮬레이터 기종 선택 → `references/ios.md`.
     iOS 에는 **사다리 ②가 없다**(논리 크기를 덮어쓸 수단이 없음) → 맞는 기종이 없으면 곧바로 ③(중단).
   - **Web:** Playwright `viewport` 를 프레임 CSS px 로 → `references/web.md`.

   dp 외에 **같이 고정할 변수**(안 하면 줄바꿈·합성 프레임이 흔들려 재현성이 떨어진다): 접근성 폰트 배율,
   테마(프레임의 다크/라이트에 맞춰 전환 — 한쪽으로 고정하면 나머지 절반이 미검증으로 남는다), 애니메이션,
   목/데모 데이터(문자열 길이가 바뀌면 줄바꿈과 오차 위치가 전부 바뀐다). Android 는 앞의 `--freeze
   --theme` 가 폰트 배율·애니메이션·테마를 함께 처리한다(원복 포함). 나머지 플랫폼은 references 참조.

1. **캡처** — 플랫폼 capture(→ references). 풀스크린 이미지. **`capture.py` 를 쓴다** — 잠든 기기(전면 검정인데
   크기가 맞아 dp 게이트도 PNG 검사도 통과한다)와 멀티 디스플레이 PNG 깨짐을 이미 처리한다. 직접
   `screencap` 을 쓰면 그 둘을 직접 해야 한다.
   ⚠️ **단색 검사는 "다른 화면"을 통과시킨다** — 런치 실패로 런처 홈이 찍히면 dp·extent·resize 게이트가
   전부 ok 인 채 `mean_diff 57.21` 만 남는다. `capture.py --expect-package <pkg>` 로 포그라운드를 검증하고,
   조용히 실패하는 `monkey` 대신 `am start -W -n` 으로 진입한다. **`--expect-package` 는 opt-in 이라
   넘기지 않으면 검증 자체가 없다 — 항상 넘긴다.**
2. **exact-crop** — 대상 창 실경계로. **임베디드/멀티윈도 호스트(dumpsys가 로컬좌표만)는 좌표 추측 금지 →
   중립 `crop.py auto <cap_full.png> figma.png <real.png>`**(figma 템플릿 content 매칭)라
   **Step 3(figma)을 먼저** 받아 넘긴다. 창이 실좌표를 주면 플랫폼 `frame`(→ references), 확정 박스 재사용은 `box`/`anchor` — 이 경우 `--figma`로 정합 게이트.
   ⚠️ `auto` 의 스케일 후보 서치는 **0단계를 대체하지 않는다** — dp 가 다르면 auto 는 오프셋을 만들어
   억지 정합하고 성공으로 보고한다(실측: 상단 50px 절단). 0단계를 건너뛴 상태에서 `auto` 를 쓰지 말 것.
3. **Figma export + metadata 저장** — MCP `download_assets(fileKey, nodeId, defaultFormat="png",
   defaultScale=k)` 로 받는다. `k` 는 0단계에서 고른 배율과 **같은 값**이어야 real 캡처와 픽셀 1:1 이다
   (360×780dp @k=3 → 1080×2340px). 응답의 `export.url` 을 그대로 내려받아 `$OUT/figma.png` 로 둔다:
   ```bash
   curl -sL -o "$OUT/figma.png" "<export.url>"     # MCP 응답이 지시하는 회수 방법. URL 은 단명한다
   ```
   ⚠️ **`get_screenshot` 은 이 용도로 못 쓴다** — 이미지를 **인라인으로만** 돌려줘 파일이 안 나오고,
   `maxDimension` 인자도 **없다**(파라미터는 `nodeId`/`contentsOnly` 뿐, 반환은 1x). 눈으로 훑을 때만 쓴다.
   **`get_metadata` 응답은 `$OUT/figma_meta.xml` 로 저장한다** — 0단계의 프레임 dp 와
   3b 의 region 열거가 이 파일을 입력으로 쓴다(0단계에서 이미 읽었다면 같은 응답 재사용, MCP 재호출 불필요).
   - **`enumerate_regions.py` 는 MCP XML·REST JSON 둘 다 받는다** — 응답을 그대로 저장하면 된다. 색
     기대값(`--emit color`)은 `fills` 가 있는 REST 쪽에서만 나온다.
   - ⚠️ **export 의 라운드 코너는 투명이다** — RGB 로 그냥 변환하면 투명이 **검정**이 되어 diff 가 ~245 씩 나고
     정합 정렬까지 끌어당긴다(실측 dy+16 오보고). `overlay.py`·`crop.py` 는 알파를 배경색으로 합성하고 투명
     픽셀을 통계에서 뺀다 — **다른 도구로 비교할 때만** 직접 flatten 할 것.
3b. **probe region 열거 (오버레이보다 먼저 · 손으로 만들지 말 것)** — `get_metadata` 응답을 **파일로 저장**한 뒤
   `enumerate_regions.py <meta.json> --scale <k>` 로 검사 대상을 **기계 생성**한다. 출력의 `counts` 가 각 ledger
   행의 **열거 수**이고, 이 수가 커버리지 근거가 된다(`ledger_gate.py` 가 검사 수와 대조).
   ```bash
   python3 scripts/enumerate_regions.py "$OUT/figma_meta.json" --scale 3 > "$OUT/regions.json"
   GLYPH=$(python3 scripts/enumerate_regions.py "$OUT/figma_meta.json" --scale 3 --emit glyph)
   EDGE=$( python3 scripts/enumerate_regions.py "$OUT/figma_meta.json" --scale 3 --emit edge)
   TEXT=$( python3 scripts/enumerate_regions.py "$OUT/figma_meta.json" --scale 3 --emit text)
   ```
   **왜 필요한가.** region 을 `overlay` 의 `suspect_regions`(면적 평균 top-N)를 보고 손으로 만들면 ① 같은 화면을
   두 번 돌려도 목록이 달라지고("probing 이 될 때도 안 될 때도 있다"의 정체는 신뢰도가 아니라 **커버리지**다),
   ② 작은 아이콘·얇은 선은 애초에 top-N 에 올라오지 않아 **구조적으로 검사에서 빠진다.** 열거를 화면 내용으로
   결정하면 세션 간 재현성이 생긴다.
4. **오버레이 + 판정** — `overlay.py figma.png real.png <outdir> --figma-dp WxH --real-dp WxH`:
   - **dp 게이트를 항상 켠다.** 두 dp 를 넘기면 불일치 시 FAIL 로 멈춘다(0단계 누락의 기계적 방어선).
     `--real-dp` 값은 Android `viewport.py verify` 가 그대로 찍어준다.
   - 검증 모드(빠른 표시): `--blend-only` → `blend50.png` 한 장, 이어서 `Read`로 표시.
   - 보정 모드(자율 루프): `--rubric 3 --grid 12,8 --top 6` → `metrics.json`·`cmp_*`로 진단.
4b. **열거 기반 probe** — 3b 의 region 으로 `glyph_id_probe.py --size-check --scale <k>`(모양+크기),
   `edge_probe.py`(엣지·full-bleed), `glyph_probe.py`(폭/weight), `color_probe.py`(의미색)를 돈다.
   각 probe 가 마지막에 찍는 `# coverage probed=N` 을 ledger 의 `coverage.probed` 로 쓴다.
5. **코드 보정** — 아래 판정·종료 계약으로 확정된 오차만 Figma 선언값으로 교체. 보정 후 2단계부터 재실행.
6. **리포트** — 위임 보정 전용(`OVERLAY-REPORT.md`). 직접/자율 호출은 리포트를 만들지 않는다.
7. **원복** — 0단계에서 기기 설정을 바꿨으면 되돌린다(Android `viewport.py reset`). 루프가 실패로 끝나도 실행한다.

## 판정 — pixel-subtract 단독 금지

diff metric은 **큰 면적·색차·평균**만 본다 → 작은 글리프, 통째 밀린 블록, 의미색, 분포(spread), 글자 폭/
weight, 스케일을 모두 놓친다. **metric이 낮을수록(정합처럼 보일수록) 이 사각이 floor에 그대로 남는다.**
그래서 metric과 **별개로** 아래를 매번 돌린다.

**A. 아티팩트 vs 코드 오차 분리 (먼저).** 다음만 불가역 floor 후보로 두고 나머지는 코드 오차 후보로:
- `resize_ratio` **양축 배율이 서로 다르면** → 패널 stretch(환경). 별도 기록.
  양축 균일 배율은 density 정규화라 정상(왜곡 아님) — **단 real 과 figma 의 dp 논리 크기가 같을 때만.**
  dp 가 다르면 배율이 거의 균일해도(실측 `[1.0, 1.0256]`) **레이아웃 자체가 다른 것**이므로 이 판정을
  신뢰하면 안 된다. `resize_ratio` 는 픽셀 비율이고 dp 차이는 density 차이가 상쇄해 숨는다 → 0단계·dp 게이트.
- 전 글자에 **크기와 무관한 균일 ≤1px halo** → 렌더러 서브픽셀 AA.

**A-1. 두 이미지 크기가 다르면 판정하지 말 것 — 지표가 정답과 반대로 움직인다.** `overlay.py` 가
`extent 게이트`로 멈춘다. dp 게이트는 프레임/기기의 **선언 dp** 를 보므로, 0단계로 dp 를 완벽히 맞춰도
crop 이 시스템 영역(상태바·제스처바·인디케이터존)을 **서로 다르게 처리하면** 통과해버린다 — iOS
프레임(인디케이터존 34dp 포함) ↔ Android 캡처(제스처바 24dp 제외) 대조에서는 거의 항상 그렇다.
그 상태의 resize 는 y 에 비례하는 유령을 만들고 **그 유령이 진짜 오차를 상쇄한다**(실측 393dp 페어:
정합본 `mean_diff 10.26` vs 8dp 오차본 `8.55` — **정합인 쪽이 더 나쁜 숫자**, crop·dp 게이트 둘 다 초록불).
이 잔차는 **어떤 정렬로도 사라지지 않는다**(top-align +10dp / bottom-align −34dp / resize +31dp) —
콘텐츠 영역의 dp 높이가 실제로 다르기 때문이다. 그래서:
- 하단 앵커 요소(플로팅 버튼 등)는 **오버레이가 아니라 dp 절대 probe** 로 판정한다 — 각 이미지를
  **자기 시스템 영역 top 기준**으로 재면 세로 길이가 달라도 성립한다(실측: 8dp 오차를 8.00dp 로 짚었다).
- 픽셀 오버레이는 **비교 가능한 영역만 crop** 해서 돈다. resize 를 없애면 노이즈 플로어가 사라진다
  (실측 상단존: 정합본 `10.26 → 0.00`, 미세결함본 `13.46 → 3.66` — 플로어 위 +31% 였던 신호가 플로어 0 에서 검출).
- **하단 앵커 존을 bottom-aligned 로 분리 합성하는 것은 해법이 아니다.** 두 crop 의 하단 기준면이
  다르면 bottom-align 잔차가 top-align 보다 오히려 크고(−34dp vs +10dp), 분리 지점을 사람이 정하므로
  값이 파라미터에 3배 흔들린다(68/102/306dp → 58.75/78.06/26.02). 기준을 만들어 맞추는 방향 자체가
  A-2 와 같은 함정이다.

**A-2. 지표가 못 보는 것을 지표로 판단하지 말 것.** `pct_over_32` 는 픽셀 diff **32 초과만** 센다. 대비가
그보다 작은 오차는 **굵기·길이와 무관하게 항상 `0.00`** 이다 — 흰 배경(255) 위 `#E5E5E5`(229) 구분선의 diff 는
**26** 이라 원리적으로 안 잡힌다(실측: full-bleed→좌우 16dp 패딩 오차가 `mean_diff 0.0` / `pct_over_32 0.00` /
`max_diff 26` / `align_probe 정렬 OK` 로 **전 검사 통과**). 디자인 시스템의 구분선·보더·비활성색은 대부분 이
대비 구간에 있다. 그리고 그 오차의 셀 평균은 `0.14` 로 floor 밴드(≈3.8)의 1/27 이라 AA 잔차만 있어도
`suspect_regions` top-N 에서 밀려나 `cmp_*.png` 조차 생성되지 않는다. → **이 카테고리는 `edge_probe.py`
(좌표 비교)로만 본다.** 면적 평균 계열로 감도를 얻으려 임계를 낮추면 AA 노이즈가 들어온다.

**B. cmp 직독이 1차 진단.** blend/heatmap은 "어디를 볼지" 후보만 좁히는 보조. 최종 per-element 판정은
`cmp_*.png`(figma|real 나란히)를 **직접 보고** + `suspect_regions` 픽셀색(darkest/mean)을 Figma 기준값과 수치 비교로 한다.
단 `suspect_regions` 는 면적 평균 랭킹이라 **얇은 선·작은 아이콘은 여기 안 올라온다** — 그 둘은 3b 열거 기반
probe 가 담당하고, cmp 직독은 그것을 대체하지 못한다.

## 종료 계약 — floor는 default가 아니라 *증명이 필요한 결론*

**핵심 규율: metric이 floor 밴드(텍스트 화면 ~1.5%, 아이콘·게이지·그래프 화면 ~2–3%)에 들어도 그건
"레이아웃 nudge를 멈출" 신호일 뿐 "끝"이 아니다.** 종료는 아래 **blind-spot ledger의 모든 행이 evidence와
함께 통과**할 때만 성립한다. 미실행 행이 하나라도 있으면 미종료다. 어떤 잔차를 "floor/AA"라 부르려면 **그
행의 probe가 clean**이어야 한다 — metric이 낮다는 이유로 floor라 default하지 말 것.

| blind-spot 카테고리 | 검사(instrument) | floor 선언 가능 조건 (evidence) |
|---|---|---|
| 글리프 매핑·형태 | `glyph_id_probe.py` 형태 IoU(모든 아이콘·소형 포함; 본체 자동 타이트닝) | IoU 임계 이상. 낮으면 **크롭 확인 후** 올바른 노드 재export(회전 대용 ❌) — 텍스트 인접 소형 글리프는 false-positive 주의 |
| **에셋 크기** | `glyph_id_probe.py --size-check --scale <k>` — 정규화 **전** ink bbox 를 **dp 절대 비교** | Δ ≤ ±1dp. ⚠️ **IoU 통과는 크기 정합이 아니다** — `norm_cells()` 가 크기를 정규화로 지우므로 10% 작은 아이콘도 `IoU=1.00` 이 나온다(실측 20dp vs 18dp). `glyph_probe` 의 ±12% **비율** 임계도 10% 를 통과시킨다 → **dp 절대값**이어야 걸린다 |
| **엣지·full-bleed** | `edge_probe.py` — 프레임 변까지 뻗는 얇은 선/보더의 ink **시작·끝 좌표** figma vs real | Δstart·Δend ≤ ±2px. ⚠️ 면적 평균(`mean_diff`·`pct_over_32`)으로는 **원리적으로** 못 본다(A-2) — 이 행을 metric 으로 대신하지 말 것. 좌표 비교는 AA 에 면역(실측 blur 2.0 에서 0px) |
| 글자 폭·weight | `glyph_probe.py`로 텍스트 run figma vs real **advance/stroke 폭** | 개별 ±12% + **전역 편차**(median width_ratio 가 ±0.5% 밖 & 편차 방향 일치) + **국소 이상치**(median 대비 ±3%). ⚠️ 개별 임계만으로는 못 잡는 오차가 있다 — 실측: letterSpacing −0.5% → 전 텍스트 0.988(1.2%), 한글 줄바꿈 차이 → 1.052(5.2%). 전역 편차가 뜨면 픽셀이 아니라 **선언값**(Figma REST `style.letterSpacing`)을 대조한다 |
| 텍스트 의미색 | `color_probe.py`로 의미색 텍스트 대표색 vs figma 토큰 hex | 거리 임계 내 일치 |
| 분포(spread) | 행별 figma `layoutMode`/`layoutGrow`/고정폭 직독 (균등 vs 고정셀) | 규칙 일치 (행 값 복붙 ❌) |
| Canvas·게이지 위치 | 게이지/그래프 영역에 `align_probe.py` + bbox 중심·세로 figma 비교 (벡터 arc는 progress 스칼라 없음 → MCP path 끝점/bbox로 각 역산) | 중심 일치 |
| 레이아웃 위치·크기 | `align_probe.py` best(dx,dy) + 큰 suspect는 **bbox 크기** 직접 비교 | size 일치 & 시프트로 MAE 안 줆 → floor (dy가 fontSize 비례면 텍스트 메트릭=codegen, 위치 아님) |

**종료 게이트 (기계적·필수).** 수렴 선언 전 `ledger.json`(행별 `{verdict: pass|fixed|floor, evidence:
probe출력/crop 경로}`)을 쓰고 `scripts/ledger_gate.py ledger.json`이 **PASS**여야 종료한다 — 미실행 행이나
근거 없는 floor가 있으면 FAIL(종료 불가).

**커버리지도 근거다.** 열거 기반 3행(`glyph_map`·`asset_size`·`edge`)은 `coverage: {enumerated, probed}`
가 **필수**이고 `probed < enumerated` 면 FAIL 한다. `enumerated` 는 3b `enumerate_regions.py` 의 `counts`,
`probed` 는 각 probe 가 찍는 `# coverage probed=N` 이다. 이 필드가 없던 때는 화면에 아이콘이 3개인데 1개만
검사해도 evidence 만 갖추면 PASS 였다 — **빠진 대상은 "검사 안 함"이지 "정합"이 아니다.**

```json
{"categories": {
  "glyph_map":  {"verdict": "pass",  "evidence": "glyph_id_probe: IoU 0.97/0.99/1.00", "coverage": {"enumerated": 3, "probed": 3}},
  "asset_size": {"verdict": "fixed", "evidence": "--size-check: icon Δ-2.00dp → 20dp 선언 교체 후 Δ0.00", "coverage": {"enumerated": 3, "probed": 3}},
  "edge":       {"verdict": "fixed", "evidence": "edge_probe: divider Δstart+48 → padding 제거 후 Δ0", "coverage": {"enumerated": 2, "probed": 2}}
}}
```

게이트 PASS 후 **반증 1패스**(독립 시각이 "한 건은 틀렸다 가정"하고 weight·색·글리프·분포·게이지를 탐색)까지
통과하면 종료. (서브에이전트 "수렴" 자가보고는 호출자가 ledger 근거로 재검증.)

## 보정 — 값 출처 & 픽셀 nudge 금지

확정된 오차만 Figma 선언값으로 교체한다. 보정값 출처 우선순위: **① Code Connect 매핑 구현값 재사용 →
② Figma MCP 직독 교체 — `get_design_context`로 layoutMode·itemSpacing·padding·gap·고정폭·layoutGrow·색
토큰, `get_metadata`로 인스턴스 bbox/box width(마스터≠인스턴스) → ③ 픽셀 엣지 역산 최후수단.** 큰 레버부터: **에셋 글리프(재export) > 레이아웃(선언값 교체) > AA floor.** 작은 letterSpacing·±2px는
전체 diff를 거의 안 움직이니 큰 suspect가 아이콘이면 위치보다 **글리프 재export**가 정답. 재export는
`figma-asset-download`로 올바른 노드 에셋을 받아 **플랫폼별 에셋 변환**(`references/<platform>.md`)을 거친다 — 손 전사 ❌.

- ⚠️ **align_probe dx/dy는 "어디가 틀렸나"만 — 정답값은 figma 선언값 직독** (숫자만 보고 ±N nudge 금지, 악화 실측).
- ⚠️ **부재·드리프트 섹션도 직독 복제 후에만 floor.** 인스턴스 선언값을 직독 교체(눈대중 스페이서 ❌)하고, **직독값이 코드와 이미 일치할 때만** 그 잔차가 floor(렌더 quirk: bbox≠콘텐츠폭·trailing·AA; 프로덕션도 같은 방식이면 안 베낌).

## 측정기 셀프테스트 (스킬을 수정할 때)

이 루프의 스크립트를 고치면 **합성 픽스처로 감도·오탐을 먼저 검사한다** — 에뮬레이터·Figma 토큰·adb 불필요.

```bash
bash scripts/selftest/verify_all.sh        # 통과 시 exit 0
```

검사하는 것: 알려진 사각 3종이 실제로 FLAG 되는가(뷰포트 dp 불일치 / 얇은 옅은색 구분선 엣지 / 아이콘 크기
10%), 기하가 맞는 golden 픽스처에서 **오탐이 없는가**(AA blur 0.4·1.2·2.0), 종료 게이트가 미실행·커버리지
미달 ledger 를 FAIL 하는가, 기존 스크립트 회귀. `crop.py` 는 자체 `selftest_crop.py`(9케이스)도 함께 돈다.

⚠️ **감도를 임계값으로 얻으려 하지 말 것.** 면적 평균 계열(`mean_diff`·`pct_over_32`)의 임계를 낮추면 AA
노이즈가 들어온다. 새 사각을 다룰 때는 **좌표·dp 절대 비교** 쪽으로 probe 를 추가하는 것이 이 스크립트들이
택한 방향이다(좌표 비교는 AA 에 사실상 면역 — 실측 blur 2.0 에서 0px).

## 위임 인터페이스 (codegen Step 7c)

codegen이 호출할 때 넘기는 입력: `file_key`·`node_id`·crop 모드+인자·**빌드/실행 정보**(모듈·
variant·실행 타겟·platform) + (선택) **집중 영역 힌트**. design-qa가 빌드·진입·캡처를 자율 수행하므로 위임자는
앱을 미리 빌드/포그라운드 해둘 필요 없이 이 정보만 넘긴다.

반환: 코드 오차 확정 리스트(분류 포함) + `OVERLAY-REPORT.md` 경로. 위임 보정의 리포트에 담을 것: 페어 메타
(크기·resize_ratio) / 영역별 정합 표(영역·figma·real·판정·근거) / **잔여차를 코드 오차 vs 환경 아티팩트로 분리** /
(반복 시) 라운드별 요약 / blind-spot ledger 결과.

<!--
Design-To-UI
Copyright (c) 2026-present NAVER Corp.
Apache-2.0
-->
