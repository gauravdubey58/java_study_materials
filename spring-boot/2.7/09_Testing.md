# 09 — Testing in Spring Boot

[← Back to Index](./README.md)

---

## Table of Contents
- [Testing Pyramid](#testing-pyramid)
- [Test Dependencies](#test-dependencies)
- [Unit Testing with Mockito](#unit-testing-with-mockito)
- [Controller Testing with MockMvc](#controller-testing-with-mockmvc)
- [Repository Testing with @DataJpaTest](#repository-testing-with-datajpatest)
- [Integration Testing with @SpringBootTest](#integration-testing-with-springboottest)
- [Testing Annotations Cheat Sheet](#testing-annotations-cheat-sheet)

---

## Testing Pyramid

```
         ┌────────────────┐
         │  E2E / UI Tests│   Few, slow, expensive
         └───────┬────────┘
        ┌────────┴──────────┐
        │ Integration Tests │   Medium number
        └────────┬──────────┘
      ┌──────────┴────────────┐
      │     Unit Tests        │   Many, fast, cheap
      └───────────────────────┘
```

---

## Test Dependencies

`spring-boot-starter-test` bundles everything you need:
- **JUnit 5** — test framework
- **Mockito** — mocking framework
- **AssertJ** — fluent assertions
- **Hamcrest** — matchers
- **MockMvc** — test MVC controllers without a server
- **TestRestTemplate** — test with a real HTTP server
- **H2** — in-memory database for `@DataJpaTest`

---

## Unit Testing with Mockito

Pure unit tests — no Spring context, very fast.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;     // Mocks injected via constructor

    // ─── Happy Path ───────────────────────────────────────────────
    @Test
    void findById_whenUserExists_returnsUser() {
        // Arrange
        User user = new User(1L, "Alice", "alice@test.com");
        given(userRepository.findById(1L)).willReturn(Optional.of(user));

        // Act
        UserDto result = userService.findById(1L);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("Alice");
        then(userRepository).should(times(1)).findById(1L);
    }

    // ─── Sad Path ─────────────────────────────────────────────────
    @Test
    void findById_whenUserNotFound_throwsException() {
        given(userRepository.findById(99L)).willReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(99L))
            .isInstanceOf(UserNotFoundException.class)
            .hasMessageContaining("99");
    }

    // ─── Void Methods ─────────────────────────────────────────────
    @Test
    void deleteUser_callsRepositoryDelete() {
        given(userRepository.existsById(1L)).willReturn(true);
        willDoNothing().given(userRepository).deleteById(1L);

        userService.delete(1L);

        then(userRepository).should().deleteById(1L);
    }

    // ─── Argument Captor ──────────────────────────────────────────
    @Test
    void createUser_sendsWelcomeEmail() {
        ArgumentCaptor<String> emailCaptor = ArgumentCaptor.forClass(String.class);
        given(userRepository.save(any())).willAnswer(inv -> inv.getArgument(0));

        userService.create(new CreateUserRequest("Bob", "bob@test.com"));

        then(emailService).should().sendWelcome(emailCaptor.capture());
        assertThat(emailCaptor.getValue()).isEqualTo("bob@test.com");
    }

    // ─── Spy (partial mock) ───────────────────────────────────────
    @Test
    void withSpy() {
        UserService spy = Mockito.spy(new UserService(userRepository, emailService));
        doReturn(new UserDto()).when(spy).findById(anyLong());
        // real methods are called unless stubbed
    }
}
```

---

## Controller Testing with MockMvc

Tests the web layer without starting a real HTTP server.

```java
@WebMvcTest(UserController.class)           // Only loads web layer
@Import(SecurityConfig.class)               // Import security if needed
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean                               // Mocks service inside Spring context
    private UserService userService;

    // ─── GET ──────────────────────────────────────────────────────
    @Test
    void getUser_returnsOkWithBody() throws Exception {
        UserDto user = new UserDto(1L, "Alice", "alice@test.com");
        given(userService.findById(1L)).willReturn(user);

        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@test.com"))
            .andDo(print());
    }

    // ─── POST ─────────────────────────────────────────────────────
    @Test
    void createUser_returnsCreatedWithLocation() throws Exception {
        CreateUserRequest request = new CreateUserRequest("Bob", "bob@test.com");
        UserDto created = new UserDto(5L, "Bob", "bob@test.com");
        given(userService.create(any())).willReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/api/users/5")))
            .andExpect(jsonPath("$.id").value(5));
    }

    // ─── Validation Error ─────────────────────────────────────────
    @Test
    void createUser_withInvalidEmail_returnsBadRequest() throws Exception {
        CreateUserRequest badRequest = new CreateUserRequest("Bob", "not-an-email");

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(badRequest)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.email").exists());
    }

    // ─── Not Found ────────────────────────────────────────────────
    @Test
    void getUser_whenNotFound_returns404() throws Exception {
        given(userService.findById(999L)).willThrow(new UserNotFoundException("Not found"));

        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }

    // ─── With Security ────────────────────────────────────────────
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminEndpoint_withAdminRole_returns200() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }

    @Test
    void adminEndpoint_withoutAuth_returns401() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## Repository Testing with @DataJpaTest

Loads only the JPA layer (repositories, entities, Hibernate). Uses an **in-memory H2 database** by default.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;    // Useful for setting up test data

    @BeforeEach
    void setUp() {
        entityManager.persistAndFlush(new User("Alice", "alice@test.com", Status.ACTIVE));
        entityManager.persistAndFlush(new User("Bob", "bob@test.com", Status.INACTIVE));
    }

    @Test
    void findByEmail_whenExists_returnsUser() {
        Optional<User> result = userRepository.findByEmail("alice@test.com");

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("Alice");
    }

    @Test
    void findByStatus_returnsMatchingUsers() {
        List<User> active = userRepository.findByStatus(Status.ACTIVE);

        assertThat(active).hasSize(1);
        assertThat(active.get(0).getName()).isEqualTo("Alice");
    }

    @Test
    void save_persistsEntity() {
        User user = new User("Charlie", "charlie@test.com", Status.ACTIVE);
        User saved = userRepository.save(user);

        assertThat(saved.getId()).isNotNull();
        assertThat(userRepository.count()).isEqualTo(3);
    }
}

// ─── Use real database instead of H2 ───────────────────────────────
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:mysql://localhost:3306/testdb",
    "spring.datasource.username=test",
    "spring.datasource.password=test"
})
class UserRepositoryWithRealDbTest { ... }
```

---

## Integration Testing with @SpringBootTest

Loads the full application context. Use `@ActiveProfiles("test")` to load test configuration.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureTestDatabase     // Use H2 for tests
@Transactional                 // Roll back after each test
class UserIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;   // Real HTTP calls

    @Autowired
    private UserRepository userRepository;

    @MockBean
    private EmailService emailService;        // Mock external service

    @Test
    void createAndFetchUser_fullFlow() {
        // Create
        CreateUserRequest request = new CreateUserRequest("Alice", "alice@test.com");
        ResponseEntity<UserDto> createResp = restTemplate.postForEntity(
            "/api/users", request, UserDto.class
        );
        assertThat(createResp.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        Long id = createResp.getBody().getId();

        // Fetch
        ResponseEntity<UserDto> getResp = restTemplate.getForEntity(
            "/api/users/" + id, UserDto.class
        );
        assertThat(getResp.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResp.getBody().getEmail()).isEqualTo("alice@test.com");
    }

    @Test
    void deleteUser_removesFromDatabase() {
        User user = userRepository.save(new User("Bob", "bob@test.com"));

        restTemplate.delete("/api/users/" + user.getId());

        assertThat(userRepository.existsById(user.getId())).isFalse();
    }
}
```

---

## Testing Annotations Cheat Sheet

| Annotation | What It Does |
|------------|-------------|
| `@SpringBootTest` | Full context; use for integration tests |
| `@WebMvcTest(Ctrl.class)` | Web layer only; use MockMvc |
| `@DataJpaTest` | JPA layer + H2; auto-rollback per test |
| `@DataMongoTest` | MongoDB repositories only |
| `@WebFluxTest` | Reactive web layer |
| `@JsonTest` | Jackson serialization/deserialization only |
| `@RestClientTest` | Test `RestTemplate` / `RestTemplateBuilder` |
| `@MockBean` | Add Mockito mock to Spring context |
| `@SpyBean` | Add Mockito spy (real + monitored) to context |
| `@TestConfiguration` | Provide extra beans for tests only |
| `@ActiveProfiles("test")` | Activate test profile |
| `@TestPropertySource` | Override properties for a test class |
| `@Sql("setup.sql")` | Run SQL before/after test |
| `@Transactional` | Roll back each test (auto in `@DataJpaTest`) |
| `@AutoConfigureMockMvc` | Add MockMvc to a `@SpringBootTest` |
| `@WithMockUser` | Run test with a mock authenticated user |
| `@WithUserDetails` | Run test with user from `UserDetailsService` |

---

[← Spring Security](./07_Spring_Security.md) &nbsp;|&nbsp; [Next → Profiles](./10_Profiles.md)
