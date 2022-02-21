---
layout: post
title: Spring Security JWT(Json Web Token) 登录实现
date: 2021-07-20
categories: 技术
tags: SpringSecurity
---

### Spring Security JWT(Json Web Token) 登录实现

使用 Spring Security 实现 JWT 登录，我们只需要在 Spring Security 的众多 Filter 中添加一个我们用于 JWT 登录的 Filter，以下是一个demo

~~~java
// JwtAuthenticationFilter.java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String tokenHeader = request.getHeader(JwtTokenUtils.TOKEN_HEADER);
        if(null == tokenHeader || !tokenHeader.startsWith(JwtTokenUtils.TOKEN_PREFIX)) {
            filterChain.doFilter(request, response);
            return;
        }
        try {
            String token = tokenHeader.replace(JwtTokenUtils.TOKEN_PREFIX, "");
            String username = JwtTokenUtils.getUserName(token);
            List<Role> roleList = JwtTokenUtils.getUserRole(token);
            List<GrantedAuthority> roles = new ArrayList<GrantedAuthority>();
            roleList.forEach(item -> {
                roles.add(new GrantedAuthority() {
                    @Override
                    public String getAuthority() {
                        return item.getName();
                    }
                });
            });
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(username, null, roles);
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        } catch (Exception e) {
            logger.error("无法验证令牌");
        }
        filterChain.doFilter(request, response);
    }
}
~~~

~~~java
// JwtAuthenticationEntryPoint.java 用来处理认证失败的情况
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) throws IOException, ServletException {
        RestUtil.response(response, SystemCode.AccessTokenError);
    }
}
~~~

~~~java
// JwtAccessDeniedHandler.java 用来处理权限不足的情况
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException, ServletException {
        RestUtil.response(httpServletResponse, SystemCode.AccessDenied);
    }
}
~~~

完成以上，我们只需要把这些配置进 Spring Security 就可以了。

~~~java
// SecurityConfig.java 
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Configuration
    public static class SecurityConfigurerAdapter extends WebSecurityConfigurerAdapter {
              @Override
        protected void configure(HttpSecurity http) throws Exception {
            //网页中允许使用 iframe 打开页面
            http.headers().frameOptions().disable();
          //关闭 Session， 既然我们使用 JWT 进行鉴权，也就不再需要 Session 保持回话鉴权了，当然此处视情况而定
            http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);

            http
                    .addFilterBefore(new JwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
                    .exceptionHandling().authenticationEntryPoint(jwtAuthenticationEntryPoint).accessDeniedHandler(jwtAccessDeniedHandler)
              //跨域限制关闭， 后期可以自己进行设置
                    .and().csrf().disable()
                    .cors();
        }
    }
}
~~~

至此，我们已经完成了 Spring Security 中使用 JWT 的基本操作了。

