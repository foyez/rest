# 5.1 HTTPS & TLS

## Why HTTPS?

**HTTP (Insecure):**
```
Client → [ Anyone can read ] → Server
Passwords visible
Tokens stolen
Data modified
```

**HTTPS (Secure):**
```
Client → [ Encrypted tunnel ] → Server
Only client and server can read
```

---

## What is TLS?

**TLS (Transport Layer Security)** = Protocol that encrypts HTTP traffic

```
HTTP + TLS = HTTPS
```

**Previously SSL:** TLS is the newer version of SSL

---

## How HTTPS Works

### TLS Handshake

```
┌─────────┐                                        ┌─────────┐
│ Client  │                                        │ Server  │
└────┬────┘                                        └────┬────┘
     │                                                  │
     │ 1. ClientHello                                   │
     │    (Supported ciphers, TLS version)              │
     │ ──────────────────────────────────────────────>  │
     │                                                  │
     │ 2. ServerHello                                   │
     │    (Chosen cipher, TLS version)                  │
     │ <────────────────────────────────────────────────│
     │                                                  │
     │ 3. Certificate                                   │
     │    (Server's public key)                         │
     │ <────────────────────────────────────────────────│
     │                                                  │
     │ 4. Verify certificate                            │
     │                                                  │
     │ 5. Key exchange                                  │
     │    (Generate session key)                        │
     │ ──────────────────────────────────────────────>  │
     │                                                  │
     │ 6. Encrypted communication begins                │
     │ ══════════════════════════════════════════════>  │
```

---

## SSL/TLS Certificate

**Certificate contains:**
- Domain name (example.com)
- Public key
- Issued by (Certificate Authority)
- Valid dates
- Signature

**Certificate Authorities (CA):**
- Let's Encrypt (free)
- DigiCert
- Cloudflare
- AWS Certificate Manager

---

## Enforcing HTTPS

### Redirect HTTP to HTTPS

```go
func RedirectHTTPSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("X-Forwarded-Proto") != "https" {
            httpsURL := "https://" + r.Host + r.RequestURI
            http.Redirect(w, r, httpsURL, http.StatusMovedPermanently)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

### HSTS (HTTP Strict Transport Security)

**Force browser to use HTTPS:**

```go
func HSTSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Strict-Transport-Security", 
            "max-age=31536000; includeSubDomains; preload")
        
        next.ServeHTTP(w, r)
    })
}
```

**Header breakdown:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

max-age=31536000      → Remember for 1 year
includeSubDomains     → Apply to all subdomains
preload               → Add to browser's preload list
```

---

## Setting Up HTTPS

### Development (Self-Signed Certificate)

```bash
# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

```go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello HTTPS!"))
    })
    
    // Start HTTPS server
    log.Fatal(http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil))
}
```

---

### Production (Let's Encrypt)

**Option 1: Automatic (autocert)**

```go
package main

import (
    "log"
    "net/http"
    
    "golang.org/x/crypto/acme/autocert"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello HTTPS!"))
    })
    
    certManager := autocert.Manager{
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
        Cache:      autocert.DirCache("/var/www/.cache"),
    }
    
    server := &http.Server{
        Addr:      ":443",
        Handler:   mux,
        TLSConfig: certManager.TLSConfig(),
    }
    
    // Redirect HTTP to HTTPS
    go http.ListenAndServe(":80", certManager.HTTPHandler(nil))
    
    // Start HTTPS server
    log.Fatal(server.ListenAndServeTLS("", ""))
}
```

---

**Option 2: Manual (certbot)**

```bash
# Install certbot
sudo apt-get install certbot

# Get certificate
sudo certbot certonly --standalone -d example.com

# Certificate saved to:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

```go
http.ListenAndServeTLS(":443",
    "/etc/letsencrypt/live/example.com/fullchain.pem",
    "/etc/letsencrypt/live/example.com/privkey.pem",
    nil)
```

---

## Security Headers

### 1. Strict-Transport-Security (HSTS)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

Forces HTTPS for 1 year.

---

### 2. Content-Security-Policy

```
Content-Security-Policy: default-src 'self'
```

Prevents XSS by restricting resource sources.

---

### 3. X-Frame-Options

```
X-Frame-Options: DENY
```

Prevents clickjacking.

---

### 4. X-Content-Type-Options

```
X-Content-Type-Options: nosniff
```

Prevents MIME type sniffing.

---

### Complete Security Headers Middleware

```go
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // HSTS
        w.Header().Set("Strict-Transport-Security", 
            "max-age=31536000; includeSubDomains; preload")
        
        // Prevent clickjacking
        w.Header().Set("X-Frame-Options", "DENY")
        
        // Prevent MIME sniffing
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // XSS protection
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        
        // CSP
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        
        // Referrer policy
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // Permissions policy
        w.Header().Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        next.ServeHTTP(w, r)
    })
}
```

---

## TLS Best Practices

### 1. Use TLS 1.2 or Higher

```go
server := &http.Server{
    TLSConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
}
```

**Avoid:**
- SSLv2 (broken)
- SSLv3 (broken)
- TLS 1.0 (deprecated)
- TLS 1.1 (deprecated)

