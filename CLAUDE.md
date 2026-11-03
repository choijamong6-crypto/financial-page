# Financial Dashboard Project

## Data Source
- **Google Spreadsheet**: https://docs.google.com/spreadsheets/d/1-O4cnK8wzGdgYf724fu4lOouxCkb-O8kWM9csuNzYtE/edit?usp=sharing
- 데이터 업데이트 요청 시 위 스프레드시트에서 최신 데이터를 가져와 `index.html`의 JavaScript `budgetData` 객체를 업데이트할 것

## GitHub Pages
- **Repository**: https://github.com/choijamong6-crypto/financial-page
- **Live Site**: https://choijamong6-crypto.github.io/financial-page/
- 변경사항은 `/tmp/financial-page/` 디렉토리에서 git push

## GitHub CLI 설정
- `export GH_CONFIG_DIR=~/gh-config` 설정 필요 (권한 문제로 기본 경로 사용 불가)

## 파일 구조
- `index.html` - 메인 대시보드 페이지 (Google Drive와 GitHub 동기화)
- `01_pictures/` - 고양이 이미지 폴더 (나몽 사진들)
- `robots.txt` - 검색 엔진 크롤링 방지

## 작업 규칙 (중요!)
1. **최신 버전은 항상 GitHub repo (`/tmp/financial-page/index.html`)에 있음**
2. **작업 전 반드시 확인**: `cd /tmp/financial-page && git pull && git log --oneline -3`
3. **수정은 `/tmp/financial-page/index.html`에서 직접 하거나, Google Drive의 `index.html` 수정 후 repo에 복사**
4. **작업 완료 후**: repo에서 commit & push 한 뒤, Google Drive `index.html`도 동기화
5. **절대 금지**: 오래된 로컬 파일로 repo 파일을 덮어쓰기

---

# Data Update Runbook (데이터 업데이트 런북)

> "데이터 업데이트해줘" 라고 하면 이 섹션을 따라가면 됨. 매번 구조를 새로 파악할 필요 없음.

## Step 0: 사전 준비

```bash
cd /tmp/financial-page && git pull && git log --oneline -3
```

## Step 1: 스프레드시트 가져오기 (xlsx, 전체 탭)

gviz CSV(`gviz/tq?tqx=out:csv`)는 **첫 번째 탭만** 준다. 2026-11 기준 첫 탭은 빈 템플릿 `2027 상반기`라서 쓸 수 없다. 전체 탭을 xlsx로 받아 openpyxl로 읽는다:

```bash
curl -sL "https://docs.google.com/spreadsheets/d/1-O4cnK8wzGdgYf724fu4lOouxCkb-O8kWM9csuNzYtE/export?format=xlsx" -o sheet.xlsx
python3 -c "import openpyxl; print(openpyxl.load_workbook('sheet.xlsx', data_only=True).sheetnames)"
```

| 탭 | 내용 | 월 블록 시작 열 |
|----|------|----------------|
| `2026 하반기` | 2026년 7~12월 | C (7월 C~H, 8월 I~N, 9월 O~T, 10월 U~Z, 11월 AA~AF, 12월 AG~AL) |
| `2026 상반기` | 2026년 1~6월 | B (1월 B~G, …, 6월 AF~AK) |
| `Sheet2` | 2025년 9월~2026년 2월 (옛 형식) | — |
| `2027 상반기` | 빈 템플릿 | C |
| `하니` | 하은 개인 통장 (사이트에 안 씀) | — |

`data_only=False`로 다시 열면 SUM 칸의 공식이 보인다. 합계가 이상할 때 범위를 확인하는 용도.

## Step 2: 스프레드시트 구조 이해

### 월 블록 — 6열 반복

```
[+0] 수입 라벨    (참새월급, 하니월급, 기타 수입, 총액, 참새통장, 냥이통장, 공동통장, Savings, ...)
[+1] 수입 금액
[+2] 지출 카테고리 (Rent, 십일조, 공과금, 보험, Vet, Grocery, 문화생활, 외식, Shopping, Gas, Parking, 오빠, 하니, Tesla, ...)
[+3] 지출 항목    (Rent, Xcelenergy, Geico, Costco, ...)
[+4] 지출 금액
[+5] SUM         (카테고리 합계 — 믿지 말 것, 아래 "합계 규칙")
```

