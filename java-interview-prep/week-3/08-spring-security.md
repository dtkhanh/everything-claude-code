# Week 3 — Day 4–5: Spring Security

> 📖 **Estimated reading time:** 45 minutes  
> 🎯 **Focus:** Filter chain, JWT authentication, OAuth2, Method security

---

## 1. Spring Security Architecture

Every HTTP request passes through a **filter chain** before reaching your controller. Authentication and authorization happen here — your controller only runs if everything in the chain passes.

```
HTTP Request
    ↓
[SecurityFilterChain]
  ├── SecurityContextPersistenceFilter  (load/save SecurityContext)
  ├── UsernamePasswordAuthenticationFilter  (form login)
  ├── BasicAuthenticationFilter           (HTTP Basic)
  ├── BearerTokenAuthenticationFilter     (JWT/OAuth2)
  ├── ExceptionTranslationFilter          (401/403 → response)
  └── FilterSecurityInterceptor           (access decision)
    ↓
DispatcherServlet → Controller
```

### 🔑 Key Classes
- `SecurityContextHolder`: thread-local holder of the current `Authentication`
- `Authentication`: who the user is + their authorities
- `UserDetails`: Spring's representation of a user
- `AuthenticationManager` → `AuthenticationProvider`: validates credentials
- `GrantedAuthority`: a permission/role the user has

---

## 2. JWT Authentication — Full Implementation

### 📌 Concept Summary
JWT (JSON Web Token) = stateless authentication. Token contains user info + signature. Server validates signature, no session storage needed.

### 💻 Code Example — JwtService
```java
@Service
public class JwtService {
    @Value("${app.jwt.secret}")
    private String secret;
    
    @Value("${app.jwt.expiration-ms:86400000}") // 24h default
    private long expirationMs;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }

    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey())
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

    private <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return claimsResolver.apply(claims);
    }
}
```

### 💻 Code Example — JWT Filter
```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
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
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### 💻 Code Example — SecurityConfig
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {
    private final JwtAuthenticationFilter jwtFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)      // stateless REST API
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) ->
                    res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized"))
                .accessDeniedHandler((req, res, e) ->
                    res.sendError(HttpServletResponse.SC_FORBIDDEN, "Forbidden"))
            )
            .build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // cost factor 12
    }
}
```

### ❓ Interview Question
> "Walk through the JWT authentication flow in a Spring Boot API."

### ✅ Model Answer
1. **Login**: Client POSTs credentials to `/api/auth/login`
2. **Validate**: `AuthenticationManager` calls `UserDetailsService.loadUserByUsername()`, verifies BCrypt password
3. **Generate token**: `JwtService.generateToken()` creates signed JWT with username + roles embedded
4. **Client stores token**: localStorage (SPA) or memory (more secure)
5. **Subsequent requests**: Client sends `Authorization: Bearer <token>` header
6. **Filter intercepts**: `JwtAuthenticationFilter` extracts token, validates signature + expiry, loads `UserDetails`, sets `SecurityContext`
7. **Authorization**: Spring checks `GrantedAuthority` against required roles — allows or `403 Forbidden`
8. **No session**: every request is independent, server is fully stateless

### ⚠️ Gotchas
- **Never store JWT in `localStorage` if XSS is possible** — use `HttpOnly` cookie instead
- **JWT can't be invalidated** before expiry unless you maintain a blacklist (redis set of revoked tokens)
- **Short expiry + refresh tokens**: access token ~15min, refresh token ~7d stored in DB

---

## 3. Method Security

```java
@Service
public class DocumentService {
    
    // Check before method runs
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public Document getDocument(Long userId, Long documentId) { ... }
    
    // Check after method runs (can inspect return value)
    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Document findById(Long id) { ... }
    
    // Filter collection parameters
    @PreFilter("filterObject.status != 'DELETED'")
    public void processBatch(List<Document> docs) { ... }
    
    // Filter return collection
    @PostFilter("filterObject.ownerId == authentication.principal.id")
    public List<Document> findAll() { ... }
}
```

---

## 4. UserDetails & UserDetailsService

```java
// Custom UserDetails — your User entity implements it
@Entity
@Table(name = "users")
public class User implements UserDetails {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String email;
    private String password; // BCrypt hashed
    
    @ElementCollection(fetch = FetchType.EAGER)
    private Set<String> roles = new HashSet<>();
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toSet());
    }
    
    @Override public String getUsername() { return email; }
    @Override public String getPassword() { return password; }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return true; }
}

// UserDetailsService — loads user for authentication
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
    }
}
```

---

## 5. CORS Configuration

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com")); // don't use * in prod
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L); // preflight cache 1h
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}

// In SecurityConfig:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

---

*Next: [09-spring-data-jpa.md](./09-spring-data-jpa.md)*


