# Worklog: Day 4 — SVG 템플릿 시스템 + 파라메트릭 그래픽 + UI 개선

**날짜**: 2026-04-01
**커밋 범위**: 150ceb8 ~ eb09b76 (19 커밋)
**이전 워크로그**: 2026-03-28-day3-canvas-preview-graphic.md

---

## 작업 요약

1. 랜덤 패턴 생성(`genPattern`) → SVG 템플릿 시스템으로 근본 재설계
2. 정적 SVG → 파라메트릭 생성 함수로 전환 (Regenerate 시 형태 자체가 변경)
3. 3개 템플릿 각각 고유한 그래픽 언어로 정교화
4. 컬러 팔레트 AI 프롬프트 대폭 개선 (2색 동일톤 + 1색 포인트)
5. 커스텀 카테고리 입력 + 한글→영어 자동 번역
6. 각 단계별 맥락 설명 헤딩 추가, "Preview" → "Visual" 명칭 변경

---

## 1. 설계 과정 (브레인스토밍 → 스펙 → 플랜)

### 문제 인식

Day 3에서 6회 반복한 랜덤 패턴 생성의 근본적 한계를 인정:
- 파라미터 조정만으로는 "의도된 디자인" 느낌을 낼 수 없음
- 디자이너가 직접 만든 검증된 구성만 사용하는 게 정답

### 브레인스토밍에서 확정된 요구사항

| 항목 | 결정 |
|------|------|
| 완성도 | 바로 쓸 수 있는 수준 (최소 Figma 미세 조정) |
| 내용 연결 | 글 내용에 맞는 그래픽 (추상 패턴 vs 구체적 심볼 선택 가능) |
| 다양성 | 글마다 눈에 띄게 달라야 함 |
| 비용 | 무료 또는 거의 무비용 |
| 그래픽 방식 | 랜덤 생성 → **검증된 템플릿 + 파라미터 변주** |

### 검토한 3가지 접근법

| 접근법 | 설명 | 판정 |
|--------|------|------|
| A. SVG 템플릿 라이브러리 | Figma에서 만든 SVG를 파싱해서 사용 | **채택** |
| B. AI SVG 직접 생성 | LLM이 SVG 코드를 출력 | 품질 불안정 → 기각 |
| C. 하이브리드 (A+B) | 골격은 템플릿, 디테일은 AI | 복잡도 높음 → 기각 |

### 설계 문서

- **스펙**: `docs/superpowers/specs/2026-04-01-svg-template-system-design.md`
- **구현 플랜**: `docs/superpowers/plans/2026-04-01-svg-template-system.md`

---

## 2. 초기 구현 (정적 SVG 템플릿 시스템)

### 플랜 8개 태스크를 subagent-driven 방식으로 실행

| Task | 내용 | 커밋 |
|------|------|------|
| 1 | 테스트 SVG 템플릿 3개 생성 | 150ceb8 |
| 2 | SVG 파서 (parseSVG, loadTemplate) | 489fa96 |
| 3 | 변주 로직 (applyVariation: 슬롯 셔플, 미러, 스케일 지터) | a1cb68c |
| 4 | genPattern 제거 → 템플릿 렌더링 교체 | 52130c9 |
| 5 | Strategy AI 프롬프트에 template 선택 추가 | 4d1e869 |
| 6 | 템플릿 선택 UI (칩 바) | 9bcb23c |
| 7 | docs/ GitHub Pages 동기화 | f6793f4 |
| 8 | 한자 사용 금지 프롬프트 | 3797431 |

### 코드 리뷰 후 수정 (7e5f191)

- `snap()` 데드 코드 제거
- `slotFromId`에 unrecognized slot 경고 추가
- AI 반환 template ID 유효성 검증
- `loadAllTemplates()`에 `.catch()` 추가

### file:// CORS 문제 해결 (c433979)

로컬 `file://` 프로토콜에서 `fetch()`로 외부 SVG를 불러올 수 없는 문제 발생.
→ SVG를 JS 인라인 문자열로 포함시켜 해결. `loadTemplate()`을 동기 함수로 변경.

---

## 3. 그래픽 품질 피드백 반영

