---
title: SpringBoot整合Shrio
date: 2020-06-27 21:13:00
categories: 
- 项目
---

# SpringBoot 整合 shiro 前后端分离

最近看了个 github 的开源项目,里面就使用到了shiro实现前后端分离权限认证,写的挺好的,并且现在也刚刚学完Vue,就那这个项目练练手了。我会将写这个项目的经验分享出来。

虽然自己还很菜,当仍在努力中!!    附github的开源项目的演示地址: http://www.zykhome.club/#/login

# 1.导入依赖,编写工具类

## Mavne 依赖 

```xml
<!-- 前后端分离 采用 jwt生成token-->
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.2.0</version>
</dependency>

<!-- shiro依赖-->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.3.2</version>
</dependency>
```

## JWTUtils 工具类

```java
public class JWTUtils {
    /**
     * 过期时间 单位 毫秒
     */
    private static  long EXPIRE_TIME;

    //初始化 token过期时间 默认 6小时
    static{
        try { //这里是从 application.properties内读取的配置  也可以直接写个产量
            EXPIRE_TIME =Integer.parseInt(PropertiesUtils.getConfig("token.expireTime"));
        } catch (NumberFormatException e) {
            EXPIRE_TIME = 1000*60*60*6;
        }
    }

    /**
     * 校验token是否正确
     * @param token 密钥
     * @param password 用户的密码
     * @return 是否正确
     */
    public static boolean verify(String token, String username, String password) {
        try {
            Algorithm algorithm = Algorithm.HMAC256(password);
            JWTVerifier verifier = JWT.require(algorithm)
                    .withClaim("username", username)
                    .build();
            DecodedJWT jwt = verifier.verify(token);
            return true;
        } catch (Exception exception) {
            return false;
        }
    }

    /**
     * 获得token中的信息无需secret解密也能获得
     * @return token中包含的用户名
     */
    public static String getUsername(String token) {
        try {
            DecodedJWT jwt = JWT.decode(token);
            return jwt.getClaim("username").asString();
        } catch (JWTDecodeException e) {
            return null;
        }
    }

    /**
     * 生成签名
     * @param username 用户名
     * @param password 用户的密码
     * @return 加密的token
     */
    public static String sign(String username, String password) {
        try {
            //设置过期时间
            Date date = new Date(System.currentTimeMillis()+EXPIRE_TIME);
            //加密密码
            Algorithm algorithm = Algorithm.HMAC256(password);
            // 附带username信息
            return JWT.create()
                    .withClaim("username", username)
                    .withExpiresAt(date)
                    .sign(algorithm);
        } catch (UnsupportedEncodingException e) {
            return null;
        }
    }

    /**
     * 判断过期
     * @param token
     * @return
     */
    public static boolean isExpire(String token){
        DecodedJWT jwt = null;
        try {
            jwt = JWT.decode(token);
        } catch (JWTDecodeException e) {
           return true;
        }
        return System.currentTimeMillis()>jwt.getExpiresAt().getTime();
    }
}
```

## MD5 加密工具类

```java
public class MD5Utils {
    /**
     * 密码加密
     */
    public static String md5Encryption(String source,String salt){
        String algorithmName = "MD5";//加密算法
        int hashIterations = 1024;//加密次数
        SimpleHash simpleHash = new SimpleHash(algorithmName,source,salt,hashIterations);
        return simpleHash+"";
    }
}
```



# 2 开始整合 

##    自定义 Realm

