# 주문서 앱 개발 스킬

> Claude + 기획자 콤비로 주문서 앱을 만드는 검증된 패턴 모음
> 영화당 떡케이크/화과자 주문서 개발 과정에서 도출된 실전 노하우

-----

## 1. 기획서 → 앱 변환 구조

### 기획서에서 뽑아야 할 것들

```
1. 가게 정보: 가게명, 연락처, 입금계좌, 계좌주
2. 상품 구조: 상품명, 사이즈/수량 옵션, 가격
3. 추가 옵션: 옵션명, 가격, 설명
4. 주문 흐름: 손님이 입력할 항목들 (이름/연락처/날짜/시간 등)
5. 수령 방법: 픽업/배달/택배 여부
6. 결제 수단: 계좌이체/카드/지역화폐 등
7. 어드민 필요 기능: 주문 확인, 상태 변경, 메모, 필터
8. 설정 변경 항목: 사장님이 직접 바꿔야 할 텍스트/가격/사진
```

### 앱 구조 설계 원칙

- 손님용 + 어드민을 한 파일로 (URL 해시로 구분: `#admin-{키워드}`)
- 설정값은 Supabase에 저장, 기본값은 DEFAULT_CFG로 코드에 내장
- 단일 HTML 파일로 완성 (CSS + JS 분리 금지)

-----

## 2. 기술 스택

### 배포

- **GitHub** → 코드 저장소
- **Cloudflare Pages** → 무료 배포, GitHub 연동 자동 배포
- URL 구조: `{프로젝트명}.pages.dev/{파일명}.html`
- 루트 접속 원하면 파일명을 `index.html`로

### DB

- **Supabase 무료 플랜** (주문 데이터, 설정 저장)
- 테이블: `cake_orders` (주문), `cake_photos2` (설정/사진)
- RLS 반드시 비활성화 (anon key로 접근)
- 사진은 base64로 DB에 저장 (용량 주의: 케이크 최대 1200px, quality 0.95)

### 핵심 패턴

```javascript
const SUPA = 'https://{프로젝트}.supabase.co/rest/v1';
const KEY = '{anon key}';
const H = {'apikey':KEY,'Authorization':'Bearer '+KEY,
           'Content-Type':'application/json','Prefer':'return=representation'};

async function api(path, method, body) {
  return fetch(SUPA+path, {method:method||'GET', headers:H,
    body:body?JSON.stringify(body):undefined});
}
```

-----

## 3. 치명적 오류 패턴 (반드시 지킬 것)

### 오류 1: JS 문자열 안 HTML 닫는 태그

브라우저가 `</script>` 로 오인해서 파싱 중단.

```javascript
// ❌ 절대 금지
h += '<div class="sec">내용</div>';

// ✅ 반드시 이스케이프
h += '<div class="sec">내용<\/div>';
```

**검증 방법**: 배포 전 반드시 아래 명령어로 체크

```bash
node --check {파일명}.js
```

### 오류 2: JS 인라인 스타일에서 CSS 변수 사용

Cloudflare, Safari 등 일부 환경에서 `var(--color)` 가 인라인 style 속성에서 작동 안 함.

```javascript
// ❌ 절대 금지 - JS 문자열 안에서
h += '<div style="color:var(--cake);">텍스트<\/div>';

// ✅ CSS 클래스로만
h += '<div class="text-cake">텍스트<\/div>';
// CSS에 미리 정의: .text-cake { color: #6b4c3b; }
```

**핵심 원칙**: JS에서 동적으로 색상 변경할 때는 **CSS 클래스 교체**만 사용

```javascript
// ✅ 올바른 패턴
const activeClass = isActive ? 'btn-active-cake' : 'btn-inactive';
h += '<button class="btn '+activeClass+'">버튼<\/button>';
```

### 오류 3: CSS 변수 자체도 hex 병행 표기

```css
/* ✅ 안전한 방식 - fallback hex 먼저 */
html, body { background: #f2f2f7; background: var(--bg); }
.card { background: #ffffff; background: var(--card); }
```

### 오류 4: 따옴표 중첩

JS 문자열 안 HTML 속성에서 따옴표가 중첩되면 파싱 오류.

```javascript
// ❌ 위험
h += '<button onclick="chgSt('confirmed')">확정<\/button>';

// ✅ 안전 - 이스케이프
h += '<button onclick="chgSt(\'confirmed\')">확정<\/button>';
```

-----

## 4. 다중 상품 구조 (SaaS 확장 설계)

### 탭 구조 패턴

```javascript
// 상품 탭 전환
let curTab = 'cake'; // 'cake' | 'hwa'
function switchTab(tab) {
  curTab = tab;
  document.querySelectorAll('.hdr-tab').forEach(el => el.classList.remove('on'));
  document.querySelector('.hdr-tab.'+tab).classList.add('on');
  if (tab === 'cake') renderCake(); else renderHwa();
}
```

### 어드민에서 탭별 색상 구분

```css
/* 상품별 고유 색상 */
--cake: #6b4c3b;  /* 떡케이크: 브라운 */
--hwa:  #3a5a7a;  /* 화과자: 네이비 */
```