---

### 2. Use Strong Cipher Suites

```go
server := &http.Server{
    TLSConfig: &tls.Config{
        CipherSuites: []uint16{
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        },
    },
}
```

---

### 3. Enable HTTP/2

```go
server := &http.Server{
    TLSConfig: &tls.Config{
        NextProtos: []string{"h2", "http/1.1"},
    },
}
```

---

### 4. Certificate Pinning (Advanced)

Pin specific certificates to prevent MITM:

```go
expectedCertHash := "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="

func verifyCertificate(rawCerts [][]byte, verifiedChains [][]*x509.Certificate) error {
    for _, cert := range rawCerts {
        hash := sha256.Sum256(cert)
        encodedHash := base64.StdEncoding.EncodeToString(hash[:])
        
        if "sha256/"+encodedHash == expectedCertHash {
            return nil
        }
    }
    
    return errors.New("certificate pin mismatch")
}
```

---

## Mixed Content

**Problem:** HTTPS page loads HTTP resources

```html
<!-- ❌ Mixed content (insecure) -->
<script src="http://example.com/script.js"></script>

<!-- ✅ Secure -->
<script src="https://example.com/script.js"></script>
```

**Solution:**
```html
<!-- Protocol-relative URL -->
<script src="//example.com/script.js"></script>
```

---

## Testing HTTPS

### Check Certificate

```bash
openssl s_client -connect example.com:443 -servername example.com
```

### Online Tools

- SSL Labs: https://www.ssllabs.com/ssltest/
- Security Headers: https://securityheaders.com/

---

## Common Issues

### Issue 1: Certificate Expired

**Error:** "Your connection is not private"

**Solution:** Renew certificate

```bash
sudo certbot renew
```

---

### Issue 2: Self-Signed Certificate Warning

**Browser:** "Your connection is not private"

**Solution:** Use proper CA certificate in production

---

### Issue 3: Mixed Content

**Console:** "Mixed Content: The page was loaded over HTTPS, but requested an insecure resource"

**Solution:** Use HTTPS for all resources

---

## HTTPS vs HTTP Performance

**Myth:** HTTPS is slower

**Reality:**
- Modern TLS is fast
- HTTP/2 only works over HTTPS
- HTTP/2 is faster than HTTP/1.1
- **Net result: HTTPS with HTTP/2 is faster**

---

## Complete Production Setup

```go
package main

import (
    "crypto/tls"
    "log"
    "net/http"
    "time"
    
    "golang.org/x/crypto/acme/autocert"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", HomeHandler)
    
    // Add security middleware
    handler := SecurityHeadersMiddleware(mux)
    handler = RedirectHTTPSMiddleware(handler)
    
    // Certificate manager
    certManager := autocert.Manager{
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("example.com"),
        Cache:      autocert.DirCache("/var/www/.cache"),
    }
    
    // HTTPS server
    server := &http.Server{
        Addr:         ":443",
        Handler:      handler,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
        TLSConfig: &tls.Config{
            MinVersion:               tls.VersionTLS12,
            PreferServerCipherSuites: true,
            CipherSuites: []uint16{
                tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
                tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            },
            GetCertificate: certManager.GetCertificate,
        },
    }
    
    // HTTP redirect server
    go func() {
        httpServer := &http.Server{
            Addr:    ":80",
            Handler: certManager.HTTPHandler(nil),
        }
        log.Fatal(httpServer.ListenAndServe())
    }()
    
    // Start HTTPS
    log.Fatal(server.ListenAndServeTLS("", ""))
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. HTTPS is HTTP with __________ encryption.
2. The __________ header forces browsers to use HTTPS.
3. Let's Encrypt provides __________ SSL certificates.
4. TLS version __________ or higher should be used.

### True/False

1. ⬜ HTTPS makes websites slower
2. ⬜ Self-signed certificates should be used in production
3. ⬜ HSTS header forces browsers to use HTTPS
4. ⬜ HTTP/2 works over HTTP
5. ⬜ Mixed content is when HTTPS page loads HTTP resources

### Multiple Choice

1. What does HSTS do?
   - A) Encrypts data
   - B) Forces HTTPS usage
   - C) Validates certificates
   - D) Compresses responses

2. Which TLS version should you avoid?
   - A) TLS 1.2
   - B) TLS 1.3
   - C) TLS 1.0
   - D) All are safe

3. What's the best free certificate provider?
   - A) DigiCert
   - B) Let's Encrypt
   - C) Cloudflare
   - D) Self-signed

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. TLS
2. Strict-Transport-Security (or HSTS)
3. free
4. 1.2

**True/False:**
1. ❌ False - HTTP/2 over HTTPS is actually faster
2. ❌ False - Use proper CA certificates in production
3. ✅ True - HSTS forces browsers to always use HTTPS
4. ❌ False - HTTP/2 requires HTTPS (in browsers)
5. ✅ True - Mixed content is security risk

**Multiple Choice:**
1. **B** - HSTS forces browsers to use HTTPS
2. **C** - TLS 1.0 is deprecated and insecure
3. **B** - Let's Encrypt is free and widely trusted

</details>

</details>