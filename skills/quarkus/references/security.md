# Security - Quarkus Security

Quarkus Security is config-driven, not a Java `@Configuration` class: there is no `SecurityFilterChain` bean to write. Path-level rules, the auth mechanism, and role mapping all come from extensions + `application.properties`.

## Security Configuration

```properties
# application.properties

# Public endpoints - no auth required
quarkus.http.auth.permission.public.paths=/api/auth/*,/q/health/*
quarkus.http.auth.permission.public.policy=permit

# Admin-only paths
quarkus.http.auth.permission.admin.paths=/api/admin/*
quarkus.http.auth.permission.admin.policy=admin-policy
quarkus.http.auth.policy.admin-policy.roles-allowed=ADMIN

# Everything else just needs to be authenticated
quarkus.http.auth.permission.default.paths=/*
quarkus.http.auth.permission.default.policy=authenticated

# CORS
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS
```

Permission sets are evaluated in the order Quarkus discovers them; the most specific path should carry the most specific policy. There is no built-in CSRF mechanism — Quarkus REST APIs are expected to be stateless and token-based, where CSRF doesn't apply; add a manual double-submit-cookie check only if the app genuinely uses cookie-based sessions.

## Self-Issued JWT with SmallRye JWT

Unlike a hand-rolled `OncePerRequestFilter`, token **validation** needs no custom filter at all — `quarkus-smallrye-jwt` reads the `Authorization: Bearer` header, verifies the signature/issuer/expiry against the config below, and populates `SecurityIdentity` automatically:

```properties
mp.jwt.verify.publickey.location=publicKey.pem
mp.jwt.verify.issuer=https://auth.example.com
```

Token **generation** uses the SmallRye JWT fluent builder:

```java
@ApplicationScoped
public class JwtService {
    private final JwtProperties jwtProperties;

    public JwtService(JwtProperties jwtProperties) {
        this.jwtProperties = jwtProperties;
    }

    public String generateToken(User user) {
        return Jwt.issuer(jwtProperties.issuer())
            .subject(user.email)
            .groups(user.roles.stream().map(r -> r.name).collect(Collectors.toSet()))
            .expiresIn(jwtProperties.expiration())
            .sign();
    }

    public String generateRefreshToken(User user) {
        return Jwt.issuer(jwtProperties.issuer())
            .subject(user.email)
            .expiresIn(jwtProperties.refreshExpiration())
            .sign();
    }
}
```

`sign()` uses the private key configured via `smallrye.jwt.sign.key.location` — never hardcode a signing key in code or in `application.properties` committed to version control.

## Authentication Resource

```java
@Path("/api/auth")
public class AuthenticationResource {
    private final AuthenticationService authenticationService;

    public AuthenticationResource(AuthenticationService authenticationService) {
        this.authenticationService = authenticationService;
    }

    @POST
    @Path("/register")
    @Status(201)
    public AuthenticationResponse register(@Valid RegisterRequest request) {
        return authenticationService.register(request);
    }

    @POST
    @Path("/login")
    public AuthenticationResponse login(@Valid LoginRequest request) {
        return authenticationService.login(request);
    }

    @POST
    @Path("/refresh")
    public AuthenticationResponse refreshToken(RefreshTokenRequest request) {
        return authenticationService.refreshToken(request);
    }
}
```

## Authentication Service (password hashing with BcryptUtil)

```java
@ApplicationScoped
public class AuthenticationService {
    private final UserRepository userRepository;
    private final JwtService jwtService;

    public AuthenticationService(UserRepository userRepository, JwtService jwtService) {
        this.userRepository = userRepository;
        this.jwtService = jwtService;
    }

    @Transactional
    public AuthenticationResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("Email already registered");
        }

        var user = new User();
        user.email = request.email();
        user.password = BcryptUtil.bcryptHash(request.password());
        user.username = request.username();
        user.active = true;
        user.roles = Set.of(Role.USER);
        userRepository.persist(user);

        return new AuthenticationResponse(
            jwtService.generateToken(user),
            jwtService.generateRefreshToken(user));
    }

    public AuthenticationResponse login(LoginRequest request) {
        User user = userRepository.findByEmail(request.email())
            .orElseThrow(() -> new NotAuthorizedException("Invalid credentials"));

        if (!BcryptUtil.matches(request.password(), user.password)) {
            throw new NotAuthorizedException("Invalid credentials");
        }

        return new AuthenticationResponse(
            jwtService.generateToken(user),
            jwtService.generateRefreshToken(user));
    }
}
```

`io.quarkus.elytron.security.common.BcryptUtil` ships with Quarkus (`quarkus-elytron-security-common`, pulled in transitively by `quarkus-smallrye-jwt`/`quarkus-oidc`) and replaces Spring Security's `BCryptPasswordEncoder` one-for-one — `bcryptHash(password)` to hash, `matches(password, hash)` to verify.

## Method Security

```java
@ApplicationScoped
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @RolesAllowed("ADMIN")
    public List<User> getAllUsers() {
        return userRepository.listAll();
    }

    @RolesAllowed({"ADMIN", "USER"})
    public User getUserById(Long userId, SecurityIdentity identity) {
        User user = userRepository.findByIdOptional(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        if (!identity.hasRole("ADMIN") && !identity.getPrincipal().getName().equals(user.email)) {
            throw new ForbiddenException();
        }
        return user;
    }

    @Authenticated
    public void updateOwnProfile(UserUpdateRequest request, SecurityIdentity identity) {
        User user = userRepository.findByEmail(identity.getPrincipal().getName())
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        // update logic
    }
}
```