- 카테고리 라벨은 다음 라벨이 나올 때까지 아래 행에 적용된다. 지출은 3행부터 카테고리 열이 `Total`인 행(보통 69행) 전까지.
- 카테고리 열의 `under` + 숫자(예: Grocery 옆 `under 150`)는 예산 메모라서 카테고리가 아니다. 날짜가 들어 있어도 무시.
- `2026 하반기` 탭의 A/B열은 고정 지출·목표 지출 **계획**이라 데이터가 아니다.
- Total 아래(70행~)의 `대혀니 저축`/`하니 저축`은 금액 칸이 비어 있으면 0. 그 밑의 Izzy, Boulder gbs 같은 줄은 메모라 무시.
- **Goal(저축 목표)**: `2026 하반기` 탭 7월 블록 74~79행(C/D열). 예전 `2026 상반기` 탭은 109~114행(A~C열).

### 합계 규칙 (중요!)

시트의 SUM 칸·Total·총액 공식은 범위가 틀린 경우가 많다. 그래서 **카테고리 sum = 항목 합, totalExpenses = 카테고리 sum의 합, income.total = 수입 항목 합**으로 직접 계산한다. 시트 값과 다르면 셀 위치와 함께 사용자에게 알린다. 2026-11-02에 찾은 예:

- 3월 총액 `=sum(O3:O4)` → NJS $202.96 누락
- 4월 Total → Parking $14 누락 (이전 세션에서 이미 항목 합 사용)
- 7월 오빠 SUM `=sum(G33:G41)` → Gas·Parking 행까지 포함 ($316.19 중복)
- 8월 Loop 금액이 텍스트 `'16,62'` → 시트 합계에서 빠짐 (16.62로 읽음)
- 10월 SUM 열 Z10·Z15·Z18·Z31에 공식 아닌 값 $192.39가 남아 Total에 더해짐

### 지출 카테고리 → JS 키 매핑

| 스프레드시트 카테고리 | JS expenses 키 | 비고 |
|----------------------|----------------|------|
| Rent | `housing` | rent (+ 1~5월 선한 Rent: 선한 = 하은 동생) |
| 십일조 | `tithing` | 감사헌금 → `thanksOffering` |
| 공과금 | `utilities` | Xcel Energy, Xfinity (애틀랜타 시절: Gfiber, GP, AT&T) |
| 보험 | `insurance` | Geico, 나몽/아몽 보험, Tesla 보험 → `teslaInsurance`, Fetch 펫보험 → `fetchPet` |
| 나몽 / 나몽&망고 / 나몽아몽 / Vet / Pettable | `pet` | 동물병원, ESA 레터 등 |
| Grocery | `grocery` | Costco, Hmart, King Soopers 등 |
| 문화생활 | `entertainment` | YouTube, Amazon Prime 등 |
| 외식 | `dining` | |
| Shopping | `shopping` | Amazon, TJ Maxx 등 |
| Gas | `gas` | 주유 + Tesla 충전(`teslaCharging`), Chargepoint, Loop |
| member | `membership` | Costco 멤버십 등 |
| 그 밖의 모든 카테고리 | `other` | **시트 카테고리 하나당 키 하나**: Parking→`parking`, 오빠(대현 개인)→`oppa`, 하니(하은 개인)→`hani`, Tesla→`tesla`, DMV→`dmv`, 여행→`sandDunes` 등. 라벨은 `SANKEY_SUB_LABELS`에 추가 |

> 카테고리는 월마다 다르다. 해당 월에 실제로 있는 것만 넣는다.

### 저축 목표 (현재 Goal 목록, 2026-11-02)

| 항목 | 금액 |
|------|------|
| Bonus return | $5,000 (병원 사인온 보너스 반납 — 부채. 사이트에서 맨 위 "먼저 갚기!") |
| Green Card | $8,000 |
| Flight | $5,500 |
| Tesla | $48,000 |
| **합계** | **$66,500** (코드가 `savingsGoals`에서 자동 계산) |

## Step 3: index.html 업데이트 위치

차트 배열·기본 월·목표 합계는 2026-11-02부터 `budgetData`에서 자동 계산된다. 새 달을 넣을 때 고칠 곳은 이게 전부:

