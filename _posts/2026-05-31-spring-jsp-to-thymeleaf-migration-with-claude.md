---
title: 클로드 코드와 함께한 JSP → Thymeleaf 마이그레이션 후기 (feat. vibe coding)
date: 2026-05-31 19:50 +0900
categories: Spring
tags: claude, thymeleaf, vibe coding
---

타임리프 마이그레이션 작업을 마치고 팀장님으로부터 발표 요청이 들어왔다.

> "AI가 뭘 해줬는지, 한 번에 잘 됐는지, 프롬프트를 어떻게 고쳐갔는지 정리해서 간단히 공유해 줬으면 좋겠어요.
> 아무리 AI가 해줬다 한들, 최후 검증은 개발자가 하잖아요."

발표 자료를 준비하면서 슬라이드로만 남기기엔 아까운 내용들이 많았다.

고민하고 삽질하며 얻은 것들을 흘려보내기 아까워서 포스트로 남긴다. 기록은 나를 위한 것이기도 하고, 언젠가 비슷한 작업을 앞둔 누군가에게 참고가 됐으면 하는 마음이다.

## 왜 갑자기 Thymeleaf였나?

솔직히 말하면 처음부터 'Thymeleaf로 가야겠다!'는 확신이 있었던 건 아니다. 발단은 Spring Boot 버전 업그레이드였다.

현재 개발팀에서 개선 중인 프로젝트는 Spring Boot 2.2.4, Java 11 기반으로 꽤 오래됐다. 그걸 Spring Boot 3.x + Java 21로 올리는 작업을 검토하다가 결정적인 문제를 마주쳤다. **Apache Tiles 3가 Spring 6 (= Spring Boot 3의 기반)에서 완전히 제거된다는 것.**

```java
// ContextConfig.java — 이 코드가 Spring Boot 3에서 통째로 동작하지 않는다
@Bean public TilesViewResolver tilesViewResolver() { ... }
@Bean public TilesConfigurer tilesConfigurer() { ... }
```

`TilesViewResolver`, `TilesConfigurer`가 `spring-webmvc`에서 사라졌기 때문에 JSP 기반 뷰 렌더링 자체가 불가능해진다. 뷰 파일만 119개, Tiles 레이아웃과 템플릿 JSP까지 합치면 143개 파일이나 영향이 가는 문제가 발생했다.

### 다른 선택지는 없었나

당연히 고민했다. 대안으로 떠올린 것들은 이렇다.

**1. JSP만 유지, Tiles 없이**<br/>
Spring Boot 3에서 JSP 자체가 완전히 불가능한 건 아니다. 내장 톰캣과의 통합이 공식적으로 지원되지 않는다는 경고만 있을 뿐, WAR 배포 방식을 쓰면 일단 동작한다. 문제는 레이아웃이다.

Tiles가 해주던 일은 단순하지 않았다. 레이아웃 파일 하나에 헤더, 네비게이션, 푸터를 정의해 두고, 각 뷰는 **어떤 레이아웃을 쓸지**만 선언하면 된다. 이 구조를 Tiles 없이 JSP로 재현하려면, 개발자가 직접 레이아웃 조합 로직을 떠안아야 한다.

```jsp
<%-- /WEB-INF/tiles/layout.jsp --%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html>
<html>
<head>
    <title>My Application</title>
</head>
<body>
    <jsp:include page="/WEB-INF/tiles/header.jsp"/>
    <jsp:include page="/WEB-INF/tiles/nav.jsp"/>

    <div class="content">
        <jsp:include page="${bodyPath}" />
    </div>

    <jsp:include page="/WEB-INF/tiles/footer.jsp"/>
</body>
</html>
```

```java
// 모든 컨트롤러 메서드마다 이 작업이 반복된다
@GetMapping("/dashboard")
public String dashboard(Model model) {
    model.addAttribute("bodyPath", "/WEB-INF/views/myroom/dashboard.jsp");
    return "tiles/layout";
}
```

Tiles가 숨겨줬던 복잡성이 컨트롤러 전역으로 퍼지는 구조다.

