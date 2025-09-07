Perfect! With PostgreSQL, Maven, Justfile, and your team's experience level, I can provide a very tailored solution. Let me give you the complete setup including authentication considerations.

## Complete Technology Stack & Setup

### Maven Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/POM/4.0.0/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.1</version>
        <relativePath/>
    </parent>
    
    <properties>
        <java.version>21</java.version>
        <testcontainers.version>1.19.3</testcontainers.version>
        <rest-assured.version>5.4.0</rest-assured.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- JWT for Auth -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.5</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${rest-assured.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <profiles>
                        <profile>local</profile>
                    </profiles>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Application Properties

```properties
# src/main/resources/application.properties
spring.application.name=playlists-api
server.port=8080

# Database (will be overridden by profiles)
spring.datasource.url=jdbc:postgresql://localhost:5432/playlists
spring.datasource.username=playlists
spring.datasource.password=playlists
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# JWT
jwt.secret=mySecretKey
jwt.expiration=86400000

# Logging
logging.level.com.yourcompany.playlists=DEBUG
logging.level.org.springframework.security=DEBUG
```

```properties
# src/main/resources/application-local.properties
spring.datasource.url=jdbc:postgresql://localhost:5432/playlists_local
spring.datasource.username=playlists_dev
spring.datasource.password=dev_password
spring.jpa.hibernate.ddl-auto=create-drop

# Local development data
spring.sql.init.mode=always
spring.sql.init.data-locations=classpath:local-data.sql
```

```properties
# src/main/resources/application-test.properties
# Test database will be provided by Testcontainers
spring.jpa.hibernate.ddl-auto=create-drop
spring.sql.init.mode=always
spring.sql.init.data-locations=classpath:test-data.sql
```

### Justfile Configuration

```justfile
# Justfile for development workflow

# Default PostgreSQL container settings
DB_CONTAINER_NAME := "playlists-postgres"
DB_NAME := "playlists_local"
DB_USER := "playlists_dev"
DB_PASSWORD := "dev_password"
DB_PORT := "5432"

# Setup local PostgreSQL database
setup-db:
    docker run --name {{DB_CONTAINER_NAME}} \
        -e POSTGRES_DB={{DB_NAME}} \
        -e POSTGRES_USER={{DB_USER}} \
        -e POSTGRES_PASSWORD={{DB_PASSWORD}} \
        -p {{DB_PORT}}:5432 \
        -d postgres:15
    echo "Waiting for database to be ready..."
    sleep 5

# Start local database (if container exists)
start-db:
    docker start {{DB_CONTAINER_NAME}}
    sleep 3

# Stop local database
stop-db:
    docker stop {{DB_CONTAINER_NAME}}

# Remove and recreate database (fresh start)
reset-db: stop-db
    docker rm {{DB_CONTAINER_NAME}}
    just setup-db

# Run unit tests only (no integration tests)
unit-tests:
    mvn test -Dtest="*Test" -DfailIfNoTests=false

# Run integration tests (includes database setup via Testcontainers)
integration-tests:
    mvn verify -Dspring.profiles.active=test -Dtest="*IntegrationTest"

# Run all tests
all-tests:
    just unit-tests
    just integration-tests

# Start API in development mode
dev:
    mvn spring-boot:run -Dspring-boot.run.profiles=local

# Build the application
build:
    mvn clean compile

# Package the application
package:
    mvn clean package -DskipTests

# Setup everything for first time development
setup: setup-db
    mvn clean install

# Quick development cycle: start db, run tests, start app
dev-cycle: start-db integration-tests dev

# Clean everything
clean-all:
    mvn clean
    docker stop {{DB_CONTAINER_NAME}} || true
    docker rm {{DB_CONTAINER_NAME}} || true
```

### Test Data Files

```sql
-- src/test/resources/test-data.sql
-- Test data for integration tests
INSERT INTO users (id, username, email, password, enabled, role) VALUES
(1, 'testuser', 'test@example.com', '$2a$10$8K3V6Vz8XcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJc', true, 'USER'),
(2, 'admin', 'admin@example.com', '$2a$10$8K3V6Vz8XcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJc', true, 'ADMIN');

INSERT INTO playlists (id, name, description, owner_id) VALUES
(1, 'Test Playlist', 'A test playlist', 1);
```

```sql
-- src/main/resources/local-data.sql
-- Local development data
INSERT INTO users (id, username, email, password, enabled, role) VALUES
(1, 'devuser', 'dev@example.com', '$2a$10$8K3V6Vz8XcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJc', true, 'USER'),
(2, 'devadmin', 'admin@dev.com', '$2a$10$8K3V6Vz8XcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJcKvJcQwJc', true, 'ADMIN');

INSERT INTO playlists (id, name, description, owner_id) VALUES
(1, 'Development Playlist', 'For local development', 1),
(2, 'Admin Playlist', 'Admin owned playlist', 2);
```

### Base Integration Test Class

```java
// src/test/java/com/yourcompany/playlists/BaseIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @LocalServerPort
    protected int port;

    @Autowired
    protected TestRestTemplate restTemplate;

    protected RequestSpecification given() {
        return RestAssured.given()
                .port(port)
                .contentType(ContentType.JSON);
    }

    // Helper methods for authentication
    protected String getAuthToken(String username, String password) {
        return given()
                .body(Map.of("username", username, "password", password))
                .when()
                .post("/api/auth/login")
                .then()
                .statusCode(200)
                .extract()
                .path("token");
    }

    protected RequestSpecification givenAuthenticated(String username, String password) {
        String token = getAuthToken(username, password);
        return given()
                .header("Authorization", "Bearer " + token);
    }
}
```

### Example Integration Test with Authentication

```java
// src/test/java/com/yourcompany/playlists/UserApiIntegrationTest.java
public class UserApiIntegrationTest extends BaseIntegrationTest {

    @Test
    void shouldGetUserProfile_WhenAuthenticated() {
        givenAuthenticated("testuser", "password")
            .when()
                .get("/api/users/profile")
            .then()
                .statusCode(200)
                .body("username", equalTo("testuser"));
    }

    @Test
    void shouldReturn401_WhenNotAuthenticated() {
        given()
            .when()
                .get("/api/users/profile")
            .then()
                .statusCode(401);
    }

    @Test
    void shouldCreatePlaylist_WhenAuthorized() {
        Map<String, Object> playlist = Map.of(
            "name", "Integration Test Playlist",
            "description", "Created by integration test"
        );

        givenAuthenticated("testuser", "password")
            .body(playlist)
            .when()
                .post("/api/playlists")
            .then()
                .statusCode(201)
                .body("name", equalTo("Integration Test Playlist"));
    }

    @Test
    void shouldReturn403_WhenAccessingAdminResource() {
        givenAuthenticated("testuser", "password")
            .when()
                .get("/api/admin/users")
            .then()
                .statusCode(403);
    }
}
```

## Development Workflow

### First Time Setup
```bash
just setup
```

### Daily Development
```bash
# Start database and run all tests
just dev-cycle

# Or individually:
just start-db
just integration-tests
just dev
```

### Testing Authentication
The `BaseIntegrationTest` class provides helper methods for JWT authentication in your tests. Each test can authenticate as different users with different roles.

### Best Practices for Your Team

1. **Test Naming**: Use descriptive test names that explain the scenario
2. **Test Data**: Keep test data minimal and focused on the test scenario
3. **Authentication**: Always test both authenticated and unauthenticated scenarios
4. **Authorization**: Test role-based access for each endpoint
5. **Database State**: Each test should be independent and not rely on state from other tests

This setup gives you:
- ✅ Real HTTP requests to running API
- ✅ Fresh PostgreSQL database for each test run
- ✅ JWT authentication testing capabilities
- ✅ Justfile for easy workflow management
- ✅ Separation of local dev and test environments
- ✅ Industry-standard technologies

The Testcontainers approach ensures your tests run against a real PostgreSQL instance, making them more reliable than in-memory databases. The Justfile provides a clean, documented way to manage your development workflow.