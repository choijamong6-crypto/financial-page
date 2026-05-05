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

## Step 1: 스프레드시트 CSV 가져오기

**CSV Export URL** (이 URL을 WebFetch로 접근):
```
https://docs.google.com/spreadsheets/d/1-O4cnK8wzGdgYf724fu4lOouxCkb-O8kWM9csuNzYtE/gviz/tq?tqx=out:csv
```

> 주의: `export?format=csv` URL은 리다이렉트되어 실패할 수 있음. 위의 `gviz/tq` URL을 사용할 것.

## Step 2: 스프레드시트 구조 이해

### 열(Column) 구조 — 월별 6열 반복

| 월 | 열 오프셋 (0-indexed, col 0은 row label) |
|----|----------------------------------------|
| 1월 | cols 1-6 |
| 2월 | cols 7-12 |
| 3월 | cols 13-18 |
| 4월 | cols 19-24 |
| 5월 | cols 25-30 |
| 6월+ | cols 31-36, ... (6칸씩 증가) |

각 월의 6열 구조:
```
[offset+0] 수입 라벨    (참새월급, 하니월급, 총액, 참새통장, 냥이통장, 공동통장, Savings, 하니, ...)
[offset+1] 수입 금액    ($4,333.87 등)
[offset+2] 지출 카테고리 (Rent, 십일조, 공과금, 보험, 나몽, Grocery, 문화생활, 외식, Shopping, Gas, member, ...)
[offset+3] 지출 항목    (Rent, Gfiber, Geico, ...)
[offset+4] 지출 금액    ($2,162.80 등)
[offset+5] 카테고리 SUM (해당 카테고리의 합계, 카테고리 마지막 항목 행에 나타남)
```

### 행(Row) 구조 — 주요 마커 (주의: 행 위치는 월마다 다를 수 있음)

CSV에서 찾아야 할 **키워드 마커**:

| 키워드 | 위치 | 의미 |
|--------|------|------|
| `참새월급` | 수입 라벨 열 | 참새(대현) 월급 |
| `하니월급` | 수입 라벨 열 | 하니 월급 |
| `총액` | 수입 라벨 열 | **해당 월 총 수입** (income.total) |
| `참새통장` | 수입 라벨 열 | 참새 통장 잔고 |
| `냥이통장` | 수입 라벨 열 | 냥이 통장 잔고 |
| `공동통장` | 수입 라벨 열 | 공동 통장 잔고 |
| `Savings` | 수입 라벨 열 | Savings 통장 잔고 |
| `하니` + `Total` | 하단 영역 | **해당 월 총 지출** (totalExpenses) |
| `대혀니 저축` | 하단 영역 | 대현이 저축액 |
| `하니 저축` | 하단 영역 | 하니 저축액 |
| `Goal` | col 0 (맨 아래) | 저축 목표 섹션 시작 |

### 지출 카테고리 → JS 키 매핑

스프레드시트의 카테고리 라벨이 JS budgetData의 어떤 키로 변환되는지:

| 스프레드시트 카테고리 | JS expenses 키 | 비고 |
|----------------------|----------------|------|
| Rent | `housing` | rent + 선한 Rent |
| 십일조 | `tithing` | |
| 공과금 | `utilities` | Gfiber, GP, AT&T 등 |
| 보험 | `insurance` | Geico, 나몽 보험, 건강보험 |
| 나몽 / 나몽&망고 | `pet` | 월마다 항목이 다름 |
| Grocery | `grocery` | Publix, Megamart, Costco 등 |
| 문화생활 | `entertainment` | Netflix, 보드게임 등 |
| 외식 | `dining` | CFA, BBQ, 샤브샤브 등 |
| Shopping | `shopping` | Amazon 등 |
| Gas | `gas` | 주유 |
| member | `membership` | Costco 멤버십 등 |
| 서빙 / Bank | `other` | 기타 (Kroger, service fee 등) |

> 중요: 카테고리 항목은 월마다 다름! 1월에 있는 카테고리가 2월에 없을 수 있고, 2월에만 있는 카테고리도 있음. CSV에서 해당 월에 실제로 나타나는 항목만 JS에 추가할 것.

### 저축 목표 (스프레드시트 맨 아래 Goal 섹션)

| 항목 | JS savingsGoals | 현재 값 |
|------|----------------|---------|
| Car | `{ name: "Car Purchase", nameKo: "차", targetNum: 30000 }` | $30,000 |
| Flight | `{ name: "Flight Ticket", nameKo: "one-way 비행기", targetNum: 5500 }` | $5,500 |
| Green Card | `{ name: "Green Card", nameKo: "영주권 (변호사+서류)", targetNum: 8000 }` | $8,000 |
| Bonus return | `{ name: "Bonus Return", nameKo: "보너스 리턴 (반납)", targetNum: 5000 }` | $5,000 |
| **합계** | `totalGoal` (코드 내 하드코딩) | **$48,500** |

> **주의**: Bonus Return은 저축 목표가 아니라, 병원에서 받은 보너스를 나중에 반납해야 하는 돈 (부채). 저축으로 모아서 갚아야 할 금액.

