# InstaDown — Claude Working Guide

## Project layout

```
insta_down/
├── backend/   FastAPI (Python 3.10, Clean Architecture) — COMPLETE
├── mobile/    Flutter (package: mobile_base_clean, GetX + Clean Architecture)
└── docs/      PRD · TDD · BACKEND.md · MOBILE.md · ROADMAP.md
```

---

## Backend (complete — do not modify)

**Run:** `cd backend && .venv/bin/uvicorn app.main:app --reload`

```
POST http://127.0.0.1:8000/api/v1/resolve
Body: { "url": "https://www.instagram.com/reel/..." }

Response 200: { content_type, username, media_items: [{index, type, url, thumbnail_url, width, height, duration_seconds}] }
Error shape:  { error_code: "INVALID_URL|CONTENT_PRIVATE|CONTENT_NOT_FOUND|CONTENT_NOT_SUPPORTED|EXTRACTION_FAILED|RATE_LIMIT_EXCEEDED", message }
```

---

## Mobile — package `mobile_base_clean`

**Flutter 3.32.8 · Dart 3.8 · GetX 4.7**  
All imports: `package:mobile_base_clean/...`

### Directory structure

```
mobile/lib/
│
├── core/                          # Framework — không sửa khi thêm feature
│   ├── data/
│   │   ├── data_source/
│   │   │   ├── local/
│   │   │   │   ├── app_hive.dart / app_hive_impl.dart   (Hive wrapper)
│   │   │   │   └── hive_keys.dart                       (box names + setting keys)
│   │   │   └── network/
│   │   │       ├── dio_builder.dart                     (Dio factory)
│   │   │       ├── rest_api_client.dart                 (RestMethod enum, request())
│   │   │       ├── non_auth_app_server_api_client.dart  (no-auth Dio instance)
│   │   │       ├── auth_app_server_api_client.dart      (auth Dio instance)
│   │   │       └── middleware/  (HeaderInterceptor, AccessTokenInterceptor)
│   │   └── model/
│   │       ├── base_response.dart     BaseResponse<T> {code, errorMessage, result}
│   │       ├── base_response_list.dart
│   │       └── server_error.dart      ServerError {code, errorMessage}
│   │
│   ├── domain/
│   │   ├── entity/  app_data.dart / app_data_impl.dart  (global app state)
│   │   ├── repository/  base_repository.dart            (CancelToken management)
│   │   └── usecase/  base_use_case.dart                 UseCase<I,O> / NoInputUseCase<O>
│   │
│   └── presentation/
│       ├── app.dart                 GetMaterialApp root widget
│       ├── bindings/
│       │   ├── app_binding.dart     global DI (AppHive, Dio, Navigator, EventBus…)
│       │   └── base_bindings.dart   abstract: bindingsRepository/UseCase/Controller
│       ├── controllers/
│       │   ├── app_controller.dart
│       │   └── base_getx_controller.dart   buildState(action, onError, onFinally)
│       ├── navigation/
│       │   └── app_navigator.dart   toNamed, back, offAllNamed, showDialog, showSnackBar…
│       └── widgets/
│           ├── base_get_page.dart   BaseGetPage<T> = GetView<T> + GetPageMixin
│           └── base_get_bts_dialog.dart
│
├── features/                      # Mỗi feature = 1 folder, độc lập hoàn toàn
│   └── [feature]/
│       ├── data/
│       │   ├── model/             *_data.dart  (JSON ↔ Dart, raw server model)
│       │   └── repository/        *_repository_impl.dart
│       ├── domain/
│       │   ├── entity/            *_entity.dart (pure Dart, no Flutter/Dio)
│       │   ├── exception/         *_exception.dart extends AppException
│       │   ├── repository/        *_repository.dart (abstract)
│       │   └── usecase/           *_use_case.dart extends UseCase<I,O>
│       └── presentation/
│           ├── binding/           *_binding.dart extends BaseBindings
│           ├── controller/        *_controller.dart extends BaseGetxController
│           ├── logic/             (optional) validators, pure business logic helpers
│           ├── *_page.dart        extends BaseGetPage<Controller>
│           └── *_widget.dart      (optional) sub-widgets
│
├── routes/
│   ├── app_routes.dart    enum AppRoutes { splash, home, preview, history, settings }
│   └── app_pages.dart     AppPages.pages — wire GetPage + Binding
│
└── shared/
    ├── constants/
    │   ├── api_url.dart       ApiUrl.baseUrl (dart-define), ApiUrl.resolve, .health
    │   ├── const.dart         connectTimeout / receiveTimeout / sendTimeout (Duration)
    │   ├── app_dimens.dart    AppDimens (padding, radius values)
    │   └── app_text_style.dart AppTextStyle (font10Re/Semi/Bo … font36Re/Semi/Bo)
    ├── exceptions/
    │   ├── base/  AppException, AppExceptionWrapper
    │   ├── remote/  RemoteException (kind: network|timeout|serverDefined|…)
    │   └── exception_handler.dart  maps exceptions → nav.showSnackBar/showErrorDialog
    ├── mappers/
    │   └── base/  BaseDataMapper<D,E>, BaseEntityMapper<E,D>, DataMapperMixin
    ├── themes/
    │   └── app_colors.dart    AppColors (neutrals + status + InstaDown brand)
    └── utils/
        ├── get_finder.dart    sl<T>() shorthand for Get.find<T>()
        └── logger.dart        logger instance
```

