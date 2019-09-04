---
layout: single
classes: wide
title:  "Zuul API Gateway with Spring Session (Redis): Authenticate and Route in same request with Apache module Mellon (SAML)"
description: 
date:   2019-06-28 14:32:49 +0200
excerpt_separator: <!--more-->
#header:
#  teaser: /assets/images/Ska.png

---
My APIGateway (Zuul) is proxied by Apache Httpd and protected by [Mellon module][1] (SAML 2.0). After a successfully authentication on the identity provider, mellon module inject correctly some headers read into the SAML response, but the first request fails with a 403 status code. 
<!--more-->
I'm also using SpringSecurity. To solve the problem I'm using a simple filter added on the security filter chain that ensure the correct creation of SecurityContext:

```java
@Component
public class MellonFilter extends OncePerRequestFilter {

    private final Logger log = LoggerFactory.getLogger(MellonFilter.class);


    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse httpServletResponse, FilterChain filterChain) throws ServletException, IOException {
 
       String mellonId=req.getHeader("mellon-nameid");

        if(mellonId==null||mellonId.isEmpty())
            ;//do filterchain
        else {
            
            UserWithRoles userWithRoles = new UserWithRoles();
            userWithRoles.setUsername(mellonId);
            SilUserDetails details = new SilUserDetails(userWithRoles);

            SilAuthenticationPrincipal silPrincipal = null;
            Collection<SimpleGrantedAuthority> authorities = new ArrayList<>();

            authorities.add(new SimpleGrantedAuthority("Some roles");

            silPrincipal = new SilAuthenticationPrincipal(details, true, authorities);
            SecurityContextHolder.clearContext();
            SecurityContextHolder.getContext().setAuthentication(silPrincipal);
        }
        filterChain.doFilter(req,httpServletResponse);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        if(SecurityContextHolder.getContext().getAuthentication()!=null&&SecurityContextHolder.getContext().getAuthentication() instanceof SilAuthenticationPrincipal)
            return true;
        return false;

    }
}
```

Then I need a ZuulFilter to save the session (on Redis) and to propagate the actual session id:


```java
public class ZuulSessionCookieFilter extends ZuulFilter {

    private final Logger log = LoggerFactory.getLogger(ZuulSessionCookieFilter.class);

    @Autowired
    private SessionRepository repository;

    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {

        return true;
    }

    @Override
    public Object run() throws ZuulException {

        RequestContext context = RequestContext.getCurrentContext();

        HttpSession httpSession = context.getRequest().getSession();
        httpSession.setAttribute(
                HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
                SecurityContextHolder.getContext()
        );
        Session session = repository.findById(httpSession.getId());
        context.addZuulRequestHeader("cookie", "SESSION=" + base64Encode(httpSession.getId()));
        log.debug("ZuulPreFilter session proxy: {} and {}", session.getId(),httpSession.getId());

        return null;
    }

    private static String base64Encode(String value) {
        byte[] encodedCookieBytes = Base64.getEncoder().encode(value.getBytes());
        return new String(encodedCookieBytes);
    }
}
```

I hope this solution will be helpful to everyone.


  [1]: https://github.com/Uninett/mod_auth_mellon