문제는 단순 반복에서 그치지 않는다. 뷰 경로가 문자열로 하드코딩되어 있어서 파일명이나 폴더 구조가 바뀌면 IDE가 잡아주지 못하고, 사용자가 해당 메뉴를 클릭하는 런타임 시점에 404가 터진다. 레이아웃 종류가 늘어나도(팝업, 프린트 등) 컨트롤러마다 분기를 추가해야 하고, 특정 페이지에서만 필요한 CSS나 JS를 `<head>`에 주입하는 것도 편법 없이 어렵다. 동작은 하지만 유지보수할수록 손댈 곳이 늘어나는 구조다.

이미 119개 파일이 Tiles 레이아웃에 기대고 있는 상태에서 이 방향은 기술 부채를 더 쌓는 것이었다.

반면, Thymeleaf의 `layout:decorate` 는 Tiles의 `extends` 와 개념이 거의 동일하다. 레이아웃은 `base.html` 한 곳에서 관리되고, 컨트롤러는 뷰 이름만 반환하는 Tiles 시절 구조로 그대로 돌아온다.

```html
<!-- Thymeleaf: 레이아웃 선언 한 줄로 Tiles의 extends와 동일한 효과 -->
<html layout:decorate="~{layout/base}">
```

```java
// 컨트롤러는 뷰 이름만 반환 — Tiles 시절과 동일
return "myroom/dashboard";
```

수정해야 하는 파일 수만 놓고 보면 두 방향의 작업량은 비슷하다. 그러나 JSP를 Tiles 없이 유지하는 건 현재의 복잡성을 그대로 안고 가는 것이고, Thymeleaf 전환은 그 복잡성을 한 번 정리하는 기회다.

어차피 두 방식의 작업 공수가 비슷하다면,  <u>작업 후 유지보수성이 높아지는 타임리프를 선택하는 편이 합리적이라 판단</u>했다.

여기에 더해 **'내츄럴 템플릿(Natural Template)'**이라는 타임리프만의 특징도 매력적이었다. 뷰 파일이 순수 HTML 형태를 유지하기 때문에, 서버를 구동하지 않고 로컬 브라우저에서 더블클릭만 해도 마크업이 깨지지 않고 렌더링된다. JSP 시절, 화면 자잘한 수정 결과를 확인하려고 매번 톰캣 리로딩을 기다리며 끊겼던 개발 흐름을 생각하면 꽤 의미 있는 생산성 향상 포인트였다.

**2. FreeMarker, Mustache 등 다른 템플릿 엔진**<br/>
Spring Boot 3에서 공식 지원하는 뷰 기술 중 Thymeleaf 외에도 FreeMarker, Mustache가 있다. 하지만 이쪽은 팀 내 레퍼런스도 없고, 커뮤니티 자료나 AI 학습 데이터도 Thymeleaf가 압도적으로 많다. 도입 후 러닝커브를 감당할 여유가 없다.

무엇보다 Thymeleaf가 스프링 공식 문서에서 가장 적극적으로 권장하는 기본 기술이라는 점이 컸다. 향후 버전 업그레이드 시 뷰 계층 호환성으로 스트레스 받을 일을 최소화하고 싶었다.

**3. React/Vue 등 SPA 전환**<br/>
장기적으로는 고려할 수 있지만 지금 당장은 현실적이지 않다. 백엔드 API로 전면 재설계해야 하고, 배포 파이프라인도 달라진다. 이번 목표는 '뷰 레이어 교체'지 "아키텍처 전환"이 아니다.

---

결국 **Thymeleaf 3.0(Spring Boot 2.x) → 3.1(Spring Boot 3.x) 간 핵심 문법이 하위 호환**된다는 점, Spring 공식 생태계 내에서 가장 잘 지원된다는 점, 그리고 팀에 기존 경험이 어느 정도 있다는 점이 결정 요인이 됐다.

더불어 뷰 레이어 전환과 Spring Boot 3 의존성 변경을 동시에 하면 디버깅이 너무 복잡해지므로, **Boot 2.2.4 환경에서 Thymeleaf 전환을 먼저 완료하고, Boot 3으로 올리는 두 단계 전략**을 채택했다.

그리고 한 가지 더. 119개 파일을 사람이 하나씩 변환하는 건 현실적으로 부담스럽다. 여기서 **Claude Code**를 활용하면 이야기가 달라진다. 반복적인 문법 변환 패턴을 학습시키면 초안 생성 속도를 크게 끌어올릴 수 있고, 그만큼 개발자는 변환 결과 검증과 예외 케이스 처리에 집중할 수 있다. 다음 섹션부터는 실제로 실제로 어떻게 활용했는지 다루겠다.

