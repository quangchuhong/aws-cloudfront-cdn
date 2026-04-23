# Tài liệu workflow các dịch vụ CDN  
## (Amazon CloudFront, Cloudflare, Akamai, Nginx DIY CDN)

Mục tiêu README này:  
- Mô tả **workflow luồng request** cho từng CDN  
- Nêu **kiến trúc high-level**, dễ dùng để onboard dev/infrastructure  
- Có thể dùng làm base cho tài liệu nội bộ / kiến trúc hệ thống

---

## 1. Khái niệm chung về CDN

**CDN (Content Delivery Network)** là mạng lưới các máy chủ (Edge/PoP) phân tán trên toàn cầu, giúp:

- Phục vụ nội dung từ **edge gần người dùng nhất**  
- Giảm tải cho **origin** (web server, API, storage)  
- Tăng tốc độ truy cập (giảm latency, tăng cache hit)  
- Hỗ trợ **bảo mật** (WAF, DDoS protection, bot mitigation) tùy từng nhà cung cấp

Luồng tổng quát:

```text
User -> DNS -> CDN Edge (cache) -> [Origin nếu cache miss] -> Edge -> User
```

---

## 2. Amazon CloudFront

### 2.1. Kiến trúc tổng quan

  - **Managed CDN của AWS**
  - Tích hợp sâu với:
    - S3, ALB, EC2, API Gateway, Media Services…
  - Edge locations & Regional Edge Caches toàn cầu
```text
User
  |
  |  www.example.com (CNAME -> dxxxx.cloudfront.net)
  v
+----------------------------+
|   CloudFront Edge POP      |
+----------------------------+
          |
   Cache HIT ? -------------+
          |                 |
        No|                 |Yes
          v                 v
+----------------------------+      (trả từ cache)
|   Regional Edge Cache      |
+----------------------------+
          |
          v
+----------------------------+
|          Origin            |
| (S3 / ALB / EC2 / API GW)  |
+----------------------------+

```

### 2.2. Workflow chi tiết

  1. User request: https://www.example.com/page.html
  2. DNS (Route 53 hoặc DNS khác) trả về dxxxx.cloudfront.net
  3. User được route tới Edge gần nhất (anycast + DNS)
  4. Edge kiểm tra cache:
    - Nếu HIT → trả response ngay
    - Nếu MISS:
      - Gửi request tới Regional Edge Cache (nếu dùng)
      - Nếu vẫn MISS → gửi đến Origin (S3/ALB/EC2…)
  5. Origin trả response:
    - CloudFront lưu nội dung vào cache (Edge/Regional) theo Cache Policy (TTL, cache key…)
    - Trả response về cho User

### 2.3. Điểm chính

  - **Behaviors**: định tuyến theo path (/static/*, /api/* → origin & cache policy khác nhau)
  - **Cache Policy**: quyết định TTL, cache key (headers/cookies/query)
  - **Signed URL / Cookies + OAI/OAC**: bảo vệ nội dung private (S3 private + chỉ cho CloudFront đọc)
  - **Lambda@Edge / CloudFront Functions**: custom logic tại edge (rewrite URL, header, auth nhẹ)
    
---

## 3. Cloudflare

### 3.1. Kiến trúc tổng quan

  - CDN + Security (WAF, DDoS, Bot, Zero Trust…)
  - Hoạt động như reverse proxy cho domain của bạn
```text
User
  |
  |  www.example.com (NS/CNAME -> Cloudflare)
  v
+----------------------------+
|     Cloudflare Edge        |
| (Proxy + WAF + Cache)      |
+----------------------------+
          |
   Cache HIT ? -------------+
          |                 |
        No|                 |Yes
          v                 v
+----------------------------+
|          Origin            |
| (Web server / Load bal)   |
+----------------------------+

```

### 3.2. Workflow

  1. Domain dùng Cloudflare (thường đổi NS về Cloudflare)
  2. User → DNS Cloudflare → trả về IP anycast của Cloudflare
  3. Request vào Cloudflare Edge:
    - Áp dụng WAF, DDoS protection, rate limiting, rules…
    - Kiểm tra cache (theo URL, headers, cache rules, page rules…)
  4. Nếu cache MISS → proxy request về origin của bạn:
    - Có thể: HTTP/HTTPS, custom ports, origin auth
  5. Origin trả về:
    - Cloudflare có thể cache (theo header hoặc rule)
    - Trả response về user
     
### 3.3. Điểm chính

  - **Page Rules / Cache Rules / Transform Rules**: điều chỉnh cache, rewrite, redirect
  - **Argo Smart Routing**: tối ưu đường truyền từ edge → origin
  - **Workers**: giống Lambda@Edge – chạy code (JS) tại edge
  - Tính năng bảo mật/cloud networking rất phong phú (WAF, Turnstile, Zero Trust…)
