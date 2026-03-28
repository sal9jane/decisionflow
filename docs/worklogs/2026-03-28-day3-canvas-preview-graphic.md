# Worklog: Day 3 — Canvas 프리뷰 구현 + 그래픽 패턴 반복 개선

**날짜**: 2026-03-28
**커밋 범위**: d048558 ~ 13aaac1 (16 커밋)

---

## 작업 요약

1. API 키 보안 강화 (base64 → XOR 인코딩)
2. 카테고리 선택 UI + WATCHA 브랜드 프롬프트 적용
3. GitHub Pages 404 해결 (docs/index.html 추가)
4. WATCHA Sans OTF woff2 변환 및 프로젝트 포함
5. Canvas 기반 2500×850 커버 프리뷰 + PNG/SVG 다운로드 구현
6. 4단계 → 3단계 → 2단계로 플로우 단순화
7. 프리뷰에서 팔레트 실시간 전환 + 컬러 피커 추가
8. 그래픽 패턴 생성 로직 6회 반복 재설계

---

## 1. 인프라 변경

### API 키 보안
- **변경 전**: base64 분할 3조각 → `atob()` 결합 — 콘솔에서 즉시 복원 가능
- **변경 후**: XOR 인코딩 (키: `WatchaDecisionFlow2024`) — 정수 배열 + XOR 연산으로 런타임 복원
- XOR 키가 소스에 노출되어 있으므로 완전한 보안은 아님, 캐주얼 노출 방지 수준

### GitHub Pages 404 해결
- 원인: GitHub Pages가 `docs/` 폴더에서 서빙하도록 설정됐는데 `docs/index.html`이 없었음
- 해결: `docs/index.html` + `docs/fonts/` 생성, 매 커밋마다 root에서 docs로 복사

### WATCHA Sans OTF
- 원본: `/Users/janemacbook/Downloads/OTF/WATCHASansOTFExtraBold.otf` (388KB)
- `fonttools + brotli`로 woff2 변환 → 195KB (50% 압축)
- `fonts/WATCHASans-ExtraBold.woff2`에 저장
- `@font-face`로 웹에서 직접 로딩 (대체 폰트 방식에서 변경 — 사용자 한정적이라 유출 우려 없음)

---

## 2. 플로우 변천사

### 4단계 (788f2a8)
```
Structure → Strategy → Graphic(팔레트) → Preview
```
- 각 단계에서 AI API 호출 (총 3회)
- 사용자 피드백: Strategy 단계가 불필요하게 느껴짐

### 3단계 (af794c3)
```
Structure → Graphic(Strategy 숨김 + 팔레트) → Preview
```
- Strategy API를 Graphic 단계 진입 시 자동 실행
- 사용자에게는 팔레트 선택만 보여줌

### 2단계 — 현재 (ac831f8)
```
Structure → Preview(Strategy+팔레트 숨김, 프리뷰에서 팔레트 전환)
```
- "Preview →" 버튼 클릭 시 Strategy + Palette API 순차 호출 → 바로 Canvas 렌더링
- 팔레트 칩 3개가 프리뷰 하단에 표시, 클릭 시 즉시 재렌더링 (API 재호출 없음)
- Base/Accent1/Accent2 컬러 피커로 직접 색상 수정 가능

---

## 3. Canvas 프리뷰 시스템

### 렌더링 스펙
- Canvas: 2500 × 850px (Figma 원본 규격)
- 카테고리 라벨: WATCHA Sans OTF ExtraBold 80px, letterSpacing -1px, y=165
- 타이틀: WATCHA Sans OTF ExtraBold 144px, lineHeight 175px, letterSpacing -4px, y=305
- 텍스트 컬러: #FFFFFF 고정
- textBaseline: 'top', textAlign: 'center', x=1250

### 타이틀 구성
- AI가 `line1`/`line2`를 직접 결정 (이전의 `label`/`headline` 대신)
- 언어 자유: EN+KR, KR+KR, EN+EN 모두 가능
- `buildCoverTitle()`: line1 + '\n' + line2 반환
- `forceTwoLines()`: 단일 라인일 때 단어 중간에서 분할 (fallback)
- **버그 수정**: `text.slice(mid)` → `text.slice(m)` (ReferenceError)

