# Mobile — InstaDown Flutter

**Stack:** Flutter 3.32 · Dart 3.8 · GetX 4.7 · Dio 5.8 · Hive 2.2  
**Package:** `mobile_base_clean`  
**Pattern:** Clean Architecture (3 layers per feature) + GetX DI/state/navigation  
**Liên quan:** [CLAUDE.md](../CLAUDE.md) · [BACKEND.md](BACKEND.md) · [ROADMAP.md](ROADMAP.md)

---

## Mục lục

1. [Clean Architecture Overview](#1-clean-architecture-overview)
2. [Directory Structure](#2-directory-structure)
3. [Core Layer](#3-core-layer)
4. [Feature Layer](#4-feature-layer)
5. [Shared Layer](#5-shared-layer)
6. [State Management](#6-state-management)
7. [Navigation](#7-navigation)
8. [Error Handling](#8-error-handling)
9. [Network](#9-network)
10. [Local Storage](#10-local-storage)
11. [Adding a New Feature](#11-adding-a-new-feature)

---

## 1. Clean Architecture Overview

```
┌─────────────────────────────────────────────┐
│  PRESENTATION  (GetX Controller, Page, Binding) │
│  ┌─────────────────────────────────────────┐ │
│  │  DOMAIN  (UseCase, Repository abstract,  │ │
│  │           Entity, Exception)             │ │
│  └─────────────────────────────────────────┘ │
│                                               │
│  DATA  (RepositoryImpl, Model, DataSource)    │
│  (implements Domain Repository)               │
└─────────────────────────────────────────────┘
```

**Dependency Rule:**
- `Presentation` → `Domain`
- `Data` → `Domain`
- `Domain` không import Flutter/Dio/Hive — chỉ thuần Dart

---

## 2. Directory Structure

```
mobile/lib/
├── core/                        # Framework dùng chung cho toàn app
│   ├── data/
│   │   ├── data_source/
│   │   │   ├── local/           AppHive, HiveKeys
│   │   │   └── network/         DioBuilder, RestApiClient,
│   │   │                        NonAuth/AuthAppServerApiClient, Interceptors
│   │   └── model/               BaseResponse<T>, BaseResponseList<T>, ServerError
│   ├── domain/
│   │   ├── entity/              AppData (global app state)
│   │   ├── repository/          BaseRepository (CancelToken mgmt)
│   │   └── usecase/             UseCase<I,O>, NoInputUseCase<O>
│   └── presentation/
│       ├── app.dart             GetMaterialApp root
│       ├── bindings/            AppBinding (global DI), BaseBindings
│       ├── controllers/         AppController, BaseGetxController
│       ├── navigation/          AppNavigator interface + impl
│       └── widgets/             BaseGetPage<T>, BaseGetBtsDialog
│
├── features/
│   └── [feature_name]/
│       ├── data/
│       │   ├── model/           *Data (JSON model, fromJson/toJson)
│       │   └── repository/      *RepositoryImpl
│       ├── domain/
│       │   ├── entity/          *Entity (pure Dart)
│       │   ├── exception/       *Exception extends AppException
│       │   ├── repository/      *Repository abstract extends BaseRepository
│       │   └── usecase/         *UseCase extends UseCase<I,O>
│       └── presentation/
│           ├── binding/         *Binding extends BaseBindings
│           ├── controller/      *Controller extends BaseGetxController
│           ├── logic/           (optional) validators, helper logic
│           ├── *_page.dart      extends BaseGetPage<Controller>
│           └── *_widget.dart    (optional) complex sub-widgets
│
├── routes/
│   ├── app_routes.dart          enum AppRoutes { splash, home, preview, history, settings }
│   └── app_pages.dart           AppPages.pages — GetPage list
│
└── shared/
    ├── constants/               ApiUrl, AppDimens, AppTextStyle, const (timeouts)
    ├── exceptions/              AppException, RemoteException, ExceptionHandler
    ├── mappers/                 BaseDataMapper<D,E>, BaseEntityMapper<E,D>
    ├── themes/                  AppColors (neutrals + InstaDown brand + shimmer)
    └── utils/                   sl<T>(), logger, keyboard, uuid
```

---

## 3. Core Layer

### BaseGetxController

Mọi controller đều kế thừa. Không bao giờ dùng `GetxController` trực tiếp.

```dart
abstract class BaseGetxController extends GetxController {
  late AppNavigator nav = sl<AppNavigator>();
  late AppData appData  = sl<AppData>();

  final isLoading        = false.obs;
  final isLoadingOverlay = false.obs;

  Future<void> buildState({
    required ActionCallback action,
    OnErrorCallback? onError,    // return null = handled; return error = propagate
    OnFinallyCallback? onFinally,
    bool showLoading = false,
    bool showLoadingOverlay = false,
    bool handleError = true,
  }) async { ... }
}
```

**Quy tắc:**
- Mọi async call phải nằm trong `buildState(action:)` và phải `await`
- `onError` trả về `null` → lỗi đã xử lý, dừng. Trả về exception → `ExceptionHandler` xử lý tiếp
- `isTestMode = true` + `onOpen()` để mock lifecycle trong unit test

### BaseBindings

```dart
abstract class BaseBindings extends Bindings {
  @override
  void dependencies() {
    bindingsRepository();   // thứ tự bắt buộc
    bindingsUseCase();
    bindingsController();
  }
}
```

### BaseGetPage

```dart
abstract class BaseGetPage<T extends BaseGetxController> extends GetView<T>
    with GetPageMixin {
  // Override buildPage() thay vì build()
  Widget buildPage(BuildContext context);
}
```

---

## 4. Feature Layer

### Domain — Entity

Thuần Dart, không import Flutter/Dio:

```dart
class MediaItemEntity {
  final int     index;
  final String  type;          // "photo" | "video"
  final String  url;
  final String  thumbnailUrl;
  final int?    width;
  final int?    height;
  final double? durationSeconds;
  const MediaItemEntity({...});
}

class ResolveResultEntity {
  final String contentType;   // "photo" | "video" | "reel" | "carousel"
  final String username;
  final List<MediaItemEntity> mediaItems;
}
```

### Domain — Repository (abstract)

```dart
abstract class ResolveRepository extends BaseRepository {
  Future<ResolveResultEntity> resolve({required String url});
}
```

### Domain — UseCase

```dart
class ResolveUseCase extends UseCase<String, ResolveResultEntity> {
  final ResolveRepository _repo;
  ResolveUseCase(this._repo);

  @override
  Future<ResolveResultEntity> execute(String url) async {
    if (url.trim().isEmpty) throw ResolveException(kind: ResolveExceptionKind.emptyUrl);
    return _repo.resolve(url: url);
  }
}
```

### Domain — Exception

```dart
enum ResolveExceptionKind { emptyUrl, invalidUrl, privateContent, notFound, failed }

class ResolveException extends AppException {
  final ResolveExceptionKind kind;
  ResolveException({required this.kind}) : super(AppExceptionType.custom);

  @override
  String? get errorMessage => switch (kind) {
    ResolveExceptionKind.emptyUrl       => 'Vui lòng nhập link Instagram',
    ResolveExceptionKind.invalidUrl     => LocaleKeys.home_invalidUrl.tr,
    ResolveExceptionKind.privateContent => LocaleKeys.home_privateContent.tr,
    ResolveExceptionKind.notFound       => LocaleKeys.home_notFound.tr,
    ResolveExceptionKind.failed         => LocaleKeys.home_extractionFailed.tr,
  };
}
```

### Data — Model

```dart
class MediaItemData {
  final int     index;
  final String  type;
  final String  url;
  final String  thumbnailUrl;
  final int?    width;
  final int?    height;
  final double? durationSeconds;

  MediaItemData.fromJson(Map<String, dynamic> j)
      : index           = j['index'],
        type            = j['type'],
        url             = j['url'],
        thumbnailUrl    = j['thumbnail_url'],
        width           = j['width'],
        height          = j['height'],
        durationSeconds = (j['duration_seconds'] as num?)?.toDouble();
}
```

### Data — RepositoryImpl

```dart
class ResolveRepositoryImpl extends ResolveRepository {
  final NonAuthAppServerApiClient _client;
  final MediaItemDataMapper _mapper;

  ResolveRepositoryImpl(this._client, this._mapper);

  @override
  Future<ResolveResultEntity> resolve({required String url}) async {
    final json = await _client.request(
      method: RestMethod.post,
      path: ApiUrl.resolve,
      body: {'url': url},
      cancelToken: cancelToken,
    ) as Map<String, dynamic>;

    if (json['error_code'] != null) _throwFromErrorCode(json['error_code']);

    return ResolveResultEntity(
      contentType: json['content_type'],
      username: json['username'] ?? '',
      mediaItems: _mapper.mapToListEntity(
        (json['media_items'] as List)
            .cast<Map<String, dynamic>>()
            .map(MediaItemData.fromJson)
            .toList(),
      ),
    );
  }

  void _throwFromErrorCode(String code) => switch (code) {
    'INVALID_URL'       => throw ResolveException(kind: ResolveExceptionKind.invalidUrl),
    'CONTENT_PRIVATE'   => throw ResolveException(kind: ResolveExceptionKind.privateContent),
    'CONTENT_NOT_FOUND' => throw ResolveException(kind: ResolveExceptionKind.notFound),
    _                   => throw ResolveException(kind: ResolveExceptionKind.failed),
  };
}
```

### Presentation — Binding

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

### Presentation — Controller

```dart
class HomeController extends BaseGetxController {
  final _resolveUseCase = sl<ResolveUseCase>();
  final urlText   = ''.obs;
  final result    = Rxn<ResolveResultEntity>();

  Future<void> resolve() async {
    await buildState(
      showLoadingOverlay: true,
      action: () async {
        result.value = await _resolveUseCase.execute(urlText.value);
        nav.toNamed(AppRoutes.preview.path, arguments: result.value);
      },
      onError: (error) {
        if (error is ResolveException) {
          nav.showSnackBar(error.errorMessage ?? '');
          return null;
        }
        return error;
      },
    );
  }
}
```

### Presentation — Page

```dart
class HomePage extends BaseGetPage<HomeController> {
  HomePage({super.key});

  @override
  Widget buildPage(BuildContext context) {
    return Scaffold(
      body: Obx(() => controller.isLoadingOverlay.value
          ? const Center(child: CircularProgressIndicator())
          : _HomeBody()),
    );
  }
}
```

---

## 5. Shared Layer

### AppColors — InstaDown brand

```dart
AppColors.igOrange           // #F77737
AppColors.igPink             // #E1306C
AppColors.igPurple           // #833AB4
AppColors.instagramGradient  // LinearGradient bottomLeft→topRight
AppColors.downloadSuccess    // #34C759
AppColors.downloadError      // #FF3B30
AppColors.shimmerBase / shimmerHighlight
```

### Mapper

```dart
class MediaItemDataMapper extends BaseDataMapper<MediaItemData, MediaItemEntity> {
  @override
  MediaItemEntity mapToEntity(MediaItemData? data) => MediaItemEntity(
    index: data!.index, type: data.type,
    url: data.url, thumbnailUrl: data.thumbnailUrl,
    width: data.width, height: data.height,
    durationSeconds: data.durationSeconds,
  );
}
```

---

## 6. State Management

**GetX Rx — không dùng `StatefulWidget` hay `setState` cho business state:**

```dart
// Controller
final mediaResult    = Rxn<ResolveResultEntity>();
final downloadedIds  = RxSet<int>({});
final isDownloading  = false.obs;

// Page
Obx(() {
  if (controller.isLoadingOverlay.value) return const LoadingOverlay();
  if (controller.mediaResult.value == null) return const EmptyState();
  return ResultView(result: controller.mediaResult.value!);
})
```

---

## 7. Navigation

Luôn dùng `nav` từ controller, không dùng `Get.to()` hay `BuildContext` trong controller:

```dart
nav.toNamed(AppRoutes.preview.path, arguments: result);
nav.back();
nav.offAllNamed(AppRoutes.home.path);
nav.showSnackBar('message');
nav.showErrorDialog(errorMessage: 'Đã xảy ra lỗi');
```

Nhận arguments:

```dart
final result = Get.arguments as ResolveResultEntity;
```

---

## 8. Error Handling

```
Exception trong Repository/UseCase
    ↓ buildState() catch
    ↓ onError?(error) — return null nếu đã xử lý
    ↓ ExceptionHandler.handleException()
    ↓ AppNavigator.showSnackBar / showErrorDialog
```

| HTTP status | Xử lý tự động |
|------------|--------------|
| 401 | offAllNamed → login |
| 400 | showErrorDialog (error400) |
| 404 | showErrorDialog (error404) |
| 429 | showErrorDialog (error429) |
| 502/503 | showErrorDialog (error502/503) |
| timeout / no internet | showErrorDialog |

---

## 9. Network

**`NonAuthAppServerApiClient`** — dùng cho InstaDown (không cần auth):

```dart
final data = await _client.request(
  method: RestMethod.post,
  path: ApiUrl.resolve,
  body: {'url': url},
  cancelToken: cancelToken,
);
```

**Base URL theo env:**

```bash
# iOS simulator
flutter run --dart-define=API_BASE_URL=http://127.0.0.1:8000
# Android emulator
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000
# Production
flutter build apk --dart-define=API_BASE_URL=https://api.instadown.app
```

---

## 10. Local Storage

```dart
// Box names & keys — HiveKeys
HiveKeys.boxHistory   // 'download_history'
HiveKeys.boxSettings  // 'app_settings'
HiveKeys.themeMode    // 'theme_mode'   value: 'light'|'dark'|'system'
HiveKeys.appLanguage  // 'app_language' value: 'vi'|'en'

// Dùng qua AppHive (sl<AppHive>())
await hive.put(boxName: HiveKeys.boxSettings, key: HiveKeys.themeMode, value: 'dark');
final mode = hive.get<String>(boxName: HiveKeys.boxSettings, key: HiveKeys.themeMode);
```

---

## 11. Adding a New Feature

Checklist khi thêm feature `resolve`:

```
features/resolve/
├── data/
│   ├── model/resolve_result_data.dart, media_item_data.dart
│   └── repository/resolve_repository_impl.dart
├── domain/
│   ├── entity/resolve_result_entity.dart, media_item_entity.dart
│   ├── exception/resolve_exception.dart
│   ├── repository/resolve_repository.dart
│   └── usecase/resolve_use_case.dart
└── presentation/
    ├── binding/resolve_binding.dart   ← thêm vào AppPages
    ├── controller/resolve_controller.dart
    └── resolve_page.dart
```

Sau khi tạo xong:
1. Thêm route vào `AppRoutes` enum nếu cần
2. Thêm `GetPage(name:, page:, binding:)` vào `AppPages.pages`
3. Thêm mapper mới vào `AppBinding._bindingMappers()` nếu có
