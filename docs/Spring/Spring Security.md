#### httpBasic
- 这是最基本的模式，你在请求的时候如果没有登录那么前端将弹出一个系统自带的登录框
```
//httpBasic认证模式
http.httpBasic()
    .and()
    .authorizeRequests()
    //任何请求都需要登录
    .anyRequest().authenticated();
```

### formLogin
##### 背景介绍
- login.html登录页面 (/login登录处理)->不需要
- 登录成功进入index.html(通过/index处理视图)
- index页面有四个链接（/biz1,/biz2,/syslog,/sysuser）
- 四个链接对应四个html页面，admin角色可以全部访问，user角色只可以访问/biz1,/biz2
##### code

```
            http.formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/login")
                .defaultSuccessUrl("/index")
                .and()
                .authorizeRequests()
                .antMatchers("/login.html", "/login").permitAll()
                .antMatchers("/biz1", "biz2").hasAnyAuthority("ROLE_admin", "ROLE_user")
                .antMatchers("/syslog", "/sysuser").hasAnyRole("admin")
                //可以通过权限ID配置 而不是角色
                //.antMatchers("/syslog").hasAnyAuthority("sys:log")
                .anyRequest().authenticated()
                .csrf().disable();//取消跨站安全处理 

//至于在哪里给用户设置角色
//可以重写configure(AuthenticationManagerBuilder auth)方法 
//在内存中定义两个账户和角色[后面我们从数据库中读取]
@Override
p rotected void configure(AuthenticationManagerBuilder auth) throws Exception {
        final BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        auth.inMemoryAuthentication()
                .withUser("user")
                .password(passwordEncoder.encode("123456"))
                .roles("user")
                //可以不给角色 给权限ID
                //.authorities("sys:log","sys:user")
                .and()
                .withUser("admin")
                .password(passwordEncoder.encode("123456"))
                .roles("admin")
                .and()
                .passwordEncoder(passwordEncoder);
    }
    
//将项目的静态资源开放出来     
@Override
public void configure(WebSecurity web) throws Exception {
    web.ignoring()
        .antMatchers("/css/**", "/img/**", "/js/**");
}
```
##### session管理及安全

```
//会话超时配置
//最小值一分钟 配置少于一分钟也会被强制成一分钟
server:
  servlet:
    session:
      timeout: 10s
spring:
  session:
    timeout: 10s
    
    
http.sessionManagement()
        .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
        .invalidSessionUrl("/login.html");
 
 //session安全管理策略 默认 每次请求都会跟换sessionid
        .sessionFixation().migrateSession();
        
//保证cookie的安全
server:
  servlet:
    session:
        cookie:
            http-only: true # 本地js无法读取cookie内容
            secure: true    #只有HTTPS可以携带cookie
// 限制最大登录用户数量，实现SessionInformationExpiredStrategy接口
            http.sessionManagement()
                .maximumSessions(1) //只允许一个用户登录
                .maxSessionsPreventsLogin(false) //同一个账号两个地方登录时 踢出第一地方的账号 true就是第二个地方登录不上
                .expiredSessionStrategy(new MyExpiredSessionStrategy()); 
/**
 * @author sulin
 */
public class MyExpiredSessionStrategy implements SessionInformationExpiredStrategy {
    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        final Api message = Api.builder().code(400).message("你的账号在其他的地方登录，被迫下线").build();
        ResJsonUtil.write(event.getResponse(), message);
    }
}
```
#####  Role-Based Access Control
- 用户：系统接口以及访问的操作者
- 权限：能后访问某接口或者某操作的授权资格
- 角色：具有一类相同操作权限的用户的总称

#### 从数据库动态加载权限
- 核心接口实现UserDetails和UserDetailsService

```java

@Service
public class MyUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //我先造点假数据
        System.out.println("param->username:" + username);

        //加载基础用户信息
        final MyUserDetails myUserDetails = new MyUserDetails();
        myUserDetails.setUsername(username);
        myUserDetails.setPassword(new BCryptPasswordEncoder().encode("123456"));
        myUserDetails.setEnabled(true);
        //加载角色 read databases; 角色是一个特殊权限 需要加入ROLE_前缀
        List<String> roles = new ArrayList<>();
        //加载用户的资源权限列表
        List<String> authorities = new ArrayList<>();
        roles = roles.stream().map(x -> "ROLE_" + x).collect(Collectors.toList());
        authorities.addAll(roles);

        myUserDetails.setAuthorities(AuthorityUtils.commaSeparatedStringToAuthorityList(String.join(",",authorities)));
        return myUserDetails;
    }
}

@Data
public class MyUserDetails implements UserDetails {


    private Collection<? extends GrantedAuthority> authorities;

    private String password;

    private String username;

    private boolean enabled;

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
}

   @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //内存模拟用户
        final BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
        auth.userDetailsService(myUserDetailsService).passwordEncoder(bCryptPasswordEncoder);
    }
```


#### 动态加载资源鉴权规则
``` java
//写一个类
@Service("rbacService")
public class MyRBACService {

    /**
     * spring security提供的工具类
     */
    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    /**
     * 判断某用户是否具有改request资源的权限
     *
     * @param request
     * @param authentication
     * @return
     */
    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {
        final Object principal = authentication.getPrincipal();
        if (principal instanceof UserDetails) {
            UserDetails userDetails = (UserDetails) principal;
            final String username = userDetails.getUsername();
            //通过username 查询数据库的对应的资源权限
            //read database;
            List<String> urls = null;
            return urls.stream().anyMatch(url -> antPathMatcher.match(url, request.getRequestURI()));
        }
        return false;
    }
}

//然后在http配置
 http.authorizeRequests()
        .antMatchers("/doLogin").permitAll()
        .antMatchers("/index").authenticated()
        .anyRequest().access("@rbacService.hasPermission(request,authentication)")
```
#### 权限表达式使用方法总结
- 打开方法上的注解@EnableGlobalMethodSecurity(prePostEnabled = true)
- @PreAuthorize("hasRole('admin')") 当前用户角色为admin的时候才能访问(注解含义：在这个方法执行前进行权限校验)
- @PostAuthorize(returnObject.name==authentication.name) 在方法调用后进行判定
- @PreFilter(filterTarget="ids",value="filterObject%2==0") 对指定参数进行规制匹配 会把不匹配的资源进行过滤
- @PostFilter 针对返回值过滤



