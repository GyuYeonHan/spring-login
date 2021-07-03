# 스프링 Login

----

## 쿠키, 세션
쿠키에는 영속 쿠키와 세션 쿠키가 있다.
- 영속 쿠키: 만료 날짜를 입력하면 해당 날짜까지 유지
- 세션 쿠키: 만료 날짜를 생략하면 브라우저 종료시 까지만 유지

~~~java
Cookie idCookie = new Cookie("memberId", String.valueOf(loginMember.getId()));
response.addCookie(idCookie); // HttpServletResponse
~~~
로그인에 성공하면 쿠키를 생성하고 HttpServletResponse 에 담는다. 쿠키 이름은 `memberId` 이고, 값은 회원의 `id`를담아둔다.
웹 브라우저는 종료 전까지 회원의 `id`를 서버에 계속 보내줄 것이다.

~~~java
@GetMapping("/")
public String homeLogin(@CookieValue(name = "memberId", required = false) Long memberId, Model model)
~~~
`@CookieValue`를 통해 편리하게 쿠키를 조회할 수 있다.
로그인 하지 않은 사용자도 접근하기 위해서 `required = false`

### 쿠키 만료 시키기
- 세션 쿠키이므로 웹 브라우저 종료시
- 서버에서 해당 쿠키의 종료 날짜를 0으로 지정

~~~java
private void expireCookie(HttpServletResponse response, String cookieName) {
      Cookie cookie = new Cookie(cookieName, null); // 어짜피 만료시킬 쿠키기 때문에 이름만 같으면 값은 상관없음
      cookie.setMaxAge(0);
      response.addCookie(cookie);
}
~~~

### 쿠키와 보안 문제
- 쿠키 값은 임의로 변경할 수 있다.
- 쿠키에 보관된 정보는 훔쳐갈 수 있다.

`로그인은 보안 문제 때문에 쿠키로 하기에는 부적합`

> 이 문제를 해결하려면 결국 중요한 정보를 모두 서버에 저장해야 한다. 그리고 클라이언트와 서버는 추정 불가능한 임의의 식별자 값으로 연결해야 한다.

> 이렇게 서버에 중요한 정보를 보관하고 연결을 유지하는 방법을 세션이라 한다.

#### 세션을 이용한 방식
![image](https://user-images.githubusercontent.com/54987488/124359669-f4fb1980-dc60-11eb-9a6e-f745fe1d3b21.png)
1. 서버 측의 세션 저장소에 `세션 ID` 생성 및 값 저장
2. 서버에서 `세션 ID`에 대한 `쿠키`를 생성하여 클라이언트 전달
3. 클라이언트 측에서 해당 `쿠키`를 이용해 로그인 정보 유지

~~~java
//세션이 있으면 있는 세션 반환, 없으면 신규 세션 생성
HttpSession session = request.getSession(); 

//세션에 로그인 회원 정보 보관
session.setAttribute(SessionConst.LOGIN_MEMBER, loginMember);
~~~
`request.getSession(true)`
- 세션이 있으면 기존 세션을 반환한다.
- 세션이 없으면 새로운 세션을 생성해서 반환한다.

`request.getSession(false)`
- 세션이 있으면 기존 세션을 반환한다.
- 세션이 없으면 새로운 세션을 생성하지 않는다. `null` 을 반환한다.

` session.invalidate()`
- 세션을 제거한다.

~~~java
@GetMapping("/")
public String homeLoginV3Spring(@SessionAttribute(name = SessionConst.LOGIN_MEMBER, required = false) Member loginMember, Model model)
~~~
스프링이 `@SessionAttribute` 어노테이션을 통해 편하게 세션 조회 기능 제공


## 필터, 인터셉터

### 서블릿 필터

- HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러 //로그인 사용자
- HTTP 요청 -> WAS -> 필터(적절하지 않은 요청이라 판단, 서블릿 호출X) //비 로그인 사용자
  
참고로 스프링을 사용하는 경우 여기서 말하는 서블릿은 스프링의 디스패처 서블릿으로 생각하면 된다

~~~java
public interface Filter {
      public default void init(FilterConfig filterConfig) throws ServletException {}
      public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException;
      public default void destroy() {}
   }
~~~
`Filter` 인터페이스를 구현 후 `doFilter` 메소드를 오버라이드 하여 서블릿 필터 구현 가능

~~~java
@Configuration
public class WebConfig { 
    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LogFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        return filterRegistrationBean;
    }
}
~~~
구현한 필터를 등록하면 HTTP 요청 시 필터 적용

### 스프링 인터셉터
스프링 인터셉터도 서블릿 필터와 같이 웹과 관련된 공통 관심 사항을 효과적으로 해결할 수 있는 기술이다. 서블릿 필터가 서블릿이 제공하는 기술이라면, 스프링 인터셉터는 스프링 MVC가 제공하는 기술이다. 둘다 웹과 관련된 공통 관심 사항을 처리하지만, 적용되는 순서와 범위, 그리고 사용방법이 다르다.

- HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터 -> 컨트롤러 //로그인 사용자
- HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터(적절하지 않은 요청이라 판단, 컨트롤러 호출 X) // 비 로그인 사용자

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {}

    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {}

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {}
}
```
서블릿 필터보다 세분화 되어 적용 가능

- `preHandle` : 컨트롤러 호출 전에 호출된다. (더 정확히는 핸들러 어댑터 호출 전에 호출된다.)
> preHandle 의 응답값이 true 이면 다음으로 진행하고, false 이면 더는 진행하지 않는다. false
인 경우 나머지 인터셉터는 물론이고, 핸들러 어댑터도 호출되지 않는다. 
- `postHandle` : 컨트롤러 호출 후에 호출된다. (더 정확히는 핸들러 어댑터 호출 후에 호출된다.)
- `afterCompletion` : 뷰가 렌더링 된 이후에 호출된다.

~~~java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**")
                .excludePathPatterns("/css/**", "/*.ico", "/error");

        registry.addInterceptor(new LoginCheckInterceptor())
                .order(2)
                .addPathPatterns("/**")
                .excludePathPatterns("/", "/members/add", "/login", "/logout", "/css/**", "/*.ico", "/error");
    }
}
~~~
위와 같이 인터셉터 등록하여 사용 가능.
`excludePathPatterns`를 제공하여 필터보다 정밀하게 URL 지정 가능

----
출처: 김영한 님의 인프런 강의 - [스프링 MVC 2편 - 백엔드 웹 개발 활용 기술](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-2)
