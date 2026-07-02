---
layout: post
title: 커밋 메시지 컨벤션 가이드
description: 용도별로 정리한 커밋 메시지 타입과 실전 팁
tags: 학습 컨벤션
categories: posts
related_posts: false
toc: 
  sidebar: right
---

<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">


협업 프로젝트나 개인 프로젝트 모두에서 일관된 커밋 메시지는 히스토리를 읽기 쉽게 만들고, 자동화된 changelog 생성이나 semantic versioning에도 도움이 됩니다.

이번 글에서는 가장 널리 쓰이는 **Conventional Commits** 규칙을 기준으로 용도별 커밋 메시지를 정리해봤습니다.

<br>

---

<br>

### 기본 형식

```
<type>(<scope>): <subject>

<body>

<footer>
```

<br>

| 요소      | 설명                                  |
| --------- | ------------------------------------- |
| `type`    | 커밋의 성격 (아래 표 참고)            |
| `scope`   | 변경 범위 (선택, 괄호로 표기)         |
| `subject` | 간결한 요약                           |
| `body`    | 무엇을/왜 변경했는지 상세 설명 (선택) |
| `footer`  | Breaking Change, 이슈 번호 등 (선택)  |

<br>

---

<br>

### 상세 예시 (전체 커밋 메시지)

실제 커밋 메시지로 전체 예시를 만들어봤습니다.

<br>

**예시 1 — 기능 추가 (Breaking Change 포함)**

```
feat(auth): 소셜 로그인 기능 추가

기존 이메일/비밀번호 로그인 방식에 더해 Google, Kakao OAuth 로그인을
지원하도록 구현했습니다. 사용자는 이제 소셜 계정으로 회원가입 없이
바로 서비스를 이용할 수 있습니다.

- Passport.js 기반 OAuth2 전략(Google, Kakao) 추가
- 기존 유저 테이블에 provider, providerId 컬럼 추가
- 로그인 성공 시 JWT 발급 로직을 소셜 로그인에도 동일 적용

BREAKING CHANGE: User 스키마의 email 필드가 nullable로 변경됨
(소셜 로그인 유저는 이메일 미제공 가능)

Closes #142
```

<br>

**예시 2 — 버그 수정**

```
fix(player): 오디오 시크바 드래그 중 위치 튐 현상 수정

시크바를 드래그하는 동안 currentTime이 매 프레임마다 재계산되면서
드래그 위치와 실제 재생 위치가 어긋나는 문제가 있었습니다.

드래그 중에는 오디오 엘리먼트의 timeupdate 이벤트 리스너를 일시
해제하고, 드래그가 끝난 시점에만 seek을 적용하도록 수정했습니다.

Fixes #98
```

<br>

**예시 3 — 리팩토링 (body만, footer 없음)**

```
refactor(store): Zustand 스토어를 도메인별로 분리

기존에 하나의 store.ts 파일에 재생 상태, 유저 상태, UI 상태가
전부 섞여 있어 유지보수가 어려웠습니다.

playerStore, userStore, uiStore로 분리하고, 각 스토어는
자신의 상태와 액션만 관리하도록 구조를 변경했습니다. 기능 변화는
없으며 상태 접근 방식(useStore 훅 사용법)도 기존과 동일합니다.
```

<br>

**예시 4 — 간단한 수정 (subject만, body/footer 생략)**

```
chore: .env.example 파일 추가
```

<br>

**요소별로 뜯어보면 (예시 1 기준)**

| 요소      | 내용                                                         |
| --------- | ------------------------------------------------------------ |
| `type`    | `feat`                                                       |
| `scope`   | `auth` (괄호 안, 영향 범위를 짧게)                           |
| `subject` | `소셜 로그인 기능 추가` (50자 이내 권장, 마침표 없음)        |
| `body`    | 빈 줄 하나 띄운 뒤, "무엇을 왜" 서술형 + 필요하면 불릿으로 세부 변경사항 나열 |
| `footer`  | Breaking Change 명시, 이슈/PR 번호 연결 (`Closes #142`, `Fixes #98`) |

<br>

`body`와 `footer`는 선택 사항이라, 예시 4처럼 간단한 변경은 `subject` 한 줄로 끝내도 됩니다. 반대로 예시 1처럼 스키마 변경이나 API 변화가 있는 커밋은 `body`로 맥락을 남기고 `footer`에 `BREAKING CHANGE`를 꼭 명시하는 게 나중에 히스토리 추적할 때 훨씬 도움이 됩니다.

<br>

---

<br>

### 타입 한눈에 보기

| 타입       | 용도                                     |
| ---------- | ---------------------------------------- |
| `feat`     | 새로운 기능 추가                         |
| `fix`      | 버그 수정                                |
| `refactor` | 기능 변화 없이 코드 구조 개선            |
| `style`    | 포맷팅 등 로직에 영향 없는 변경          |
| `docs`     | 문서 관련 변경                           |
| `test`     | 테스트 코드 추가/수정                    |
| `chore`    | 빌드, 패키지 매니저, 설정 등 잡무성 변경 |
| `ci`       | CI/CD 설정 변경                          |
| `build`    | 빌드 시스템, 외부 의존성 관련            |
| `perf`     | 성능 개선                                |
| `revert`   | 이전 커밋 되돌리기                       |

