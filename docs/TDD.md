# TDD — InstaDown v1.0

Stack: Flutter 3.x · GetX · Dio · Hive (mobile) · Python FastAPI (backend) · Stateless backend

---

## System Architecture

```
Flutter App
  └── ApiService (Dio)
        │ HTTPS POST /api/v1/media/resolve
        ▼
  FastAPI Backend
    ├── Rate Limiter (slowapi, 30 req/min/IP)
    ├── UrlValidator (whitelist + anti-SSRF)
    └── InstagramService
          ├── [1] YtDlpExtractor (primary)   ← yt-dlp trong thread pool
          └── [2] OgMetaExtractor (fallback)  ← httpx + BeautifulSoup4
                │ returns: metadata + CDN URLs
                ▼
  Flutter App tải file TRỰC TIẾP từ Instagram CDN
  (backend không cache/lưu media)
        │
        ▼
  GallerySaver → Photos/Gallery hệ thống
  Hive Box     → Lịch sử download (local only)
```

**Nguyên tắc thiết kế:**
- Backend stateless — có thể scale horizontally
- Strategy Pattern cho extractors: swap khi Instagram thay đổi, không ảnh hưởng router
- File tải thẳng từ CDN Instagram qua Dio — backend không làm proxy

---

## Backend — FastAPI

### Cấu trúc thư mục
```
app/
├── main.py              # factory, middleware, exception handlers
├── config.py            # Pydantic BaseSettings, .env
├── api/v1/endpoints/
│   ├── media.py         # POST /media/resolve
│   └── health.py        # GET /health
├── services/
│   ├── instagram_service.py  # orchestration: validate → extract → format
│   └── url_validator.py      # domain whitelist + SSRF check
├── extractors/
│   ├── base.py               # BaseExtractor ABC
│   ├── ytdlp_extractor.py    # primary (yt-dlp)
│   └── og_meta_extractor.py  # fallback (httpx + BS4)
├── schemas/
│   ├── media.py              # ResolveRequest, MediaResponse
│   └── errors.py             # ErrorResponse, ExtractionError
└── utils/url_utils.py        # normalize, detect type, reject SSRF
```

### Endpoints

#### `POST /api/v1/media/resolve`
```json
// Request
{ "url": "https://www.instagram.com/p/ABC123/" }

// Response 200
{
  "success": true,
  "data": {
    "content_type": "photo|video|reel|carousel",
    "username": "string",
    "media_items": [
      {
        "index": 0,
        "type": "photo|video",
        "url": "https://scontent-*.cdninstagram.com/...",
        "thumbnail_url": "https://...",
        "width": 1080, "height": 1080,
        "duration_seconds": null
      }
    ]
  }
}

// Error
{ "success": false, "error": "ERROR_CODE", "message": "..." }
```

| HTTP | error_code | Khi nào |
|---|---|---|
| 400 | `INVALID_URL` | URL không phải Instagram |
| 400 | `CONTENT_NOT_SUPPORTED` | Stories (out of scope v1) |
| 403 | `CONTENT_PRIVATE` | Bài đăng private |
| 404 | `CONTENT_NOT_FOUND` | URL không tồn tại |
| 429 | `RATE_LIMITED` | Vượt giới hạn request |
| 503 | `EXTRACTION_FAILED` | yt-dlp + fallback đều thất bại |

#### `GET /api/v1/health`
```json
{ "status": "ok", "version": "1.0.0" }
```

### Key Implementation Notes

**YtDlpExtractor** — chạy `yt-dlp` trong `asyncio.run_in_executor` (thread pool) vì là blocking call. Chọn format mp4 resolution cao nhất cho video.

**OgMetaExtractor** — dùng User-Agent `facebookexternalhit/1.1` để Instagram trả về OG tags đầy đủ.

**InstagramService** — nếu extractor raise `CONTENT_PRIVATE` hoặc `CONTENT_NOT_FOUND` thì dừng ngay (không fallback vì kết quả sẽ giống nhau).

**URL Validation** — sau khi whitelist domain, resolve DNS và reject nếu IP nằm trong `10.x`, `172.16-31.x`, `192.168.x`, `127.x` (chống SSRF).

**Rate limiting** — 30 req/min/IP + 200 req/hour/IP qua `slowapi`. Header `Retry-After` trong response 429.