```java
@Component
public class UserRealm extends AuthorizingRealm {
	//这里同 concurrentHashMap简单实现缓存, 项目中建议将用户权限信息放入redis缓存中
    private static final Map<String,ActiveUser> userCache = new ConcurrentHashMap();

    @Autowired
    private UserService userService; //这里是我自己项目中的 userService 

    //必须重写该方法 否则 用自定义token 会不匹配
    @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof JwtToken;
    }

    // 授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {

        ActiveUser activeUser = (ActiveUser) SecurityUtils.getSubject().getPrincipal();

        List<Role> roles = activeUser.getRoles();
        List<Menu> menus = activeUser.getMenus();

        activeUser.setRoles(roles);
        activeUser.setMenus(menus);

        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();

        for(Role temp : roles){
            String roleName = temp.getRoleName();
            info.addRole(roleName);
        }

        for(Menu menu : menus){
            String perms = menu.getPerms();
            info.addStringPermission(perms);
        }
        return info;
    }

    //认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        //获取token(在登录接口会生成,往后看就能明白)
        String token = (String)authenticationToken.getPrincipal();
        //如果token为null 抛出异常 交给统一异常处理类来处理异常返回结果(@ExceptionHandler)
        if(token==null){
            throw new AuthenticationException("请登陆");
        }
		//解析token 获取用户名
        String username = JWTUtils.getUsername(token);
		//如果解析失败
        if(username==null){
            throw new AuthenticationException("Token 解析错误");
        }
		//通过用户名查询用户信息
        User user = userService.getUserByUserName(username);
        //如果查询不到,说明用户不存在
        if(user==null){
            throw new UnknownAccountException("账户不存在");
        }
		//这里根据项目的业务逻辑进行配置
        if(user.getStatus()==1){
            throw new LockedAccountException("账号被锁定");
        }
		//判断token是否过期
        if(JWTUtils.isExpire(token)){
            throw new AccountException("token已过期,请重新登陆");
        }
        //验证
        if(!JWTUtils.verify(token,username,user.getPassword())){
            throw new CredentialsException("密码错误");
        }
        //先从缓存内获取用户信息 可换成redis 逻辑是一样的
        ActiveUser cacheUser = userCache.get(user.getUsername());

        //如果缓存不为null 直接返回
        if(cacheUser!=null){
            SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(cacheUser,token,getName());
            return info;
        }
		//查询数据库
        ActiveUser activeUser = new ActiveUser();
        activeUser.setUsername(user.getUsername());
        activeUser.setUserId(user.getId());
        activeUser.setUser(user);

        //获取用户角色和权限
        Long userId = user.getId();

        List<Role> roles = userService.getUserRoleById(userId);
        List<Menu> menus = userService.getUserMenuById(userId);

        activeUser.setRoles(roles);
        activeUser.setMenus(menus);

        //放入缓存
        userCache.put(user.getUsername(),activeUser);
        SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(activeUser,token,getName());
		//返回 SimpleAuthenticationInfo 交给shiro处理 底层调用equals判断
        return info;
    }
}
```

## 自定义  Token 

```java
public class JwtToken implements AuthenticationToken {
    //自定义简单实现即可,主要功能是为了验证时的数据交互
    private String token;

    public JwtToken(String token) {
        this.token = token;
    }

    @Override
    public Object getPrincipal() {
        return token;
    }

    @Override
    public Object getCredentials() {
        return token;
    }
}
```



## 自定义 Filter

