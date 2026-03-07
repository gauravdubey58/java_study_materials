# 07 — Spring Security

[← Back to Index](./README.md)

---

## Table of Contents
- [Security Concepts](#security-concepts)
- [SecurityFilterChain — Boot 2.7 Style](#securityfilterchain--boot-27-style)
- [UserDetailsService](#userdetailsservice)
- [JWT Authentication](#jwt-authentication)
- [Method-Level Security](#method-level-security)
- [OAuth2 & OIDC](#oauth2--oidc)
- [CSRF Protection](#csrf-protection)
- [CORS Configuration](#cors-configuration)
- [Password Encoding](#password-encoding)

---

## Security Concepts

| Term | Definition |
|------|-----------|
| **Authentication** | Verifying identity — "who are you?" (username + password, JWT, OAuth2) |
| **Authorization** | Verifying permissions — "what can you do?" (roles, authorities) |
| **Principal** | The currently authenticated entity (user) |
| **SecurityContext** | Thread-local storage holding the current `Authentication` |
| **GrantedAuthority** | A permission/role granted to a principal (e.g., `ROLE_ADMIN`) |
| **Filter Chain** | Ordered list of security filters every HTTP request passes through |
| **CSRF** | Cross-Site Request Forgery — malicious site tricks browser to make requests |
| **CORS** | Cross-Origin Resource Sharing — allows requests from different origins |

### Spring Security Filter Order (simplified)

```
Request
  │
  ├─ CorsFilter
  ├─ CsrfFilter
  ├─ UsernamePasswordAuthenticationFilter  ← form login
  ├─ BasicAuthenticationFilter             ← Basic Auth
  ├─ BearerTokenAuthenticationFilter       ← JWT / OAuth2
  ├─ ...your custom filters...
  └─ AuthorizationFilter                   ← final access decision
         │
         ▼
  Controller / Handler
```

---

## SecurityFilterChain — Boot 2.7 Style

> ⚠️ **`WebSecurityConfigurerAdapter` is deprecated in Spring Boot 2.7 / Spring Security 5.7.**  
> Use `SecurityFilterChain` beans instead.

### Stateless REST API (JWT)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtAuthFilter jwtAuthFilter,
                          UserDetailsService userDetailsService) {
        this.jwtAuthFilter    = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF for stateless REST APIs
            .csrf(csrf -> csrf.disable())

            // Stateless session — no HttpSession created
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .antMatchers("/api/auth/**").permitAll()            // Public endpoints
                .antMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")      // ROLE_ADMIN
                .antMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")
                .antMatchers("/api/users/**").hasAuthority("user:read")
                .anyRequest().authenticated()
            )

            // Add JWT filter before the username/password filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

### Form-Based (Session) Security

```java
@Bean
public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .antMatchers("/login", "/register", "/css/**", "/js/**").permitAll()
            .antMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginPage("/login")
            .loginProcessingUrl("/do-login")
            .defaultSuccessUrl("/dashboard", true)
            .failureUrl("/login?error=true")
        )
        .logout(logout -> logout
            .logoutUrl("/logout")
            .logoutSuccessUrl("/login?logout")
            .invalidateHttpSession(true)
            .clearAuthentication(true)
            .deleteCookies("JSESSIONID")
        )
        .rememberMe(remember -> remember
            .key("uniqueAndSecret")
            .tokenValiditySeconds(86400)  // 1 day
        );
    return http.build();
}
```

---

## UserDetailsService

Spring Security loads user details via `UserDetailsService`.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPassword())                  // BCrypt-encoded
            .roles(user.getRole().name())                  // Adds ROLE_ prefix
            // or
            .authorities(user.getAuthorities())            // Full authority strings
            .accountExpired(!user.isActive())
            .credentialsExpired(user.isPasswordExpired())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

---

## JWT Authentication

### JWT Auth Filter

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        final String authHeader = request.getHeader("Authorization");

        // Skip if no Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7);
        final String username = jwtService.extractUsername(jwt);

        // Only authenticate if not already authenticated
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### JWT Service

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration:86400000}")   // Default 24h
    private long jwtExpiration;

    public String generateToken(UserDetails userDetails) {
        return generateToken(Map.of(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return Jwts.builder()
            .setClaims(extraClaims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        return resolver.apply(extractAllClaims(token));
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
        );
        UserDetails user = userDetailsService.loadUserByUsername(request.getEmail());
        String token = jwtService.generateToken(user);
        return ResponseEntity.ok(new AuthResponse(token));
    }

    @PostMapping("/register")
    public ResponseEntity<UserDto> register(@Valid @RequestBody RegisterRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(authService.register(request));
    }
}
```

---

## Method-Level Security

```java
// Enable (Spring Boot 2.7 / Security 5.7+)
@EnableMethodSecurity(prePostEnabled = true)
@Configuration
public class MethodSecurityConfig { }
```

```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    // Only ADMIN can access
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/all")
    public List<Document> getAll() { ... }

    // User can only access their own documents
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    @GetMapping("/user/{userId}")
    public List<Document> getUserDocs(@PathVariable Long userId) { ... }

    // Check returned object against current user
    @PostAuthorize("returnObject.author == authentication.name")
    @GetMapping("/{id}")
    public Document getById(@PathVariable Long id) { ... }

    // Filter input list — only process files owned by current user
    @PreFilter("filterObject.owner == authentication.name")
    @DeleteMapping("/batch")
    public void deleteFiles(@RequestBody List<File> files) { ... }

    // Filter output list — only return non-confidential documents
    @PostFilter("filterObject.confidential == false or hasRole('ADMIN')")
    @GetMapping
    public List<Document> getAll2() { ... }
}
```

---

## OAuth2 & OIDC

### Resource Server (validate JWT from Auth Server)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com   # or your auth server
          # Or provide public key directly:
          # public-key-location: classpath:public.pem
```

```java
@Bean
public SecurityFilterChain resourceServerChain(HttpSecurity http) throws Exception {
    http
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter()))
        )
        .authorizeHttpRequests(auth -> auth
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        );
    return http.build();
}

@Bean
public JwtAuthenticationConverter jwtConverter() {
    JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
    grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
    grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
    return converter;
}
```

---

## CSRF Protection

| Scenario | CSRF Needed? |
|----------|-------------|
| Browser-based app with cookies/session | ✅ Yes |
| Stateless REST API with JWT in header | ❌ No (disable it) |
| Mobile app | ❌ No |

```java
// Stateless API — disable CSRF
.csrf(csrf -> csrf.disable())

// Stateful app — use CSRF token
.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())  // for SPA
)

// Ignore specific paths
.csrf(csrf -> csrf
    .ignoringAntMatchers("/api/webhooks/**")
)
```

---

## CORS Configuration

```java
// Option 1: Global config (preferred)
@Configuration
public class CorsConfig {
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:3000", "https://myapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(List.of("Authorization", "X-Custom-Header"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}

// Option 2: Controller-level
@CrossOrigin(origins = "http://localhost:3000")
@RestController
public class ProductController { ... }

// Option 3: Method-level
@CrossOrigin(origins = "*", methods = {RequestMethod.GET})
@GetMapping("/public/products")
public List<Product> publicProducts() { ... }
```

---

## Password Encoding

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // Work factor 12 (higher = slower = more secure)
}

// Registration
public void registerUser(CreateUserRequest req) {
    User user = new User();
    user.setEmail(req.getEmail());
    user.setPassword(passwordEncoder.encode(req.getPassword()));   // Store encoded
    userRepository.save(user);
}

// Comparison (login)
boolean matches = passwordEncoder.matches(rawPassword, encodedPassword);  // Never decode
```

### Password Encoder Options

| Encoder | Algorithm | Notes |
|---------|-----------|-------|
| `BCryptPasswordEncoder` | BCrypt | ✅ Recommended, adaptive |
| `Argon2PasswordEncoder` | Argon2 | Best; memory-hard |
| `Pbkdf2PasswordEncoder` | PBKDF2 | FIPS-compliant |
| `SCryptPasswordEncoder` | SCrypt | Memory-hard |
| `DelegatingPasswordEncoder` | Multiple | Migrate between encoders |
| `NoOpPasswordEncoder` | Plain text | ❌ NEVER in production |

---

[← Spring Data JPA](./06_Spring_Data_JPA.md) &nbsp;|&nbsp; [Next → Actuator & Monitoring](./08_Actuator.md)