## Step 3: 코드 내 업데이트 위치 (index.html)

> 모든 라인 번호는 대략적임. `grep`으로 확인 필요.

### 3-1. `budgetData` 객체 (~line 1172)

```javascript
const budgetData = {
    jan: { income: {...}, expenses: {...}, totalExpenses: N, savings: {...}, travel: {...} },
    feb: { ... },
    // 새 달 추가 시 여기에 추가
};
```

**각 월 객체의 구조:**
```javascript
monthKey: {
    income: {
        chamSalary: N,      // 참새월급 (없으면 생략)
        haniSalary: N,      // 하니월급 (없으면 생략)
        // 기타 수입 항목 (월마다 다름)
        total: N             // 총액 행의 값
    },
    expenses: {
        housing: { rent: N, seonhanRent: N, sum: N },
        tithing: { tithing: N, sum: N },
        utilities: { gfiber: N, gp: N, att: N, sum: N },
        insurance: { geico: N, namongInsurance: N, healthInsurance: N, sum: N },
        pet: { namong: N, ..., sum: N },
        grocery: { publix: N, megamart: N, ..., sum: N },
        entertainment: { netflix: N, ..., sum: N },
        dining: { ..., sum: N },
        shopping: { ..., sum: N },
        gas: { ..., sum: N },
        membership: { ..., sum: N },
        other: { ..., sum: N }
        // 해당 월에 존재하는 카테고리만 포함!
    },
    totalExpenses: N,        // "Total" 행의 값 (모든 카테고리 SUM의 합)
    savings: { daehyuni: N, hani: N, total: N },  // 저축 행
    travel: { ... }          // 여행 섹션 (있는 경우만)
}
```

### 3-2. `savingsGoals` 배열 (~line 1274)

```javascript
const savingsGoals = [
    { name: "Green Card", nameKo: "영주권 (변호사+서류)", target: "$8,000", targetNum: 8000 },
    { name: "Flight Ticket", nameKo: "one-way 비행기", target: "$5,500", targetNum: 5500 },
    { name: "Car Purchase", nameKo: "차", target: "$30,000", targetNum: 30000 },
    { name: "Bonus Return", nameKo: "보너스 리턴", target: "$5,000", targetNum: 5000 }
];
```
- 스프레드시트 맨 아래 `Goal` 섹션이 변경되면 여기도 업데이트

### 3-3. `currentSavingsBalance` (~line 1282)

```javascript
const currentSavingsBalance = 4500.08; // $4,500.08
```
- **Savings 통장 잔고만** 사용 (참새통장/냥이통장/공동통장은 각자 비상금이라 합산 X)
- 스프레드시트에서 가장 최근 달의 수입 라벨 열에서 `Savings` 행의 금액을 읽어올 것
- 주석도 함께 업데이트

### 3-4. `updateTrendChart()` 함수 (~line 1330)

```javascript
const months = ['Sep', 'Oct', 'Nov', 'Dec', 'Jan', 'Feb'];
const incomeData = [4153, 5300, 4333.87, 5461.22, 15353.08, 3458.22];
const expenseData = [5369.42, 4405.71, 3426.90, 6555.99, 8476.59, 7442.08];
```
- `months` 배열에 새 달 추가 (영문 약어: 'Mar', 'Apr', 등)
- `incomeData`에 해당 월 income.total 추가
- `expenseData`에 해당 월 totalExpenses 추가
- **순서**: 시간순 (오래된 달 → 최신 달)

### 3-5. `updateExpenseBarChart()` 함수 (~line 1441)

```javascript
const months = ['Sep', 'Oct', 'Nov', 'Dec', 'Jan', 'Feb'];
// ...
{ label: 'Housing',   data: [3200, 3200, 1200, 3200, 3362.80, 3353.44] },
{ label: 'Utilities', data: [272, 220, 0, 180, 255.99, 228.46] },
{ label: 'Insurance', data: [140, 140, 0, 140, 1487.93, 173.00] },
{ label: 'Other',     data: [1757.42, 845.71, 2226.90, 3035.99, 3369.87, 3687.18] }
```
- `months` 배열에 새 달 추가
- 각 dataset의 `data` 배열에 해당 월 값 추가:
  - **Housing** = `expenses.housing.sum`
  - **Utilities** = `expenses.utilities.sum`
  - **Insurance** = `expenses.insurance.sum`
  - **Other** = `totalExpenses - housing.sum - utilities.sum - insurance.sum`

### 3-6. 월 선택 버튼 HTML (~line 995-1000)

```html
<button class="month-btn" onclick="selectMonth('sep')">9월</button>
<button class="month-btn" onclick="selectMonth('oct')">10월</button>
<button class="month-btn" onclick="selectMonth('nov')">11월</button>
<button class="month-btn" onclick="selectMonth('dec')">12월</button>
<button class="month-btn" onclick="selectMonth('jan')">1월</button>
<button class="month-btn active" onclick="selectMonth('feb')">2월</button>
```
- 새 달 추가 시: 새 버튼 추가하고, `active` 클래스를 최신 달로 이동

