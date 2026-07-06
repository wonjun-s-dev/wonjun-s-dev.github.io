---
layout: page
title: starbucks-homepage
description: 스타벅스 코리아 공식 홈페이지 UI/인터랙션을 재현한 정적 프론트엔드 클론 프로젝트
img: assets/img/sc-01.png
importance: 4
category: study
---
<div style="word-break: keep-all; overflow-wrap: break-word;" markdown="1">

스타벅스 코리아 공식 홈페이지의 UI와 인터랙션을 그대로 재현한 정적 프론트엔드 클론 프로젝트입니다.

**배포**: [바로가기](https://euphonious-manatee-8cb890.netlify.app/) <br>
**레포지토리**: [바로가기](https://github.com/wonjun-s-dev/starbucks-homepage) <br>
**Stack**: HTML5, CSS3, Vanilla JavaScript, GSAP, Swiper, ScrollMagic <br>
**기간 / 인원**: 2026.07.02 ~ 2026.07.06 · 1인 개발 <br>

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/sc-01.png" title="메인 화면" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/sc-03.png" title="로그인 화면" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    좌측은 메인 페이지 비주얼 섹션, 우측은 별도 경로(/signin)로 구현한 로그인 페이지입니다.
</div>

## 프로젝트 소개

프론트엔드 퍼블리싱 실력과 스크롤/애니메이션 라이브러리 활용 능력을 기르기 위해 진행한 클론 프로젝트입니다. (패스트캠퍼스 강의 실습)

- 메인 페이지의 12개 이상 섹션(비주얼, 공지, 리워드, 유튜브, 시즌 상품, 리저브 커피, Find the store, Awards 등) 마크업/스타일 재현
- GSAP 기반 스크롤 인터랙션(헤더 뱃지 페이드, Top 버튼, 플로팅 오브젝트, 스크롤 스파이)
- Swiper를 활용한 3종 캐러셀(공지사항 세로 슬라이드, 프로모션, 어워드)
- 로그인 페이지 UI 별도 구현

**왜 만들었는가**

실제 서비스 화면을 그대로 뜯어보며 레이아웃 구현, CSS 애니메이션, 서드파티 라이브러리(GSAP/Swiper/ScrollMagic) 연동 경험을 쌓기 위해 제작했습니다.

**성과 / 결과**

학습 목적의 클론 프로젝트로, 실사용자 지표는 없습니다.

---

## 기술 스택

| 구분 | 기술 | 선택 이유 |
|---|---|---|
| Markup / Style | HTML5, CSS3 | 프레임워크 없이 순수 마크업/스타일링 연습 |
| Animation | GSAP 3.5.1, ScrollToPlugin | 스크롤 트리거 애니메이션과 부드러운 스크롤 이동 구현 |
| Carousel | Swiper 6.8.4 | 공지/프로모션/어워드 등 다양한 슬라이드 옵션(방향, autoplay, pagination)을 한 라이브러리로 통일 |
| Scroll Effect | ScrollMagic 2.0.8 | `scroll-spy` 클래스 토글로 섹션 진입 시 애니메이션 트리거 |
| Utility | Lodash | 스크롤 이벤트 `throttle` 처리로 성능 저하 방지 |
| 기타 | YouTube IFrame API, Google Fonts(Nanum Gothic), Material Icons | 영상 임베드 및 디자인 요소 재현 |
| Tooling | Prettier | 코드 포맷 통일 (`.prettierrc`) |

이번 프로젝트에서 **처음 써본 기술**은 ScrollMagic과 GSAP ScrollToPlugin입니다. CSS만으로는 어려운 "스크롤 위치에 따른 클래스 토글"과 "특정 좌표로 부드럽게 스크롤"을 라이브러리로 해결하는 경험을 쌓았습니다.

---

## 프로젝트 구조

```
starbucks-homepage/
├── index.html          # 메인 페이지
├── signin/
│   └── index.html       # 로그인 페이지
├── css/
│   ├── common.css       # 헤더/푸터 등 공통 스타일
│   ├── main.css        # 메인 페이지 섹션별 스타일
│   └── signin.css       # 로그인 페이지 스타일
├── js/
│   ├── common.js       # 검색창 인터랙션, 연도 표시
│   ├── main.js         # GSAP/Swiper/ScrollMagic 인터랙션
│   └── youtube.js       # YouTube IFrame API 초기화
├── images/             # 슬라이드, 배경, 아이콘 등 에셋
└── favicon.ico / favicon.png
```

---

## 주요 기능

**1. 헤더 인터랙션**

검색창 focus 시 확장 애니메이션, 스크롤 500px 이상 시 상단 뱃지 페이드아웃 및 Top 버튼 노출.

**2. 메인 비주얼**

GSAP를 이용한 텍스트/이미지 순차 페이드인 효과.

**3. 캐러셀 3종**

공지사항(세로 자동 슬라이드), 프로모션(가운데 정렬 + 토글 숨김 버튼), Awards(5개 노출 + 좌우 네비게이션) — 모두 Swiper로 구현.

**4. 스크롤 스파이 애니메이션**

`section.scroll-spy` 요소가 뷰포트에 80% 진입하면 ScrollMagic이 `show` 클래스를 토글해 섹션 등장 애니메이션을 트리거.

**5. 플로팅 오브젝트**

`floating1~3` 요소를 GSAP로 랜덤한 딜레이/폭으로 상하 반복 이동시켜 자연스러운 부유 효과 연출.

**6. YouTube 임베드**

YouTube IFrame API를 동적으로 로드해 자동재생 + 무음 + 반복재생 플레이어 삽입.

**7. 로그인 페이지**

메인 페이지와 분리된 `/signin` 경로에 별도 UI 구현.

---

## 실행 방법

빌드 도구 없이 정적 파일로 구성되어 있어 별도 설치 과정이 필요 없습니다.

```bash
git clone [저장소_URL]
cd starbucks-homepage

# 방법 1: 브라우저로 바로 열기
open index.html

# 방법 2: 로컬 서버로 실행 (권장 — CORS/상대경로 이슈 방지)
npx live-server
```

---

## 회고

**배운 점**

GSAP·Swiper·ScrollMagic 세 라이브러리를 한 페이지에서 같이 써보면서 각각의 역할을 명확히 구분하는 법을 익혔습니다. 특히 스크롤 이벤트에 애니메이션을 직접 걸면 성능이 급격히 떨어진다는 걸 체감하고 나서, `lodash.throttle`로 스크롤 핸들러 실행 빈도를 제한하는 습관을 들이게 됐습니다.

**아쉬운 점**

화면 크기를 데스크톱 기준으로만 만들어서 모바일/태블릿 대응이 전혀 안 되어 있습니다. 로그인 페이지도 UI만 있고 실제 인증 로직은 없어서, 지금은 "마크업+인터랙션 재현"에 그친 상태입니다.

**다음 계획**

반응형 브레이크포인트 추가 → 이미지 lazy loading 적용 → 로그인 폼에 최소한의 유효성 검사 붙이기 순으로 진행할 예정입니다.

### 향후 개선 계획

- [ ] 반응형 레이아웃 대응 (모바일/태블릿)
- [ ] 로그인 페이지 실제 인증 로직 연동
- [ ] 이미지 lazy loading 적용

</div>
