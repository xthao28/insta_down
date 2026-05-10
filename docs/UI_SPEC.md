# UI/UX Spec — InstaDown v1.0

Flutter mobile (iOS + Android). Font: **Inter**. State management: Riverpod.

---

## Design Tokens

### Colors (xem `core/values/collection.dart`)

| Token | Light | Dark | Dùng cho |
|---|---|---|---|
| `primary` | `#E1306C` | `#E1306C` | CTA, active state, link |
| `gradientStart/Mid/End` | `#833AB4` `#E1306C` `#F77737` | — | Button CTA, progress bar |
| `background` | `#FAFAFA` | `#000000` | Scaffold |
| `surface` | `#FFFFFF` | `#121212` | Card, AppBar, sheet |
| `surfaceVariant` | `#F0F0F0` | `#1E1E1E` | Input bg, chip |
| `onSurface` | `#262626` | `#EDEDED` | Text chính |
| `textSecondary` | `#737373` | `#8E8E8E` | Caption, metadata |
| `textDisabled` | `#BBBBBB` | `#4D4D4D` | Placeholder, disabled |
| `border` | `#DBDBDB` | `#363636` | Input border, card border |
| `divider` | `#EBEBEB` | `#2A2A2A` | List separator |
| `success` | `#00C853` | `#69F0AE` | Download done |
| `error` | `#E53935` | `#EF5350` | Lỗi |
| `info` | `#1E88E5` | — | Info banner |

### Spacing (xem `core/values/const.dart` → `AppSpacing`)
`xs=4` · `sm=8` · `md=12` · `lg=16` · `xl=20` · `xxl=24` · `h1=32`  
Screen horizontal padding: **20dp**

### Border Radius (xem `AppRadius`)
`xs=4` · `sm=8` · `md=12` · `lg=16` · `full=999`

### Sizes (xem `AppSizes`)
Button height: **56dp** · AppBar: **56dp** · BottomNav: **64dp** · Tap min: **44dp**  
Icon: sm=16 · md=20 · lg=24 · xl=32 · hero=80

### Durations (xem `AppDuration`)
fast=150ms · normal=250ms · slow=350ms

### Typography
Font: Inter · Scale: headlineMedium=22 · titleLarge=16 · bodyMedium=14 · labelLarge=14(w600) · bodySmall=12

---

## Màn hình 1 — Home Screen

**Layout (top→bottom, padding horizontal 20dp):**
1. Logo + App name (`headlineMedium`, căn trái)
2. Subtitle caption (`bodyMedium`, `textSecondary`)
3. **URL Input Area** — TextField (`surfaceVariant` bg, radius `md`, border đổi màu theo state) + icon `link-2` bên trái + nút Clear `[✕]` khi có text
4. **Paste Button** — `OutlinedButton` full-width 48dp, icon clipboard
5. **CTA Button** — `FilledButton` gradient 56dp full-width, icon `download-cloud`
6. Divider "hoặc"
7. **Share Hint Card** — bg `infoLight`, icon `share-2`, hướng dẫn share từ Instagram

**States:**

| State | Input border | CTA button | Ghi chú |
|---|---|---|---|
| Idle | `border` 1dp | Disabled (flat grey) | Check clipboard tự động khi mở app |
| Valid URL | `success` 1.5dp | Enabled (gradient) | — |
| Loading | `border` | Spinner trắng + "Đang phân tích..." | Input + buttons disabled |
| Error | `error` 1.5dp | Re-enabled | Error banner slide-down bên dưới input |

---

## Màn hình 2 — Preview & Download Screen

**AppBar:** `[←]` Back · `@username` (titleLarge, truncate) · `[↗]` Share icon

**Media Preview Area** (chiếm ~45% screen height):
- **Photo:** `CachedNetworkImage`, AspectRatio theo width/height API (fallback 1:1), radius `lg`
- **Video/Reel:** thumbnail + overlay play button (circle 64dp, 20% white) + duration badge bottom-left + "Reels" pill badge top-left nếu là reel
- **Carousel:** `PageView`, prev/next arrows 36dp, dot indicator (active=8dp gradient, inactive=6dp border), badge `"2/4"` top-right

**Media Info Section:**
```
[icon]  Carousel · 4 items              ← bodyMedium, textSecondary
        3 ảnh + 1 video
[👤]   @username                         ← titleMedium
```