```java
public class JwtFilter extends AccessControlFilter {


    @Override  //在访问前判断 如果 返回true 直接放行 允许访问
    protected boolean isAccessAllowed(ServletRequest servletRequest, ServletResponse servletResponse, Object o) throws Exception {
        Subject subject = SecurityUtils.getSubject();

        HttpServletRequest request = (HttpServletRequest)servletRequest;
        String token = request.getHeader("token");
        if(token ==null || JWTUtils.isExpire(token)) return false;

        return subject.isAuthenticated();
    }

    @Override  //当上面方法返回false 会调用 onAccessDenied 进行验证 如果返回true 也放行 
    protected boolean onAccessDenied(ServletRequest servletRequest, ServletResponse servletResponse) throws Exception {
        System.out.println("验证不通过再次验证==>");
        HttpServletRequest request = (HttpServletRequest)servletRequest;
        
        String token = request.getHeader("token");
        JwtToken jwtToken = new JwtToken(token);

        Subject subject = SecurityUtils.getSubject();

        //登陆
        try {
            subject.login(jwtToken);
        } catch (ShiroException e) {
            e.printStackTrace();
            System.out.println( e.getLocalizedMessage());
            outputError((HttpServletResponse)servletResponse,e);
        }
        return subject.isAuthenticated();
    }

    /**
     * 对跨域提供支持 防止跨域时获取不到 token请求头
     */
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        httpServletResponse.setHeader("Access-control-Allow-Origin", httpServletRequest.getHeader("Origin"));
        httpServletResponse.setHeader("Access-Control-Allow-Methods", "GET,POST,OPTIONS,PUT,DELETE");
        httpServletResponse.setHeader("Access-Control-Allow-Headers", httpServletRequest.getHeader("Access-Control-Request-Headers"));
        // 跨域时会首先发送一个option请求，这里我们给option请求直接返回正常状态
        if (httpServletRequest.getMethod().equals(RequestMethod.OPTIONS.name())) {
            httpServletResponse.setStatus(HttpStatus.OK.value());
            return false;
        }
        return super.preHandle(request, response);
    }

	//用于输出异常 只是一个普通方法
    public void outputError(HttpServletResponse response, Exception e){
        response.setContentType("application/json; charset=utf-8");

        try (PrintWriter writer = response.getWriter()){
            ObjectMapper objectMapper = new ObjectMapper();
            writer.write(objectMapper.writeValueAsString(ResponseBean.error(4001,e.getMessage())));
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }

}
```

## Shiro配置

```java
@Configuration
public class ShiroConfig {

    @Bean  
    public DefaultWebSecurityManager webSecurityManager(@Autowired UserRealm userRealm){
        //配置 自定义的realm
        DefaultWebSecurityManager manager = new DefaultWebSecurityManager();
        manager.setRealm(userRealm);
        
        //禁用session
        DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
        DefaultSessionStorageEvaluator defaultSessionStorageEvaluator = new DefaultSessionStorageEvaluator();
        defaultSessionStorageEvaluator.setSessionStorageEnabled(false);
        subjectDAO.setSessionStorageEvaluator(defaultSessionStorageEvaluator);
        manager.setSubjectDAO(subjectDAO);
        return manager;
    }

    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(@Autowired DefaultWebSecurityManager defaultWebSecurityManager){
        //配置过滤器
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        bean.setSecurityManager(defaultWebSecurityManager);

        Map filters = new HashMap();
        //注册自己的filter 进行验证 
        filters.put("myFilter",new JwtFilter());

        bean.setFilters(filters);

        Map chainMap = new LinkedHashMap<>();
        chainMap.put("/swagger-ui.html", "anon");
        chainMap.put("/webjars/**","anon");
        chainMap.put("/swagger-resources/**","anon");
        chainMap.put("/v2/**","anon");
        chainMap.put("/404.html","anon");
        chainMap.put("/user/login", "anon");
        chainMap.put("/**", "myFilter");//使用我们自定义的filter过滤器
	
        bean.setFilterChainDefinitionMap(chainMap);
        return bean;
    }

}
```



# 3登录接口

```java
@PostMapping("/login")
    public ResponseBean login(HttpServletRequest request, @Validated String username,  String password){

        log.info("用户:{}开始登陆",username);
        User user = userService.getUserByUserName(username);
        if(user == null){
            throw new UnknownAccountException("账户不存在");
        }
        String token = JWTUtils.sign(username, MD5Utils.md5Encryption(password, user.getSalt()));
        JwtToken jwtToken = new JwtToken(token);
        Subject subject = SecurityUtils.getSubject();
        subject.login(jwtToken);

        userService.addLoginLog(request);
        if(subject.isAuthenticated()){
            ResponseBean bean = ResponseBean.success("登陆成功");
            bean.setData(token);
            log.info("用户:{}登录成功",username);
            return bean;
        }
        return ResponseBean.error("登陆失败");
    }
```