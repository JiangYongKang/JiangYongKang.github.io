---
title: Spring Boot 中使用 Spring Security + JWT 构建身份认证系统
date: 2018.08.28 17:23:42
tags: SpringBoot
categories: 服务端开发
---

权限控制是非常常见的功能，在各种后台管理里权限控制更是重中之重。在 Spring Boot 中使用 Spring Security 构建权限系统是非常轻松和简单的。Spring Security 是一个能够为基于 Spring 的企业应用系统提供声明式的安全访问控制解决方案的安全框架。它提供了一组可以在 Spring 应用上下文中配置的 Bean，为应用系统提供声明式的安全访问控制功能，减少了为企业系统安全控制编写大量重复代码的工作。

<!-- more -->

### 添加依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### UserDetails
按照官方文档的说法，为了定义我们自己的认证管理，我们可以添加 `UserDetailsService`, `AuthenticationProvider`, `AuthenticationManager` 这种类型的 Bean。实现的方式有多种，这里我选择最简单的一种（因为本身我们这里的认证授权也比较简单）通过定义自己的 `UserDetailsService` 从数据库查询用户信息，至于认证的话就用默认的。
```java
/**
 * Author: vincent
 * Date: 2018-06-12 16:50:00
 * Comment:
 */

public class User implements UserDetails {

    private Integer id;
    private String username;
    private String password;
    private Date lastPasswordResetDate;
    private List<Role> roles = new ArrayList<>();

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @JsonIgnore
    public List<Role> getRoles() {
        return roles;
    }

    public void setRoles(List<Role> roles) {
        this.roles = roles;
    }

    @Override
    @JsonIgnore
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return roles.stream().map(Role::getName).map(SimpleGrantedAuthority::new).collect(Collectors.toList());
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    @JsonIgnore
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    @JsonIgnore
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    @JsonIgnore
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    @JsonIgnore
    public boolean isEnabled() {
        return true;
    }

    @JsonIgnore
    public Date getLastPasswordResetDate() {
        return lastPasswordResetDate;
    }

    public void setLastPasswordResetDate(Date lastPasswordResetDate) {
        this.lastPasswordResetDate = lastPasswordResetDate;
    }
}
```

### 开启权限验证
```java
/**
 * Author: vincent
 * Date: 2018-06-12 17:02:00
 * Comment:
 * @EnableWebSecurity 开启权限验证注解
 * @EnableGlobalMethodSecurity(prePostEnabled = true) 开启方法级别的权限验证
 */

@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 因为我们实现了 UserDetailsService，Spring 会自动在项目中查找它的实现类，
     * 这里注入的也自然是我们自定义的 UserDetailsServiceImpl 类
     */
    @Resource
    private UserDetailsService userDetailsService;

    @Resource
    private AuthenticationTokenFilter authenticationTokenFilter;

    /**
     * 设置我们自己实现的的 UserDetailsService，并设置密码的加密方式，加密方式不是必须的，也就是说，这里是可以明文密码
     * @param authenticationManagerBuilder
     * @throws Exception
     */
    @Override
    public void configure(AuthenticationManagerBuilder authenticationManagerBuilder) throws Exception {
        authenticationManagerBuilder.userDetailsService(this.userDetailsService).passwordEncoder(passwordEncoder());
    }

    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        // 因为我们是基于 token 进行验证的，所以 csrf 和 session 我们这里都不需要
        httpSecurity.csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .authorizeRequests()

                // 获取 token 的接口所有人都可以访问
                .antMatchers("/authentication/**").permitAll()

                // 除了上面的接口，其它接口都必须鉴权认证
                .anyRequest().authenticated();
        httpSecurity.headers().cacheControl();

        // 将我们实现的过滤器添加进去，方法名可以很轻松的看出来是前置过滤
        httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 集成 JWT
想要将 JWT（Json Web Token） 在 spring boot 中集成起来，需要创建一个 filter，继承自 OncePerRequestFilter。
并且将它添加到 WebSecurityConfig，需要注意的是，这个过滤器是作为权限验证，必须添加到 before 集合中。
```java
/**
 * Author: vincent
 * Date: 2018-06-12 17:13:00
 * Comment:
 */

@Component
public class AuthenticationTokenFilter extends OncePerRequestFilter {