### 주문 카드 왼쪽 컬러 라인으로 구분

```css
.order-card-cake { border-left: 3px solid #6b4c3b; }
.order-card-hwa  { border-left: 3px solid #3a5a7a; }
```

-----

## 5. 설정 시스템 패턴

### 기본값 + DB 저장 구조

```javascript
const DEFAULT_CFG = {
  shop_name: '가게명',
  shop_phone: '010-0000-0000',
  bank_account: '은행 계좌번호',
  bank_holder: '예금주',
  // 상품별 옵션들...
};

let CFG = JSON.parse(JSON.stringify(DEFAULT_CFG));

async function loadSettings() {
  // Supabase에서 설정 불러와서 CFG에 병합
}
async function saveSettings(data) {
  // Supabase에 저장
}
```

### 설정 가능 항목 체크리스트

- [ ] 가게명, 연락처, 계좌
- [ ] 상품명, 사이즈/수량, 가격
- [ ] 추가 옵션 (추가/삭제 가능하게)
- [ ] 안내 문구 (픽업 안내, 보관 방법 등)
- [ ] 예시 문구 (레터링 예시 등)
- [ ] 디자인 설명 텍스트, placeholder
- [ ] 상품별 사진 (최대 3장, 슬라이드)
- [ ] 팝업 사진 + 설명 (초콜릿 장식 등)
- [ ] 포장 옵션 (추가/삭제 가능하게)

-----

## 6. 사진 업로드 패턴

### 이미지 리사이즈 (반드시 사용)

```javascript
function resizeImage(file, maxW, maxH, cb) {
  const reader = new FileReader();
  reader.onload = function(e) {
    const img = new Image();
    img.onload = function() {
      const canvas = document.createElement('canvas');
      let w = img.width, h = img.height;
      if (w > maxW) { h = h * maxW / w; w = maxW; }
      if (h > maxH) { w = w * maxH / h; h = maxH; }
      canvas.width = w; canvas.height = h;
      canvas.getContext('2d').drawImage(img, 0, 0, w, h);
      cb(canvas.toDataURL('image/jpeg', 0.95)); // quality 0.95
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
}
```

- 케이크/화과자 상품 사진: 1200x1200
- 팝업 사진 (초콜릿 장식 등): 1200x1600 (세로형)
- 손님 디자인 참고 사진: 1200x1200

-----

## 7. 애플 스타일 디자인 시스템

### 색상 팔레트

```css
--bg: #f2f2f7;        /* iOS 시스템 배경 */
--card: #ffffff;       /* 카드 배경 */
--text: #1c1c1e;       /* 본문 */
--text2: #8e8e93;      /* 보조 텍스트 */
--text3: #c7c7cc;      /* 힌트 텍스트 */
--sep: rgba(60,60,67,0.1); /* 구분선 */
--r: 13px;             /* 기본 border-radius */
```

### 핵심 디자인 원칙

- 이모지 남발 금지: 꼭 필요한 곳(성공화면, 빈 화면)만
- 섹션 헤더: uppercase, 12px, font-weight:600, color:#8e8e93
- 카드: background:#fff, border-radius:13px, margin:0 16px
- 버튼 활성화: CSS 클래스로만 (`on-cake`, `on-hwa`)
- 토글 스위치: 활성화 시 상품 색상으로

### 위계 구조

```
헤더 (sticky, blur)
  └ 상품 탭
본문
  └ 섹션 헤더 (12px, uppercase, 보조색)
    └ 카드 (16px margin, 13px radius)
      └ 행 (48px min-height, 0.5px 구분선)
```

-----

## 8. 데모 파일 제작 패턴

### 실제 DB → 메모리 mock으로 교체

```javascript
// 실제 Supabase 코드 대신
let _demoOrders = [ /* 샘플 주문 3개 */ ];
let _demoSettings = null;

async function api() { return {ok:true, json:async function(){return[];}}; }
async function getOrders(type) {
  await new Promise(r=>setTimeout(r,300)); // 로딩 느낌
  return _demoOrders.filter(o=>o.type===type);
}
async function addOrder(o) { _demoOrders.unshift(o); }
async function updateOrder(id,p) {
  const i = _demoOrders.findIndex(o=>o.id===id);
  if (i>=0) _demoOrders[i] = Object.assign({}, _demoOrders[i], p);
}
async function delOrder(id) {
  _demoOrders = _demoOrders.filter(o=>o.id!==id);
}
```

### 개인정보 제거 체크리스트

- [ ] 전화번호 → 010-0000-0000
- [ ] 계좌번호 → 000-0000-0000
- [ ] 예금주 → 홍길동
- [ ] 가게명 → 샘플 케이크샵
- [ ] 주소 → 제거 또는 00으로 대체
- [ ] 지역명 (부여 등) → 제거

### 데모 배너

```javascript
// 상단에 데모임을 명시
'<div style="background:linear-gradient(135deg,#8b4a4a,#4a6b8b);color:white;...">'
+ '데모 버전 · 실제 주문은 저장되지 않아요'
+ '</div>'
```

-----