---

## Key patterns

### 1. Controller — `buildState`

```dart
class HomeController extends BaseGetxController {
  final _resolveUseCase = sl<ResolveUseCase>();
  final mediaItems = RxList<MediaItemEntity>([]);

  Future<void> resolve(String url) async {
    await buildState(
      showLoadingOverlay: true,
      action: () async {
        final items = await _resolveUseCase.execute(ResolveInput(url: url));
        mediaItems.assignAll(items);
        nav.toNamed(AppRoutes.preview.path);
      },
      onError: (error) {
        if (error is ResolveException) {
          nav.showSnackBar(error.errorMessage ?? '');
          return null; // handled — stop propagation
        }
        return error; // unhandled — let ExceptionHandler deal with it
      },
    );
  }
}
```

### 2. UseCase

```dart
class ResolveUseCase extends UseCase<ResolveInput, List<MediaItemEntity>> {
  final ResolveRepository _repository;
  ResolveUseCase(this._repository);

  @override
  Future<List<MediaItemEntity>> execute(ResolveInput input) async {
    if (input.url.trim().isEmpty) throw ResolveException(kind: ResolveExceptionKind.emptyUrl);
    return _repository.resolve(url: input.url);
  }
}
```

### 3. Repository

```dart
// domain/repository/resolve_repository.dart
abstract class ResolveRepository extends BaseRepository {
  Future<List<MediaItemEntity>> resolve({required String url});
}

// data/repository/resolve_repository_impl.dart
class ResolveRepositoryImpl extends ResolveRepository {
  final NonAuthAppServerApiClient _client;
  final ResolveDataMapper _mapper;

  ResolveRepositoryImpl(this._client, this._mapper);

  @override
  Future<List<MediaItemEntity>> resolve({required String url}) async {
    final data = await _client.request(
      method: RestMethod.post,
      path: ApiUrl.resolve,
      body: {'url': url},
      cancelToken: cancelToken,
    );
    return _mapper.mapToListEntity(data['media_items'] as List);
  }
}
```

### 4. Binding

```dart
class HomeBinding extends BaseBindings {
  @override
  void bindingsRepository() {
    Get.lazyPut<ResolveRepository>(() => ResolveRepositoryImpl(sl(), sl()));
  }

  @override
  void bindingsUseCase() {
    Get.lazyPut(() => ResolveUseCase(sl()));
  }

  @override
  void bindingsController() {
    Get.lazyPut(() => HomeController());
  }
}
```

### 5. Page

```dart
class HomePage extends BaseGetPage<HomeController> {
  HomePage({super.key});

  @override
  Widget buildPage(BuildContext context) {
    return Scaffold(
      body: Obx(() => controller.isLoadingOverlay.value
        ? const CircularProgressIndicator()
        : _buildContent()),
    );
  }
}
```

### 6. Mapper

```dart
// data/model/media_item_data.dart
class MediaItemData {
  final int index;
  final String type;
  final String url;
  final String thumbnailUrl;

  MediaItemData.fromJson(Map<String, dynamic> json)
      : index = json['index'],
        type = json['type'],
        url = json['url'],
        thumbnailUrl = json['thumbnail_url'];
}

// shared/mappers/
class MediaItemDataMapper extends BaseDataMapper<MediaItemData, MediaItemEntity> {
  @override
  MediaItemEntity mapToEntity(MediaItemData? data) => MediaItemEntity(
        index: data!.index, type: data.type,
        url: data.url, thumbnailUrl: data.thumbnailUrl,
      );
}
```

---

## Colors (InstaDown brand)

```dart
AppColors.igOrange          // #F77737
AppColors.igPink            // #E1306C
AppColors.igPurple          // #833AB4
AppColors.instagramGradient // LinearGradient orange→pink→purple
AppColors.downloadSuccess   // #34C759
AppColors.downloadError     // #FF3B30
AppColors.shimmerBase / shimmerHighlight
```

## API runtime config

```bash
flutter run --dart-define=API_BASE_URL=http://127.0.0.1:8000   # iOS sim
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000    # Android emu
```

## Locale keys (InstaDown)

`home_*` · `preview_*` · `history_*` · `settings_*`  
Defined in `assets/locales/vi_VN.json` — regenerate `locales.g.dart` via:
```bash
flutter pub run get_cli generate locales assets/locales
```

## Rules

| Rule | Detail |
|---|---|
| DI | `sl<T>()` = `Get.find<T>()`. Never `Get.put()` inside a widget/page. |
| State | `Rx` variables + `Obx()`. No `setState`, no `StatefulWidget` for business state. |
| Errors | `buildState(onError:)` — return `null` if handled, return exception if not. |
| Navigation | `nav.toNamed()` / `nav.back()` from controller. Never `Get.to()` in widgets. |
| Async ops | Always `await` inside `buildState(action:)` — otherwise errors won't be caught. |
| CancelToken | Use `cancelToken` from `BaseRepository` — auto-cancelled on `onClose()`. |
| Testing | Set `controller.isTestMode = true`, call `controller.onOpen()` to simulate lifecycle. |

---

## Docs

- `docs/MOBILE.md` — architecture detail, per-layer rules
- `docs/ROADMAP.md` — Steps 9–19 (mobile implementation order)
- `docs/BACKEND.md` — API contract reference
