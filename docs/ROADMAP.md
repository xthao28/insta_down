# Lộ trình prompt cho Claude — InstaDown

---

## Phase 1 — Backend (thực hiện theo thứ tự)

**Bước 1 — Scaffold project**
```
Tạo toàn bộ cấu trúc thư mục cho backend theo BACKEND.md:
tạo các file __init__.py, requirements.txt, .env.example,
Dockerfile, docker-compose.yml, và config.py.
```

**Bước 2 — Domain layer**
```
Viết toàn bộ domain layer theo BACKEND.md:
- domain/entities/ (ContentType, MediaInfo, MediaItem)
- domain/value_objects/instagram_url.py
- domain/ports/media_extractor.py
- domain/exceptions.py
```

**Bước 3 — Infrastructure layer**
```
Viết 2 extractor theo BACKEND.md:
- infrastructure/extractors/ytdlp_extractor.py
- infrastructure/extractors/og_meta_extractor.py
```

**Bước 4 — Application layer**
```
Viết application layer theo BACKEND.md:
- application/dtos/media_dto.py
- application/use_cases/resolve_media.py
```

**Bước 5 — Presentation layer**
```
Viết presentation layer theo BACKEND.md:
- presentation/schemas/ (requests.py, responses.py)
- presentation/middleware/ (cors.py, rate_limiter.py)
- presentation/api/v1/endpoints/ (media.py, health.py)
- presentation/api/v1/router.py
```

**Bước 6 — Wiring**
```
Viết app/dependencies.py và app/main.py theo BACKEND.md
để wire infrastructure vào application.
```

**Bước 7 — Tests**
```
Viết toàn bộ tests theo BACKEND.md:
- tests/conftest.py (FakeExtractor, fixtures)
- tests/domain/test_instagram_url.py
- tests/application/test_resolve_media.py
- tests/presentation/test_media_endpoint.py
```

**Bước 8 — Chạy và kiểm tra**
```
Chạy backend, test thử với URL Instagram thực,
sửa lỗi nếu có.
```

---

## Phase 2 — Mobile (thực hiện theo thứ tự)

**Bước 9 — Cập nhật pubspec + rename**
```
Cập nhật pubspec.yaml theo MOBILE.md: đổi name thành insta_down,
thêm tất cả dependencies còn thiếu (gallery_saver, cached_network_image,
hive_flutter, receive_sharing_intent, permission_handler, build_runner,
hive_generator).
```

**Bước 10 — Base classes**
```
Viết các base class theo MOBILE.md:
- core/base/controller/base_controller.dart
- core/base/model/base_response.dart
- core/base/repository/base_repositoty.dart
- core/base/repository/base_send_request.dart
- core/enum/enum.src.dart (AppStatus, ContentType, DownloadStatus)
```

**Bước 11 — Router**
```
Cập nhật core/router/app_route.dart (thêm home, preview, history routes)
và core/router/router_page.dart theo MOBILE.md.
```

**Bước 12 — Dio client**
```
Tạo shares/network/dio_client.dart theo MOBILE.md
(Dio instance với timeout, interceptor, DioLogSds).
```

**Bước 13 — History module**
```
Viết toàn bộ history module theo MOBILE.md (đơn giản nhất, chỉ Hive):
domain/ → data/ → controller → binding → page → widgets.
Chạy build_runner để generate Hive adapter.
```

**Bước 14 — Download module**
```
Viết download module theo MOBILE.md:
service/download_service.dart → controller → binding → widgets/download_progress_sheet.
```

**Bước 15 — Preview module**
```
Viết preview module theo MOBILE.md (phức tạp nhất):
domain/ → data/ → controller (với selected.obs cho carousel)
→ binding → page → widgets (media_preview, carousel_view, selection_controls).
```

**Bước 16 — Home module**
```
Viết home module theo MOBILE.md:
controller (URL validate, paste, share intent)
→ binding → page → widgets (url_input_bar, share_hint_card).
```

**Bước 17 — main.dart + wiring**
```
Viết lại main.dart theo MOBILE.md:
Hive init, register adapter, Dio singleton Get.put,
GetMaterialApp với AppTheme, routes, defaultTransition.
```

**Bước 18 — iOS / Android config**
```
Thêm các config platform cần thiết:
- iOS Info.plist: NSPhotoLibraryAddUsageDescription,
  NSPhotoLibraryUsageDescription, share extension
- Android Manifest: READ_MEDIA_IMAGES, WRITE_EXTERNAL_STORAGE,
  intent-filter cho share intent
```

**Bước 19 — Test end-to-end**
```
Chạy app trên simulator, test luồng:
paste URL → preview → download → gallery → lịch sử.
Báo lỗi nếu có để sửa.
```

---

## Lưu ý khi prompt

- Mỗi bước **chỉ làm 1 việc** — đừng gộp nhiều bước vào 1 prompt
- Nếu Claude hỏi clarify, trả lời ngắn rồi tiếp tục
- Sau mỗi bước có thể nói **"chạy thử và báo lỗi"** để Claude debug luôn
- Nếu có gì khác so với docs, nói **"ưu tiên theo code thực tế"** để Claude bám vào codebase thay vì docs
