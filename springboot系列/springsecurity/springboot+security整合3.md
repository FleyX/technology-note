&emsp;&emsp;这篇讲解如何自定义鉴权过程，实现根据数据库查询出的url和method是否匹配当前请求的url和method来决定有没有权限。security鉴权过程如下：
![鉴权流程][鉴权流程]

##一、 重写metadataSource类

1. 编写MyGranteAuthority类，让权限包含url和method两个部分。
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
2. 编写MyConfigAttribute类，实现ConfigAttribute接口，代码如下：
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
3. 编写MySecurityMetadataSource类，获取当前url所需要的权限
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

##二、 编写MyAccessDecisionManager类

&emsp;&emsp;实现AccessDecisionManager接口以实现权限判断,直接return说明验证通过，如不通过需要抛出对应错误，代码如下：
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

##三、 编写MyFilterSecurityInterceptor类
&emsp;&emsp;该类继承AbstractSecurityInterceptor类，实现Filter接口,代码如下：
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

## 四、 加入到security的过滤器链中
```java
.addFilterBefore(urlFilterSecurityInterceptor,FilterSecurityInterceptor.class)
```
完成


















[鉴权流程]:data:image/jpeg;base64,/9j/4QAYRXhpZgAASUkqAAgAAAAAAAAAAAAAAP/sABFEdWNreQABAAQAAAAQAAD/4QOTaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wLwA8P3hwYWNrZXQgYmVnaW49Iu+7vyIgaWQ9Ilc1TTBNcENlaGlIenJlU3pOVGN6a2M5ZCI/PiA8eDp4bXBtZXRhIHhtbG5zOng9ImFkb2JlOm5zOm1ldGEvIiB4OnhtcHRrPSJBZG9iZSBYTVAgQ29yZSA1LjMtYzAxMSA2Ni4xNDU2NjEsIDIwMTIvMDIvMDYtMTQ6NTY6MjcgICAgICAgICI+IDxyZGY6UkRGIHhtbG5zOnJkZj0iaHR0cDovL3d3dy53My5vcmcvMTk5OS8wMi8yMi1yZGYtc3ludGF4LW5zIyI+IDxyZGY6RGVzY3JpcHRpb24gcmRmOmFib3V0PSIiIHhtbG5zOnhtcE1NPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvbW0vIiB4bWxuczpzdFJlZj0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL3NUeXBlL1Jlc291cmNlUmVmIyIgeG1sbnM6ZGM9Imh0dHA6Ly9wdXJsLm9yZy9kYy9lbGVtZW50cy8xLjEvIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6RTk3RUIwQzU4RTQwMTFFODk1NkNFMDY2NjI4REM4NjAiIHhtcE1NOkluc3RhbmNlSUQ9InhtcC5paWQ6RTk3RUIwQzQ4RTQwMTFFODk1NkNFMDY2NjI4REM4NjAiIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENTNiBXaW5kb3dzIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0UmVmOmluc3RhbmNlSUQ9InV1aWQ6ZmFmNWJkZDUtYmEzZC0xMWRhLWFkMzEtZDMzZDc1MTgyZjFiIiBzdFJlZjpkb2N1bWVudElEPSIxOEQzQTk4QkIxMTFERTA1MjY5MkYxNzEyNTZEMzJEMSIvPiA8ZGM6Y3JlYXRvcj4gPHJkZjpTZXE+IDxyZGY6bGk+ZnhiPC9yZGY6bGk+IDwvcmRmOlNlcT4gPC9kYzpjcmVhdG9y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