### Dependencies
```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
httpx==0.27.2
beautifulsoup4==4.12.3
lxml==5.3.0
yt-dlp==2024.10.7        # update thường xuyên!
slowapi==0.1.9
```

---

## Frontend — Flutter

### Cấu trúc thư mục
```
lib/
├── main.dart                        # Hive init, Dio singleton, GetMaterialApp
├── assets.dart                      # Auto-generated asset paths
├── core/
│   ├── base/                        # BaseController, BaseResponse, BaseSendRequest mixin
│   ├── enum/enum.src.dart           # AppStatus, ContentType, DownloadStatus
│   ├── router/
│   │   ├── app_route.dart           # AppRoutes constants (/home, /preview, /history)
│   │   └── router_page.dart         # GetPage list với Binding
│   └── values/
│       ├── collection.dart          # AppColors, AppGradients
│       ├── const.dart               # AppSpacing, AppRadius, AppSizes, AppDuration, AppConst
│       ├── app_theme.dart           # AppTheme.light / AppTheme.dark
│       ├── app_api.dart             # AppApi.baseUrl, endpoints, timeouts
│       └── key.dart                 # AppKey (Hive box names)
├── modules/
│   ├── splash/                      # Khởi động app
│   ├── home/                        # URL input, paste, share intent
│   │   └── controller/ binding/ widgets/
│   ├── preview/                     # Media preview + selection + download trigger
│   │   └── controller/ binding/ domain/ data/ widgets/
│   ├── download/                    # Progress sheet, DownloadService, GallerySaver
│   │   └── controller/ binding/ service/ widgets/
│   └── history/                     # Lịch sử Hive
│       └── controller/ binding/ domain/ data/ widgets/
└── shares/
    ├── function/                    # logger, SecureStorage
    ├── package/export_package.dart  # re-export: get, dio, logger, secure_storage
    └── widgets/                     # AppBar, BottomSheet, Button, Input, SizedBox shortcuts
```

### State Management — GetX
Mỗi module có 1 `Controller` (extends `BaseController` extends `GetxController`). State dùng `.obs`:

| Controller | State chính | Vai trò |
|---|---|---|
| `HomeController` | `RxBool isValidUrl` · `TextEditingController` | URL input, paste, validate |
| `PreviewController` | `Rx<AppStatus>` · `Rx<MediaInfo?>` · `RxSet<int> selected` | Gọi API, chọn carousel items |
| `DownloadController` | `RxList<DownloadTask>` (mỗi task có `RxDouble progress`) | Queue tải, progress per-file |
| `HistoryController` | `RxList<HistoryEntry>` | CRUD Hive box |

**Binding** là nơi duy nhất `Get.lazyPut` — controller không tự inject dependency:
```dart
class PreviewBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => MediaRemoteDataSource(Get.find<Dio>()));
    Get.lazyPut<MediaRepository>(() => MediaRepositoryImpl(Get.find()));
    Get.lazyPut(() => PreviewController(repository: Get.find()));
  }
}
```

**Clean Architecture per module:**
- `domain/entities/` — pure Dart class, không import package
- `domain/repository/` — abstract interface
- `data/models/` — extends entity, thêm `fromJson`
- `data/datasources/` — Dio call trực tiếp
- `data/repository/` — implement interface, dùng `BaseSendRequest` mixin

### Download Flow
```
DownloadController.startDownload(items, username)
  → DownloadService._ensurePermission()   # permission_handler
  → dio.download(cdnUrl, tempPath, onReceiveProgress)
  → task.progress.value = received/total  # Rx update → UI tự rebuild
  → GallerySaver.saveImage/Video(tempPath, albumName: 'InstaDown')
  → File(tempPath).delete()               # dọn temp
  → HistoryRepository.add(entry)          # lưu Hive
```

### Local Storage — Hive
```dart
// Domain entity (pure Dart)
class HistoryEntry {
  final String username, thumbnailUrl, contentType;
  final int mediaCount;
  final DateTime downloadedAt;
}

// Data model (@HiveType, extends entity)
@HiveType(typeId: AppConst.hiveHistoryTypeId)  // typeId = 0
class HistoryEntryModel extends HistoryEntry with HiveObjectMixin {
  @HiveField(0) final String   username;
  @HiveField(1) final String   thumbnailUrl;
  @HiveField(2) final int      mediaCount;
  @HiveField(3) final String   contentType;
  @HiveField(4) final DateTime downloadedAt;
}
// Box name: AppKey.hiveHistoryBox = 'download_history'
// Generate adapter: dart run build_runner build
```

