# Worklog: Day 2 — Level 2 구현 + 3단계 플로우 + 디자인 리디자인

**날짜**: 2026-03-26
**커밋 범위**: a0a0c42 ~ e451145 + 미커밋 변경 (Pentagram 스타일)

---

## 작업 요약

1. Level 2 (Expression Strategy) 기능 구현
2. 3단계 UI 플로우로 구조 변경
3. 디자인 4회 반복 (다크→Anthropic→Swiss→Pentagram)
4. 디자인 스킬 설치 시도

---

## 1. Level 2 — Expression Strategy 추가

### 배경
Level 1(Structure)은 텍스트를 label/headline/sub로 정리하는 것이었음.
Level 2는 **"어떻게 표현할지"에 대한 전략**까지 도출하는 단계.

사용자 철학: "한글이냐 영문이냐, 어떤 톤이냐 — 이런 판단까지 포함해서 디자인 의사결정을 돕고 싶다"

### 추가된 출력 필드 (Expression Strategy)
| 필드 | 값 옵션 | 설명 |
|------|---------|------|
| `primary_language` | ko / en / mix | 텍스트 언어 방향 |
| `emphasis` | label / headline / sub | 시각적으로 가장 강조할 요소 |
| `tone` | information / emotion / blend | 정보 vs 감성 |
| `readability` | strong / medium / low | 가독성 수준 |
| `visual_weight` | text-centric / graphic-support / graphic-dominant | 텍스트 vs 그래픽 비중 |
| `reasoning` | 자유 텍스트 (한국어) | 선택 이유 2-3문장 |

### 프롬프트 구조
Step별로 별도 AI 호출. 이전 단계 결과를 다음 단계 입력으로 전달:
- Step 1: 원본 텍스트 → Structure JSON
- Step 2: Structure JSON → Strategy JSON
- Step 3: Structure + Strategy JSON → Graphic Options JSON

---

## 2. 3단계 UI 플로우

### 구조
```
Step 1: Input → Structure 결과 확인 → [다음]
Step 2: Strategy 자동 생성 → 결과 확인 → [다음]
Step 3: Graphic Options 자동 생성 → Color/Layout/Style 선택
```

### Step 3 — Graphic Options (신규)
AI가 제안하는 그래픽 방향:
- **Color Mood**: 3개 색상 무드 (이름 + hex 코드)
- **Layout**: 3개 레이아웃 제안 (한국어)
- **Style Direction**: 3개 스타일 방향 (한국어)
- **Note**: 그래픽 방향 설명 1-2문장

각 옵션은 **클릭해서 선택/해제** 가능 (`.selected` 클래스 토글)

### 네비게이션
- 각 단계에서 이전/다음 자유 이동
- 상단 스텝 표시 (현재/완료 상태)
- "처음으로" 리셋 기능
- "전체 복사" — 3단계 결과를 한번에 클립보드 복사

---

## 3. 디자인 반복 이력

### 반복 1: 다크 모드 + Inter
- 초기 MVP. 기본적인 다크 테마.
- 카드 기반 결과 표시.

### 반복 2: DM Sans + 그레인 텍스처
- frontend-design 스킬 가이드라인 적용
- 그레인 오버레이, accent 그라데이션 라인, 컬러 태그 시스템
- 사용자 피드백: "업계 최고 UXUI 전문가가 만든 것처럼"

### 반복 3: Swiss/Dieter Rams 스타일
- IBM Plex Sans/Mono, 밑줄 입력 필드, 라인 기반 구분
- 사용자 피드백: **"박스가 안 보이고 텍스트가 오밀조밀해서 좋지 않다"** → 거부

### 반복 4: Anthropic 사이트 스타일
- 따뜻한 크림 배경(#f7f5f2), 카드+그림자, 세그먼트 컨트롤 탭
- 뱃지 시스템, 넉넉한 여백 → **GitHub에 배포 완료 (e451145)**

### 반복 5: Pentagram 스타일 (현재, 미커밋)
- 순백 배경, 흑백 고대비
- 세리프 Headline (Noto Serif KR 48px/900)
- 카드/박스 없이 1px 라인으로만 영역 구분
- 사각 칩 (border-radius: 0), 선택 시 검정 반전
- 상단 네비게이션 바 (브랜드 + 01/03 페이지 표시)
- **아직 커밋/배포되지 않은 상태**

### 사용자의 디자인 취향 정리
- 유행을 타지 않는 클래식한 스타일 선호
- 타이포그래피 중심의 미니멀리즘
- 디자이너 포트폴리오 사이트 같은 룩
- Graphic 영역은 Figma처럼 도식화된 툴 느낌 원함
- 박스가 보여야 함 (Swiss 스타일의 라인 only는 거부)
- Pentagram 참조: "깔끔한 기본, 유행이 느껴지지 않는"

---

## 4. 스킬 설치

### 설치 완료
- **UI-UX-Pro-Max** (`nextlevelbuilder/ui-ux-pro-max-skill@ui-ux-pro-max`)

### 이미 내장되어 있던 것
- **frontend-design** — Anthropic 공식 스킬 (`.claude/plugins/marketplaces/` 경로에 존재)

### 설치 실패 (저장소 404)
- `anthropics/frontend-design` (별도 GitHub repo는 없음, 플러그인으로만 존재)
- `supercent-io/skill-web-accessibility`
- `vercel-labs/agents-skill-web-design-guidelines`

---

## 5. 현재 상태

### 파일
```
decisionflow/
├── index.html                    # 전체 앱 — Pentagram 스타일 (미커밋)
├── docs/
│   └── worklogs/
│       ├── 2026-03-25-day1-init.md
│       └── 2026-03-26-day2-level2-redesign.md
└── .git/
```

### 배포 상태
- GitHub Pages 활성: `https://sal9jane.github.io/decisionflow/`
- 현재 배포 버전: **Anthropic 스타일 (e451145)**
- 로컬 최신: **Pentagram 스타일 (미커밋)**

### API
- Groq API 키: base64 인코딩으로 코드에 내장
- 모델: `llama-3.3-70b-versatile`
- 3단계 각각 별도 API 호출 (총 3회)

---

## 다음 작업 시 참고

### 우선 해야 할 것
1. Pentagram 스타일 최종 확인 후 커밋/배포
2. 사용자 피드백에 따라 디자인 미세 조정 가능성 있음

### 고려할 수 있는 방향
- Step 3 Graphic Options를 더 시각적으로 표현 (컬러 프리뷰 확대, 레이아웃 도식화 등)
- 선택한 옵션을 기반으로 실제 그래픽 프리뷰 생성 (Level 3)
- 히스토리/저장 기능 — 이전 변환 결과를 localStorage에 보관
- 프롬프트 튜닝 — 다양한 입력에 대한 결과 품질 검증

### 사용자 특성 (작업 시 유의)
- 디자이너, 개발 경험 없음
- 기술 용어 없이 쉽게 설명 필요
- 한국어로 소통
- 디자인 의사결정 과정 자체를 중요하게 생각
- "완벽하지 않아도 실제 작업에 바로 쓸 수 있는 70% 수준이면 충분"