## 9. Cloudflare Pages 배포

### GitHub 연동 배포

1. Cloudflare Dashboard → Workers & Pages → Create
1. “Import Git repository” 선택
1. 레포 선택, 빌드 명령어 비워두기
1. 파일 접근: `{프로젝트}.pages.dev/{파일명}.html`
1. 루트 접속: 파일명을 `index.html`로

### 자동 배포

- GitHub main 브랜치에 push하면 자동 재배포
- 영화당 실제 앱: `youngwhadang.pages.dev/dduck.html`
- 데모: `cakee-order-demo.pages.dev/demo.html`

-----

## 10. SaaS 확장 설계 (향후)

### shop_id 기반 멀티 가게 구조

```sql
-- cake_orders 테이블에 shop_id 컬럼 추가
ALTER TABLE cake_orders ADD COLUMN shop_id text;

-- 가게별 필터링
SELECT * FROM cake_orders WHERE shop_id = 'shop_001';
```

### 무료 운영 가능 규모 (Supabase 무료 플랜 기준)

- DB 용량: 500MB → 텍스트 주문 데이터 수년치 가능
- API 요청: 월 200만건 → 1,000곳 × 하루 10주문 × 월 30일 = 약 90만건
- 사진은 Cloudflare R2 무료 플랜 (월 10GB) 별도 저장 권장

### 가격 전략

- 초기: 일시불 세팅비 5만원
- 확장: 월 9,900원 구독 (커피 두 잔)
- 기능 추가 시: 월 19,900원

-----

## 11. 가게별 전용 앱 디자인 시스템 (플로앙 v1)

플로앙 앙금플라워 케이크 전용으로 만든 디자인 시스템. 따뜻하고 우아한 느낌.

### 컬러 팔레트

```css
--bg: #f5f2ef;          /* 따뜻한 아이보리 배경 */
--card: #ffffff;
--text: #1a1714;        /* 따뜻한 블랙 */
--text2: #8a837c;       /* 보조 텍스트 */
--text3: #c4bdb7;       /* 힌트 */
--sep: rgba(50,40,35,0.08);  /* 구분선 */
--accent: #7a4f3a;      /* 딥 로즈브라운 포인트 */
--accent2: rgba(122,79,58,0.07);
```

### 핵심 디자인 원칙

- border-radius **16px** (카드), **14px** (버튼/시트), **22px** (칩/토스트)
- 그림자: `0 1px 0 rgba(50,40,35,0.06), 0 2px 8px rgba(50,40,35,0.04)` (얇고 섬세하게)
- 버튼 주요 액션: `box-shadow:0 4px 16px rgba(122,79,58,0.25)` (색상 그림자)
- 토글 스프링: `cubic-bezier(0.34,1.56,0.64,1)` (iOS 느낌)
- 버튼 탭: `transition: opacity 0.15s, transform 0.12s` + `:active { transform:scale(0.97); }`
- 헤더 블러: `backdrop-filter:blur(24px) saturate(1.8)`
- 토스트: `backdrop-filter:blur(8px)` 추가

### 선택 박스(sz) 테두리 잘림 방지

```css
/* sz-scroll에 padding 여유 확보 */
.sz-scroll { padding:4px 16px 6px; }

/* 선택 시 box-shadow 링 쓰지 말 것 → border만 사용 */
.sz.on-cake { border-color:#7a4f3a; background:rgba(122,79,58,0.05); }
/* ❌ box-shadow:0 0 0 3px ... 는 overflow로 잘림 */
```

### 가게 특화 섹션 추가 방법

기존 사이즈 선택과 동일한 sz 슬라이드 패턴으로 추가 선택 항목 구현:

```javascript
// 스타일 선택 (돔/크레센트/리스 등)
const styles = [{id:'dome', name:'돔', desc:'가득찬 둥근형', e:'🌸'}, ...];
styles.forEach(function(st) {
  const isOn = fd.style === st.id;
  h += '<div class="sz'+(isOn?' on-cake':'')+'" onclick="fd.style=\''+st.id+'\';renderCake()">';
  // ... 내용
});

// fd 초기화에 추가 필드 포함
let fd = {sizes:{}, style:'', taste:'', delivery:false, ...};

// 주문 제출 전 유효성 검사
if (!fd.style) { toast('스타일을 선택해주세요'); return; }
if (!fd.taste) { toast('맛을 선택해주세요'); return; }
```

### 불필요 섹션 제거 시 주의

- renderCake() 함수에서 해당 섹션 h += 블록 통째로 제거
- DEFAULT_CFG에서도 관련 필드 제거 또는 유지 (설정 화면에 영향)
- fd 초기화, submitCake() 유효성 검사, addOrder() 호출부도 함께 확인

-----

## 12. 향후 추가할 기능 (피드백 기반)

실제 사용 가게 생기면 추가할 기능들:

1. **카카오톡 알림** - 주문 들어오면 사장님 카톡으로 알림
1. **예약 캘린더** - 픽업 날짜별 주문 수 한눈에 보기
1. **손님 주문 확인 링크** - 제출 후 주문 내역 다시 볼 수 있게