1. **`budgetData`에 월 객체 추가.** 키 규칙: 2025년 9~12월 = `sep`~`dec`, 2026년 1~7월 = `jan`~`jul`, **2026년 8월부터는 연도 접미사** (`aug26`, `sep26`, `oct26`, `nov26`, `dec26`, `jan27`, …)
2. **`SANKEY_MONTH_INFO`에 같은 키 추가 — 시간순으로.** 이 순서가 추이/월별 막대 차트의 x축 순서다 (`{ name: 'Nov', year: 2026, days: 30 }`)
3. **월 버튼 추가** + `active` 클래스를 최신 달로 이동. 2025년 버튼은 "25년 9월"처럼 연도 표시
4. **`let currentMonth = '<최신 키>';`** — `init()`이 이 값으로 첫 화면을 그린다
5. **`currentSavingsBalance`** — 최신 달 `Savings` 행 값만 (참새/냥이/공동통장 합산 X). 비어 있으면 그 전 달 값을 쓰고 주석에 몇 월 값인지 적는다
6. 새 항목 키의 라벨이 어색하면 `SANKEY_SUB_LABELS`(수입은 `SANKEY_INCOME_LABELS`)에 추가. 없으면 camelCase → Title Case로 표시된다
7. Goal 목록이 바뀌었으면 `savingsGoals`만 고친다 (인사이트·나몽 진행바 합계는 자동)

기존 달 수정은 `budgetData`의 그 달만 고치면 차트까지 따라간다.

**각 월 객체의 구조:**
```javascript
monthKey: {
    income: { chamSalary: N, haniSalary: N, /* 기타 수입 */ total: N },   // total = 수입 항목 합
    expenses: {
        housing: { rent: N, sum: N },
        // ... 해당 월에 있는 카테고리만. sum = 항목 합
        other: { parking: N, oppa: N, ..., sum: N }
    },
    totalExpenses: N,                              // 카테고리 sum의 합
    savings: { daehyuni: N, hani: N, total: N },
    travel: { ... }                                // 옛 형식(상반기) 여행 섹션만. totalExpenses에 안 들어감
}
```

## Step 4: 검증

- [ ] 모든 달: 카테고리 `sum` = 항목 합, `totalExpenses` = 카테고리 sum의 합, `income.total` = 수입 항목 합 (예외: 2025년 12월 `travel`은 옛 구조라 항목이 합과 안 맞음 — 그대로 둠)
- [ ] `budgetData` 키 = `SANKEY_MONTH_INFO` 키 = 버튼 순서, `active` 버튼 = `currentMonth` = 최신 달
- [ ] 새 달 카테고리 합이 시트 항목 합과 일치 (시트 Total과 다르면 사용자에게 보고)
- [ ] 브라우저에서 비밀번호 입력 후 모든 월 버튼 클릭 → 콘솔 오류 없음, Sankey 그려짐

## Step 5: Commit, Push, Sync

```bash
cd /tmp/financial-page
git add index.html CLAUDE.md
git commit -m "N월 데이터 업데이트"
export GH_CONFIG_DIR=~/gh-config
git push
```

그 후 Google Drive 동기화 (Drive의 `CLAUDE.md`도 같이 — Claude Code가 이 폴더에서 열릴 때 런북을 바로 읽도록):
```bash
cp /tmp/financial-page/index.html "/Users/choidaehyun/Library/CloudStorage/GoogleDrive-daehyeonchoi@gmail.com/My Drive/Favorites/Financial/index.html"
cp /tmp/financial-page/CLAUDE.md "/Users/choidaehyun/Library/CloudStorage/GoogleDrive-daehyeonchoi@gmail.com/My Drive/Favorites/Financial/CLAUDE.md"
```

---

# Cash Flow Sankey 다이어그램

> 월별 자금 흐름을 시각화하는 섹션. 수입 소스 → Income 허브 → 지출 카테고리 → 서브카테고리 형태의 4단 흐름.

## 위치 / 의존성
- HTML 카드: `<div class="sankey-card">` — 월 선택 버튼과 summary cards 사이
- SVG 컨테이너: `<svg id="sankeyChart">`, 날짜 라벨: `<div id="sankeyDate">`
- CDN 의존성 (`<head>`):
  ```html
  <script src="https://cdn.jsdelivr.net/npm/d3@7"></script>
  <script src="https://cdn.jsdelivr.net/npm/d3-sankey@0.12.3"></script>
  ```
- 핵심 JS 함수:
  - `getSankeyData(monthData)` — budgetData를 sankey 노드/링크로 변환
  - `updateSankey(month, monthData)` — d3-sankey 레이아웃 + SVG 렌더링
  - 호출 위치: `init()` 및 `selectMonth()` 양쪽에서 호출

## 데이터 흐름 구조

