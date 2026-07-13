---
layout: post
title: iPad Homepage 프로젝트에서 생각해봐야할 것들 (2) — 푸터 아코디언 편
date: 2026-07-11
tags: [CSS, JavaScript, Accordion, Accessibility]
inline: false
related_posts: false
toc:
  sidebar: right
---

<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">

지난 글에서 스프라이트 아이콘, `IntersectionObserver`, 마스킹 같은 개별 트릭을 하나씩 뜯어봤다면, 이번엔 컴포넌트 하나를 처음부터 끝까지 관통해서 본다. 애플 홈페이지 하단에 있는 "사이트맵" 아코디언 — 클릭하면 펼쳐지고 다시 클릭하면 접히는 그 메뉴 — 를 만든 코드는 JS 5줄, CSS 20줄 남짓이다. 짧지만 그 안에 "동적 렌더링", "height 트랜지션의 한계", "브레이크포인트별 역할 분리", "접근성 트레이드오프"가 전부 들어있다.

## 이 컴포넌트가 하는 일

`data/navigations.js`에는 "쇼핑 및 알아보기", "서비스", "계정" 같은 8개 카테고리가 배열로 정의돼 있고, 각 카테고리는 `title`과 `maps`(링크 목록)를 가진다.

```js
export default [
  {
    title: '쇼핑 및 알아보기',
    maps: [
      { name: '스토어', url: '/shop/goto/store' },
      { name: 'Mac', url: '/mac' },
      // ...
    ]
  },
  // ... 총 8개 카테고리
]
```

`js/main.js`는 이 배열을 순회하며 `<div class="map">` 8개를 만들어 `footer .navigations`에 밀어 넣는다.

```js
const navigationsEl = document.querySelector('footer .navigations');
navigations.forEach(function (nav) {
  const mapEl = document.createElement('div');
  mapEl.classList.add('map');

  let mapList = '';
  nav.maps.forEach(function (map) {
    mapList += `<li><a href="${map.url}">${map.name}</a></li>`;
  });

  mapEl.innerHTML = `
    <h3>
      <span class="text">${nav.title}</span>
      <span class="icon">+</span>
    </h3>
    <ul>
      ${mapList}
    </ul>
  `;

  navigationsEl.appendChild(mapEl);
});
```

그리고 렌더링이 끝난 뒤, 방금 만든 `.map` 요소들을 다시 한번 순회하며 클릭 핸들러를 붙인다.

```js
const mapEls = document.querySelectorAll('footer .navigations .map');
mapEls.forEach(function (el) {
  const h3El = el.querySelector('h3');
  h3El.addEventListener('click', function () {
    el.classList.toggle('active');
  });
});
```

핵심 로직은 이게 전부다. `h3`를 클릭하면 부모 `.map`에 `active` 클래스를 토글한다. 펼치고 접는 애니메이션, 아이콘이 회전하는 것, 심지어 "아코디언처럼 보이는 것" 자체도 JS는 전혀 모른다. 전부 CSS가 한다.

---

## 1. `height: auto`는 트랜지션이 안 된다 — 그래서 우회한다

아코디언을 처음 만들어보면 누구나 이 함정에 걸린다. "펼쳐진 목록의 실제 높이"는 항목 개수에 따라 다 다른데(3개짜리 카테고리도 있고 10개짜리도 있다), CSS `transition`은 `height: 0`에서 `height: auto`로 가는 애니메이션을 지원하지 않는다. `auto`는 애니메이션 가능한 값이 아니라서, 브라우저는 그 구간을 보간 없이 즉시 점프시켜 버린다.

이 프로젝트는 그 한계를 정면돌파하는 대신 우회한다. `height`는 그냥 순간적으로 바뀌게 놔두고, **실제로 부드럽게 보이는 효과는 `transform`과 `opacity`가 전담**한다.

```css
/* 접힌 상태 (기본) */
footer .navigations .map ul {
  height: 0;
  overflow: hidden;
  transition:
    transform 0.6s,
    opacity 0.4s;
  transform: translate(0, -20px);
  opacity: 0;
}

/* 아코디언 메뉴 */
footer .navigations .map.active ul {
  height: auto;
  overflow: visible;
  padding: 6px 0 18px;
  transform: translate(0, 0);
  opacity: 1;
}
```

### 왜 이게 "괜찮게" 보이는가

