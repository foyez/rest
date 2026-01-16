# API Design & Development Guide

A comprehensive guide to API design, security, and best practices.

## Table of Contents

### 1. Fundamentals
- [1.1 REST API Basics](01-fundamentals/01-rest-api-basics.md)
- [1.2 HTTP Methods](01-fundamentals/02-http-methods.md)
- [1.3 HTTP Status Codes](01-fundamentals/03-http-status-codes.md)
- [1.4 API Design Principles](01-fundamentals/04-api-design-principles.md)

### 2. Authentication & Authorization
- [2.1 Authentication vs Authorization](02-auth/01-auth-basics.md)
- [2.2 Hashing & Salting](02-auth/02-hashing-salting.md)
- [2.3 JWT Tokens](02-auth/03-jwt-tokens.md)
- [2.4 Refresh Tokens](02-auth/04-refresh-tokens.md)
- [2.5 OAuth 2.0](02-auth/05-oauth.md)
- [2.6 Session-Based Authentication](02-auth/06-session-auth.md)

### 3. API Versioning
- [3.1 Versioning Strategies](03-versioning/01-versioning-strategies.md)
- [3.2 Breaking vs Non-Breaking Changes](03-versioning/02-breaking-changes.md)

### 4. Request & Response Patterns
- [4.1 Pagination](04-patterns/01-pagination.md)
- [4.2 Filtering & Sorting](04-patterns/02-filtering-sorting.md)
- [4.3 Error Handling](04-patterns/03-error-handling.md)
- [4.4 Rate Limiting](04-patterns/04-rate-limiting.md)

### 5. Security Best Practices
- [5.1 HTTPS & TLS](05-security/01-https-tls.md)
- [5.2 CORS](05-security/02-cors.md)
- [5.3 API Keys](05-security/03-api-keys.md)
- [5.4 Input Validation](05-security/04-input-validation.md)
- [5.5 SQL Injection Prevention](05-security/05-sql-injection.md)

### 6. Advanced Topics
- [6.1 Idempotency](06-advanced/01-idempotency.md)
- [6.2 Caching Strategies](06-advanced/02-caching.md)
- [6.3 Webhooks](06-advanced/03-webhooks.md)
- [6.4 GraphQL vs REST](06-advanced/04-graphql-vs-rest.md)

### 7. Testing & Documentation
- [7.1 API Testing Strategies](07-testing/01-testing-strategies.md)
- [7.2 API Documentation](07-testing/02-documentation.md)

### 8. Real-World Examples
- [8.1 Building a Secure API in Go](08-examples/01-secure-api-go.md)
- [8.2 Common API Patterns](08-examples/02-common-patterns.md)

---

## Quick Reference

- [HTTP Status Code Cheat Sheet](01-fundamentals/03-http-status-codes.md#status-code-cheat-sheet)
- [Authentication Flow Diagrams](02-auth/01-auth-basics.md#authentication-flows)
- [API Design Checklist](01-fundamentals/04-api-design-principles.md#design-checklist)