```
수입 소스 (참새 월급, 하니 월급, NJS 등)
   ↓
Income 허브 ($총수입)
   ↓
카테고리 (Housing, Tithing, Utilities, Pet, ..., Travel, Savings, 남은 돈)
   ↓
서브카테고리 (Rent, 선한 Rent, Geico, Costco 등)
```

### 균형 보정
- 수입 < 지출(+저축+여행)이면 좌측에 `🏦 통장에서` 노드 자동 추가 (값 = deficit)
- 수입 > 지출(+...)이면 우측에 `✨ 남은 돈` 카테고리 자동 추가 (값 = surplus)
- 다이어그램이 항상 균형되도록 보장 (d3-sankey가 imbalance를 시각적으로 깨뜨리는 걸 방지)

### Travel / Savings 처리
- `travel`: 여러 destination(`{ portland, destin, denver }` 등)을 합산한 카테고리. 서브카는 destination 별 total
- `savings`: `{ daehyuni, hani }` 둘이 서브카 (양쪽 다 양수일 때만)

## 서브카테고리 묶음 규칙 (중요!)

서브카가 너무 많아 라벨이 겹치는 걸 방지하기 위해 다음 단계로 묶음:

### 1단계: misc 합산
키 패턴 `^misc\d*$` (misc1, misc2, misc 등) 항목들을 모두 한 항목으로 합산. 라벨은 `기타`. 키는 `misc`.

### 2단계: 동일 라벨 합산
`SANKEY_SUB_LABELS` 매핑에서 같은 표시 라벨로 매핑되는 키들을 합산:
- `gas1, gas2, gas3, gas4, gas5` → 모두 `Gas`로 매핑됨 → **하나로 합산**
- `cfa, cfa2` → `CFA`로 합산
- `dancingGoats, dancingGoats1, dancingGoats2` → `Dancing Goats`로 합산
- `costco, costco2` → `Costco`로 합산

### 3단계: $100 미만 항목 묶기
값이 **$100 미만**인 항목들이 **2개 이상**이면 한 노드로 합쳐서 라벨링:
- 라벨 형식: `{묶인 항목 중 가장 큰 것의 이름} 등 {N}건`
- 예: `Costco 등 4건 $129` (publix + walmart + traderJoes + costco)
- 예: `BBQ 등 10건 $247` (Dining 전체)
- **예외**: 가장 큰 항목의 라벨이 `기타`(misc 합산)이면 두 번째로 큰 항목의 이름 사용
  - 이유: "기타 등 N건"은 부자연스러움
  - 예: Dining에서 misc 합 $51이 BBQ $42보다 크지만 라벨은 `BBQ 등 10건`
- 작은 항목 1개만 있으면 묶지 않고 그대로 표시

### 라벨 우선순위
서브카 라벨 결정 순서:
1. `_displayLabel` (코드에서 동적으로 계산된 라벨; 묶음/dedupe 결과)
2. `SANKEY_SUB_LABELS[key]` (정적 매핑)
3. `sankeyTitleCase(key)` (camelCase → Title Case fallback)

## 라벨 매핑 테이블 추가 시
새로운 서브카 키가 추가되면 `SANKEY_SUB_LABELS` 객체에 매핑 추가:
```js
const SANKEY_SUB_LABELS = {
    rent: 'Rent', seonhanRent: '선한 Rent',
    geico: 'Geico', namongInsurance: '나몽 보험',
    // ... 새 키: '표시 라벨'
};
```

같은 표시 라벨로 매핑하면 자동 dedupe됨 (gas1~5처럼).

수입 소스 라벨도 별도 매핑: `SANKEY_INCOME_LABELS` (chamSalary, haniSalary, irs, fetch 등).

카테고리 라벨/색상: `SANKEY_CATEGORY_LABELS`, `SANKEY_CATEGORY_COLORS`.

## 레이아웃 파라미터

```js
const NODE_PADDING = 28;        // 노드 사이 수직 간격 (2줄 라벨 ~28px와 일치)
const PER_NODE = 36;            // 노드 1개당 최소 수직 공간
const calcHeight = maxCol * PER_NODE + 60;  // maxCol = sources/cats/subs 중 최대 노드 수
```

- `nodeAlign(d3.sankeyLeft)` — 모든 카테고리가 동일 컬럼에 정렬되도록 (sankeyJustify 사용 시 sub 없는 카테고리가 우측 끝으로 밀림)
- 높이는 leaf 수에 따라 자동 가변 (Mar의 경우 ~1900px). 카드가 길어지면 페이지 스크롤로 확인
- 라벨은 모든 노드에 표시 (작은 노드도 가시성 보장). 노드 패딩이 라벨 높이만큼 확보되어 겹치지 않음

