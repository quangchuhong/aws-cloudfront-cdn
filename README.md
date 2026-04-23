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
