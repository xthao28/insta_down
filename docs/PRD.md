# PRD — InstaDown v1.0

Mobile app tải ảnh/video Instagram. Flutter (iOS + Android) + Python FastAPI.

---

## Mục tiêu & Metrics

| Mục tiêu | Đo bằng |
|---|---|
| Hoàn thành tải trong ≤ 3 bước | UX flow audit |
| Hỗ trợ 100% content type public | Test coverage |
| Không cần đăng nhập Instagram | Kiến trúc stateless |

**KPI:** 10k downloads / 30 ngày · Rating ≥ 4.3 · D7 retention ≥ 30% · Parse success ≥ 95% · Latency P95 < 3s

---

## User Personas

**Minh Châu, 24t — Content creator nghiệp dư**
Dùng iPhone, theo dõi tài khoản thời trang, cần lưu ảnh carousel chất lượng cao làm mood board. Pain: screenshot mất chất lượng, app hiện có nhiều quảng cáo.
> "Copy link → dán → xem trước → lưu. Xong."

**Tuấn Anh, 38t — Phụ huynh**
Dùng Samsung, muốn lưu video nấu ăn xem offline và chia sẻ qua Zalo. Pain: không xem offline được, app phức tạp.
> "Mở ra là biết ngay phải làm gì."

---

## Functional Requirements

### Must Have
| ID | Tính năng |
|---|---|
| F01 | Paste URL hoặc nhận Share Intent từ Instagram |
| F02 | Parse URL → trả metadata + CDN URL (backend) |
| F03 | Preview ảnh/video trước khi tải |
| F04 | Carousel: swipe xem + checkbox chọn từng ảnh |
| F05 | Download → lưu thẳng vào Photos/Gallery hệ thống |
| F06 | Thanh progress khi tải |
| F07 | Lịch sử tải (local, Hive) |

### Should Have
| ID | Tính năng |
|---|---|
| F08 | Download nền + push notification khi xong |
| F09 | Swipe-to-delete lịch sử |

### Nice to Have
| ID | Tính năng |
|---|---|
| F10 | Mở file từ lịch sử |
| F11 | Share file ngay sau khi tải xong |
| F12 | Dark mode · Đa ngôn ngữ (VI/EN) |

---

## Non-Functional Requirements

| Nhóm | Yêu cầu |
|---|---|
| Performance | Parse < 3s P95 · Cold start < 2s · App size < 30MB |
| Security | HTTPS/TLS 1.3 · Validate URL (whitelist domain + anti-SSRF) · Không lưu credential · Rate limit 30 req/min/IP |
| Compatibility | iOS 14+ · Android 8+ (API 26+) |
| Reliability | Backend uptime ≥ 99.5% · Crash rate < 0.5% |

---

## Out of Scope (v1.0)

- Tài khoản private
- Instagram Stories
- Các nền tảng khác (TikTok, YouTube…)
- Tài khoản người dùng / đăng nhập InstaDown
- Lịch sử cloud sync
- Batch download toàn bộ profile
- Phiên bản web
- Quảng cáo

---

## Risks

| Rủi ro | Xác suất | Ảnh hưởng | Biện pháp |
|---|---|---|---|
| Instagram thay đổi API/HTML | Cao | Nghiêm trọng | Strategy pattern extractor, monitor success rate, hotfix SLA 24h |
| Vi phạm ToS Meta | Trung bình | Cao | Chỉ support public content, không cache media trên server, có ToS/disclaimer rõ |
| App Store từ chối | Trung bình | Cao | Nghiên cứu guidelines trước, chuẩn bị APK sideload fallback |
| SSRF attack | Thấp (nếu validate đúng) | Nghiêm trọng | Whitelist domain + reject private IP sau DNS resolve |
| Backend quá tải | Trung bình | Trung bình | Redis cache kết quả parse (TTL 5min), auto-scale container |