## 119개 파일, AI 없이 하면 어땠을까

타임리프 변환 작업의 기계적인 부분을 먼저 정리해보면 이렇다.

- JSP 지시어(`<%@ page %>`, `<%@ taglib %>`) 제거
- JSTL 태그를 Thymeleaf 속성으로 치환 (`<c:if>` → `th:if`, `<c:forEach>` → `th:each` 등)
- Tiles `layout:decorate` 선언 추가
- `<script>` 내부 EL 표현식 처리 (`th:inline="javascript"` + `[[${변수}]]`)
- `<fmt:formatDate>`, `<fmt:formatNumber>` → Thymeleaf 유틸리티 함수로 교체

이걸 119개 파일에 수작업으로 한다고 상상해보자. JSTL 함수(`${fn:}`)만 32개 파일 59곳에 사용 중이었고, Map 접근 패턴도 22개 파일 153건이었다. 단순 반복 작업이지만 파일 하나당 수백 줄짜리 JSP가 적지 않다. 직접 작업할 경우 분명히 실수가 나온다. 다른 업무를 병행하면서 이걸 수작업했다면 야근과 주말을 갈아냈어야 할 것이다.

## AI를 쓰면 다 해결된다고 생각했다

처음엔 그랬다. 계획서 마크다운을 만들고, 변환 규칙을 죄다 적어서 Claude Code에게 던졌다. 'T-3 단계 변환해줘' 한 마디면 되겠지 싶었다.

초반 몇 개는 잘 됐다. 그런데 파일이 많아지고 대화가 길어질수록 문제가 쌓이기 시작했다.

**[컨텍스트 폭발]**<br/>
계획서가 워낙 방대하다 보니 매번 명령 복사-붙여넣기를 해야 했고, 대화 히스토리가 길어지면 AI가 앞서 정한 규칙을 까먹기 시작했다. 레이아웃 판별 규칙이 있는데도 파일명이나 내용만 보고 추측해서 잘못된 레이아웃을 붙이는 경우가 생겼다.

**[무조건 검증해야 한다!]**<br/>
빌드가 통과되었음에도 화면이 깨졌다. 에러 로그도 없이 페이지가 중간에서 잘리거나, 특정 데이터일 때만 뻑이 나거나, 차트가 아예 그려지지 않거나. AI가 잘 변환했다 해도 실제 브라우저에서 확인하면 전혀 다른 이야기였다.

---

그제서야 직감했다. *AI를 도구로 잘 쓰는 방법 자체를 설계해야 한다!*

## 서브에이전트 분업

![images](https://1drv.ms/i/c/9251ef56e0951664/IQRpHouNJH1zSL8U4l1Dult0ATSwpXWEta_gb0b-jJuX5og)

Claude에게 역으로 물었다. 'Claude Code를 가장 효율적으로 쓰는 방법이 무엇인가?' 이 질문이 작업 방식을 바꿨다.

계획서를 통째로 넘기는 대신, 작업을 **세 가지 전문화된 서브에이전트**로 쪼갰다.

사실, 서브에이전트 구조 자체도 Claude가 역할에 맞게 설계해줬다. Claude Code 사용법을 깊이 공부하지 않아도 충분히 활용할 수 있었다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSBk-zb9OAuToI26g-9hxr3AaoLF9sAHRAaMPQNLzV6JyI)

### layout-resolver — 변환 전 사전 분석

변환을 시작하기 전에 컨트롤러 코드를 grep해서 어떤 레이아웃을 써야 하는지 먼저 판별한다.

```
grep -r "classroom/" src/main/java
→ return "myroom/web/classroom/main/index" 확인
→ prefix: myroom/ → layout_basic.html 사용
```

파일 내용이나 경로 이름만 보고 레이아웃을 추측하다가 틀리는 일이 없어졌다. AJAX로 부분 로딩되는 파일들처럼 레이아웃을 붙이면 안 되는 경우도 미리 걸러낼 수 있었다.

### jsp-converter — 파일 단위 변환

한 번에 큰 파일을 통째로 변환하려다가 토큰 초과로 누락이 생기는 문제가 있었다. 이걸 파일 단위로 쪼개서 호출하고, 변환이 끝날 때마다 한 줄 보고만 메인으로 돌려받았다. 메인 컨텍스트가 보존됐다.