`height`는 `0`과 `auto` 사이를 순간이동하지만, `overflow: hidden`이 걸려 있는 동안은 어차피 내용물이 보이지 않는다. 그리고 `transform: translate(0, -20px) → translate(0, 0)`과 `opacity: 0 → 1`은 정상적으로 애니메이션되는 속성이라, 눈에는 "위에서 살짝 내려오면서 페이드인" 하는 것처럼 보인다. 즉 이 아코디언은 진짜로 "펼쳐지는" 게 아니라, **height는 접혔다 폈다를 툭툭 스냅하고, 그 위에 transform+opacity로 만든 슬라이드-페이드 효과를 덧씌운** 것이다.

이 트릭의 한계는 항목이 아주 많은 카테고리(예시 데이터 기준 "Apple Store" 카테고리는 10개 링크)에서 드러난다. `height: 0 → auto`가 순간 전환되기 때문에, 링크가 10개인 목록이나 3개인 목록이나 "펼쳐지는 속도감"은 동일하다 — 원래 height 자체를 애니메이션했다면 내용이 많을수록 더 크게 열리는 느낌을 줄 수 있었겠지만, 여기선 그 디테일은 포기하고 구현 단순성을 택한 것이다.

### 심화 — `grid-template-rows: 0fr → 1fr`라는 대안

최근 브라우저(2023년 이후 Chrome/Firefox/Safari)에서는 `fr` 단위가 트랜지션 가능한 값이 되면서, `height` 대신 CSS Grid로 이 문제를 더 깔끔하게 풀 수 있게 됐다.

```css
.map ul {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.6s;
}
.map.active ul {
  grid-template-rows: 1fr;
}
.map ul > .inner {
  overflow: hidden;
}
```

이 방식은 `height`가 실제로 콘텐츠 높이만큼 부드럽게 늘어나기 때문에, `translate` + `opacity`로 눈속임할 필요 없이 "진짜 펼쳐지는" 느낌을 준다. 다만 `ul` 안에 `overflow: hidden`을 가진 래퍼 한 겹(`.inner`)이 추가로 필요하다는 제약이 있다. 이 프로젝트가 만들어진 시점을 생각하면 `translate` + `opacity` 조합이 더 안전한 선택이었을 수 있지만, 지금 새로 만든다면 `grid-template-rows` 쪽이 코드도 더 단순하고 실제 높이감도 살아있어 더 나은 선택이다.

---

## 2. 아이콘 "+"가 "×"로 변신하는 방법

각 카테고리 제목 옆의 `+` 아이콘은 별도의 이미지나 SVG가 아니라 그냥 텍스트 `+`다. 열렸을 때 `×`처럼 보이는 것도 이미지 교체가 아니라 순수 CSS 회전이다.

```css
footer .navigations .map.active h3 .icon {
  transform: scale(1.2) rotate(45deg);
}
```

`+` 글자를 45도 돌리면 가로줄과 세로줄이 대각선으로 눕는데, 이게 시각적으로 `×`(닫기) 기호처럼 읽힌다. `scale(1.2)`는 회전하면서 살짝 작아 보이는 착시를 보정하려고 더해진 값으로 보인다. 아이콘을 여닫을 때마다 이미지 스와핑이나 별도의 `.icon--close` 클래스 없이, 문자 하나 + `transform` 두 줄로 열림/닫힘 상태를 표현한 셈이다. 실무에서 "아이콘 상태 변화"를 만들 때 아이콘 폰트나 SVG 두 벌을 준비하는 대신 이런 순수 CSS 변형으로 처리할 수 있다는 걸 보여주는 예시다.

---

## 3. 같은 이벤트 리스너, 다른 시각적 결과 — 브레이크포인트가 동작을 결정한다

`h3El.addEventListener('click', ...)`는 화면 크기와 무관하게 항상 등록된다. 데스크톱에서 이 링크 목록을 클릭해도 `classList.toggle('active')`는 똑같이 실행된다. 그런데 실제 애플 사이트에서 데스크톱 화면의 푸터 사이트맵은 애초에 접혀있지 않고 8개 카테고리가 전부 펼쳐진 채로 보인다. 아코디언처럼 동작하는 건 모바일(740px 이하)뿐이다.

이걸 가능하게 하는 건 아이콘과 `ul`에 걸린 규칙이 전부 미디어쿼리 안에 있기 때문이다.