### 피드백 1: 끝점 정렬 + 슬리버 제거 + 컬러 정교화 (0dd0e42)

**문제**: 스케일 지터(±5%)가 미세 어긋남을 발생시킴. AI 팔레트가 3색 제각각이라 촌스러움.

**그래픽 수정:**
- 3개 SVG 템플릿 좌표 재설계 (뾰족한 영역 제거)
- **스케일 지터 제거** — 변주는 슬롯 셔플 + 좌우 미러링만 유지

**컬러 팔레트 프롬프트 전면 개편:**
- 기존: "Duo-tone / Monochromatic / Analogous 중 선택"
- 변경: **base + accent1 = 같은 색상 계열 (톤온톤) + accent2 = 대비 포인트 컬러**
- HSL 기준 명시: base 명도 ≤30%, accent1 30-50%, accent2는 색상환 60°+ 차이
- 구체적 좋은 팔레트 예시 4개 추가

### 피드백 2: Column Arc를 아크 전용으로 (7af32fc)

**문제**: 사각형 컬럼 + 타원 1개 → Grid Mosaic과 구분 안 됨.

**수정**: 사각형 전부 제거, 원/타원만으로 구성:
- 캔버스 양쪽 가장자리 대형 타원
- 하단에서 올라오는 호, 상단에서 내려오는 호
- 소형 원 악센트 2개

### 피드백 3: Diagonal Cross를 사선 전용으로 (927bcb5)

**문제**: 사각형 배경 위에 사선 2개만 → 사각형 기반처럼 보임.

**수정**: 전체 구성을 대각선 polygon으로만:
- 좌하/우상 대각 삼각형
- 교차 평행사변형 밴드
- 겹치는 사선 스트라이프

---

## 4. 파라메트릭 생성 함수로 전환 (c1c6f6f)

### 문제

정적 SVG + 슬롯 셔플/미러 = 최대 6가지 조합 → Regenerate가 단조로움.

### 해결: TEMPLATE_GENERATORS

정적 SVG 문자열 대신 **시드 기반 파라메트릭 생성 함수**로 교체:

```js
const TEMPLATE_GENERATORS = {
  'grid-mosaic': function(rand) { ... },
  'diagonal-cross': function(rand) { ... },
  'column-arc': function(rand) { ... }
};
```

각 함수가 `mulberry32` 시드 RNG를 받아 매번 다른 형태를 생성:

| 템플릿 | 파라미터화 요소 |
|--------|----------------|
| grid-mosaic | 컬럼 수(3-4), 컬럼 폭 비율, 로우 수(2-3), 로우 높이, 오버랩 악센트 위치/크기 |
| diagonal-cross | 밴드 수(방향별 2), 드리프트 각도, 밴드 폭, 시작 위치, 교차점 다이아몬드 |
| column-arc | 아크 수(3-5), 중심 위치(5가지 소스), 반지름, 소형 원 수(1-2) |

### 삭제된 코드

- `TEMPLATE_SVG` 객체 (인라인 SVG 문자열)
- `parseSVG()`, `loadTemplate()`, `loadAllTemplates()`, `slotFromId()`
- `templateCache` 객체
- `applyVariation()` 함수

→ 모두 `TEMPLATE_GENERATORS` + `generateTemplate()` 함수로 대체.

`generateTemplate(id, seed)`가 생성 함수 호출 + 슬롯 셔플 + 미러링을 한번에 처리.

---

## 5. 템플릿별 후속 정교화

### Column Arc: 겹침 교차 영역 (8c50063)

**추가된 규칙:**
- 타원 쌍마다 정규화 거리로 겹침 여부 판단 (`dist < 1.2`)
- 겹치는 두 타원의 중간점에 작은 교차 타원 배치
- 교차 영역은 부모 두 타원과 다른 세 번째 색상 사용
- 최소 크기 보장 (`irx > 80 && iry > 60`)

벤 다이어그램처럼 겹치면서 깊이감 생기는 효과.

### Diagonal Cross: 양방향 교차 + 슬리버 제거 (17c1618 → ddbdc24)

**1차 (17c1618):**
- 단일 방향 → 두 방향 교차 (좌상→우하 + 우상→좌하)
- 교차 지점에 다이아몬드(마름모) 악센트