`@RolesAllowed`/`@PermitAll`/`@DenyAll` are the standard `jakarta.annotation.security` annotations — not Quarkus- or Spring-specific — and work on CDI beans and JAX-RS resource methods alike. `@Authenticated` (`io.quarkus.security.Authenticated`) is Quarkus's own equivalent of Spring Security's `isAuthenticated()` expression: any valid identity, no specific role required. There's no SpEL-style `@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")` expression language — the ownership check above is a plain `if`, which is also easier to unit test without bootstrapping a security context.

## OIDC Resource Server (External Identity Provider)

For Keycloak, Auth0, Entra ID, or any OpenID Connect provider, `quarkus-oidc` replaces both the self-issued JWT setup above and a hand-written `JwtDecoder` bean:

```properties
quarkus.oidc.auth-server-url=https://auth.example.com/realms/myrealm
quarkus.oidc.client-id=my-service
quarkus.oidc.application-type=service

quarkus.http.auth.permission.public.paths=/public/*
quarkus.http.auth.permission.public.policy=permit
quarkus.http.auth.permission.default.paths=/*
quarkus.http.auth.permission.default.policy=authenticated
```

No Java configuration is required beyond this; `@RolesAllowed`/`SecurityIdentity` work the same way regardless of whether the token was validated via `quarkus-smallrye-jwt` or `quarkus-oidc`.

## Keycloak Dev Services — Zero-Config Local Auth

This is the security-specific version of the same Dev Services mechanism behind the automatic database/Kafka containers in `testing-patterns`: add `quarkus-oidc` (or `quarkus-oidc-client`) with **no** `quarkus.oidc.auth-server-url` configured, and running `quarkus:dev` or any test automatically starts a real Keycloak container, pre-provisioned and ready to issue tokens — no manual Keycloak setup, no realm export/import step, nothing to configure by hand to get a working login.

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

Leaving `quarkus.oidc.auth-server-url` unset in dev/test mode is enough to trigger it. Out of the box, Dev Services for Keycloak creates:

- a realm named `quarkus`
- a client `quarkus-app` with client secret `secret`
- two users: `alice` (roles `admin` + `user`) and `bob` (role `user`), both with password equal to the username
- an admin console login of `admin`/`admin`

```java
@QuarkusTest
class OrderResourceTest {

    @Test
    void adminCanDeleteOrder() {
        String token = given()
            .contentType(ContentType.URLENCODED)
            .formParam("grant_type", "password")
            .formParam("client_id", "quarkus-app")
            .formParam("client_secret", "secret")
            .formParam("username", "alice")
            .formParam("password", "alice")
            .when().post("/realms/quarkus/protocol/openid-connect/token") // against the Dev Services Keycloak
            .then().extract().path("access_token");

        given()
            .auth().oauth2(token)
            .when().delete("/api/v1/orders/1")
            .then().statusCode(204);
    }
}
```

To use a project's real realm/client/roles instead of the generic defaults, drop an exported realm JSON file on the classpath and point Dev Services at it — it takes priority over the generated default realm:

```properties
# application.properties
quarkus.keycloak.devservices.realm-path=quarkus-realm.json
```

Other useful knobs: `quarkus.keycloak.devservices.create-realm=false` to skip realm creation entirely (e.g. when the container is only needed for something else), and `quarkus.keycloak.devservices.access-token-lifespan` / `.sso-session-max-lifespan` to tune token/session expiry for faster expiry-related test scenarios. As with every other Dev Services, this only activates in dev/test mode and requires Docker/Podman — it's never part of a production build.

## TLS & Certificate Management

Covered in depth by the `security-engineer` agent: the unified TLS registry (`quarkus.tls.*`) replaces SSL Bundles, with `reload-period` for zero-downtime certificate rotation and the same cert-manager-on-Kubernetes story.

## Quick Reference

| Annotation / Property | Purpose |
|---|---|
| `quarkus.http.auth.permission.<name>.*` | Path-level access rules (replaces `authorizeHttpRequests`) |
| `@RolesAllowed` | Role-based method/class security (standard `jakarta.annotation.security`) |
| `io.quarkus.security.Authenticated` | Require any authenticated identity, no specific role |
| `SecurityIdentity` | Inject to read the current principal/roles/attributes programmatically |
| `BcryptUtil.bcryptHash()` / `.matches()` | Password hashing (replaces `BCryptPasswordEncoder`) |
| `Jwt.issuer(...).sign()` | Build and sign a JWT (SmallRye JWT Build) |
| `quarkus-smallrye-jwt` / `quarkus-oidc` | Validate bearer tokens declaratively — no custom filter |
| Keycloak Dev Services (add `quarkus-oidc`, leave `auth-server-url` unset) | Auto-starts a pre-provisioned Keycloak container for dev/test, zero config |

## Security Best Practices

- Always use HTTPS in production
- Use the TLS registry with a `reload-period` for zero-downtime certificate rotation
- Store JWT signing keys/OIDC client secrets in environment variables or mounted secrets, never in `application.properties`
- Use `BcryptUtil` for password hashing (never roll your own)
- Implement a token refresh mechanism
- Add `@RateLimit` (SmallRye Fault Tolerance) to authentication endpoints
- Validate all user inputs with Bean Validation
- Log security-relevant events
- Keep extensions updated (`./mvnw quarkus:update`)
- Implement proper token/session expiry