```css
/* 기본(데스크톱/태블릿): 아이콘 자체를 숨김 */
footer .navigations .map h3 .icon {
  display: none;
}

@media (max-width: 740px) {
  footer .navigations .map {
    margin-top: 0;
    border-bottom: 1px solid var(--color-border);
  }
  footer .navigations .map h3 {
    padding: 12px 0;
    font-weight: 400;
    display: flex;
  }
  footer .navigations .map:hover h3 {
    font-weight: 600;
  }
  footer .navigations .map h3 .text {
    flex-grow: 1;
  }
  footer .navigations .map h3 .icon {
    display: block;
    padding: 0 10px;
    color: var(--color-font-darkgray);
  }
  footer .navigations .map ul {
    height: 0;
    overflow: hidden;
    transition: transform 0.6s, opacity 0.4s;
    transform: translate(0, -20px);
    opacity: 0;
  }
  footer .navigations .map.active ul {
    height: auto;
    overflow: visible;
    padding: 6px 0 18px;
    transform: translate(0, 0);
    opacity: 1;
  }
}
```

`height: 0`, `overflow: hidden` 같은 "접힘" 규칙 자체가 740px 이하에서만 존재한다. 데스크톱/태블릿에는 이 규칙이 아예 없으니 `ul`은 항상 기본값(`height: auto`나 다름없는 자연스러운 블록 높이)으로 펼쳐져 있고, `active` 클래스가 붙든 안 붙든 시각적으로 달라질 게 없다. `h3 .icon`도 기본이 `display: none`이라 데스크톱에서는 `+`/`×` 아이콘 자체가 렌더링되지 않는다.

### 왜 이 설계가 영리한가

"아코디언이냐 아니냐"를 JS가 판단해서 분기 처리(`if (window.innerWidth <= 740) { ... }`)하는 방법도 있었을 것이다. 하지만 그렇게 했다면 리사이즈 이벤트마다 상태를 다시 계산하고, 데스크톱에서 모바일로 전환되는 순간의 엣지 케이스(이미 열려 있던 게 갑자기 접혀야 하는지 등)를 신경 써야 했다. 대신 이 프로젝트는 **"클릭했을 때 상태를 뒤집는다"는 사실 하나만 JS가 책임지고, 그 상태가 화면에 어떻게 반영될지는 전적으로 CSS 미디어쿼리에 위임**했다. 덕분에 리사이즈 핸들러도, 뷰포트 분기 로직도 필요 없다 — 브라우저가 미디어쿼리를 재평가하는 걸 그냥 이용하는 것이다. `js/main.js` 안에서 리사이즈 이벤트를 직접 구독하는 코드는 검색바(`.searching` 클래스 제거)에만 있고, 아코디언 쪽에는 전혀 없다는 게 이 설계를 뒷받침한다.

---

## 4. 아쉬운 점 — "하나만 열리는" 아코디언이 아니다, 그리고 접근성

### 배타적 열림(exclusive open)이 구현돼 있지 않다

`forEach`로 8개의 `.map`에 각각 독립적인 리스너를 붙였을 뿐, "다른 걸 열면 나머지는 닫는다"는 로직이 없다. 그래서 사용자가 8개 카테고리를 전부 클릭하면 8개가 동시에 펼쳐진 채로 남는다. 애플 실사이트도 실제로 이렇게 동작하기 때문에 "버그"는 아니지만, 흔히 알려진 "아코디언 = 하나만 열림" 패턴을 기대하고 구현하면 놓치기 쉬운 지점이다. 배타적으로 만들고 싶다면 클릭 시 형제 요소를 순회하며 닫아주는 코드가 필요하다.

```js
mapEls.forEach(function (el) {
  const h3El = el.querySelector('h3');
  h3El.addEventListener('click', function () {
    const willOpen = !el.classList.contains('active');
    mapEls.forEach(function (other) {
      other.classList.remove('active');
    });
    if (willOpen) {
      el.classList.add('active');
    }
  });
});
```

### `<h3>`는 클릭 가능한 요소가 아니다

클릭 핸들러가 `<button>`이 아니라 `<h3>`에 달려 있다. 마우스로는 문제없이 동작하지만, 키보드만으로 이 페이지를 탐색하는 사용자는 `Tab`으로 이 요소에 포커스를 옮길 수조차 없다(`h3`는 기본적으로 포커스 가능한 요소가 아니다). 스크린 리더 사용자 입장에서도 이 항목이 "펼치고 접을 수 있는 토글 버튼"이라는 정보(`aria-expanded`)가 전혀 없어서, 그냥 제목처럼 읽히고 지나간다. README의 회고에도 "접근성(ARIA, 포커스 트랩) 속성 보강은 진행하지 않았다"고 스스로 짚었는데, 이 아코디언이 바로 그 사례다.