JSTL EL과 SpEL의 차이도 이 단계에서 명시적으로 규칙화했다. 대표적인 함정이 Map 접근이다.

```
JSTL EL:  ${params.searchCode}    → 키 없으면 null (조용히 통과)
SpEL:     ${params.searchCode}    → 키 없으면 EL1008E 예외 ❌
SpEL:     ${params['searchCode']} → 키 없으면 null ✅
```

기존 JSP는 Map 키가 없어도 그냥 null로 처리해줬기 때문에 개발자들이 이걸 의식하지 않고 썼다. Thymeleaf로 바꾸면서 이게 런타임 예외로 터진다. 153건을 일일이 고치는 게 아니라 규칙을 `CLAUDE.md`에 박아두고 에이전트가 자동으로 적용하게 했다.

### spel-checker — 정적 점검

변환 후 런타임에서 터질 위험 패턴을 미리 찾는다. JSTL 태그 잔재, Map 도트 접근, `th:text`/`th:utext` 혼용 등을 스캔한다.

## AI로 변환할 때 진짜 좋았던 것들

### 기계적 반복 작업의 속도

JSTL `<c:forEach>` 수십 개를 `th:each`로, `<fmt:formatDate>`를 `#dates.format()`으로 치환하는 작업은 AI가 확실히 빠르다. 손으로 할 때 집중력이 떨어지면 반드시 실수가 나오는데, 이런 패턴 치환은 AI가 훨씬 안정적이다.

### 에러 로그 그대로 붙여넣기

런타임 오류 발생 시, 서버 스택트레이스를 통째로 던지면 몇 번째 줄의 어떤 표현식에서 어떤 이유로 터졌는지 빠르게 찾아줬다. 혼자 씨름하면 30분 걸릴 걸 5분 만에 원인까지 짚어주는 경우가 많았다.

### 대량 파일의 일괄 치환

같은 패턴이 16곳, 24곳씩 퍼져있는 경우 `replace_all`로 한 번에 처리했다. 정규식 작성하고 검토하고 실행하는 과정을 AI가 대신했다.

### 계획서 작성 자체를 AI에게 맡기기

작업 전에 "이 파일들을 변환할 건데 어떤 순서로, 어떤 방식으로 하는 게 좋겠어?"를 물어보는 것 자체가 유용했다. 빠뜨리기 쉬운 고려사항들을 짚어줬다. 처음부터 인간이 모든 계획을 짜지 않아도 됐다.

## 그런데 이것만큼은 AI가 못 잡았다

