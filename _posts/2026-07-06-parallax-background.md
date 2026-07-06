---
layout: post
title: CSS만으로 패럴랙스 배경 만들기
description: background-attachment fixed 파헤치기
tags: 학습 CSS
categories: posts
related_posts: false
toc: 
  sidebar: right

---

<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">

# CSS만으로 패럴랙스 배경 만들기: `background-attachment: fixed` 파헤치기

JavaScript 라이브러리 없이도 CSS 속성 하나로 그럴듯한 패럴랙스(parallax) 효과를 만들 수 있습니다. 이번 글에서는 실무에서 자주 쓰는 다음 패턴을 뜯어보고, 왜 동작하는지 그리고 언제 문제가 생기는지 정리해봅니다.

```css
.pick-your-favorite {
  background-image: url('../images/favorite_bg.jpg');
  background-repeat: no-repeat;
  background-position: center;
  background-attachment: fixed;
  background-size: cover;
}

.pick-your-favorite .inner {
  padding: 110px 0;
}
```

## 핵심은 `background-attachment: fixed`

`background-attachment` 속성은 배경 이미지가 스크롤에 반응하는 방식을 결정합니다.

| 값                | 동작                                                         |
| ----------------- | ------------------------------------------------------------ |
| `scroll` (기본값) | 배경 이미지가 요소와 함께 스크롤됨                           |
| `fixed`           | 배경 이미지가 **뷰포트 기준으로 고정**됨                     |
| `local`           | 요소 자체의 내부 스크롤에 반응 (내부에 스크롤바가 있는 경우) |

`fixed`로 설정하면 배경 이미지는 브라우저 화면에 못박힌 것처럼 고정되고, 그 위의 콘텐츠만 스크롤되며 지나갑니다. 이 "레이어가 분리되어 움직이는" 느낌이 바로 패럴랙스 효과의 정체입니다.

## 나머지 속성들의 역할

```css
background-repeat: no-repeat;   /* 이미지 타일링 방지 */
background-position: center;    /* 이미지를 요소 중앙에 배치 */
background-size: cover;         /* 비율 유지하며 요소를 완전히 채움 (넘치면 크롭) */
```

`cover`와 `fixed`를 함께 쓰면, 배경이 뷰포트를 기준으로 그려지기 때문에 화면 크기가 바뀌어도 이미지가 항상 화면을 꽉 채우면서 비율이 깨지지 않습니다.

## `.inner`의 `padding: 110px 0`은 왜 필요한가

배경이 `fixed`이기 때문에, 실제 이미지 크기와 무관하게 **이 요소의 높이만큼만** 배경이 보이는 "창문" 영역이 결정됩니다. 안에 콘텐츠가 있다면 그 위아래로 여백을 주는 동시에, 섹션 자체의 세로 공간을 확보하는 역할을 합니다. 만약 이 padding이 없고 내부 콘텐츠도 적다면, 섹션 높이가 작아져서 패럴랙스 효과가 거의 안 보이게 됩니다.

## 실무에서 반드시 알아야 할 함정 3가지

### 1. iOS Safari에서 제대로 동작하지 않는다

`background-attachment: fixed`는 오랫동안 모바일 Safari에서 정상적으로 지원되지 않았습니다. 스크롤 시 배경이 깨지거나, `scroll`처럼 그냥 같이 움직여버리는 경우가 많습니다. 실무에서는 보통 미디어 쿼리로 분기 처리합니다.

```css
.pick-your-favorite {
  background-attachment: fixed;
}

@media (max-width: 768px) {
  .pick-your-favorite {
    background-attachment: scroll;
  }
}
```

더 확실한 방법은 `position: fixed`인 별도의 `div`를 배경 전용 레이어로 분리하고, `z-index`로 콘텐츠보다 뒤에 배치하는 방식입니다. 이 경우 뷰포트 지원 이슈 없이 동일한 시각 효과를 낼 수 있습니다.

### 2. 스크롤 성능 이슈

`fixed` 배경은 스크롤할 때마다 브라우저가 다시 그려야 하는 영역이 늘어나서 리페인트(repaint) 비용이 큽니다. 저사양 기기에서는 스크롤이 버벅이는 원인이 될 수 있습니다. 최근에는 CSS만으로 처리하기보다, `Intersection Observer`로 요소가 뷰포트에 들어왔는지 감지하고 `transform: translateY()`를 이용해 GPU 가속 애니메이션으로 패럴랙스를 구현하는 방식이 더 널리 쓰입니다.

```javascript
const target = document.querySelector('.parallax-layer');

const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        window.addEventListener('scroll', updateParallax);
      } else {
        window.removeEventListener('scroll', updateParallax);
      }
    });
  },
  { threshold: 0 }
);

observer.observe(target);

function updateParallax() {
  const scrollY = window.scrollY;
  target.style.transform = `translateY(${scrollY * 0.3}px)`;
}
```

`transform`은 레이아웃 재계산 없이 GPU 합성 레이어에서 처리되기 때문에, `background-attachment: fixed`보다 스크롤 성능이 유리합니다.

### 3. 겹쳐진 다른 배경과의 쌓임 순서 확인

패럴랙스 섹션 위에 오버레이나 텍스트를 얹는 경우가 많은데, 이때 부모에 `position: relative`가 없거나 자식 요소에 `z-index`가 없으면 겹침 순서가 꼬일 수 있습니다. `position`이 걸린 요소끼리는 stacking context 규칙을 따른다는 걸 기억해두면 디버깅이 빨라집니다.

## 정리

| 방식                                     | 장점                            | 단점                              |
| ---------------------------------------- | ------------------------------- | --------------------------------- |
| `background-attachment: fixed`           | 코드 3줄로 구현 가능, JS 불필요 | iOS Safari 이슈, 스크롤 성능 저하 |
| `position: fixed` 배경 레이어            | 모바일 호환성 좋음              | 별도 마크업/CSS 구조 필요         |
| JS + `transform` (Intersection Observer) | 성능 우수, 세밀한 제어 가능     | 구현 복잡도 증가                  |

간단한 히어로 섹션 정도라면 `background-attachment: fixed`로 충분하지만, 프로덕션 환경에서 모바일 트래픽 비중이 크다면 처음부터 `transform` 기반 JS 패럴랙스를 고려하는 게 유지보수 측면에서 더 안전합니다.

</div>
