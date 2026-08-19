---
layout: post
title: 테스트 피라미드, 실무에서는 이렇게 굴러간다 — 비율·도구·CI 배치까지
date: 2026-08-19
tags: [Testing, Vitest, Playwright, Testcontainers, MSW, CI/CD]
inline: false
related_posts: false
toc:
  sidebar: right
---

<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">

"테스트를 어떻게 짜야 하나"라는 질문도 "상태관리 뭐 쓸까"와 비슷한 함정이 있다. 유닛/통합/E2E를 뭉뚱그려서
"테스트"라고 부르면, 결국 셋 다 어중간하게 짜거나 하나에 전부 몰아넣게 된다. 실무에서는 이 세 레벨을 역할이
다른 도구로 명확히 분리하고, 그 비율과 CI 배치 방식까지 팀 컨벤션으로 정해두는 경우가 많다. 이 글은 그 비율,
레벨별로 실무에서 실제로 쓰는 도구, 그리고 왜 그 도구를 쓰는지를 정리한다.

## 1. "테스트 몇 개 짜야 하나"라는 질문이 틀린 이유

레벨별로 검증 대상과 속도가 완전히 다른데, 이걸 구분하지 않으면 "E2E 하나로 다 검증하면 되지 않나"라는
생각에 빠지기 쉽다. 문제는 E2E가 느리고(브라우저 부팅, 네트워크 왕복 포함) 깨지기 쉬워서(flaky), 이걸로
모든 분기를 커버하려 하면 테스트 스위트 하나 돌리는 데 몇 시간이 걸리고 원인 불명 실패가 계속 쌓인다.

레벨을 나누는 기준은 "이 테스트가 무엇을 진짜라고 가정하는가"다.

1. 유닛(Unit) — 함수/모듈 하나만 진짜, 나머지는 전부 mock
2. 통합(Integration) — DB/HTTP 레이어는 진짜, 브라우저는 없음
3. E2E(End-to-End) — 브라우저 포함 전체 스택이 진짜

이 구분을 무시하고 셋을 섞으면, 유닛 테스트인데 실제 DB에 연결해서 느려지거나, E2E인데 내부 API를 mock해서
"진짜로 붙는지"를 검증 못 하는 애매한 테스트가 나온다.

## 2. 실무에서 흔히 쓰는 비율 — 70/20/10 피라미드

| 레벨 | 비율 | 실행 속도 | 실행 시점 |
| --- | --- | --- | --- |
| 유닛 | ~70% | 밀리초~초 단위, 수천 개도 몇 초 | 매 커밋/저장 시 (watch 모드) |
| 통합 | ~20% | 초~분 단위 | PR마다 CI에서 |
| E2E | ~10% | 분 단위 | merge 직전 또는 야간 스케줄 |

이 비율이 절대 법칙은 아니고, 팀이나 도메인에 따라 통합 테스트 비중을 더 크게 가져가는 "테스트 다이아몬드"
형태(마이크로서비스 간 계약이 중요한 조직)도 흔하다. 다만 공통된 원칙은 "위로 갈수록 개수는 줄고, 신뢰도는
더 현실적이 되고, 비용(시간·유지보수)은 늘어난다"는 것이다.

## 3. 레벨별로 실무에서 실제로 쓰는 도구와 이유

### 3.1 백엔드 유닛 테스트 — 서비스 레이어를 제일 두껍게

비즈니스 로직(예: "이름이 중복되면 409를 던진다", "보호 그룹은 삭제 불가")은 DB 없이도 검증 가능하므로,
이 레이어에 테스트를 가장 많이 몰아준다. DB 접근 레이어(Prisma/TypeORM 등)는 mock 처리한다.

```ts
vi.mock('@/lib/prisma/hive.js', () => ({ hivePrisma: mockPrisma }));

it('보호 그룹은 409를 던진다', async () => {
  mockPrisma.group.findUnique.mockResolvedValue({ id: 'g1', name: 'admin' });
  await expect(groupsService.remove('g1')).rejects.toMatchObject({ status: 409 });
});
```

이유는 명확하다. 분기 하나하나(404, 409, 정상 케이스)를 DB 없이 밀리초 단위로 검증할 수 있고, 개발자가
저장할 때마다(watch 모드) 즉시 피드백을 받을 수 있다.

### 3.2 백엔드 통합 테스트 — 진짜 DB, 다만 격리된 진짜 DB

통합 테스트는 "DB 쿼리가 실제로 맞게 나가는가", "미들웨어 체인(인증→권한→검증)이 실제로 연결돼 동작하는가"를
검증한다. 여기서 실무가 두 갈래로 나뉜다.

