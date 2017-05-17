---
layout: post
title:  SpringBoot中拦截器和解析器的使用
date:   2017-05-17 22:18:00 +0800
tags: 能工巧匠集
---

学习过SpringMVC的同学，对于HandlerInterceptorAdapter（拦截器）和HandlerMethodArgumentResolver（解析器）都不陌生，但是对于初涉Spring Boot的我，关于它们的使用却有点不知所措。

其实Spring Boot关于拦截器，解析器的使用与SpringMVC是一样的，主要是需要把拦截器和解析器配置进项目，在传统的SpringMVC项目中，拦截器和解析器的配置是配置在xml文件中的。它们可能像这样：

interceptorAdapter:

    <bean id="authorizationIntercepter" class="com.babywolf.authorization.interceptor.AuthorizationIntercepter">
    
    </bean>
    <bean class="org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="authorizationIntercepter"/>
        </list>
    </property>
    </bean>
    
argumentResolver:

    <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="customArgumentResolvers">
        <list>
            <bean class="com.babywolf.authorization.resolvers.CurrentUserMethodArgumentResolver" />
        </list>
    </property>
    </bean>

但是在Spring Boot的配置写法更为简单没有繁琐的xml配置，我们只需要在继承WebMvcConfigurerAdapter添加你的拦截器和解析器即可。上面的两段配置，它们可以被写成这样：

    @Configuration
    public class MvcConfig extends WebMvcConfigurerAdapter {
    
        @Autowired
        private AuthorizationIntercepter authorizationIntercepter;
    
        @Autowired
        private CurrentUserMethodArgumentResolver currentUserMethodArgumentResolver;
    
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(authorizationIntercepter);
        }
    
        @Override
        public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
            argumentResolvers.add(currentUserMethodArgumentResolver);
        }
    }
    

这里的AuthorizationIntercepter与CurrentUserMethodArgumentResolver是我自定义的拦截器和解析器。值得注意的是@Configuration注解千万不能少，不然MvcConfig不会作为配置文件加载类被调用，使得项目确实配置项，完整的代码：

    @Component
    public class AuthorizationIntercepter extends HandlerInterceptorAdapter {
        @Autowired
        TokenManager manager;
    
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            if (!(handler instanceof HandlerMethod)) {
                return true;
            }
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            Method method = handlerMethod.getMethod();
    
            String authorization = request.getHeader(Constants.AUTHORIZATION);
    
            TokenModel model = manager.getToken(authorization);
            if (manager.checkToken(model)) {
                request.setAttribute(Constants.CURRENT_USER_ID, model.getUserId());
                return true;
            }
            if (method.getAnnotation(Authorization.class) != null) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                return false;
            }
            return true;
        }
    }

    @Component
    public class CurrentUserMethodArgumentResolver implements HandlerMethodArgumentResolver {
    
        @Autowired
        private UserRepository userRepository;
    
        @Override
        public boolean supportsParameter(MethodParameter methodParameter) {
            if (methodParameter.getParameterType().isAssignableFrom(User.class) &&
                    methodParameter.hasParameterAnnotation(CurrentUser.class)) {
                return true;
            }
            return false;
        }
    
        @Override
        public Object resolveArgument(MethodParameter methodParameter, ModelAndViewContainer modelAndViewContainer, NativeWebRequest nativeWebRequest, WebDataBinderFactory webDataBinderFactory) throws Exception {
            Long currentUserId = (Long) nativeWebRequest.getAttribute(Constants.CURRENT_USER_ID, RequestAttributes.SCOPE_REQUEST);
            if (currentUserId != null) {
                //从数据库中查询并返回
                return userRepository.findOne(currentUserId);
            }
            throw new MissingServletRequestPartException(Constants.CURRENT_USER_ID);
        }
    }
    