**Selection Panel** (chỉ với carousel):
- Header: "Chọn nội dung tải xuống" + TextButton "Chọn tất cả" / "Bỏ chọn"
- `ListTile` mỗi item: thumbnail 40dp + tên + resolution + `Checkbox` gradient
- Counter: "Đã chọn 3/4" bottom
- **Mặc định: tất cả checked**

**Download Button:** gradient CTA, text cập nhật theo selection ("Tải xuống" / "Tải xuống (3 ảnh)")

**Download Progress — Modal Bottom Sheet** (isDismissible: false khi đang tải):
- Mỗi item: tên + `LinearProgressIndicator` gradient + speed "1.2 MB/s"
- Done: checkmark fade-in (`success` color)
- All done: header → "Hoàn tất!" + nút "Xem trong Thư viện" slide-up

---

## Màn hình 3 — History Screen

**AppBar:** "Lịch sử tải xuống" + TextButton "Xóa hết" (chỉ hiện khi list có data)

**List Item:**
```
[thumbnail 72×72, radius sm]  @username            Hôm nay 14:32
                               Carousel · 4 items
                               [🖼 3]  [🎬 1]        ← chips
```
- Timestamp: "Hôm nay HH:mm" / "Hôm qua HH:mm" / "DD/MM/YYYY"
- **Swipe left to delete** (`Dismissible`): background đỏ + icon trash + haptic medium

**Empty State:**
Icon `clock` 80dp (`textDisabled`) · "Chưa có lịch sử tải xuống" · CTA "Đến trang tải xuống"

---

## Global Components

**Bottom Navigation Bar** (2 tab):
- Tab 1: icon `house` · label "Tải về"
- Tab 2: icon `clock` · label "Lịch sử"
- Active: icon + label gradient color + pill indicator top
- Background: `surface`, border-top 1dp `divider`

**Loading Skeleton:** `shimmer` package, base/highlight color theo dark/light, cùng layout với màn hình thực

**Snackbar** (floating, radius `md`, margin bottom 80dp):

| Type | Icon | Color |
|---|---|---|
| success | check-circle | success |
| error | alert-circle | error |
| info | info | info |

**Error Banner** (inline, persistent):
bg `errorLight`, border 1dp error 30%, icon `alert-circle`, text `bodySmall`, nút "Thử lại" nếu `isRetryable`

**Permission Dialog:** Khi cần Photos — custom dialog trước khi request. Nếu permanently denied → dialog với nút "Mở Cài đặt" (`openAppSettings()`).

---

## Micro-interactions

| Hành động | Animation | Haptic |
|---|---|---|
| Paste URL | Text type 200ms + border green 300ms + CTA gradient fade 350ms | selection |
| CTA tap | scale 0.97 100ms | light |
| Error xuất hiện | Banner slide-down 300ms bounce | heavy |
| Carousel swipe | Spring physics, dot animate 200ms | — |
| Checkbox toggle | scale 0.9→1.0 spring, checkmark draw 200ms | selection |
| "Chọn tất cả" | Staggered check mỗi item cách 40ms | light (1 lần) |
| Download tap | Bottom sheet slide-up 400ms spring | medium |
| Từng file xong | Progress fill → checkmark scale spring 400ms | light |
| Tất cả xong | Header crossfade → "Hoàn tất!" + action slide-up | heavy + notification |
| Swipe delete | Red bg reveal, item slide-out 300ms | medium |

**Haptic utility:** `AppHaptics.light()` / `.medium()` / `.heavy()` / `.selection()`

---

## Platform Notes

| | iOS | Android |
|---|---|---|
| Back | Swipe cạnh trái + nút AppBar | Hardware/gesture + nút AppBar |
| Progress | `CupertinoActivityIndicator` | `CircularProgressIndicator` |
| Gallery | Photos app, album "InstaDown" | MediaStore, folder "InstaDown" |
| Permission | iOS native dialog | Custom rationale → system dialog |
| Share | `share_plus` → iOS native sheet | `share_plus` → Android sharesheet |

**Safe Area:** `SafeArea(top: true, bottom: false)` trong Scaffold body; NavigationBar tự handle bottom inset.

**Text scale:** clamp 0.8–1.3 để tránh vỡ layout.

**Dark mode:** `ThemeMode.system`, dùng `AppTheme.light` / `AppTheme.dark` từ `core/values/app_theme.dart`.