<br>

---

<br>

### 타입별 상세 예시

<br>

#### feat — 새로운 기능 추가

사용자에게 보이는 새로운 기능이나 화면을 추가했을 때 사용합니다.

```
feat: 사용자 로그인 기능 추가
feat(auth): JWT 토큰 갱신 로직 구현
feat(player): 오디오 시크바 드래그 기능 추가
```

<br>

#### fix — 버그 수정

의도하지 않은 동작이나 오류를 고쳤을 때 사용합니다.

```
fix: MSW가 Range 요청을 가로채는 문제 해결
fix(api): Prisma 쿼리에서 N+1 문제 수정
fix: ARM64/X86_64 아키텍처 불일치로 인한 배포 실패 해결
```

<br>

#### refactor — 기능 변화 없이 코드 구조 개선

동작은 그대로 두고 코드 가독성이나 구조만 개선했을 때 사용합니다.

```
refactor: GraphQL resolver 로직 분리
refactor(store): Zustand 스토어 구조 정리
```

<br>

#### style — 포맷팅 등 로직에 영향 없는 변경

코드 로직에는 영향 없는 공백, 세미콜론, 들여쓰기 등을 수정했을 때 사용합니다.

```
style: ESLint 규칙에 따라 들여쓰기 수정
style: 불필요한 공백 제거
```

<br>

#### docs — 문서 관련 변경

README, 주석, API 문서 등 문서만 수정했을 때 사용합니다.

```
docs: README에 배포 가이드 추가
docs(api): GraphQL 스키마 주석 보완
```

<br>

#### test — 테스트 코드 추가/수정

테스트 코드를 추가하거나 수정했을 때 사용합니다.

```
test: MSW 핸들러 테스트 추가
test(auth): 로그인 실패 케이스 테스트 작성
```

<br>

#### chore — 빌드, 패키지 매니저, 설정 등 잡무성 변경

소스 코드 로직과 무관한 설정 파일, 패키지 관리 등을 변경했을 때 사용합니다.

```
chore: pnpm workspace 설정 추가
chore(deps): Prisma 버전 업데이트
chore: .env.example 파일 추가
```

<br>

##### ci — CI/CD 설정 변경

GitHub Actions, Jenkins 등 CI/CD 파이프라인 설정을 변경했을 때 사용합니다.

```
ci: GitHub Actions 워크플로우에 ECR 푸시 단계 추가
ci(fargate): 배포 전 헬스체크 스텝 추가
ci: Docker 빌드 캐시 최적화
```

<br>

#### build — 빌드 시스템, 외부 의존성 관련

빌드 도구나 외부 의존성 설정을 변경했을 때 사용합니다.

```
build: esbuild 설정에 alias 추가
build(docker): 멀티 스테이지 빌드로 이미지 크기 최적화
```

<br>

#### perf — 성능 개선

속도, 메모리 등 성능을 개선했을 때 사용합니다.

```
perf: 이미지 lazy loading 적용
perf(db): 인덱스 추가로 조회 속도 개선
```

<br>

#### revert — 이전 커밋 되돌리기

이전 커밋을 취소하고 되돌릴 때 사용합니다.

```
revert: "feat: 소셜 로그인 추가" 되돌림
```

<br>

---

<br>

### 실전 팁

**1. scope는 괄호로 표기**

`feat(auth):`, `fix(player):` 처럼요. 모노레포 구조라면 특히 유용합니다. 예를 들어 `feat(server):`, `feat(client):`처럼 패키지 단위로 구분할 수 있습니다.

<br>

**2. body는 "무엇을/왜" 중심으로**

제목 아래 빈 줄을 하나 두고 작성합니다. "어떻게"는 코드 자체가 설명해주기 때문에 body에 담을 필요는 없습니다.

<br>

**3. Breaking Change 표시**

타입 뒤에 `!`를 붙이거나 본문에 `BREAKING CHANGE:`를 명시합니다.

```
feat!: API 응답 구조 변경
```

<br>

**4. 컨벤션 강제하기**

`commitlint` + `husky` 조합을 추천합니다. pre-commit 훅에서 커밋 메시지 형식을 검사해 규칙에 어긋나면 커밋 자체를 막아줍니다.

<br>

---

<br>

### 마무리

처음에는 번거롭게 느껴질 수 있지만, 몇 번 습관을 들이면 커밋 히스토리만 봐도 프로젝트의 변화 흐름이 한눈에 들어옵니다.

특히 CI/CD 파이프라인이나 릴리즈 자동화를 구축할 때는 `type`을 기준으로 changelog를 자동 생성할 수도 있어서 데브옵스 워크플로우와도 잘 맞물립니다.

</div>