## 월별 날짜 라벨
`SANKEY_MONTH_INFO` 객체에 월 → `{ name, year, days }` 매핑. 새 달 추가 시 갱신 (**시간순** — 차트 x축 순서도 이걸 따름):
```js
sep: { name: 'Sep', year: 2025, days: 30 },
jun: { name: 'Jun', year: 2026, days: 30 },
```

## selectMonth 연동
`selectMonth(month)` 내부에서 `updateSankey(month, monthData)` 호출. 새 달 추가 시 별도 수정 불필요 — 자동 동작.

## 디버깅 팁
- 라벨이 안 보이면: `SANKEY_SUB_LABELS`에 키 매핑 누락 → fallback `sankeyTitleCase`
- 다이어그램이 짧으면: leaf 수가 적은 달 (예: 6월). 정상 동작
- 노드가 안 그려지면: `data.nodes.length <= 1` 체크 — 빈 달은 "데이터가 없는 달이에요 🐱" 메시지 표시
- 윈도우 리사이즈 시: 디바운스 200ms 후 자동 리렌더 (`window.addEventListener('resize', ...)`)

---

# 비밀번호 게이트 (Password Gate)

사이트 진입 시 4자리 비밀번호 인증 필수. 코드 위치: `index.html` `<head>`의 `/* ===== 비밀번호 게이트 ===== */` CSS 블록 + `<body>` 시작 직후 `<div id="auth-gate">` HTML + 인라인 `<script>`.

## 정책
- **비밀번호**: `1023` (4자리 숫자)
- **SHA-256 해시**: `6629ddae3736e894e89cb4a1300a9d2c5c0fad418f8ea06a341b81f2a98bb491`
- **영속 저장 X**: 매 페이지 로드(새로고침/재방문)마다 입력 필요. localStorage 안 씀
- **인증 상태**: `document.body.classList.add('authed')` — 메모리 only

## 비밀번호 변경 시
1. 새 해시 계산: `echo -n "NEW_PW" | shasum -a 256`
2. `index.html`의 `const HASH = '...'` 한 줄만 교체

## 게이트 디자인 (현재)
- 사진 슬롯 5개 (메인 원형 1 + 데코 폴라로이드 4모서리)
- 슬롯마다 layer 2개 겹쳐 cross-fade (`transition: opacity 2s`)
- 3초마다 새 5장 랜덤 (`setInterval(refreshPhotos, 3000)`)
- PIN 박스 4개, 자동 다음칸 포커스, 4번째 입력 시 자동 검증

---

# 사진 최적화 정책 (Photo Optimization)

## 원칙
- **Drive `01_pictures/`**: 사용자가 업로드하는 원본 (수 MB) — 백업으로 보존
- **Repo `01_pictures/`**: 압축본 (수백 KB) — 사이트 로딩 속도 위해 필수

## 새 사진 추가 시 절차
Drive 원본 그대로 repo에 복사하면 페이지 로딩 매우 느림. 반드시 압축 후 commit.

```bash
cp "DRIVE/new.jpg" /tmp/financial-page/01_pictures/
sips -Z 1200 -s formatOptions 85 /tmp/financial-page/01_pictures/new.jpg
# 그리고 index.html의 allNamongPhotos 배열에 추가
```

## 일괄 압축 (Drive에서 새로 받은 사진들 한꺼번에)
```bash
cd /tmp/financial-page/01_pictures
find . -maxdepth 1 \( -iname "*.jpg" -o -iname "*.jpeg" \) -print0 | while IFS= read -r -d '' f; do
  sips -Z 1200 -s formatOptions 85 "$f" >/dev/null 2>&1
done
```

## 주의
- **GIF는 건드리지 말 것** (애니메이션 보존)
- 이미 압축된 사진을 또 돌려도 거의 안 줄어듦 (idempotent)
- 기준 효과: 132MB → 16MB (88% 감소)

---

# 디렉토리 복구 절차

`/tmp/financial-page/.git`이 손상되었거나 디렉토리가 비어있을 때 (실제 발생한 적 있음):

```bash
mv /tmp/financial-page /tmp/financial-page.broken-$(date +%s)
export GH_CONFIG_DIR=~/gh-config
git clone https://github.com/choijamong6-crypto/financial-page.git /tmp/financial-page
```

`rm -rf`는 사용자 승인 거절될 수 있으니 `mv`로 백업하는 게 안전.
