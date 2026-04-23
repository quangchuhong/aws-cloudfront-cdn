# AWS CloudFront – README Kiến Trúc & Workflow

Tài liệu này tổng hợp lại các nội dung đã trao đổi, dùng làm **tài liệu nội bộ** để:
- Onboard dev/infra về **CloudFront**
- Thiết kế CDN cho **báo điện tử / web traffic lớn**
- Hiểu workflow, caching, bảo mật, chi phí, và tích hợp Lambda@Edge

---

## 1. CloudFront là gì?

AWS CloudFront là **CDN managed** của AWS:

- Phân phối nội dung (HTML, CSS, JS, ảnh, video, API, file tải…) từ **Edge Location gần user**.
- Giảm tải origin (S3, ALB, EC2, on‑prem web server…).
- Hỗ trợ **cache thông minh, SSL/TLS, Signed URL, WAF, Lambda@Edge…**

---

## 2. Khái niệm: Distribution & Origin

### 2.1. Distribution

- Distribution = **“tuyến phát nội dung”/CDN endpoint** của bạn.
- Có domain dạng: `dxxxxx.cloudfront.net`.
- Cấu hình:
  - **Origins** (nguồn dữ liệu gốc)
  - **Behaviors** (routing theo path)
  - **Cache / Origin Request Policies**
  - **SSL/TLS, WAF, logging…**

### 2.2. Origin

Origin = **nơi chứa nội dung gốc**:

- AWS:
  - S3 (static files, website tĩnh)
  - ALB / EC2 / API Gateway
- On‑prem:
  - Web server trong DC (Nginx/Apache, F5, v.v.) qua **Custom Origin**
- Điều kiện: CloudFront phải **truy cập được** origin qua Internet (HTTP/HTTPS).

---

## 3. Workflow request CloudFront (tổng quát)

```text
User
  |
  |  www.example.com
  v
DNS (Route 53 / DNS khác)
  |
  |  CNAME / ALIAS -> dxxxx.cloudfront.net
  v
+-----------------------------+
| CloudFront Edge Location    |
+-----------------------------+
       |
       |  Cache HIT ? -------+
       |                     |
      No                     | Yes
       v                     v
+-----------------------------+   (trả từ cache)
| Regional Edge Cache (nếu có)|
+-----------------------------+
       |
       v
+-----------------------------+
|           Origin            |
| (S3 / ALB / EC2 / on-prem)  |
+-----------------------------+
```

Luồng:

  1. User → DNS → CloudFront distribution.
  2. CloudFront đưa user đến Edge gần nhất.
  3. Edge check cache:
    - HIT → trả về ngay.
    - MISS → (có thể) check Regional Edge Cache → nếu vẫn MISS → gọi Origin.
  4. Origin trả response → CloudFront cache lại theo TTL → trả cho user.

---

## 4. Behaviors & Routing theo path

**Behavior** = rule áp dụng cho 1 nhóm đường dẫn (path pattern):

