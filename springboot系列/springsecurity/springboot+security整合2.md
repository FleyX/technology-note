[id]:2018-08-21
[type]:javaee
[tag]:java,spring,springsecurity,scurity

&emsp;&emsp;紧接着上一篇，上一篇中登录验证都由security帮助我们完成了，如果我们想要增加一个验证码登录或者其它的自定义校验就没办法了，因此这一篇讲解如何实现这个功能。

##一、 实现自定义登录校验类

&emsp;&emsp;继承UsernamePasswordAuthenticationFilter类来拓展登录校验，代码如下：
```java
public class MyUsernamePasswordAuthentication extends UsernamePasswordAuthenticationFilter{
	
	private Logger log = LoggerFactory.getLogger(this.getClass());

	@Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException {
		//我们可以在这里进行额外的验证，如果验证失败抛出继承AuthenticationException的自定义错误。
		log.info("在这里进行验证码判断");
        //只要最终的验证是账号密码形式就无需修改后续过程
		return super.attemptAuthentication(request, response);
	}

	@Override
	public void setAuthenticationManager(AuthenticationManager authenticationManager) {
		// TODO Auto-generated method stub
		super.setAuthenticationManager(authenticationManager);
	}
}
```

##二、 将自定义登录配置到security中
&emsp;&emsp;编写自定义登录过滤器后，configure Bean修改为如下：
```java
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http
		.csrf() //跨站
		.disable() //关闭跨站检测
        //自定义鉴权过程，无需下面设置
		.authorizeRequests()//验证策略
			.antMatchers("/public/**").permitAll()//无需验证路径
           .antMatchers("/user/**").permitAll()
           .antMatchers("/login").permitAll()//放行登录
			.antMatchers(HttpMethod.GET, "/user").hasAuthority("getAllUser")//拥有权限才可访问
			.antMatchers(HttpMethod.GET, "/user").hasAnyAuthority("1","2")//拥有任一权限即可访问
			//角色类似，hasRole(),hasAnyRole()
			.anyRequest().authenticated()
		.and()
        //自定义异常处理
		.exceptionHandling()
            .authenticationEntryPoint(myAuthenticationEntryPoint)//未登录处理
			.accessDeniedHandler(myAccessDeniedHandler)//权限不足处理
		.and()
        //加入自定义登录校验
        .addFilterBefore(myUsernamePasswordAuthentication(),UsernamePasswordAuthenticationFilter.class)
        .rememberMe()//默认放在内存中
            .rememberMeServices(rememberMeServices())
            .key("INTERNAL_SECRET_KEY")
//       重写usernamepasswordauthenticationFilter后，下面的formLogin()设置将失效，需要手动设置到个性化过滤器中
//        .and()
//		.formLogin()
//			.loginPage("/public/unlogin") //未登录跳转页面,设置了authenticationentrypoint后无需设置未登录跳转页面
//			.loginProcessingUrl("/public/login")//登录api
//            .successForwardUrl("/success")
//            .failureForwardUrl("/failed")
//            .usernameParameter("id")
//            .passwordParameter("password")
//			.failureHandler(myAuthFailedHandle) //登录失败处理
//			.successHandler(myAuthSuccessHandle)//登录成功处理
//            .usernameParameter("id")
		.and()
		.logout()//自定义登出
			.logoutUrl("/public/logout")
            .logoutSuccessUrl("public/logoutSuccess")
			.logoutSuccessHandler(myLogoutSuccessHandle);
	}
```
然后再编写Bean，代码如下：
```java
@Bean
public MyUsernamePasswordAuthentication myUsernamePasswordAuthentication(){
    MyUsernamePasswordAuthentication myUsernamePasswordAuthentication = new MyUsernamePasswordAuthentication();
    myUsernamePasswordAuthentication.setAuthenticationFailureHandler(myAuthFailedHandle); //设置登录失败处理类
    myUsernamePasswordAuthentication.setAuthenticationSuccessHandler(myAuthSuccessHandle);//设置登录成功处理类
    myUsernamePasswordAuthentication.setFilterProcessesUrl("/public/login");
    myUsernamePasswordAuthentication.setRememberMeServices(rememberMeServices()); //设置记住我
    myUsernamePasswordAuthentication.setUsernameParameter("id");
    myUsernamePasswordAuthentication.setPasswordParameter("password");
    return myUsernamePasswordAuthentication;
}
```
完成。