### 다운로드
- **PNG**: `canvas.toDataURL('image/png')` → `<a>` 클릭
- **SVG**: 직접 SVG 문자열 생성, `font-family="WATCHA Sans OTF"` 지정 → Figma에서 열면 로컬 폰트 자동 매칭
- SVG에서 polygon 타입 지원 (`<polygon points="..."/>`)
- 파일명: `{line1}_{line2}.png/.svg`

### AI 프롬프트 변경사항

**Structure (Step 1)**:
```
line1/line2 형식으로 변경. 언어 제한 없음.
한자 사용 금지 필요 (현재 미적용 — 다음 세션에서 추가).
```

**Palette (Step 2, 내부)**:
```
3가지 서로 다른 색 필수 (동일색 금지).
base는 어두운 톤만 (밝기 40% 이하).
duo-tone / monochromatic / analogous 전략 선택.
```

---

## 4. 그래픽 패턴 생성 — 반복 이력

### 반복 1: Zone 기반 배치 (788f2a8)
- 8개 zone에 5-8개 형태 배치
- **문제**: 형태가 너무 적고 가장자리에만 배치, 캔버스 대부분이 단색 배경

### 반복 2: 3-layer 풀 커버리지 (a51924c)
- Layer 1: 세로 컬럼으로 전체 채움
- Layer 2: 7-11개 대형 형태 오버레이
- Layer 3: 소형 악센트
- **문제**: 투명도 겹침으로 탁한 색상, 형태끼리 관계 없이 단순 적재

### 반복 3: 재귀 분할 테셀레이션 (83eef91)
- `subdivide()` 함수로 캔버스를 재귀적으로 분할
- 50px 그리드 스냅, 15-30% 대각선 분할
- **문제**: 분할이 너무 잘게 되어 노이즈, 세이프존 고려 부족

### 반복 4: 세이프존 보호 추가 (e87c7f8)
- `inSafe()` 함수로 중앙 영역 분할 억제
- 대각선/원형 악센트 가장자리로 제한
- **문제**: 인접 블록이 같은 색 → 경계 소실

### 반복 5: 3존 + 교차 빔 (b8259a1 → 2190d61)
- 좌마진 + 중앙 + 우마진 구조
- 긴 삼각형 빔이 가장자리에서 관통
- 중앙 사각형 추가 → 썸네일에서 단색으로 보여 제거
- **문제**: 빔이 "랜덤 슬래시"처럼 보임, 의도된 구성이 아닌 느낌

### 반복 6: 그리드 셀 기반 — 현재 (8da439a → 13aaac1)
- 3-4 컬럼 × 2-3 로우 그리드
- 인접 셀 색상 교차 보장 (`diffColor`)
- 2-3개 겹침 악센트 블록
- **문제**: 아직 완성도 부족 — 중앙/가장자리 밀도 차이 없음, "의도된 구성" 느낌 미달

---

## 5. 사용자 그래픽 피드백 종합 (매우 중요)

### 핵심 원칙
1. **밀도 차이**: 중앙은 느슨(큰 블록), 가장자리는 조밀(작은 블록) — WATCHA PEDIA 커버 참조
2. **형태 수 10-15개**: 너무 많으면 노이즈, 너무 적으면 심심
3. **미세 어긋남 = 디자인 실수**: 모든 경계 정확히 정렬
4. **같은 색 인접 금지**: 블록 경계가 반드시 보여야 함
5. **opacity 항상 1**: 투명도 겹침 = 탁함
6. **썸네일 크롭 고려**: 세이프존 밖이 잘려도 그래픽이 보여야 함 (중앙 단색 블록 금지)

### 레퍼런스 분석

**WATCHA PEDIA (핑크 커버)**:
- 기본 구조: 3-4 세로 컬럼 + 가로 분할
- 좌우: 작은 사각형들이 겹쳐 밀도 높음
- 중앙: 큰 블록 위에 텍스트, 2-3개 작은 악센트로 심심함 방지
- 컬러: 핑크 4-5톤 (매우 연한 핑크 ~ 진한 마젠타)

**CTI Report 시리즈 (5개 변형)**:
- 3존 구조 (좌/중앙/우)
- 교차하는 긴 삼각형들이 X 패턴 생성
- 같은 구도를 2색/3색 다양하게 적용
- 중앙은 큰 사각형으로 텍스트 배경

**Brand & Content Balance**:
- 세로 컬럼 + 큰 아크/곡선 오버레이
- 톤온톤 핑크 계열
- 전체 캔버스 빈틈없이 채움