**방식 A — 공유 테스트 DB**: `hive_test`처럼 고정된 DB를 하나 두고, 매 테스트 전후로 `deleteMany`로 정리.
설정이 간단하지만, 병렬로 테스트를 돌리면(여러 CI job이 같은 DB를 공유) 데이터 충돌이 날 수 있다.

**방식 B — Testcontainers**: 테스트 실행 시점에 도커로 MySQL/Postgres 컨테이너를 매번 새로 띄우고, 끝나면
버린다.

```ts
import { MySqlContainer } from '@testcontainers/mysql';

let container: StartedMySqlContainer;

beforeAll(async () => {
  container = await new MySqlContainer('mysql:8').start();
  process.env.HIVE_DB_HOST = container.getHost();
  process.env.HIVE_DB_PORT = String(container.getPort());
}, 60_000); // 컨테이너 부팅 시간 고려해서 타임아웃 넉넉히

afterAll(async () => {
  await container.stop();
});
```

컨테이너마다 완전히 새 DB라서 격리가 완벽하고, CI에서 여러 job이 병렬로 돌아도 서로 안 건드린다. 단점은
컨테이너 부팅 시간(수 초~수십 초)이 매 실행마다 붙는다는 것과, CI 환경에 도커 실행 권한이 있어야 한다는 것.
스타트업/소규모 팀은 방식 A로 시작했다가, CI 병렬화가 필요해지는 시점에 방식 B로 옮기는 경우가 많다.

컨트롤러/라우트 레벨은 supertest로 실제 HTTP 요청을 흉내 낸다.

```ts
const res = await request(app).post('/groups').send({ name: 'dup-group' });
expect(res.status).toBe(409);
```

여기서 검증하는 건 "서비스 로직이 맞는가"가 아니라(그건 이미 유닛에서 검증됨) "라우팅 → 인증 미들웨어 →
zod 검증 → 컨트롤러 → 서비스 → 응답 직렬화"라는 배관이 실제로 이어져 있는가다.

### 3.3 프론트 컴포넌트 테스트 — 화면에 보이는 것만 검증한다

React Testing Library의 핵심 철학은 "구현 디테일을 테스트하지 마라"다. `useState` 변수 이름이나 내부 함수
호출 여부를 검증하는 게 아니라, 사용자가 실제로 보고 클릭하는 것만 검증한다.

```tsx
// 나쁜 예: 구현 디테일 테스트
expect(component.state('isModalOpen')).toBe(true);

// 좋은 예: 사용자 관점 테스트
render(<GroupHeaderContainer />);
await userEvent.click(screen.getByRole('button', { name: '삭제' }));
expect(screen.getByText('정말로 삭제하시겠습니까')).toBeInTheDocument();
```

이렇게 짜면 내부 구현을 리팩터링해도(예: `useState`를 Zustand로 바꿔도) 테스트는 안 깨진다 — 사용자가 보는
화면과 동작이 그대로면 테스트도 그대로 통과해야 한다는 게 원칙이기 때문이다.

API 호출은 함수 mock(`vi.fn()`으로 fetch 자체를 가짜 함수로 바꿔치기) 대신 **네트워크 레벨**에서 가로채는
MSW(Mock Service Worker)가 요즘 표준이다.

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.post('/api/groups', () => HttpResponse.json({ id: 'g1', name: 'dup-group' }, { status: 409 })),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

함수 mock은 "우리 코드가 fetch를 어떻게 호출하는지"에 종속되지만(구현이 axios에서 fetch로 바뀌면 mock도
다시 짜야 함), MSW는 실제 URL/메서드/응답 형태를 그대로 다루기 때문에 구현이 바뀌어도(axios ↔ fetch) 테스트
코드는 그대로 유지된다 — 리팩터링에 훨씬 강하다.

### 3.4 E2E — 정말 핵심 시나리오만

E2E는 "이게 깨지면 서비스가 망가진다" 수준의 시나리오에만 쓴다. 로그인, 회원가입, 결제, 그리고 이 프로젝트
기준으로는 "관리자가 그룹 권한을 잘못 건드려서 전체 시스템 접근이 막히는" 케이스 정도가 여기 해당한다.

```ts
test('admin 그룹은 삭제 시도 시 에러 스낵바가 뜬다', async ({ page }) => {
  await page.goto('/system/groups');
  await page.locator('.ag-row', { hasText: 'admin' }).locator('input[type="checkbox"]').check();
  await page.getByRole('button', { name: '삭제' }).click();
  await expect(page.getByText('기본 그룹은 삭제할 수 없습니다')).toBeVisible();
});
```