![images](https://1drv.ms/i/c/9251ef56e0951664/IQT58DR5X56MS6yfFK5gEYyDAbXmq7ByYqkDP5-fPvCzHDo)

![images](https://1drv.ms/i/c/9251ef56e0951664/IQTTQXwPmyNnQ5bHAzq8fYDqAaTE_i_LXQl4euyCDuPH8FE)

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQNHzJMnFETRaNgfIwAMhecAbICbhANE71pJNtrHGM_ZS4)

### 빌드 성공 ≠ 화면 정상

가장 당혹스러웠던 부분이다. 빌드가 깔끔하게 통과하고 spel-checker도 CLEAN이 뜨는데, 브라우저에서는 화면이 이상하다.

**[항상 같은 분기만 타는 조용한 버그]**<br/>
이게 제일 위험하다. 에러도 안 나고 로그도 없고 그냥 특정 분기가 항상 false로 처리되는 것이다.

```
JSP EL:   ${score eq '0'}  → score가 Integer 0이어도 문자열 '0'과 비교해서 true 반환
SpEL:     ${score eq '0'}  → Integer와 String 타입 불일치 → false (조용히)
```

이런 경우 '에러 없이 틀린다'는 표현이 딱 맞다. 코드 리뷰로도 잘 안 잡힌다. 실제 데이터를 넣어보지 않으면 모른다.

**[데이터 의존 런타임 에러]**<br/>
`null`이 들어오는 경우, 특정 사용자의 데이터 조합에서만 발생하는 경우는 정적 분석으로 잡기 어렵다. 개발 환경에서 일반 계정과 엣지케이스 데이터를 직접 넣어보는 수밖에 없었다.

**[차트 렌더링 실패]**<br/>

JS 인라인 표현식 변환에서 구조 문자가 소실되거나, JSON 배열을 직렬화 방식이 달라져서 차트 데이터가 깨지는 경우가 있었다. 이건 브라우저 콘솔을 직접 열어봐야 드러난다.

---

결론: **AI 점검은 "빌드-타임 버그"를 잡는 데 유용하고, "런타임-데이터 버그"는 사람이 실제로 써봐야 잡힌다.**

### 레이아웃 판단 실수

초기에 서브에이전트 없이 작업할 때, AI가 컨트롤러를 확인하지 않고 파일 경로나 내용만 보고 레이아웃을 결정하는 경우가 있었다. 파일명이 `popup`이 들어가면 팝업 레이아웃이겠거니 추측하는 식. 그런데 실제로는 컨트롤러가 어떤 prefix로 return하느냐가 레이아웃을 결정한다. 이 규칙을 명시적으로 CLAUDE.md에 못박기 전까지 오변환이 꽤 있었다.

## 한 번 잡은 버그는 AI에게 학습시키자

![images](https://1drv.ms/i/c/9251ef56e0951664/IQR1CkDlLVWTTZkwSqU2-Qg0AVO3LmYYZhsc_4nbrirWmRo)

한 번 디버깅한 오류는 다시 디버깅하지 않겠다는 생각으로 `learned-patterns.md`를 만들었다. 오류가 날 때마다 다음 포맷으로 기록했다.

물론 이 포맷도 클로드 코드가 작성해줬다.

```
## [패턴 이름]
### 증상
### 원인
### 표준 처리 패턴
### 점검 명령 (grep이나 spel-checker 플래그)
```

예를 들어 `$.['key']` 안전 탐색 인덱서 조합이 SpEL 파싱 오류를 낸다는 것, `.?[condition]` 안에서 외부 반복 변수를 참조하려면 `#vars['변수명']`을 써야 한다는 것 같은 내용들이 쌓였다.

이 파일을 다음 모듈 변환 시 AI에게 컨텍스트로 함께 넘겼더니, 같은 함정에 걸리는 일이 크게 줄었다. 팀원이 비슷한 작업을 할 때 온보딩 문서로도 쓸 수 있다는 게 부가적인 이점이었다.

## 점진적 배포 전략 없이는 불가능했다

![images](https://1drv.ms/i/c/9251ef56e0951664/IQTiE-WuLkGtSY-Zkme3VDo5AW9jV9NVJK-bK37BrO_u3ek)

100개 넘는 화면을 한 번에 배포하면 어떻게 됐을까. 생각만 해도 아찔하다.

Thymeleaf와 Tiles를 공존시키는 방식을 썼다. `ViewResolver` 우선순위를 Thymeleaf(order=1), Tiles(order=2)로 설정하면, 컨트롤러가 view name을 반환할 때 Thymeleaf가 `templates/` 하위에서 `.html` 파일을 먼저 찾고, 없으면 Tiles가 JSP를 렌더링한다.

```
[컨트롤러 return "project/web/study/index"]
   ↓
ThymeleafViewResolver → templates/project/web/study/index.html 있으면 렌더링
   ↓ (없으면)
TilesViewResolver → tiles3.xml 정의 → JSP 렌더링
```

한 모듈씩 변환하고 검증이 끝나면 `spring.thymeleaf.view-names`에 해당 패턴을 추가해서 검증이 끝난 모듈부터 순서대로 Thymeleaf가 렌더링하도록 전환했다. 모든 모듈 검증이 완료된 T-9 단계에서 Tiles/JSP 의존성을 전부 제거했다.

한 모듈씩 변환이 완료되면 `spring.thymeleaf.view-names`에 해당 패턴을 추가해서 Thymeleaf가 렌더링하도록 단계별로, 점진적으로 전환했다. 모든 모듈 검증이 완료된 T-9 단계에서 Tiles/JSP 의존성을 전부 제거했다.

이 전략 덕분에 운영 중인 서비스에 영향을 주지 않고 모듈별로 안전하게 전환할 수 있었다.

## 느낀 점

**[AI는 초안 생성기다.]**<br/>
AI는 빠르고 지치지 않고 일관적이다. 하지만 코드를 완성해주지는 않는다. 비슷해 보이는 JSP EL과 SpEL의 미묘한 차이처럼, 런타임에서 드러나는 실수는 AI가 만들어줬어도 개발자가 책임져야 한다.

**[규칙은 코드보다 문서로 먼저 정의해야 한다.]**<br/>
AI에게 '타임리프로 바꿔줘' 한 마디로 통하려면, 어떤 레이아웃을 쓸지, Map 접근은 어떻게 할지, 레이아웃 판별은 어떤 기준으로 할지가 문서화되어 있어야 한다. 별도 마크다운에 규칙을 쌓아두어야 비로소 AI가 일관되게 동작한다.

**[대량 작업에서는 컨텍스트 관리가 핵심이다.]**<br/>
긴 대화, 많은 파일, 복잡한 규칙이 섞이면 AI는 앞에서 정한 것을 잊는다. 이걸 막는 방법은 역할을 쪼개고, 메인 컨텍스트에는 요약만 받고, 상태를 파일에 저장해두는 것이었다.

**[에러 없이 틀리는 버그가 가장 위험하다.]**<br/>
빌드가 통과하고 정적 점검도 통과했는데 화면 분기가 조용히 잘못 타는 경우. 이건 코드 리뷰로도 AI 점검으로도 안 잡힌다. 결국 개발자가 실제 데이터로 직접 확인해야 한다. 이 검증 단계는 개발자가 직접 해야 한다.

**[삽질은 문서로 남기자. AI에게 학습시키자.]**<br/>
AI가 생긴 이후로 문서화의 의미가 달라졌다. 예전엔 팀원이 나중에 읽을 것을 기대하며 썼다면, 이제는 다음 변환 작업에서 AI에게 입력하는 컨텍스트가 된다. 회고가 즉각적인 품질 향상으로 돌아오는 루프, 이게 이번 작업에서 가장 인상 깊었다.

## 마치며

![images](https://1drv.ms/i/c/9251ef56e0951664/IQTw9BXBQnoyQKRDjwvZTt1-AaCj-8wEQRedS5_0a8uds1A)

AI 덕분에 코딩 자체에 드는 시간은 확실히 줄었다. 그런데 솔직히 말하면, 수작업으로 했을 때와 비교해 전체 작업 시간이 획기적으로 단축된 것 같지는 않다.

코드를 AI가 써주는 만큼, 그 **코드를 검수하는 시간이 생각보다 많이 들었다.** 단순히 브라우저에서 화면을 눌러보는 것만이 아니라, AI가 변환한 코드 자체를 한 줄씩 읽으며 의도대로 변환됐는지 확인하는 작업도 만만치 않았다. 게다가 처음부터 내가 구축한 프로젝트가 아니었기 때문에 세부 기능을 파악하는 데도 시간이 따로 들었다. 다른 업무를 병행하면서 틈틈이 진행한 것도 속도를 늦춘 요인 중 하나였다.

그럼에도 체감상 달라진 건 분명히 있다. **AI 덕분에 이 작업을 진행하면서 다른 업무도 함께 소화할 수 있었다.** 집중 코딩이 필요한 구간을 AI가 채워주는 동안, 나는 다른 일을 병행할 수 있었다. 단순히 한 작업의 시간이 줄어든 게 아니라, 같은 시간에 더 많은 일을 할 수 있게 된 것이다.

다음 작업인 Spring Boot 3.x 버전 업그레이드에도 이번 경험이 밑거름이 될 것 같다. `javax.*` → `jakarta.*` 일괄 치환, MyBatis 버전 업, 내부 라이브러리 호환성 확인 작업 등이 기다리고 있는데, 작업을 쪼개고 계획서를 먼저 문서화하는 방식, 그리고 이번에 익힌 Claude Code의 파일 참조(`@경로`)나 에러 로그 직접 붙여넣기 같은 활용법은 어떤 작업에서도 써먹을 수 있다고 생각한다.


## References
- https://docs.spring.io/spring-boot/upgrading.html
- https://www.thymeleaf.org/doc/tutorials/3.0/thymeleafspring.html
- https://www.baeldung.com/spring-boot-3-spring-6-new
- https://code.claude.com/docs/en/sub-agents
- Claude Sonet 4.6
- NotebookLM

---

*이 글은 Claude와 함께 작성되었습니다. 코드와 경험은 제 것이고, Claude는 글을 다듬는 데 도움을 주었습니다.*