Dưới đây là **Checklist triển khai Phase 0 (Keycloak + API Gateway + User Service)** theo dạng **task-by-task** (có **file tree + config + snippets**) để bạn làm theo như “playbook” triển khai chuẩn production.

> Mục tiêu Phase 0:
> ✅ Dựng Keycloak (OIDC) + Postgres + realm/roles/client export
> ✅ Dựng API Gateway (Spring Cloud Gateway) verify JWT + RBAC route policy + propagate headers
> ✅ Dựng spa-user-service (DB + Liquibase + API /me) dùng `user_id = sub`
> ✅ Chạy E2E: login -> gọi `/me` thành công

---

# Phase 0 — PLAYBOOK

## 0) Quy ước & thông số dùng chung

* **Realm**: `spa-booking`
* **Roles (realm roles)**: `CUSTOMER`, `STAFF`, `ADMIN`
* **Client (OIDC)**:

  * `spa-web` (public client) — dùng cho front-end (Authorization Code + PKCE)
  * `spa-api-gateway` (confidential) — optional (nếu bạn muốn gateway làm BFF). Với API gateway thuần route, chỉ cần resource-server verify token.
* **Issuer URL (Docker)**: `http://keycloak:8080/realms/spa-booking`
* Headers Gateway propagate:

  * `X-User-Id` = `sub`
  * `X-User-Roles` = roles (realm_access.roles)
  * `X-User-Email` = email (optional)

---

# 1) Repo `infra-keycloak`

## ✅ 1.1 Create repo structure

```text
infra-keycloak/
  docker-compose.yml
  keycloak/
    realm-export/
      spa-booking-realm.json   (export sau khi tạo realm)
  README.md
```

---

## ✅ 1.2 docker-compose Keycloak + Postgres

**infra-keycloak/docker-compose.yml**

```yaml
services:
  keycloak-db:
    image: postgres:16
    container_name: keycloak-db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
    ports:
      - "5433:5432"
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:26.0.7
    container_name: keycloak
    depends_on:
      - keycloak-db
    command:
      - start-dev
      - --import-realm
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8081:8080"
    volumes:
      - ./keycloak/realm-export:/opt/keycloak/data/import

volumes:
  keycloak_db_data:
```

**Run**

```bash
docker compose up -d
```

---

## ✅ 1.3 Tạo Realm/Client/Roles (manual lần đầu)

Truy cập: `http://localhost:8081`
Login: `admin/admin`

### Tạo Realm

* Realm name: `spa-booking`

### Tạo Roles (Realm Roles)

* `CUSTOMER`
* `STAFF`
* `ADMIN`

### Tạo Client cho FE (public client)

* Client ID: `spa-web`
* Client type: OpenID Connect
* Access type: **public**
* Standard flow: ON
* PKCE: ON
* Valid redirect URIs:

  * `http://localhost:3000/*`
* Web origins:

  * `http://localhost:3000`

### Tạo Users demo

* `alice` role `CUSTOMER`
* `bob` role `STAFF`
* `admin1` role `ADMIN`

---

## ✅ 1.4 Export realm để versioning

Trong Keycloak UI:

* Realm settings → **Export** → Export realm
  Lưu vào:

```text
infra-keycloak/keycloak/realm-export/spa-booking-realm.json
```

> Tip: lần sau chạy `--import-realm` sẽ tự dựng lại.

---

## ✅ 1.5 README cho infra-keycloak (tối thiểu)

**infra-keycloak/README.md** (nội dung gợi ý)

* how to run
* issuer url
* users demo
* how to export/import realm

---

# 2) Repo `spa-api-gateway`

## ✅ 2.1 Create repo + file tree

```text
spa-api-gateway/
  pom.xml
  src/main/java/.../GatewayApplication.java
  src/main/java/.../security/SecurityConfig.java
  src/main/java/.../security/UserContextGlobalFilter.java
  src/main/resources/application.yml
  src/main/resources/application-dev.yml (optional)
  README.md
```

---

## ✅ 2.2 Dependencies (pom.xml)

Tối thiểu:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

> Bạn cần thêm BOM Spring Cloud tương thích với Spring Boot 3.5.x (để build không lỗi).

---

## ✅ 2.3 application.yml (JWT validation + routes)

**spa-api-gateway/src/main/resources/application.yml**

```yaml
server:
  port: 8080

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/spa-booking

  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://spa-user-service:8082
          predicates:
            - Path=/me/**

        # Public read examples (sau này)
        - id: catalog-public
          uri: http://spa-catalog-service:8083
          predicates:
            - Path=/services/**

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

> Trong local dev bạn có thể dùng `http://localhost:8081/...` nếu gateway chạy ngoài docker.
> Trong docker network: dùng `http://keycloak:8080/...`

---

## ✅ 2.4 SecurityConfig (RBAC route policy)

**SecurityConfig.java**

