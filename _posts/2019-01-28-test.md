---
layout: post
title: Interceptor 적용 경험기
author: 박준하
---


## 인터셉터(Interceptor) 적용 경험기
-----

메일 서비스의 로그인 부분을 맡아서 개발하면서 세션 검사를 어떻게 해줘야 하나 고민하던 중에
멘토님과 팀원이 인터셉터라는 것을 알려줬습니다.
인터셉터는 특정 URI로 요청시 Controller로 가는 요청을 가로채는 역할을 합니다.
같은 어플리케이션 내에서만 접근하는 필터(Filter)와는 다르게 스프링에서 관리되기 때문에 스프링 내의
모든 객체에 접근이 가능합니다.
모든 컨트롤러에서 세션이 유효한지 확인을 해야하는데 코드의 중복을 막기위해 인터셉터를 이용해서
로그인 정보처리를 하면 중복 코드가 확 줄어서 사용할 수 있습니다.

스프링에서 인터셉터를 지원하기 위해서 HandlerInterceptor 인터페이스와 HandlerInterceptorAdapter 추상 클래스를 지원합니다.
추상 클래스에서는 3가지 메소드를 제공하는데 이 3가지 메소드를 오버라이딩해서 이용할 수 있습니다.

1) public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)

: Controller로 요청이 들어가기 전에 수행됩니다.

 : request, response, handler 등의 매개변수를 이용가능한데 우리가 아는 HttpServletRequest, HttpServletResponse,

  이고, 나머지 하나는 이 preHandle() 메서드를 수행하고 수행될 컨트롤러 메서드에 대한 정보를 담고 있는 handle

  입니다.

2) postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,

ModelAndView modelAndView)

: 컨트롤러의 메서드의 처리가 끝나 return 되고 화면을 띄워주는 처리가 되기 직전에 이 메서드가 수행됩니다.

: ModelAndView 객체에 컨트롤러에서 전달해 온 Model 객체가 전달됨으로 컨트롤러에서 작업 후 

: postHandle() 에서 작업할 것이 있다면 ModelAndView를 이용하면 됩니다.

3) afterCompletion()

: 컨트롤러가 수행되고 화면처리까지 끝난 뒤 호출됩니다.



package com.nhnent.rookie.dabada;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
 
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
 

public class LoginInterceptor extends HandlerInterceptorAdapter{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
    	if("/login".equals(request.getRequestURI())) return true;
    	if("/".equals(request.getRequestURI())) return true;

        HttpSession session = request.getSession();
        Object obj = session.getAttribute("id");
        
        if ( obj == null ){
            response.sendRedirect("/login");
            return false;
        }
        session.setMaxInactiveInterval(30*60);
        return true;
    }
}

이게 실제로 적용한 인터셉터 코드인데 request가 기본값이거나 로그인 페이지에서 온 경우 컨트롤러로 그냥 보내주고
그 외의 경우에서는 세션 검사를 해서 로그인 페이지로 보내거나 시간을 초기화 해주고 컨트롤러로 보내줍니다.


package com.nhnent.rookie.dabada;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;


//@EnableWebMvc <-- Spring boot를 사용하는 경우 사용할 필요가 없는 어노테이션
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter{
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new LoginInterceptor())
				.addPathPatterns("/**");
	}
}

이게 설정 파일인데 인터셉터를 하나 생성해주고 .addPathPatterns("/**");를 통해 모든 Url경로에 적용 시켜줍니다.
여기서 제가 애를 먹은 부분이 저 @EnableWebMvc 어노테이션 때문인데 스프링 부트를 사용하면 굳이 필요하지 않는 어노테이션인데
여기저기 검색해보면서 코드들을 보면 저 어노테이션을 매번 사용하는데 저걸 사용하면 property에 추가해준 prefix나 suffix가 
잘 동작하지 않는 등 Config파일 외의 설정이 잘 안먹었습니다.
처음에는 몰라서 Config파일에 prefix, suffix 설정을 추가해주고 뷰 처리도 해주고 했는데도 잘 동작하지 않아서 
이걸 어떻게 해야하나 싶었는데 코드를 모두 지웠는데도 잘 동작하지 않는 것을 보고 어노테이션 문제일거라는
팀원의 추측으로 @EnableWebMvc를 지워보니 설정이 잘 동작하는 것을 확인했습니다.
이후 위의 코드처럼 인터셉터를 추가해주고 메일 서비스를 돌려보니 인터셉터가 잘 적용 되었습니다!
