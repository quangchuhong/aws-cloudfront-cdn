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
