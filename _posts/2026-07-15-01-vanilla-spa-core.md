---
layout: post
title: omdb-movie-app에서 생각해봐야할 것들 (1) — 프레임워크 없이 SPA 만들기 편
date: 2026-07-15
tags: [TypeScript, SPA, State Management, Router, Frontend]
inline: false
related_posts: false
toc:
  sidebar: right
---

<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">

React를 몇 년째 쓰면서도 정작 `useState`를 호출하면 왜 화면이 다시 그려지는지, 그 안에서 무슨 일이 벌어지는지를 설명하지 못한다는 걸 깨달은 적이 있다. 프레임워크가 대신 풀어주던 문제를 한 번도 직접 풀어본 적이 없었기 때문이다. 그래서 2일짜리 사이드 프로젝트(omdb-movie-app)에서는 React도, Vue도, 어떤 상태관리 라이브러리도 쓰지 않고 `Component`, `Store`, `Router` 세 가지를 직접 만들었다. 짧은 코드지만 그 안에 "반응형 상태란 무엇인가", "구독 기반 렌더링", "배열 반응성의 한계", "URL과 화면을 동기화하는 법"이 전부 들어있다.

## 1. 왜 프레임워크 없이 만들어보기로 했나

이 프로젝트를 시작할 때 목표는 "빠르게 완성"이 아니라 "블랙박스를 열어보기"였다. React의 `useState`, Vue의 `ref`/`reactive`는 전부 "상태가 바뀌면 관련된 화면만 다시 그린다"는 동일한 문제를 풀고 있는데, 그 내부 구현을 한 번도 직접 만들어보지 않고서는 왜 특정 패턴(예: 배열을 직접 `push`하면 안 되고 새 배열로 교체해야 한다는 규칙)이 존재하는지 감으로만 알고 있을 뿐이었다. 그래서 이번엔 라이브러리 없이 `Component`(렌더 생명주기), `Store`(반응형 상태), `Router`(URL 동기화) 세 가지 코어를 직접 설계했다.

## 2. 반응형 상태란 무엇인가 — `Object.defineProperty`로 직접 구현

"반응형 상태"란 결국 "값이 바뀌는 순간을 코드가 알아챌 수 있는 상태"다. 일반 객체의 프로퍼티는 값이 바뀌어도 아무도 알림을 받지 못한다. 이걸 감지하려면 값을 읽고 쓰는 지점에 코드를 끼워 넣어야 하는데, `Object.defineProperty`가 바로 그 지점을 만들어준다.

```ts
// src/core/core.ts (핵심만 정리)
type Observer = () => void;

export class Store<S extends Record<string, unknown>> {
  state: S;
  private observers: Partial<Record<keyof S, Observer[]>> = {};

  constructor(initialState: S) {
    this.state = {} as S;

    (Object.keys(initialState) as (keyof S)[]).forEach((key) => {
      let value = initialState[key];

      Object.defineProperty(this.state, key, {
        get: () => value,
        set: (newValue) => {
          value = newValue;
          this.notify(key);
        },
      });
    });
  }

  subscribe(key: keyof S, callback: Observer) {
    if (!this.observers[key]) this.observers[key] = [];
    this.observers[key]!.push(callback);
  }

  private notify(key: keyof S) {
    this.observers[key]?.forEach((cb) => cb());
  }
}
```

원리는 단순하다. `initialState`의 각 key를 순회하며 진짜 값은 클로저 변수(`value`)에 숨겨두고, `state.movies`처럼 그 key에 접근할 때는 `get`이, `state.movies = [...]`처럼 값을 대입할 때는 `set`이 호출되도록 만든다. `set` 안에서 `notify`를 호출해 그 key를 구독하고 있는 콜백들을 전부 실행하면, "값이 바뀌면 관련된 곳에 알림이 간다"는 반응형의 최소 조건이 완성된다.

## 3. 구독(subscribe) 패턴으로 필요한 컴포넌트만 다시 그리기

상태가 바뀔 때마다 페이지 전체를 다시 그리는 건 간단하지만 비효율적이다. 그래서 컴포넌트는 자신이 실제로 관심 있는 key만 구독하도록 만들었다.

```ts
// src/components/MovieList.ts
export class MovieList extends Component {
  constructor() {
    super();
    movieStore.subscribe('movies', () => this.render());
    movieStore.subscribe('isLoading', () => this.render());
  }
}
```

`Component` 자체는 아주 단순한 생명주기만 가진다.

```ts
// src/core/core.ts
export abstract class Component<P = unknown> {
  el: HTMLElement;

  constructor(protected props?: P) {
    this.el = document.createElement('div');
    this.render();
  }

  abstract template(): string;

  render() {
    this.el.innerHTML = this.template();
    this.setEvent?.();
  }

  setEvent?(): void;
}
```

`movieStore.state.movies = newMovies`가 실행되면 `Store`의 setter가 `notify('movies')`를 호출하고, 그걸 구독하고 있던 `MovieList.render()`만 다시 실행된다. `Chatbot`이나 `TheHeader`처럼 `movies` key와 무관한 컴포넌트는 전혀 영향을 받지 않는다. 이게 "전체 리렌더 대신 필요한 컴포넌트만 다시 그린다"는 것의 실체다 — React의 재조정(reconciliation)과는 결이 다르지만, "관심 있는 상태만 구독한다"는 목적은 동일하다.

## 4. 한계 체감: 배열 반응성이 안 되는 이유

이 구조를 만들고 나서 바로 부딪힌 문제가 있다.

```ts
movieStore.state.movies.push(newMovie); // 아무 일도 일어나지 않는다
```