Ví dụ:

  - /static/* → Origin S3 (cache mạnh).
  - /images/* → Origin S3 (TTL dài).
  - /api/* → Origin ALB (hầu như không cache).
  - * (Default) → Origin ALB/S3 (HTML trang báo, TTL trung bình).
      
Mỗi Behavior cấu hình:

  - Path pattern
  - Origin gắn kèm
  - Cache Policy (TTL, cache key)
  - Origin Request Policy (gửi gì xuống origin)
  - Viewer Protocol Policy (HTTP/HTTPS)
  - Allowed methods, compression, Lambda@Edge, v.v.
---

## 5. Caching trong CloudFront

### 5.1. Cache key

Cache key quyết định: 2 request có dùng chung cache không.

Thành phần:

  - Bắt buộc: Protocol + Host + Path + Method (GET/HEAD)
  - Tùy chọn (qua Cache Policy):
    - Query strings
    - Headers
    - Cookies
      
Nguyên tắc:

  - Càng thêm nhiều (headers/cookies/query) vào key → cache bị “vỡ vụn” → hit ratio giảm.
  - Static (CSS/JS/ảnh): thường không dùng cookie, ít header, query chỉ khi cần thiết (VD version file).
    
### 5.2. TTL (Time To Live)

Đặt trong Cache Policy:
  - DefaultTTL, MinTTL, MaxTTL
  - Có thể:
    - Tôn trọng header origin: Cache-Control, Expires
    - Hoặc override TTL tại CloudFront.
      
Gợi ý:

  - Static: 1h–24h+ (thậm chí 7 ngày nếu dùng tên file có hash).
  - HTML: 30–120s cho báo điện tử (update nhanh nhưng vẫn cache).
  - API: TTL=0 hoặc vài giây (micro-caching).
    
### 5.3. Caching theo Request Headers

Khi cần phân biệt theo header (ít dùng, phải cẩn thận):

  - Accept-Language: khi cùng 1 URL nhưng nội dung theo ngôn ngữ.
  - Một số header device (client hints) nếu render khác nhau mobile/desktop.
    
KHÔNG nên:

  - Cho toàn bộ User-Agent vào cache key (vỡ cache).
  - Cho header Authorization vào cache key cho public content (dễ leak & nát cache). → Thường: request có auth thì không cache.
---

## 6. Signed URLs, OAI & OAC

### 6.1. CloudFront Signed URLs / Signed Cookies

Dùng khi cần kiểm soát truy cập nội dung private qua CloudFront:

  - Backend kiểm tra quyền (login, đã mua…).
  - Backend tạo Signed URL (có expiry, policy, signature).
  - CloudFront validate signature:
    - Hợp lệ → cho tải nội dung.
    - Không hợp lệ/expired → 403.
      
### 6.2. OAI (Origin Access Identity)

  - “User đặc biệt” của CloudFront để đọc S3 private.
  - Dùng với S3:
    - S3 bucket private.
    - Bucket policy chỉ cho OAI s3:GetObject.
    - User phải đi qua CloudFront (có thể kèm Signed URL).
      
### 6.3. OAC (Origin Access Control) – khuyến nghị mới

  - Thay thế dần OAI cho S3.
  - Ký request từ CloudFront → S3 bằng SigV4.
  - Cho phép policy chặt chẽ hơn, là recommended cho triển khai mới.
    
Kết hợp chuẩn:

  - S3 private + OAC/OAI + CloudFront + Signed URL ⇒ không thể tải trực tiếp từ S3, chỉ qua CloudFront & có quyền.

---

## 7. SSL/TLS & SNI trong CloudFront

### 7.1. Viewer → CloudFront

  - Domain: www.example.com → CNAME/ALIAS → CloudFront.
  - Certificate:
    - Default: *.cloudfront.net.
    - Custom: ACM certificate cho www.example.com (ở us-east-1).
      
**Viewer Protocol Policy:**

  - HTTP and HTTPS
  - Redirect HTTP to HTTPS (khuyến nghị)
  - HTTPS only
    
### 7.2. SNI (Server Name Indication)

  - Giúp CloudFront phục vụ nhiều domain HTTPS trên cùng 1 IP.
  - CloudFront sử dụng SNI-only (mặc định, nên dùng):
    - Hầu hết browser/app hiện đại hỗ trợ.
    - Không cần dedicated IP, chi phí rẻ.

### 7.3. CloudFront → Origin

  - Có thể HTTP only, HTTPS only, hoặc Match Viewer.
  - Nếu origin dùng HTTPS:
    - Cấu hình min TLS (ví dụ TLSv1.2).
    - Verify cert và SNI theo origin domain name.
---

## 8. Lambda@Edge

**Lambda@Edge làm được gì?**

Bạn có thể:

- Sửa request của viewer trước khi CloudFront xử lý:

  - Thêm/bỏ header, cookie, query
  - Chuyển hướng URL, rewrite path (/ → /index.html, SEO URL,…)
  - Chặn/bật truy cập theo IP, country, user-agent…
    
- Sửa request từ CloudFront đến origin:

  - Thêm auth header (HMAC, token…)
  - Thay đổi path đến origin
  - Route đến origin khác theo logic riêng
    
- Sửa response từ origin trước khi trả cho viewer:

  - Thêm/bỏ header (CSP, HSTS, caching,…)
  - Thay đổi nội dung HTML (inject banner, AB test, personalisation đơn giản…)
    
- Sửa response cache lại (origin response) trước khi CloudFront lưu vào cache.
  

Lambda@Edge = chạy Lambda function tại Edge (gần user) để can thiệp:

**4 điểm hook:**

  1. Viewer Request – trước khi CloudFront xử lý request.
  2. Origin Request – trước khi CloudFront gọi origin (cache miss).
  3. Origin Response – khi vừa nhận response từ origin.
  4. Viewer Response – ngay trước khi trả lại cho viewer.

Use case:

  - URL rewrite kiểu báo: /tin-tuc/abc → /tin-tuc/abc/index.html.
  - Redirect logic (SEO, HTTPS, canonical).
  - Thêm/bớt header (CSP, HSTS).
  - Geo-based routing đơn giản.
  - Chặn bot theo UA/headers/country.
    
Lưu ý:

  - Lambda@Edge phải được deploy ở us-east-1 và publish = true.
  - AWS tự replicate function tới tất cả Edge – bạn không tự “build” edge.
    
Ví dụ Lambda@Edge (Viewer Request) rewrite URL:
```js
'use strict';

exports.handler = async (event, context, callback) => {
    const request = event.Records[0].cf.request;
    let uri = request.uri;

    if (uri.match(/\.(html|css|js|png|jpe?g|gif|webp|ico|svg)$/i)) {
        return callback(null, request);
    }

    if (uri === '/' || uri === '') {
        request.uri = '/index.html';
        return callback(null, request);
    }

    if (!uri.endsWith('/')) {
        uri = uri + '/';
    }
    request.uri = uri + 'index.html';

    return callback(null, request);
};

```
