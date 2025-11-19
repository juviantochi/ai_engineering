## Partner Filter Ordering Strategy

Goal: Allow multiple partners to share `/fs/brick/service/**` while keeping each partner’s filter independent, and ensure no unauthenticated request reaches the controller.

### Ordering Layout
1. `@Order(3)` – Veefin filter  
   - Checks both path *and* Veefin-specific headers.  
   - If path matches `/fs/brick/service/veefin/**` **and** headers exist → validate signature.  
   - If path matches but headers are missing → reject 401 (Veefin request without signature).  
   - If path doesn’t match, or headers are absent, simply skip down the chain so other partners can try.

2. `@Order(4)` – Partner B filter  
   - Same logic, but keyed on Partner B’s path/header combo.

3. … repeat for additional partners …

4. `@Order(n+1)` – Catch-all partner guard  
   - Dynamically derives the protected patterns by aggregating `pathPatterns` from every partner’s configuration (no need for a duplicate list).  
   - If the request URI matches any of those collected patterns **and** no earlier partner filter set `request.setAttribute("partner.authenticated", true)`, return 401 (“Missing partner authentication”).  
   - Otherwise fall through to controllers.

### Implementation Notes
- Each partner filter, after successful verification, sets an attribute (`request.setAttribute("partner.authenticated", true)` and maybe `partnerId`) so the catch-all knows the request was authenticated.
- The catch-all filter can reuse Spring’s `AntPathMatcher` and obtain its pattern list directly from the partner configs (e.g., `veefin.pathPatterns`, `partnerB.pathPatterns`). Example:
  ```java
  @Component
  @Order(Ordered.LOWEST_PRECEDENCE - 10)
  public class PartnerCatchAllFilter extends OncePerRequestFilter {

    private final AntPathMatcher matcher = new AntPathMatcher();
    private final Set<String> protectedPatterns;

    public PartnerCatchAllFilter(PartnerSignatureConfig config) {
      this.protectedPatterns =
          config.getPartners().values().stream()
              .flatMap(pc -> pc.getPathPatterns().stream())
              .collect(Collectors.toUnmodifiableSet());
    }

    @Override
    protected void doFilterInternal(
        HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws IOException, ServletException {
      boolean protectedPath =
          protectedPatterns.stream().anyMatch(p -> matcher.match(p, request.getRequestURI()));
      boolean authenticated = Boolean.TRUE.equals(request.getAttribute("partner.authenticated"));

      if (protectedPath && !authenticated) {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.getWriter().write("{\"code\":\"UNAUTHORIZED\",\"message\":\"Missing partner signature\"}");
        return;
      }
      chain.doFilter(request, response);
    }
  }
  ```
  Each partner filter simply sets `request.setAttribute("partner.authenticated", true)` after successful validation, so the catch-all knows the request is safe.
- This lets partners be added incrementally (new filter @Order(n)) without rewriting existing ones, while ensuring the protected endpoints never bypass authentication.
