# 11 — Exception Handling & REST APIs

[← Back to Index](./README.md)

---

## Table of Contents
- [Exception Hierarchy](#exception-hierarchy)
- [Global Exception Handler](#global-exception-handler)
- [Custom Exceptions](#custom-exceptions)
- [REST API Best Practices](#rest-api-best-practices)
- [Full REST Controller Example](#full-rest-controller-example)
- [HTTP Status Code Reference](#http-status-code-reference)

---

## Exception Hierarchy

```
Throwable
├── Error                          (JVM-level, don't catch)
└── Exception
    ├── IOException                (checked)
    ├── SQLException               (checked)
    └── RuntimeException           (unchecked — Spring handles these)
        ├── IllegalArgumentException
        ├── IllegalStateException
        ├── DataAccessException    (Spring's data access hierarchy)
        └── Your custom exceptions
```

---

## Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ─── 404 Not Found ────────────────────────────────────────────
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiError handleNotFound(ResourceNotFoundException ex, WebRequest request) {
        return ApiError.builder()
            .status(404)
            .error("NOT_FOUND")
            .message(ex.getMessage())
            .path(extractPath(request))
            .timestamp(LocalDateTime.now())
            .build();
    }

    // ─── 400 Validation Error ─────────────────────────────────────
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid value",
                (e1, e2) -> e1    // Keep first error per field
            ));

        return ApiError.builder()
            .status(400)
            .error("VALIDATION_FAILED")
            .message("Input validation failed")
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
    }

    // ─── 409 Conflict ─────────────────────────────────────────────
    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ApiError handleConflict(DuplicateResourceException ex) {
        return ApiError.of(409, "CONFLICT", ex.getMessage());
    }

    // ─── 403 Forbidden ────────────────────────────────────────────
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ApiError handleAccessDenied(AccessDeniedException ex) {
        return ApiError.of(403, "FORBIDDEN", "You do not have permission to perform this action");
    }

    // ─── 400 Bad Request (path variable type mismatch) ────────────
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        return ApiError.of(400, "BAD_REQUEST",
            "Parameter '" + ex.getName() + "' must be of type " + ex.getRequiredType().getSimpleName());
    }

    // ─── 500 Catch-All ────────────────────────────────────────────
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiError handleAll(Exception ex) {
        log.error("Unexpected error", ex);
        return ApiError.of(500, "INTERNAL_SERVER_ERROR", "An unexpected error occurred");
    }

    private String extractPath(WebRequest request) {
        return request.getDescription(false).replace("uri=", "");
    }
}
```

---

## Custom Exceptions

```java
// Base exception
public abstract class AppException extends RuntimeException {
    private final HttpStatus status;

    protected AppException(String message, HttpStatus status) {
        super(message);
        this.status = status;
    }

    public HttpStatus getStatus() { return status; }
}

// Specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resourceName, Object id) {
        super(resourceName + " not found with id: " + id, HttpStatus.NOT_FOUND);
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String message) {
        super(message, HttpStatus.CONFLICT);
    }
}

public class BusinessRuleViolationException extends AppException {
    public BusinessRuleViolationException(String message) {
        super(message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

```java
// Error response DTO
@Data
@Builder
public class ApiError {
    private int status;
    private String error;
    private String message;
    private String path;
    private LocalDateTime timestamp;
    private Map<String, String> fieldErrors;

    public static ApiError of(int status, String error, String message) {
        return ApiError.builder()
            .status(status).error(error).message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

---

## REST API Best Practices

### 1. Consistent URL Structure

```
GET    /api/v1/users           → list all
GET    /api/v1/users/{id}      → get one
POST   /api/v1/users           → create
PUT    /api/v1/users/{id}      → full replace
PATCH  /api/v1/users/{id}      → partial update
DELETE /api/v1/users/{id}      → delete

GET    /api/v1/users/{id}/orders       → nested resource list
GET    /api/v1/users/{id}/orders/{oid} → nested resource
```

### 2. Versioning

```
/api/v1/users   ← URI versioning (most common, visible)
/api/v2/users

# Or via header: Accept: application/vnd.myapi.v2+json
```

### 3. Request / Response DTOs

Never expose entities directly. Use separate request and response objects.

```java
// Request DTO
public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @NotBlank @Size(min = 8) String password
) {}

// Response DTO (never include password!)
public record UserResponse(
    Long id,
    String name,
    String email,
    String status,
    LocalDateTime createdAt
) {}
```

### 4. Use ResponseEntity for Control

```java
// Explicit status + headers
return ResponseEntity
    .status(HttpStatus.CREATED)
    .header("X-Resource-Id", created.getId().toString())
    .location(URI.create("/api/v1/users/" + created.getId()))
    .body(created);

// Simple cases
return ResponseEntity.ok(resource);
return ResponseEntity.noContent().build();   // 204
return ResponseEntity.notFound().build();    // 404
```

---

## Full REST Controller Example

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public ResponseEntity<Page<UserResponse>> getAll(
        @RequestParam(defaultValue = "0")    int page,
        @RequestParam(defaultValue = "20")   int size,
        @RequestParam(defaultValue = "id")   String sortBy,
        @RequestParam(defaultValue = "asc")  String direction
    ) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.fromString(direction), sortBy));
        return ResponseEntity.ok(userService.findAll(pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getById(@PathVariable @Positive Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> replace(
        @PathVariable Long id,
        @Valid @RequestBody UpdateUserRequest request
    ) {
        return ResponseEntity.ok(userService.replace(id, request));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<UserResponse> patch(
        @PathVariable Long id,
        @RequestBody Map<String, Object> fields
    ) {
        return ResponseEntity.ok(userService.patch(id, fields));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/{id}/orders")
    public ResponseEntity<Page<OrderResponse>> getUserOrders(
        @PathVariable Long id,
        Pageable pageable
    ) {
        return ResponseEntity.ok(userService.getOrders(id, pageable));
    }
}
```

---

## HTTP Status Code Reference

| Code | Name | When to Use |
|------|------|-------------|
| **2xx Success** | | |
| 200 | OK | GET, PUT, PATCH succeeded |
| 201 | Created | POST created a resource |
| 202 | Accepted | Async operation accepted |
| 204 | No Content | DELETE succeeded; or PUT with no body |
| **3xx Redirect** | | |
| 301 | Moved Permanently | Resource permanently moved |
| 302 | Found | Temporary redirect |
| 304 | Not Modified | ETag/If-Modified-Since matched |
| **4xx Client Errors** | | |
| 400 | Bad Request | Malformed JSON, validation failure |
| 401 | Unauthorized | Not authenticated (no/invalid token) |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported |
| 409 | Conflict | Duplicate resource, state conflict |
| 410 | Gone | Resource permanently deleted |
| 415 | Unsupported Media Type | Wrong Content-Type |
| 422 | Unprocessable Entity | Semantically invalid (business rule) |
| 429 | Too Many Requests | Rate limit exceeded |
| **5xx Server Errors** | | |
| 500 | Internal Server Error | Unexpected server failure |
| 502 | Bad Gateway | Upstream service failure |
| 503 | Service Unavailable | App not ready (overloaded / down) |
| 504 | Gateway Timeout | Upstream service timeout |

---

[← Profiles](./10_Profiles.md) &nbsp;|&nbsp; [Next → Caching](./12_Caching.md)