실무에서 같은 컴포넌트를 만든다면 최소한 이 정도는 갖추는 걸 권장한다.

```html
<h3>
  <button class="text" aria-expanded="false" aria-controls="map-panel-0">
    쇼핑 및 알아보기
  </button>
  <span class="icon" aria-hidden="true">+</span>
</h3>
<ul id="map-panel-0">
  ...
</ul>
```

```js
h3El.addEventListener('click', function () {
  const isActive = el.classList.toggle('active');
  const btn = h3El.querySelector('button');
  btn.setAttribute('aria-expanded', String(isActive));
});
```

`<button>`으로 바꾸면 키보드 포커스와 `Enter`/`Space` 클릭이 브라우저 기본 동작으로 딸려 오고, `aria-expanded`를 토글해주면 스크린 리더가 "접혀 있음 / 펼쳐져 있음" 상태를 그대로 읽어준다. 코드량은 몇 줄 늘지 않지만 체감되는 접근성 차이는 크다.

---

## 정리 표

| 항목 | 구현 방식 | 강점 | 아쉬운 점 |
| --- | --- | --- | --- |
| 데이터 렌더링 | `navigations.js` 배열 → `forEach` + 템플릿 리터럴 | 카테고리·링크 추가/삭제가 데이터 수정만으로 끝남 | - |
| 펼침/접힘 로직 | `classList.toggle('active')` | JS 코드가 5줄로 끝남 | 배타적 열림(하나만 열기) 미구현 |
| 펼침/접힘 표현 | `height: 0/auto` + `transform`/`opacity` 트랜지션 | `height: auto` 트랜지션 불가 문제를 우회 | 항목 수와 무관하게 항상 같은 속도로 열림 |
| 아이콘 상태 변화 | 텍스트 `+`를 `rotate(45deg)` | 별도 이미지/아이콘 세트 불필요 | - |
| 브레이크포인트 분리 | 아코디언 CSS 전체가 `@media (max-width: 740px)` 안에만 존재 | JS는 뷰포트를 몰라도 됨, 리사이즈 핸들러 불필요 | - |
| 접근성 | `<h3>` 클릭, `aria-*` 없음 | - | 키보드 포커스 불가, 스크린 리더 정보 없음 |

---

## TypeScript 프로젝트에서의 참고

React + TypeScript로 옮긴다면, 이 컴포넌트가 안고 있던 문제들을 상태 관리와 타입으로 자연스럽게 해소할 수 있다.

- **배타적 열림**: `activeIndex: number | null`을 부모에서 `useState`로 관리하고, 각 `.map`은 `isActive={activeIndex === index}` prop만 받아 렌더링하면 "다른 걸 열면 나머지가 닫힌다"는 로직이 클릭 핸들러 한 곳(`setActiveIndex(prev => prev === index ? null : index)`)으로 모인다. 지금처럼 `forEach`로 DOM을 순회하며 클래스를 직접 지우는 명령형 코드가 필요 없어진다.
- **접근성**: `aria-expanded`, `aria-controls`, `<button>` 시맨틱을 매번 손으로 챙기는 대신 `useAccordionItem(id: string)` 같은 커스텀 훅으로 감싸면, 훅이 `buttonProps`/`panelProps` 객체를 반환하고 컴포넌트는 스프레드만 하면 되도록 만들 수 있다. `aria-expanded: boolean`처럼 boolean 타입을 명시해두면 문자열 `'true'`/`'false'`를 실수로 넣는 것도 컴파일 타임에 걸러진다.
- **height 트랜지션**: CSS Grid의 `grid-template-rows: 0fr → 1fr` 트랜지션은 Framer Motion 같은 애니메이션 라이브러리 없이도 순수 CSS로 처리 가능하니, 굳이 JS로 `scrollHeight`를 측정해서 `style.height`에 픽셀 값을 넣는(리플로우를 유발하는) 방식은 피하는 게 좋다.
- **데이터 타입**: `navigations.js`의 배열 구조를 그대로 옮긴다면 `interface NavGroup { title: string; maps: { name: string; url: string }[] }` 정도로 타입을 선언해두면, 렌더링 컴포넌트 쪽에서 `nav.maps.map(...)`을 돌 때 `name`/`url` 오타를 자동완성과 타입 체크로 방지할 수 있다.

</div>