E2E를 많이 만들지 않는 이유는 셋이다. 느리다(브라우저 부팅+렌더링+네트워크), 깨지기 쉽다(타이밍 이슈, 셀렉터가
CSS 클래스명 변경에 취약), 그리고 실패했을 때 원인 파악이 오래 걸린다(스택 전체가 얽혀 있어서 어느 레이어
문제인지 바로 안 보임). "테스트 유지보수 비용이 테스트가 주는 신뢰보다 커지는 지점"이 바로 E2E를 과하게
늘렸을 때 제일 먼저 온다.

## 4. CI 파이프라인에 배치하는 법

| 트리거 | 돌리는 테스트 | 이유 |
| --- | --- | --- |
| 로컬 저장 시 (watch) | 유닛 | 즉각적인 피드백 |
| PR 생성/업데이트 시 | 유닛 + 통합 | 빠르게(수 분 내) 리뷰어에게 신호를 줘야 함 |
| merge 직전 또는 야간 스케줄 | E2E | 무거워서 매 PR마다 돌리면 병목이 됨 |

PR마다 E2E까지 돌리는 팀도 있지만, 이 경우 보통 "핵심 시나리오 5~10개만" 골라서 돌리지 전체 E2E 스위트를
매번 돌리진 않는다. 야간 배치로 전체 E2E(회귀 포함)를 돌리고 실패하면 다음 날 아침에 확인하는 방식이 흔하다.

## 5. 커버리지 숫자보다 중요한 것

"커버리지 80% 달성"을 목표로 잡으면, 의미 없는 라인(단순 getter, 로깅 코드)까지 억지로 커버해서 숫자만
채우는 테스트가 늘어난다. 실무에서 더 흔한 접근은 두 가지다.

**핵심 로직 우선순위화**: 결제, 권한 판정, 데이터 정합성처럼 "틀리면 사고로 이어지는" 로직은 커버리지
수치와 무관하게 무조건 테스트한다. 반대로 단순 CRUD의 getter성 코드는 통합 테스트 한 번으로 간접 커버되면
충분하다고 보고 유닛 테스트를 따로 안 만드는 경우도 많다.

**버그 하나 = 회귀 테스트 하나**: 프로덕션에서 버그가 발견되면, 고치기 전에 그 버그를 재현하는 테스트를
먼저 작성한다(레드 → 그린). 이렇게 쌓인 테스트 스위트는 "실제로 터졌던 문제들의 목록"이라, 커버리지 %보다
실질적인 안전망이 된다.

## 6. 결론 — 세 레벨은 대체 관계가 아니라 역할 분담

유닛 테스트가 많다고 통합/E2E가 필요 없어지는 게 아니고, E2E 하나로 유닛/통합을 대신할 수도 없다. 유닛은
"이 함수가 맞게 짜였는가", 통합은 "이 레이어들이 실제로 연결됐는가", E2E는 "사용자가 실제로 이 흐름을 완주할
수 있는가" — 셋은 서로 다른 질문에 답한다. 그래서 "테스트 뭐부터 짜야 하나"의 답도 상태관리와 마찬가지로
"먼저 검증 대상의 성격을 구분한다"는 원칙에서 출발한다.

---

## 정리 표

| 레벨 | 비율 목표 | 진짜인 것 | mock하는 것 | 주 도구 | 실행 시점 |
| --- | --- | --- | --- | --- | --- |
| 유닛 | ~70% | 함수/모듈 하나 | DB, 외부 API 전부 | Vitest/Jest | 저장할 때마다 |
| 통합 | ~20% | DB, HTTP 레이어 | 브라우저 | Vitest+supertest+Testcontainers | PR마다 |
| E2E | ~10% | 전체 스택(브라우저 포함) | 없음(실제 환경) | Playwright/Cypress | merge 직전/야간 |

## 다음 단계로 갈 수 있는 방향

지금 정리는 "레벨을 나누고 비율을 잡는다"는 원칙 수준이고, 실제로 이 프로젝트(hive-react-backend/frontend)의
그룹 기능에 세 레벨을 전부 적용해보면 더 구체적인 트레이드오프가 드러날 것 같다. Testcontainers 도입 전후로
CI 실행 시간이 실제로 얼마나 벌어지는지, MSW로 짠 컴포넌트 테스트가 axios→fetch 전환 같은 리팩터링에 실제로
얼마나 안 깨지는지, 그리고 "버그 하나 = 회귀 테스트 하나" 습관을 몇 주 유지했을 때 회귀 버그 재발률이
실제로 줄어드는지를 다음 글에서 직접 측정해볼 계획이다.


</div>
