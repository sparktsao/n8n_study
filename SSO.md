curl -v -X POST "https://sso.trendops.co/tokens/verify" \
  -H "Content-Type: application/json" \
  -d '{
    "request":{"token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjZKeXEzS282IiwidHlwIjoiSldUIn0.eyJzdWIiOiJ1c2VyX2E2Mjc4YWFiMzE5ODQ2MWYiLCJpc3MiOiJUcmVuZCBTU08gSldUIiwiYXVkIjoiVHJlbmQgU1NPIEpXVCIsImV4cCI6MTc1MjUxMjc5MSwiaWF0IjoxNzUyNTEwOTkxLCJuYmYiOjE3NTI1MTA5OTEsImp0aSI6Imp3dF9mMGU1ZGU0NTRmNWM0MTJlIiwidG9rZW5fdHlwZSI6ImFjY2VzcyIsInVzZXJfaWQiOiJ1c2VyX2E2Mjc4YWFiMzE5ODQ2MWYiLCJlbWFpbCI6InNwYXJrX3RzYW9AdHJlbmRtaWNyby5jb20iLCJuYW1lIjoiU3BhcmsgVHNhbyIsInNjb3BlIjpbInJlYWQ6cHJvZmlsZSIsIndyaXRlOnByb2ZpbGUiXSwicGVybWlzc2lvbnMiOlsidXNlcjpyZWFkIiwidXNlcjp3cml0ZSJdLCJyb2xlcyI6WyJ1c2VyIl0sImlwX2FkZHJlc3MiOiIzMi4xNDIuMTcwLjE0NiIsInVzZXJfYWdlbnQiOiJNb3ppbGxhLzUuMCAoTWFjaW50b3NoOyBJbnRlbCBNYWMgT1MgWCAxMF8xNV83KSBBcHBsZVdlYktpdC81MzcuMzYgKEtIVE1MLCBsaWtlIEdlY2tvKSBDaHJvbWUvMTM4LjAuMC4wIFNhZmFyaS81MzcuMzYiLCJtZXRhZGF0YSI6eyJvYXV0aF9wcm92aWRlciI6Imdvb2dsZSIsIm9hdXRoX3N1YiI6IjEwODMwOTE5NDIyMTk0NzM4Njk4MiIsImxvZ2luX21ldGhvZCI6Im9hdXRoX2NhbGxiYWNrIn19.AZ04ulsvFbKEyuj4GeBBem43uR00oyyczdtZk0s2HtOOz_kdwD5POmSN8rz8VUM3iKxWvFdeLUS4T79F9xjHs412CqZRq6rhf7Y__wEmBayEELHOqBQsVLQYHNVKZgiMElPW4JNkjYcnoGcQdJ4wCTotgiY5eLl8-9ORyKoH9_-Ve9neyNwJLl8ptfrcZ9JdDi7pdiF89uFddKfcLTrKUBqGFHcTB_TU6gNMeijf5Y48lPHNps9f5oRfVtX2Ba3HG0TSylH62_imzdgv7dzTPpN8aYRSNJiXqcHvAyLjdQagFo16CoyJtOXlMlzGVMgzcDgffxKwM4oZg_B9iWzzUg",
    "token_type": "access"
  }}'


  Note: Unnecessary use of -X or --request, POST is already inferred.
* Host sso.trendops.co:443 was resolved.
* IPv6: (none)
* IPv4: 34.217.152.160
*   Trying 34.217.152.160:443...
* Connected to sso.trendops.co (34.217.152.160) port 443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=sso.trendops.co
*  start date: Jul 14 12:33:27 2025 GMT
*  expire date: Oct 12 12:33:26 2025 GMT
*  subjectAltName: host "sso.trendops.co" matched cert's "sso.trendops.co"
*  issuer: C=US; O=Let's Encrypt; CN=R10
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://sso.trendops.co/tokens/verify
* [HTTP/2] [1] [:method: POST]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: sso.trendops.co]
* [HTTP/2] [1] [:path: /tokens/verify]
* [HTTP/2] [1] [user-agent: curl/8.7.1]
* [HTTP/2] [1] [accept: */*]
* [HTTP/2] [1] [content-type: application/json]
* [HTTP/2] [1] [content-length: 1312]
> POST /tokens/verify HTTP/2
> Host: sso.trendops.co
> User-Agent: curl/8.7.1
> Accept: */*
> Content-Type: application/json
> Content-Length: 1312
> 
* upload completely sent off: 1312 bytes
< HTTP/2 200 
< content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://accounts.google.com https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdn.jsdelivr.net; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:; connect-src 'self' https://accounts.google.com https://oauth2.googleapis.com; frame-src 'self' https://accounts.google.com; worker-src 'self' blob:; object-src 'none'; base-uri 'self';
< content-type: application/json
< date: Mon, 14 Jul 2025 16:36:48 GMT
< permissions-policy: geolocation=(), microphone=(), camera=(), payment=(), usb=(), magnetometer=(), gyroscope=()
< referrer-policy: strict-origin-when-cross-origin
< server: uvicorn
< strict-transport-security: max-age=315360000; includeSubDomains; preload
< x-content-type-options: nosniff
< x-frame-options: DENY
< x-process-time: 0.0016112327575683594
< x-ratelimit-limit: 100
< x-ratelimit-remaining: 99
< x-ratelimit-reset: 2025-07-14T16:37:48.741736
< x-ratelimit-window: 60
< x-xss-protection: 1; mode=block
< content-length: 712
< 
* Connection #0 to host sso.trendops.co left intact
{"valid":true,"token_data":{"sub":"user_a6278aab3198461f","iss":"Trend SSO JWT","aud":"Trend SSO JWT","exp":1752512791,"iat":1752510991,"nbf":1752510991,"jti":"jwt_f0e5de454f5c412e","token_type":"access","user_id":"user_a6278aab3198461f","email":"spark_tsao@trendmicro.com","name":"Spark Tsao","scope":["read:profile","write:profile"],"permissions":["user:read","user:write"],"roles":["user"],"session_id":null,"device_id":null,"ip_address":"32.142.170.146","user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36","metadata":{"oauth_provider":"google","oauth_sub":"108309194221947386982","login_method":"oauth_callback"}},"error":null}%      