```java
@EnableWebFluxSecurity
public class SecurityConfig {

  @Bean
  public SecurityWebFilterChain security(ServerHttpSecurity http) {
    return http
      .csrf(ServerHttpSecurity.CsrfSpec::disable)
      .authorizeExchange(ex -> ex
        .pathMatchers("/actuator/health", "/services/**").permitAll()
        .pathMatchers("/admin/**").hasRole("ADMIN")
        .pathMatchers("/staff/**").hasAnyRole("STAFF","ADMIN")
        .pathMatchers("/me/**").authenticated()
        .anyExchange().authenticated()
      )
      .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
      .build();
  }
}
```

> Lưu ý: `hasRole("ADMIN")` tương đương role name `ROLE_ADMIN`.
> Với Keycloak realm roles, Spring thường map thành `ROLE_<role>`.
> Nếu bạn thấy mismatch, bạn sẽ cần converter (mục 2.6).

---

## ✅ 2.5 GlobalFilter propagate headers (X-User-Id, X-User-Roles, X-User-Email)

**UserContextGlobalFilter.java**

```java
@Component
public class UserContextGlobalFilter implements GlobalFilter, Ordered {

  @Override
  public int getOrder() {
    return -1;
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return exchange.getPrincipal()
      .cast(JwtAuthenticationToken.class)
      .map(auth -> {
        Jwt jwt = auth.getToken();

        String sub = jwt.getSubject();
        String email = jwt.getClaimAsString("email");

        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        @SuppressWarnings("unchecked")
        List<String> roles = realmAccess == null ? List.of()
          : (List<String>) realmAccess.getOrDefault("roles", List.of());

        ServerHttpRequest req = exchange.getRequest().mutate()
          .header("X-User-Id", sub)
          .header("X-User-Email", email == null ? "" : email)
          .header("X-User-Roles", String.join(",", roles))
          .build();

        return exchange.mutate().request(req).build();
      })
      .defaultIfEmpty(exchange)
      .flatMap(chain::filter);
  }
}
```

---

## ✅ 2.6 (Quan trọng) Map roles từ Keycloak vào Spring Authorities

Nếu `hasRole("ADMIN")` không hoạt động (thường gặp), thêm converter:

**KeycloakRoleConverter.java**

```java
@Component
public class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {

  @Override
  public Collection<GrantedAuthority> convert(Jwt jwt) {
    Map<String, Object> realmAccess = jwt.getClaim("realm_access");
    if (realmAccess == null) return List.of();

    @SuppressWarnings("unchecked")
    List<String> roles = (List<String>) realmAccess.getOrDefault("roles", List.of());

    return roles.stream()
      .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
      .toList();
  }
}
```

Trong `SecurityConfig`:

```java
@Bean
public ReactiveJwtAuthenticationConverter jwtAuthConverter(KeycloakRoleConverter converter) {
  ReactiveJwtAuthenticationConverter c = new ReactiveJwtAuthenticationConverter();
  c.setJwtGrantedAuthoritiesConverter(converter);
  return c;
}

@Bean
public SecurityWebFilterChain security(ServerHttpSecurity http,
    ReactiveJwtAuthenticationConverter jwtAuthConverter) {
  return http
    ...
    .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter)))
    .build();
}
```

---

## ✅ 2.7 Quick Test JWT (manual)

Lấy token từ Keycloak (password grant chỉ để dev):

```bash
curl -X POST "http://localhost:8081/realms/spa-booking/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=spa-web" \
  -d "grant_type=password" \
  -d "username=alice" \
  -d "password=<password>"
```

Copy `access_token` để gọi gateway.

---

# 3) Repo `spa-user-service`

## ✅ 3.1 Create repo + file tree

```text
spa-user-service/
  pom.xml
  src/main/java/.../UserServiceApplication.java
  src/main/java/.../controller/MeController.java
  src/main/java/.../service/MeService.java
  src/main/java/.../repository/UserRepository.java
  src/main/java/.../entity/UserEntity.java
  src/main/java/.../dto/MyProfileResponse.java
  src/main/java/.../dto/UpdateMyProfileRequest.java
  src/main/resources/application.yml
  src/main/resources/db/changelog/db.changelog-master.xml
  src/main/resources/db/changelog/changes/V1__init.xml
  README.md
```

---

## ✅ 3.2 Dependencies (pom.xml)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

> User-service **không cần Spring Security** nếu bạn “trust gateway” hoàn toàn và service chỉ chạy internal network. (Prod nên chặn direct access bằng network policy.)

---

## ✅ 3.3 application.yml (DB + liquibase)

```yaml
server:
  port: 8082

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/user_db
    username: user
    password: user
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        format_sql: true
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

---

## ✅ 3.4 Liquibase DDL (users table)

**db.changelog-master.xml**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
  <include file="db/changelog/changes/V1__init.xml" relativeToChangelogFile="false"/>
</databaseChangeLog>
```