`Object.defineProperty`는 `state.movies`라는 **프로퍼티 자체**에 대한 get/set만 감시한다. `movies`가 가리키는 배열 안에서 `push`가 실행되는 건 "프로퍼티 재할당"이 아니라 "배열 인스턴스 내부 메서드 호출"이기 때문에 setter를 전혀 거치지 않는다. 그래서 이 프로젝트에서는 항상 새 배열로 통째로 교체하는 방식을 강제했다.

```ts
movieStore.state.movies = [...movieStore.state.movies, newMovie]; // 이렇게 "재할당"해야 감지된다
```

이 한계를 직접 겪어보고 나서야 Vue2가 왜 `push`, `pop`, `splice` 같은 배열 변경 감지를 위해 배열 메서드 자체를 오버라이드하는 별도의 트릭을 써야 했는지, 그리고 Vue3와 대부분의 최신 상태관리 라이브러리가 왜 `Proxy`로 옮겨갔는지가 코드 레벨에서 이해됐다. `Proxy`는 프로퍼티 단위가 아니라 객체 전체에 대한 `get`/`set`/`deleteProperty` 트랩을 걸 수 있어서, 배열 인덱스 변경이나 메서드 호출까지 더 근본적으로 감지할 수 있다.

## 5. 라우팅도 결국 "URL 상태"를 동기화하는 문제다

`react-router` 없이 URL과 화면을 맞추려면 최소한 두 가지만 있으면 된다. "URL이 바뀌었다는 신호"와 "그 URL에 맞는 화면을 그리는 함수"다.

```ts
// src/core/core.ts
export function createRouter(routes: Route[]) {
  const routeRender = () => {
    const hash = location.hash.slice(1) || '/';
    const matched = routes.find((route) => new RegExp(`${route.path}/?$`).test(hash));
    (matched?.component ?? NotFound).render();
  };

  window.addEventListener('popstate', routeRender);
  window.addEventListener('hashchange', routeRender);
  routeRender();
}
```

`location.hash` + `hashchange`/`popstate` 이벤트만으로 라우팅이 동작한다. 다만 라우트 매칭을 정규식으로 처리하다 보니 `/movie/:id` 같은 동적 세그먼트는 지원하지 않는다. 그래서 영화 상세 페이지로 이동할 때는 URL 파라미터 대신 `history.state`에 값을 담아 우회했다.

```ts
// src/routes/Movie.ts (진입 시)
history.replaceState({ id: imdbID }, '');
location.hash = '#/movie';

// src/routes/Movie.ts (렌더링 시)
const { id } = history.state;
```

URL 쿼리스트링(`#/movie?id=...`)으로 직접 처리하는 방법도 있었지만, `history.state`를 쓰면 파싱 코드 없이 객체를 그대로 주고받을 수 있어 더 단순했다. 대신 링크를 그대로 복사해서 새 창에서 열면 `history.state`가 없으니 상세 정보가 비는 한계는 그대로 남는다 — 이 프로젝트에서는 SNS 공유나 직접 링크 접근 요구사항이 없어서 문제가 되지 않았을 뿐이다.

## 6. Proxy로 바꾼다면?

지금 구조의 다음 개선 방향은 명확하다. `Object.defineProperty` 기반 `Store`를 `Proxy` 기반으로 바꾸면, 배열 `push`나 새로운 key 추가까지 감지할 수 있다.

```ts
function reactive<S extends object>(target: S, onChange: (key: keyof S) => void): S {
  return new Proxy(target, {
    set(obj, key, value) {
      obj[key as keyof S] = value;
      onChange(key as keyof S);
      return true;
    },
  });
}
```

`Proxy`는 객체 전체를 감싸기 때문에 `Object.defineProperty`처럼 생성 시점에 모든 key를 미리 알아야 하는 제약도 없다. Vue3의 `reactive()`, Solid.js의 signal, Zustand 같은 최신 상태관리 도구들이 공통적으로 이 방향(값 자체를 프록시하거나, 값 변경을 구독 가능한 원자 단위로 쪼개는 것)으로 수렴한 이유를 이번에 직접 만들어보고서야 체감했다.

---

## 정리 표

| 항목 | 구현 방식 | 강점 | 아쉬운 점 |
| --- | --- | --- | --- |
| 반응형 상태 | `Object.defineProperty` getter/setter + pub-sub | 외부 라이브러리 없이 구독 기반 부분 렌더링 구현 | 배열 `push`/`splice` 등 메서드 변경은 감지 못 함 |
| 구독/렌더링 | `store.subscribe(key, callback)` | 컴포넌트가 관심 있는 key만 구독해 불필요한 렌더 방지 | 구독 해제(unsubscribe) 로직이 없어 메모리 누수 여지 |
| 컴포넌트 생명주기 | 생성자에서 `render()` 호출하는 단순 구조 | 코드량이 적고 이해하기 쉬움 | 마운트/언마운트 훅이 없어 정리(cleanup) 시점을 못 잡음 |
| 라우팅 | `location.hash` + `popstate`/`hashchange`, 정규식 매칭 | react-router 없이 SPA 라우팅 구현 | 동적 세그먼트(`:id`) 미지원, `history.state`로 우회 |

## 실무 프로젝트에 적용한다면

이 정도 규모를 넘어서는 프로젝트라면 직접 만든 `Store`보다는 `Proxy` 기반 반응성을 갖춘 도구(Vue3의 `reactive`, Zustand, Jotai, Solid.js의 signal)를 쓰는 게 합리적이다. 다만 이번처럼 한 번 직접 만들어보면, 그 도구들의 API 뒤에서 실제로 무슨 일이 벌어지는지 — "구독", "알림", "변경 감지의 단위"가 왜 그런 형태로 설계됐는지 — 를 문서가 아니라 코드로 이해하게 된다. 도구를 잘 쓰기 위해 도구 없이 한 번 만들어보는 경험의 값어치는 여기에 있었다.

</div>