**Automatic Labeling System (내용 기반)**:
- 클라우드 모양 → "클라우드 비용" 주제
- 하향 화살표 → "비용 절감" 의미
- 디지털화된 픽셀 느낌의 형태

### 현재 한계 인식
- **랜덤 생성의 근본적 한계**: 파라미터 조정만으로는 "의도된 디자인" 불가
- 매 반복마다 조금씩 나아지지만 근본적 품질 격차는 좁혀지지 않음
- **다음 세션 방향**: 검증된 컴포지션 템플릿 + 파라미터 변주 접근 필요

---

## 6. 현재 파일 구조

```
decisionflow/
├── index.html                              # 전체 앱 (2단계 플로우)
├── fonts/
│   └── WATCHASans-ExtraBold.woff2          # WATCHA 폰트 (195KB)
├── docs/
│   ├── index.html                          # GitHub Pages용 복사본
│   ├── fonts/
│   │   └── WATCHASans-ExtraBold.woff2
│   └── worklogs/
│       ├── 2026-03-25-day1-init.md
│       ├── 2026-03-26-day2-level2-redesign.md
│       └── 2026-03-28-day3-canvas-preview-graphic.md
├── index.html.backup
├── index_test.html
└── .git/
```

### 배포 상태
- GitHub Pages: `https://sal9jane.github.io/decisionflow/`
- 현재 배포 버전: 2단계 플로우 + 그리드 셀 패턴 (13aaac1)

### API 호출 구조
```
[사용자 입력] → Step 1 AI (Structure: line1/line2/sub)
           → Step 2 AI (Strategy: 내부용, 화면에 미노출)
           → Step 3 AI (Palette: 3개 팔레트 제안)
           → Canvas 렌더링 (genPattern + drawShape + 텍스트)
```

---

## 7. 다음 세션 TODO

### 즉시 수정 필요
- [ ] AI 프롬프트에 "한자(漢字) 사용 금지" 추가 — 현재 간헐적으로 한자 출력됨
- [ ] `forceTwoLines()` 단일 단어 케이스 테스트 — 버그 수정은 했으나 검증 필요

### 그래픽 패턴 근본 재설계
현재 접근(랜덤 생성)의 한계를 인정하고, **검증된 컴포지션 템플릿** 방식으로 전환:

1. **템플릿 3-5개 설계** — 수동으로 좋은 구성을 정의
   - Type A: 그리드 모자이크 (WATCHA PEDIA 스타일) — 가장자리 조밀, 중앙 느슨
   - Type B: 세로 컬럼 + 곡선 (Brand & Content 스타일)
   - Type C: 교차 삼각형 (CTI 스타일)
   - Type D: 큰 블록 + 악센트 (스마트TV 스타일)

2. **각 템플릿의 구조는 고정, 파라미터만 변주**
   - 컬럼 폭 비율
   - 블록 크기 비율
   - 색상 배치
   - 악센트 위치

3. **가변 밀도 구현**
   - 중앙(세이프존): 큰 블록 1-2개 — 텍스트 가독성 확보
   - 가장자리: 작은 블록 다수 — 시각적 풍부함
   - 밀도 차이가 자연스럽게 텍스트를 돋보이게 함

### 고도화 (이후)
- [ ] 트랙 B: 내용 기반 심볼 라이브러리 (클라우드, 화살표, 별, 돋보기 등)
- [ ] AI가 글 내용에서 적합한 심볼을 선택하는 로직
- [ ] Slack 봇 또는 Figma 플러그인 확장

---

## 8. 기술 참고

### Canvas letterSpacing
- `ctx.letterSpacing = '-1px'` — Chrome 99+, Firefox 115+, Safari 17+
- 미지원 브라우저에서는 무시됨 (fallback: 간격 없음)
- `'letterSpacing' in ctx` 로 체크 후 적용

### seededRandom
- `mulberry32(seed)` 사용 — 결정적 난수 생성
- `hashStr(string)` — 문자열을 정수 시드로 변환
- 같은 타이틀 + 같은 patternSeed → 항상 같은 패턴

### SVG 텍스트 좌표
- Canvas `textBaseline='top'`, y=165(카테고리), y=305(타이틀)
- SVG는 기본 alphabetic baseline 사용, y=215(카테고리), y=420+i*175(타이틀)
- Figma에서 열 때 약간의 위치 차이 가능 → SVG는 편집용이므로 허용 범위
