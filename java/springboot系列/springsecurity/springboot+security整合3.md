---
id="2018-08-22-10-38"
title="springboot+security整合（3）"
headWord="文接上篇，上篇说了那个啥自定义校验的功能，这篇来学学如何自定义鉴权。感觉都定义到这个地步，都不太需要security框架了，再自己整整缓存方面的功能就是一个功能完成的鉴权模块了。"
tags=["java", "spring","springboot","spring-security","security"]
category="java"
serie="spring boot学习"
---

&emsp;&emsp;这篇讲解如何自定义鉴权过程，实现根据数据库查询出的 url 和 method 是否匹配当前请求的 url 和 method 来决定有没有权限。security 鉴权过程如下：
![鉴权流程](./picFolder/pic2.png)

## 一、 重写 metadataSource 类

1. 编写 MyGranteAuthority 类，让权限包含 url 和 method 两个部分。

```java
public class MyGrantedAuthority implements GrantedAuthority {
    private String method;
    private String url;

    public MyGrantedAuthority(String method, String url) {
        this.method = method;
        this.url = url;
    }

    @Override
    public String getAuthority() {
        return url;
    }

    public String getMethod() {
        return method;
    }

    public String getUrl() {
        return url;
    }

    @Override
    public boolean equals(Object obj) {
        if(this==obj) return true;
        if(obj==null||getClass()!= obj.getClass()) return false;
        MyGrantedAuthority grantedAuthority = (MyGrantedAuthority)obj;
        if(this.method.equals(grantedAuthority.getMethod())&&this.url.equals(grantedAuthority.getUrl()))
            return true;
        return false;
    }
}
```

2. 编写 MyConfigAttribute 类，实现 ConfigAttribute 接口，代码如下：

```java
public class MyConfigAttribute implements ConfigAttribute {
    private HttpServletRequest httpServletRequest;
    private MyGrantedAuthority myGrantedAuthority;

    public MyConfigAttribute(HttpServletRequest httpServletRequest) {
        this.httpServletRequest = httpServletRequest;
    }

    public MyConfigAttribute(HttpServletRequest httpServletRequest, MyGrantedAuthority myGrantedAuthority) {
        this.httpServletRequest = httpServletRequest;
        this.myGrantedAuthority = myGrantedAuthority;
    }

    public HttpServletRequest getHttpServletRequest() {
        return httpServletRequest;
    }

    @Override
    public String getAttribute() {
        return myGrantedAuthority.getUrl();
    }

    public MyGrantedAuthority getMyGrantedAuthority() {
        return myGrantedAuthority;
    }
}
```

3. 编写 MySecurityMetadataSource 类，获取当前 url 所需要的权限

```java
@Component
public class MySecurityMetadataSource implements FilterInvocationSecurityMetadataSource {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private JurisdictionMapper jurisdictionMapper;
    private List<Jurisdiction> jurisdictions;

    private void loadResource() {
        this.jurisdictions = jurisdictionMapper.selectAllPermission();
    }


    @Override
    public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
        if (jurisdictions == null) this.loadResource();
        HttpServletRequest request = ((FilterInvocation) object).getRequest();
        Set<ConfigAttribute> allConfigAttribute = new HashSet<>();
        AntPathRequestMatcher matcher;
        for (Jurisdiction jurisdiction : jurisdictions) {
            //使用AntPathRequestMatcher比较可让url支持ant风格,例如/user/*/a
            //*匹配一个或多个字符，**匹配任意字符或目录
            matcher = new AntPathRequestMatcher(jurisdiction.getUrl(), jurisdiction.getMethod());
            if (matcher.matches(request)) {
                ConfigAttribute configAttribute = new MyConfigAttribute(request,new MyGrantedAuthority(jurisdiction.getMethod(),jurisdiction.getUrl()));
                allConfigAttribute.add(configAttribute);
                //这里是获取到一个权限就返回,根据校验规则也可获取多个然后返回
                return allConfigAttribute;
            }
        }
        //未匹配到，说明无需权限验证
        return null;
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return FilterInvocation.class.isAssignableFrom(clazz);
    }
}
```

## 二、 编写 MyAccessDecisionManager 类

&emsp;&emsp;实现 AccessDecisionManager 接口以实现权限判断,直接 return 说明验证通过，如不通过需要抛出对应错误，代码如下：

```java
@Component
public class MyAccessDecisionManager implements AccessDecisionManager{
    private Logger log = LoggerFactory.getLogger(this.getClass());

	@Override
	public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
			throws AccessDeniedException, InsufficientAuthenticationException {
	    //无需验证放行
	    if(configAttributes==null || configAttributes.size()==0)
	        return;
	    if(!authentication.isAuthenticated()){
	        throw new InsufficientAuthenticationException("未登录");
        }
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
        for(ConfigAttribute attribute : configAttributes){
            MyConfigAttribute urlConfigAttribute = (MyConfigAttribute)attribute;
            for(GrantedAuthority authority: authorities){
                MyGrantedAuthority myGrantedAuthority = (MyGrantedAuthority)authority;
                if(urlConfigAttribute.getMyGrantedAuthority().equals(myGrantedAuthority))
                    return;
            }
        }
        throw new AccessDeniedException("无权限");
	}

	@Override
	public boolean supports(ConfigAttribute attribute) {
		return true;
	}

	@Override
	public boolean supports(Class<?> clazz) {
		return true;
	}
}
```

## 三、 编写 MyFilterSecurityInterceptor 类
&emsp;&emsp;该类继承 AbstractSecurityInterceptor 类，实现 Filter 接口,代码如下：

```java
@Component
public class MyFilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

    //注入上面编写的两个类
    @Autowired
    private MySecurityMetadataSource mySecurityMetadataSource;

    @Autowired
    public void setMyAccessDecisionManager(MyAccessDecisionManager myAccessDecisionManager) {
        super.setAccessDecisionManager(myAccessDecisionManager);
    }

    @Override
    public void init(FilterConfig arg0) throws ServletException {
    }


    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        FilterInvocation fi = new FilterInvocation(request, response, chain);
        invoke(fi);
    }

    public void invoke(FilterInvocation fi) throws IOException, ServletException {
        //这里进行权限验证
        InterceptorStatusToken token = super.beforeInvocation(fi);
        try {
            fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
        } finally {
            super.afterInvocation(token, null);
        }
    }

    @Override
    public void destroy() {
    }

    @Override
    public Class<?> getSecureObjectClass() {
        return FilterInvocation.class;
    }

    @Override
    public SecurityMetadataSource obtainSecurityMetadataSource() {
        return this.mySecurityMetadataSource;
    }
}
```

## 四、 加入到 security 的过滤器链中

```java
.addFilterBefore(urlFilterSecurityInterceptor,FilterSecurityInterceptor.class)
```

完成