**2차 슬리버 수정 (ddbdc24):**
- 밴드 최소 폭 500px 이상으로 증가
- 방향별 2개 고정 + 일정 간격 배치
- 드리프트 랜덤 흔들림 제거
- 다이아몬드 최소 크기 확대

---

## 6. UI/UX 개선

### 커스텀 카테고리 (d3729ac)

- Category 영역에 **Custom** 버튼 추가
- 클릭 시 텍스트 입력 필드 노출
- **한글 입력 → AI가 영어로 번역** (별도 AI 호출 1회 추가)
  - 예: "데이터 엔지니어링" → "Data Engineering"
- 영어 라벨이 커버에 표시됨
- `getCatLabel()`, `getCatContext()` 함수로 기존/커스텀 카테고리 통합 처리

### 맥락 설명 헤딩 (5d29f66)

각 패널 상단에 Decision Flow의 흐름을 설명하는 제목 추가:

| 패널 | 헤딩 |
|------|------|
| 입력 단계 (p1) | **텍스트를 디자인 구조로** |
| 구조 결과 확인 (p1r) | **커버에 들어갈 구조** |
| 비주얼 결과 (p2r) | **구조를 시각 언어로** |

스텝 탭 명칭 변경: "Preview" → **"Visual"**
버튼 라벨: "Preview →" → **"Visual →"**
로딩 텍스트: "Generating Preview" → **"Generating Visual"**

### 무한 재귀 버그 수정 (eb09b76)

`replace_all`로 `CAT_LABELS[cat]` → `getCatLabel()` 교체 시, `getCatLabel()` 함수 내부의 `return CAT_LABELS[cat]`까지 교체되어 자기 자신을 무한 호출하는 버그 발생. 함수 내부를 원본 딕셔너리 참조로 복원.

---

## 7. 현재 파일 구조

```
decisionflow/
├── index.html                              # 전체 앱 (단일 파일)
├── templates/                              # 테스트 SVG (현재 미사용 — 파라메트릭 생성으로 대체)
│   ├── grid-mosaic.svg
│   ├── diagonal-cross.svg
│   └── column-arc.svg
├── fonts/
│   └── WATCHASans-ExtraBold.woff2
├── docs/
│   ├── index.html                          # GitHub Pages용 복사본
│   ├── fonts/
│   │   └── WATCHASans-ExtraBold.woff2
│   ├── templates/                          # GitHub Pages용 복사본 (미사용)
│   ├── superpowers/
│   │   ├── specs/
│   │   │   └── 2026-04-01-svg-template-system-design.md
│   │   └── plans/
│   │       └── 2026-04-01-svg-template-system.md
│   └── worklogs/
│       ├── 2026-03-25-day1-init.md
│       ├── 2026-03-26-day2-level2-redesign.md
│       ├── 2026-03-28-day3-canvas-preview-graphic.md
│       └── 2026-04-01-day4-svg-template-system.md
├── index.html.backup
└── index_test.html
```

### 배포 상태
- GitHub Pages: `https://sal9jane.github.io/decisionflow/`
- 현재 배포 버전: eb09b76 (파라메트릭 템플릿 + UI 개선)

### API 호출 구조 (현재)
```
[사용자 입력]
  → (Custom 카테고리 시) AI: 한글→영어 번역
  → AI 1: Structure (line1/line2/sub 추출)
  → AI 2: Strategy (톤 + 템플릿 선택)
  → AI 3: Palette (3개 팔레트, 2색 톤온톤 + 1색 포인트)
  → generateTemplate(id, seed) → Canvas 렌더링
```

---

## 8. 현재 그래픽 시스템 아키텍처

### 렌더링 파이프라인

```
TEMPLATE_GENERATORS[id](rand)     ← 시드 기반 형태 생성
         ↓
generateTemplate(id, seed)        ← 슬롯 셔플 + 미러링 적용
         ↓
drawShape(ctx, shape, colors)     ← Canvas에 그리기 (슬롯→실제 색상)
         ↓
shapeToSVG(shape, colors)        ← SVG 다운로드용 문자열 생성
```

