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
---

## 4. Akamai

### 4.1. Kiến trúc tổng quan

  - Một trong các CDN lớn & lâu đời nhất, rất nhiều PoP toàn cầu
  - Mạnh về:
    - Media delivery
    - Enterprise security
    - Các khách hàng enterprise: bank, OTT, e-commerce lớn…
      
```text
User
  |
  |  www.bank.com (CNAME -> bank.akamai.net)
  v
+----------------------------+
|         Akamai Edge        |
|  (WAF + DDoS + Cache)      |
+----------------------------+
          |
   Cache HIT ? -------------+
          |                 |
        No|                 |Yes
          v                 v
+----------------------------+
|          Origin            |
| (DC của khách hàng / Cloud)|
+----------------------------+

```

### 4.2. Workflow tổng quan

  1. DNS: www.site.com → CNAME xxx.akamai.net
  2. User được route tới Akamai Edge gần nhất
  3. Edge:
    - Chạy Kona Site Defender / WAF / Bot Manager (tùy license)
    - Kiểm tra cache theo cấu hình (Property Configuration)
  4. Cache MISS:
    - Proxy về origin (DC on-prem, cloud, hybrid)
  5. Origin trả về:
    - Akamai cache (nếu rule cho phép)
    - Trả về cho user


### 4.3. Điểm chính

  - Cấu hình qua Property Manager (mạnh & chi tiết, nhưng phức tạp hơn)
  - Thường được dùng với:
    - Bank / Financial (tích hợp bảo mật, SLA cao)
    - Media streaming lớn (HLS/DASH, live, VOD)
  - Tích hợp với SIEM/SOC, logging rất chi tiết

---

## 5. Nginx DIY CDN (tự xây CDN bằng Nginx)

### 5.1. Kiến trúc tổng quan

Ở đây không có “dịch vụ CDN managed”. Bạn tự triển khai:

  - Nhiều node Nginx ở nhiều region / DC (PoP)
  - DNS / Anycast để điều hướng user tới PoP gần nhất
  - Nginx làm reverse proxy + cache đến origin trung tâm
```text
         Internet / Users Global
     +-------------------------------+
     |                               |
User VN                        User US/EU
  |                                  |
  | www.example.com                  | www.example.com
  v                                  v
+----------+                   +----------+
|   DNS    | --- Geo/Latency-->|   DNS    |
+----+-----+                   +----+-----+
     |                              |
     v                              v
+--------------+              +--------------+
| Nginx Edge   |              | Nginx Edge   |
| Asia (SG/VN) |              | US/EU        |
+------+-------+              +------+-------+
       \                             /
        \                           /
         v                         v
              +----------------+
              |    Origin      |
              |  (Central DC)  |
              +----------------+

```

### 5.2. Workflow

  1. User request https://www.example.com/image.jpg
  2. DNS (GeoDNS / latency-based) trả IP Nginx Edge gần nhất
  3. Request tới Nginx Edge:
    - Check cache (proxy_cache)
    - Nếu HIT → trả file
    - Nếu MISS → proxy tới origin (central DC/S3/cluster khác)
  4. Origin trả về → Nginx Edge cache file → trả user

### 5.3. Ví dụ config Nginx Edge (rút gọn)
```text
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cdn_cache:100m
                 max_size=20g inactive=1d use_temp_path=off;

upstream origin_backend {
    server origin.example.internal:80;
}

server {
    listen 80;
    server_name www.example.com;

    location ~* \.(png|jpe?g|gif|css|js)$ {
        proxy_pass http://origin_backend;

        proxy_cache        cdn_cache;
        proxy_cache_valid  200 301 302 1h;
        proxy_cache_valid  404         5m;

        add_header X-Cache $upstream_cache_status always;
    }

    location / {
        proxy_pass http://origin_backend;
        # có thể cache HTML ngắn hạn (micro-caching) tùy use case
    }
}

```

### 5.4. Đặc điểm

Ưu:

  - Quyền kiểm soát tuyệt đối (security, location, compliance, data residency)
  - Phù hợp tổ chức lớn có DC nhiều nơi, đội ngũ ops mạnh
    
Nhược:

  - Bạn phải lo mọi thứ:
    - Mua/bố trí server global
    - DNS / Anycast
    - Monitoring, alerting, autoscaling, failover
    - Bảo mật (WAF, DDoS…) – thường phải mua/thêm sản phẩm khác
  - Khó đạt scale + reliability như CloudFront/Cloudflare/Akamai nếu không đầu tư lớn

---

## 6. So Sánh Nhanh

| Tiêu chí | CloudFront | Cloudflare | Akamai | Nginx DIY CDN |
|----------|------------|------------|--------|---------------|
| **Loại dịch vụ** | Managed CDN (AWS) | Managed CDN + Security Suite | Managed CDN Enterprise | Tự xây, tự vận hành |
| **Edge global** | Có (PoP AWS) | Có (mạng riêng Cloudflare) | Có (rất rộng, truyền thống) | Tuỳ bạn triển khai |
| **Origin phổ biến** | S3, ALB, EC2, API GW | Web server, LB bất kỳ | DC on-prem, cloud, hybrid | Origin riêng (DC, cloud) |
| **Tích hợp AWS** | Rất sâu | Gián tiếp (qua DNS/proxy) | Gián tiếp | Bạn tự tích hợp |
| **Bảo mật tích hợp** | AWS WAF, Shield, SigV4, OAC | WAF, DDoS, Bot, Zero Trust | Kona/WAF, DDoS, Bot | Phải thêm giải pháp riêng |
| **Edge compute** | Lambda@Edge, CF Functions | Cloudflare Workers | Akamai EdgeWorkers (tuỳ plan) | Không native, tự viết app Nginx |
| **Quản lý / vận hành** | Thấp | Thấp – trung | Trung – cao (tùy tổ chức) | Cao (tự lo hết) |
| **Use case điển hình** | App/website trên AWS, media | Web global, security, SaaS, startup | Bank, OTT lớn, enterprise phức tạp | Bank/tổ chức cần full control |
