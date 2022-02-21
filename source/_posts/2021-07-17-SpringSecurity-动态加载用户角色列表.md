---
layout: post
title: SpringSecurity 动态加载用户角色列表
date: 2021-07-17
categories: 技术
tags: SpringSecurity
---

### SpringSecurity 动态加载用户角色列表

通过实现 AccessDecisionManager 接口和 FilterInvocationSecurityMetadataSource 接口

- 实现 AccessDecisionManager 接口

~~~java

@Component
public class CustomAccessDecisionManager implements AccessDecisionManager {
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {

        for(ConfigAttribute configAttribute : collection) {
            //如果你请求的url在数据库中不具备角色，即不存在限制
            if("ROLE_def".equals(configAttribute.getAttribute())) {
                //匿名用户拒绝访问
                if(authentication instanceof AnonymousAuthenticationToken) {
                    throw new AccessDeniedException("权限不足，无法访问");
                } else {
                    //登录用户放行
                    return;
                }
            }
            //如果你访问的路径在数据库中存在角色
            Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
            for (GrantedAuthority authority : authorities) {
                //查看用户是否拥有该权限,如果有则直接放行
                if(authority.getAuthority().equals(configAttribute.getAttribute()))
                    return;
            }
        }
        //登录却没有相应权限
        throw new AccessDeniedException("权限不足，无法访问");
    }

    @Override
    public boolean supports(ConfigAttribute configAttribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
~~~

- 实现 AccessDecisionManager 接口

~~~java

@Component
public class CustomFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {

    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    private UserRoleService userRoleService;

    @Autowired
    public CustomFilterInvocationSecurityMetadataSource(UserRoleService userRoleService) {
        this.userRoleService = userRoleService;
    }

    @Override
    public Collection<ConfigAttribute> getAttributes(Object obj) throws IllegalArgumentException {
        //获取当前用户请求的url
        String requestUrl = ((FilterInvocation) obj).getRequestUrl();
        //数据库中查询出所有路径
        List<Role> roleList = userRoleService.getAllRole();
        List<String> roles = new ArrayList<>();
        for(Role role : roleList) {
            if(antPathMatcher.match(role.getPath(),requestUrl)) {
                roles.add(role.getName());
            }
        }
        String[] roleStr = new String[roles.size()];
        roles.toArray(roleStr);
        if(roles.size() != 0)
            return SecurityConfig.createList(roleStr);
        else
            return SecurityConfig.createList("ROLE_def");
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}

~~~

- 配置Spring Security

~~~ java
           http
                    //动态获取角色权限
                    .authorizeRequests()
                    .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                        @Override
                        public <O extends FilterSecurityInterceptor> O postProcess(O obj) {
                            obj.setSecurityMetadataSource(customFilterInvocationSecurityMetadataSource);
                            obj.setAccessDecisionManager(customAccessDecisionManager);
                            return obj;
                        }
                    })
~~~