### 지원하는 도형 타입
- `rect` — 사각형
- `circle` — 원
- `ellipse` — 타원
- `polygon` — 다각형 (사선 밴드, 다이아몬드 등)
- `path` — SVG path (현재 미사용, Figma SVG 대비)

### 변주 시스템
- **슬롯 셔플**: base/accent1/accent2 역할 회전 (3가지)
- **좌우 미러링**: 50% 확률 수평 반전
- **파라메트릭 변주**: 시드가 바뀌면 형태 자체가 변경 (Regenerate)

---

## 9. 설계 문서와 현재 코드의 차이

원래 스펙은 "Figma에서 만든 SVG를 파싱하는 시스템"이었으나, 실제로는 **파라메트릭 생성 함수**로 진화함:

| 스펙 (2026-04-01) | 현재 코드 |
|-------------------|-----------|
| 정적 SVG 파일을 fetch/파싱 | 파라메트릭 JS 함수가 직접 shapes 배열 생성 |
| `parseSVG()`, `loadTemplate()` | `TEMPLATE_GENERATORS[id](rand)` |
| `applyVariation()` (슬롯 셔플 + 미러 + 스케일 지터) | `generateTemplate()` (슬롯 셔플 + 미러, 지터 제거) |
| `templateCache` 객체 | 캐시 없음 (매번 생성) |
| Figma SVG 교체 워크플로우 | 코드 내 생성 함수 수정으로 대응 |

**Figma SVG 파싱 코드(`parseSVG` 등)는 삭제됨.** 나중에 Figma SVG import가 필요하면 재구현 필요.

`templates/` 폴더의 SVG 파일은 여전히 존재하지만 현재 코드에서 참조하지 않음.

---

## 10. 사용자 피드백 이력 (그래픽 관련, 중요)

| 피드백 | 대응 | 상태 |
|--------|------|------|
| 랜덤 생성으로는 디자인 느낌이 안 남 | 검증된 템플릿 접근으로 전환 | 완료 |
| Column Arc가 Grid Mosaic과 비슷 | 아크/타원 전용으로 재설계 | 완료 |
| Diagonal Cross도 사각형 기반처럼 보임 | 대각선 polygon 전용으로 재설계 | 완료 |
| Regenerate가 단조로움 | 파라메트릭 생성 함수로 전환 | 완료 |
| 좁고 뾰족한 영역이 실수처럼 보임 | 밴드 최소 폭 확대, 지터 제거 | 완료 |
| 컬러 3색이 촌스러움 | 2색 동일톤 + 1색 포인트 규칙 | 완료 |
| 겹치는 영역에 다른 컬러 | Column Arc에 교차 영역 감지 추가 | 완료 |

### 디자인 품질 기준 (여전히 유효)
- 끝점이 정확히 맞아야 함 (미세 어긋남 = 오류)
- 좁고 뾰족한 영역 금지 (슬리버 = 오류)
- 형태 다양성 확보하되 "우연히 생긴" 느낌은 안 됨
- 중앙 텍스트 영역의 가독성 최우선

---

## 11. 다음 세션 TODO

### 즉시 할 수 있는 것
- [ ] `templates/` 폴더와 `docs/templates/` 정리 (현재 미사용 SVG 파일)
- [ ] Grid Mosaic도 겹침 교차 영역 규칙 적용 검토
- [ ] 각 템플릿의 Regenerate 결과를 10회+ 돌려보며 슬리버/어긋남 케이스 점검
- [ ] Custom 카테고리 입력 시 영어 번역 결과가 커버에 정상 반영되는지 E2E 테스트

### 검토할 방향
- [ ] 그래픽 2트랙 중 **내용 기반 심볼** (클라우드→구름, 비용→화살표 등) — 별도 설계 필요
- [ ] 컬러 팔레트가 여전히 기대에 못 미치면, 검증된 외부 팔레트 소스 연동 검토
- [ ] Figma SVG import 워크플로우 재도입 여부 — 현재 파라메트릭으로 충분한지 평가

### 고도화 (이후)
- [ ] Slack 봇 연동 (요청 경로가 Slack이므로)
- [ ] WATCHA Pedia 광고 이미지 유스케이스 추가
- [ ] 히스토리/저장 기능 (localStorage)