### Share Intent
Dùng `receive_sharing_intent`: khi user nhấn Share trong Instagram app → InstaDown nhận URL → `HomeController` tự điền input → `Get.toNamed(AppRoutes.preview, arguments: url)`.

### Dependencies
```yaml
get: ^4.6.6                          # State management + navigation + DI
dio: ^5.4.2+1                        # HTTP client + download
dio_log_sds:                         # Dio logger (internal package)
  git: ...
cached_network_image: ^3.4.1         # Thumbnail caching
gallery_saver: ^2.3.2                # Lưu file vào Photos/Gallery
path_provider: ^2.1.4                # Temp directory
permission_handler: ^11.3.1          # Photos/Storage permission
hive_flutter: ^1.1.0                 # Local storage (lịch sử)
receive_sharing_intent: ^1.8.0       # Nhận URL share từ Instagram app
flutter_secure_storage: ^9.0.0       # (đã có) Settings bảo mật
logger: ^2.2.0                       # (đã có) Logging
# dev
build_runner: ^2.4.12
hive_generator: ^2.0.1
```

---

## Data Flow — Paste URL → File Saved

```
1. User paste/share URL
2. HomeController.onUrlChanged() → isValidUrl.value = true → CTA enabled
3. HomeController.fetchUrl() → Get.toNamed('/preview', arguments: url)
4. PreviewController.onInit() → _resolve(url)
5.   MediaRepositoryImpl.resolveUrl(url)
       → MediaRemoteDataSource.resolve(url)
       → POST /api/v1/media/resolve
6.   Backend: validate → yt-dlp extract → format → return CDN URLs
7. status.value = AppStatus.success → Obx rebuild PreviewPage
8. User chọn items (selected.obs) → tap Download
9. DownloadController.startDownload(selectedItems, username)
10.  DownloadService._ensurePermission() → request Photos/Storage
11.  dio.download(cdnUrl) → tempPath (progress → task.progress.value)
12.  GallerySaver.save(tempPath) → Photos/Gallery
13.  File(tempPath).delete()
14.  HistoryRepository.add() → Hive box
15. allDone → Get.snackbar("Đã lưu X ảnh vào thư viện")
```

---

## Error Handling

```dart
// BaseSendRequest mixin (core/base/repository/base_send_request.dart)
// Bắt DioException → trả BaseResponse(success: false, error: 'ERROR_CODE')

// Controller nhận BaseResponse và cập nhật state:
final result = await _repository.resolveUrl(url);
if (result.isSuccess) {
  status.value = AppStatus.success;
} else {
  errorCode.value = result.error ?? 'UNKNOWN_ERROR';
  status.value    = AppStatus.error;
}
```

**Luồng xử lý lỗi:**  
`DioException` → `BaseSendRequest._handleDioError()` → `BaseResponse(success: false, error: code)`  
→ `Controller.errorCode.obs` cập nhật → `Obx()` trong Page rebuild → hiển thị ErrorView

**error_code → message tiếng Việt** (map trong widget hoặc extension):

| error_code | Message hiển thị |
|---|---|
| `INVALID_URL` | Link không hợp lệ. Vui lòng kiểm tra lại. |
| `CONTENT_PRIVATE` | Nội dung này ở chế độ riêng tư hoặc đã bị xóa. |
| `CONTENT_NOT_FOUND` | Không tìm thấy nội dung. Link có thể đã bị xóa. |
| `CONTENT_NOT_SUPPORTED` | Loại nội dung này chưa được hỗ trợ (Stories). |
| `RATE_LIMITED` | Bạn đang tải quá nhiều. Vui lòng chờ một chút. |
| `CONNECTION_TIMEOUT` | Kết nối quá chậm. Kiểm tra mạng và thử lại. |
| `NO_NETWORK` | Không có kết nối mạng. |
| `SERVER_ERROR` / `EXTRACTION_FAILED` | Lỗi máy chủ. Vui lòng thử lại sau. |