### 3-7. `init()` 함수의 기본 월 (~line 1760)

```javascript
updateSummaryCards(budgetData.feb);
updateTrendChart();
updateCategoryChart(budgetData.feb);
updateExpenseBarChart();
updateSavingsChart();
updateInsights(budgetData.feb);
updateExpenseTable(budgetData.feb);
```
- 모든 `budgetData.feb` → `budgetData.{최신월}` 로 변경
- `currentMonth` 변수 (~line 1284)도 최신 월로 변경: `let currentMonth = 'feb';`

### 3-8. `updateInsights()` 내 totalGoal (~line 1537)

```javascript
const totalGoal = 48500; // Green Card + Flight + Car + Bonus
```
- savingsGoals의 targetNum 합계가 변경되면 이 하드코딩 값도 변경

### 3-9. `updateNamongProgress()` 내 totalGoal (~line 1725)

```javascript
const totalGoal = savingsGoals.reduce((sum, g) => sum + g.targetNum, 0);  // $48,500
```
- 이 줄은 자동 계산이므로 savingsGoals만 맞으면 OK
- 주석의 `$48,500`이 실제 합계와 다르면 주석 업데이트

## Step 4: 검증 체크리스트

모든 업데이트 후 반드시 확인:

- [ ] **지출 합계 검증**: 모든 카테고리의 `sum` 값을 더한 것 = `totalExpenses`
  - 예: `housing.sum + tithing.sum + utilities.sum + insurance.sum + pet.sum + grocery.sum + entertainment.sum + dining.sum + shopping.sum + gas.sum + membership.sum + other.sum = totalExpenses`
- [ ] **트렌드 차트 일치**: `incomeData` 배열의 각 값 = 해당 월 `budgetData.{month}.income.total`
- [ ] **트렌드 차트 일치**: `expenseData` 배열의 각 값 = 해당 월 `budgetData.{month}.totalExpenses`
- [ ] **바 차트 Other 계산**: Other = totalExpenses - Housing - Utilities - Insurance (각 월별로)
- [ ] **저축 잔고**: `currentSavingsBalance` = 최신 월의 Savings 통장 잔고만 (다른 통장 합산 X)
- [ ] **저축 목표 합계**: savingsGoals의 targetNum 합 = `updateInsights()`의 `totalGoal` 하드코딩 값
- [ ] **기본 월 참조**: `init()` 내 모든 `budgetData.XXX`와 `currentMonth` 변수가 최신 달을 가리킴
- [ ] **월 버튼**: 최신 달 버튼에 `active` 클래스가 있음
- [ ] **months 배열 길이**: trendChart와 barChart의 months/data 배열 길이가 동일

## Step 5: Commit, Push, Sync

```bash
cd /tmp/financial-page
git add index.html
git commit -m "N월 데이터 업데이트"
export GH_CONFIG_DIR=~/gh-config
git push
```

그 후 Google Drive 동기화:
```bash
cp /tmp/financial-page/index.html "/Users/choidaehyun/Library/CloudStorage/GoogleDrive-daehyeonchoi@gmail.com/My Drive/Favorites/Financial/index.html"
```

## 새 달 추가 시 추가 작업 요약

새 달(예: 3월)을 추가할 때 변경해야 할 **모든** 위치:

1. `budgetData` 객체에 `mar: { ... }` 추가
2. `currentSavingsBalance` 값을 3월 통장 잔고로 업데이트
3. `updateTrendChart()` — months, incomeData, expenseData 배열 각각에 값 추가
4. `updateExpenseBarChart()` — months 배열 + Housing/Utilities/Insurance/Other data 배열 각각에 값 추가
5. 월 선택 버튼 HTML — 새 버튼 추가 + `active` 클래스 이동
6. `init()` 함수 — 모든 `budgetData.feb` → `budgetData.mar`
7. `currentMonth` 변수 — `'feb'` → `'mar'`
8. `savingsGoals` — Goal 섹션이 변경된 경우만 업데이트
9. `totalGoal` 하드코딩 — savingsGoals 합계가 변경된 경우만 업데이트

## 기존 달 수정 시 작업 요약

기존 달(예: 1월 데이터 수정)의 경우:

1. `budgetData` 내 해당 월 객체 수정
2. trendChart의 incomeData/expenseData 해당 인덱스 수정
3. barChart의 Housing/Utilities/Insurance/Other 해당 인덱스 수정
4. 해당 월이 최신 월이면 `currentSavingsBalance`도 확인

---

## 데이터 업데이트 절차 (요약)
1. `cd /tmp/financial-page && git pull`
2. WebFetch로 CSV 데이터 가져오기 (gviz URL 사용)
3. CSV 파싱하여 각 월 데이터 추출
4. 위 Step 3의 모든 코드 위치 업데이트
5. Step 4 검증 체크리스트 확인
6. git commit & push
7. Google Drive `index.html` 동기화

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
`SANKEY_MONTH_INFO` 객체에 월 → `{ name, year, days }` 매핑. 새 달 추가 시 갱신:
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