**V1__init.xml** (tối thiểu cho Phase 0)

* `users` (PK=user_id=Keycloak sub)
* unique email/phone partial
* role_snapshot check

*(Bạn có thể dùng bản Liquibase mình đã đưa ở trước cho user-service.)*

---

## ✅ 3.5 Entity + Repository

**UserEntity.java**

```java
@Entity
@Table(name = "users")
public class UserEntity {

  @Id
  @Column(name = "user_id", nullable = false)
  private UUID userId; // = Keycloak sub

  @Column(name = "full_name", nullable = false)
  private String fullName;

  @Column(name = "email")
  private String email;

  @Column(name = "phone")
  private String phone;

  @Column(name = "role_snapshot", nullable = false)
  private String roleSnapshot;

  @Column(name = "created_at", nullable = false)
  private OffsetDateTime createdAt;

  @Column(name = "updated_at", nullable = false)
  private OffsetDateTime updatedAt;

  // getters/setters
}
```

**UserRepository.java**

```java
public interface UserRepository extends JpaRepository<UserEntity, UUID> {}
```

---

## ✅ 3.6 Business logic `/me` (lazy-create profile)

**MeService.java**

```java
@Service
public class MeService {

  private final UserRepository repo;

  public MeService(UserRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public UserEntity getOrCreate(UUID userId, String email, String rolesHeader) {
    return repo.findById(userId).orElseGet(() -> {
      UserEntity u = new UserEntity();
      u.setUserId(userId);
      u.setEmail((email == null || email.isBlank()) ? null : email);
      u.setFullName("New User"); // default, client can update later
      u.setRoleSnapshot(resolvePrimaryRole(rolesHeader));
      u.setCreatedAt(OffsetDateTime.now());
      u.setUpdatedAt(OffsetDateTime.now());
      return repo.save(u);
    });
  }

  private String resolvePrimaryRole(String rolesHeader) {
    // Example priority: ADMIN > STAFF > CUSTOMER
    if (rolesHeader == null) return "CUSTOMER";
    String r = rolesHeader.toUpperCase();
    if (r.contains("ADMIN")) return "ADMIN";
    if (r.contains("STAFF")) return "STAFF";
    return "CUSTOMER";
  }
}
```

---

## ✅ 3.7 Controller `/me`

**MeController.java**

```java
@RestController
@RequestMapping("/me")
public class MeController {

  private final MeService service;

  public MeController(MeService service) {
    this.service = service;
  }

  @GetMapping
  public MyProfileResponse me(
      @RequestHeader("X-User-Id") UUID userId,
      @RequestHeader(value = "X-User-Email", required = false) String email,
      @RequestHeader(value = "X-User-Roles", required = false) String roles
  ) {
    UserEntity u = service.getOrCreate(userId, email, roles);
    return MyProfileResponse.from(u);
  }
}
```

---

# 4) E2E Checklist (điểm kết thúc Phase 0)

## ✅ 4.1 Run infra-keycloak

```bash
cd infra-keycloak
docker compose up -d
```

## ✅ 4.2 Run user-service DB (local dev)

Bạn có thể dùng 1 Postgres riêng cho `user_db` (docker compose khác), ví dụ nhanh:

```bash
docker run --name user-db -e POSTGRES_DB=user_db -e POSTGRES_USER=user -e POSTGRES_PASSWORD=user -p 5432:5432 -d postgres:16
```

## ✅ 4.3 Start spa-user-service

```bash
cd spa-user-service
mvn clean spring-boot:run
```

* Liquibase chạy và tạo bảng thành công.

## ✅ 4.4 Start spa-api-gateway

```bash
cd spa-api-gateway
mvn clean spring-boot:run
```

## ✅ 4.5 Get token từ Keycloak (dev)

```bash
curl -X POST "http://localhost:8081/realms/spa-booking/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=spa-web" \
  -d "grant_type=password" \
  -d "username=alice" \
  -d "password=<password>"
```

## ✅ 4.6 Call `/me` through gateway

```bash
curl -H "Authorization: Bearer <ACCESS_TOKEN>" http://localhost:8080/me
```

**Expected**

* `200 OK`
* user record auto-created with `user_id = sub`

---

# 5) Definition of Done (Phase 0)

✅ Keycloak realm/roles/users ok (export file committed)
✅ Gateway verify JWT ok (issuer-uri)
✅ RBAC hoạt động (ADMIN route bị chặn với CUSTOMER)
✅ Gateway propagate `X-User-*` headers ok
✅ User-service DB created by Liquibase
✅ `/me` works end-to-end via gateway

---

Nếu bạn muốn, bước tiếp theo mình có thể viết **docker-compose Phase 0** gom **Keycloak + gateway + user-service + user-db** vào 1 stack để bạn chạy 1 lệnh là lên toàn bộ.