    @Resource
    private UserDetailsService userDetailsService;

    @Resource
    private TokenUtil tokenUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String authorization = request.getHeader(AUTHORIZATION);
        if (!StringUtils.isEmpty(authorization) && authorization.startsWith(TokenUtil.TOKEN_PREFIX)) {
            String token = authorization.substring(TokenUtil.TOKEN_PREFIX.length());
            String username = tokenUtil.getUsernameFromToken(token);
            if (!StringUtils.isEmpty(username) && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
                if (tokenUtil.validateToken(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities()
                    );
                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

```java
/**
 * Author: vincent
 * Date: 2018-06-12 17:17:00
 * Comment:
 */

@Component
public class TokenUtil implements Serializable {
    public static final String TOKEN_PREFIX = "Bearer ";

    static final String CLAIM_KEY_USERNAME = "sub";
    static final String CLAIM_KEY_CREATED = "iat";

    private static final long serialVersionUID = -3301605591108950415L;

    private Clock clock = DefaultClock.INSTANCE;

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
    }

    public Date getIssuedAtDateFromToken(String token) {
        return getClaimFromToken(token, Claims::getIssuedAt);
    }

    public Date getExpirationDateFromToken(String token) {
        return getClaimFromToken(token, Claims::getExpiration);
    }

    public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = getAllClaimsFromToken(token);
        return claimsResolver.apply(claims);
    }

    private Claims getAllClaimsFromToken(String token) {
        return Jwts.parser()
                .setSigningKey(secret)
                .parseClaimsJws(token)
                .getBody();
    }

    private Boolean isTokenExpired(String token) {
        final Date expiration = getExpirationDateFromToken(token);
        return expiration.before(clock.now());
    }

    private Boolean isCreatedBeforeLastPasswordReset(Date created, Date lastPasswordReset) {
        return (lastPasswordReset != null && created.before(lastPasswordReset));
    }

    private Boolean ignoreTokenExpiration(String token) {
        return false;
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return doGenerateToken(claims, userDetails.getUsername());
    }

    private String doGenerateToken(Map<String, Object> claims, String subject) {
        final Date createdDate = clock.now();
        final Date expirationDate = calculateExpirationDate(createdDate);

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(createdDate)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }

    public Boolean canTokenBeRefreshed(String token, Date lastPasswordReset) {
        final Date created = getIssuedAtDateFromToken(token);
        return !isCreatedBeforeLastPasswordReset(created, lastPasswordReset)
                && (!isTokenExpired(token) || ignoreTokenExpiration(token));
    }

    public String refreshToken(String token) {
        final Date createdDate = clock.now();
        final Date expirationDate = calculateExpirationDate(createdDate);

        final Claims claims = getAllClaimsFromToken(token);
        claims.setIssuedAt(createdDate);
        claims.setExpiration(expirationDate);

        return Jwts.builder()
                .setClaims(claims)
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }

    public Boolean validateToken(String token, UserDetails userDetails) {
        if (userDetails instanceof User) {
            User user = (User) userDetails;
            final String username = getUsernameFromToken(token);
            final Date created = getIssuedAtDateFromToken(token);
            return (
                    username.equals(user.getUsername())
                            && !isTokenExpired(token)
                            && !isCreatedBeforeLastPasswordReset(created, user.getLastPasswordResetDate())
            );
        } else {
            System.out.println("类型转换失败！");
            return false;
        }
    }

    private Date calculateExpirationDate(Date createdDate) {
        return new Date(createdDate.getTime() + expiration * 1000);
    }
}
```

### UserDetailsService

```java
/**
 * Author: vincent
 * Date: 2018-06-12 16:49:00
 * Comment: 这个接口只定义了一个方法 loadUserByUsername，顾名思义，就是提供一种从用户名可以查到用户并返回的方法。
 * 注意，不一定是数据库，文本文件、xml文件等等都可能成为数据源，这也是为什么 Spring 提供这样一个接口的原因：保证你可以采用灵活的数据源。
 */

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Resource
    private UserMapper userMapper;

    @Resource
    private RoleMapper roleMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userMapper.findByUsername(username);
        if (user == null)
            throw new UsernameNotFoundException("Cannot find user with username, username = " + username);
        user.setRoles(roleMapper.selectByUserId(user.getId()));
        return user;
    }
}
```
