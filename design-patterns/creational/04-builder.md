[← Abstract Factory](./03-abstract-factory.md) | [Index](../index.md)

---

# Pattern 4 — Builder
**Category:** Creational

---

## Intent

Separate the **construction** of a complex object from its **representation**, allowing the same construction process to create different representations.

---

## The Problem It Solves

A class with many optional parameters leads to telescoping constructors (`new Pizza("large", true, false, true, false, "extra cheese", null, 3)`). This is unreadable, error-prone, and inflexible. Builder solves this by providing a fluent API to set only the parameters you need.

---

## Java Implementation

```java
// The complex object
public class HttpRequest {
    private final String  method;
    private final String  url;
    private final Map<String, String> headers;
    private final String  body;
    private final int     timeoutMs;
    private final boolean followRedirects;

    // Private constructor — only the Builder can call it
    private HttpRequest(Builder builder) {
        this.method          = builder.method;
        this.url             = builder.url;
        this.headers         = Collections.unmodifiableMap(builder.headers);
        this.body            = builder.body;
        this.timeoutMs       = builder.timeoutMs;
        this.followRedirects = builder.followRedirects;
    }

    // Getters...
    public String getMethod()  { return method; }
    public String getUrl()     { return url; }
    public String getBody()    { return body; }

    @Override
    public String toString() {
        return String.format("%s %s (timeout=%dms, redirects=%b)%nHeaders: %s%nBody: %s",
                method, url, timeoutMs, followRedirects, headers, body);
    }

    // Static inner Builder class
    public static class Builder {
        // Required parameters
        private final String method;
        private final String url;

        // Optional parameters with defaults
        private Map<String, String> headers = new HashMap<>();
        private String  body            = null;
        private int     timeoutMs       = 5000;
        private boolean followRedirects = true;

        public Builder(String method, String url) {
            this.method = method;
            this.url    = url;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;    // fluent chaining
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int timeoutMs) {
            this.timeoutMs = timeoutMs;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public HttpRequest build() {
            // Validation
            if (method == null || method.isBlank())
