# Hướng dẫn phát triển MultiApp

## Đối tượng và trạng thái

Tài liệu này dành cho người tiếp tục phát triển fork `LamPPKK/MultiApp`. Baseline hiện tại dùng `compileSdk`/`targetSdk` 36, AGP 8.13.2, Gradle 8.13, Kotlin 2.3.20, Java 17 và Compose BOM 2026.06.00. Debug, release và unit-test task đã build thành công; chưa có kết quả chạy thực tế trên thiết bị Android 14–16.

Repo có ba module và một submodule Git:

```text
app (master 64-bit, UI Compose)
  +-- opensdk: API Java, manifest proxy, bootstrap engine
  +-- assets/assist.apk: APK 32-bit được build từ module assist

assist (assistant 32-bit)
  +-- opensdk với flavor assistant

opensdk (Git submodule)
  +-- source wrapper/API
  +-- assets/moon.jar và native binaries của engine
```

## Hành vi và điểm vào chính

- `app/src/main/java/org/waxmoon/MoonApplication.java` kế thừa `HackApplication` để bootstrap engine.
- `app/src/main/java/org/waxmoon/AppListActivity.kt` là launcher mẫu: liệt kê app host, cài clone và gọi `HackApi.startActivity`.
- `opensdk/src/main/java/com/hack/opensdk/HackApplication.java` gọi `HackRuntime.install(...)` trong `attachBaseContext`.
- `opensdk/src/main/java/com/hack/opensdk/HackApi.java` là facade cho install, package query, lifecycle và start/stop app.
- `opensdk/src/main/java/com/hack/server/core/HackServerProvider.java` bootstrap server process; `ForgroundService` giữ runtime hoạt động bằng FGS `specialUse`.
- `assist` là package 32-bit dùng cùng shared UID/signature với master để hỗ trợ app `armeabi-v7a`.

## Hợp đồng build quan trọng

- Luôn clone với submodule. Commit của MultiApp chỉ hợp lệ khi commit OpenSDK tương ứng đã được push tới remote trong `.gitmodules`.
- `app` dùng OpenSDK flavor `master`; `assist` dùng flavor `assistant`. Không gộp hai manifest flavor.
- `:app:preDebugBuild` phụ thuộc `:assist:syncAssistDebugApk`; `:app:preReleaseBuild` phụ thuộc `:assist:syncAssistApk`.
- Vì vậy APK master phải chứa `assets/assist.apk` giống byte-for-byte với APK assist cùng build type.
- Master và assistant hiện dựa vào cùng `android:sharedUserId` và debug certificate. Đổi applicationId/signing ở một bên phải đổi đồng bộ bên còn lại.

## Chuẩn bị môi trường

Yêu cầu:

- JDK 17 đầy đủ.
- Android SDK Platform 36.
- Git có hỗ trợ submodule.
- Kết nối tới Google Maven và Maven Central cho lần resolve dependency đầu tiên.

Clone phiên bản chính:

```bash
git clone --recurse-submodules https://github.com/LamPPKK/MultiApp.git
cd MultiApp
git submodule update --init --recursive
```

Ví dụ biến môi trường:

```bash
export JAVA_HOME=/path/to/jdk-17
export ANDROID_HOME=/path/to/Android/sdk
```

## Build và xác minh

```bash
./gradlew :app:assembleDebug :app:testDebugUnitTest
./gradlew :app:assembleRelease
```

Output:

- `app/build/outputs/apk/debug/app-debug.apk`
- `app/build/outputs/apk/release/app-release.apk`
- `assist/build/outputs/apk/debug/assist-debug.apk`
- `assist/build/outputs/apk/release/assist-release.apk`

Kiểm tra assistant được nhúng đúng:

```bash
unzip -p app/build/outputs/apk/release/app-release.apk assets/assist.apk | shasum -a 256
shasum -a 256 assist/build/outputs/apk/release/assist-release.apk app/src/main/assets/assist.apk
```

Ba hash release phải giống nhau. Lặp lại với debug khi sửa task graph.

## Những thay đổi Android 14–16 đã có

- File `moon.jar` và native extraction dùng read-only-before-write; workspace cũ writable được tạo lại.
- Master/assistant manifest được tách bằng product flavor, có exported flag và các property tương thích cần thiết.
- Server runtime khởi động FGS đúng API, tạo notification channel và khai báo `specialUse`.
- Launcher xử lý system-bar insets; application opt out predictive back để bảo toàn proxy flow hiện tại.
- SDK 36 nullable platform types đã được xử lý trong launcher.
- Assistant APK luôn được build và đồng bộ trước khi package master debug/release.

## Bất biến, giới hạn và điểm dễ lỗi

- `opensdk/src/main/assets/moon.jar` và các native binary liên quan là prebuilt/closed. Không sửa được hành vi lõi chỉ bằng wrapper Java.
- Native libraries hiện có ELF alignment 4 KB. `android:pageSizeCompat="enabled"` chỉ bật compatibility mode; đây không phải bảo đảm chạy native ổn định trên mọi thiết bị Android 16 dùng page size 16 KB.
- Release hiện ký bằng debug certificate. Cần signing riêng trước khi phát hành, nhưng master và assistant phải tiếp tục dùng cùng certificate.
- `sharedUserId` là cơ chế legacy; không tự ý bỏ nếu chưa thiết kế lại IPC, file/data sharing và luồng cài assistant.
- Host target 36 có thể ép edge-to-edge cho proxy Activity của guest; engine proxy nằm trong `moon.jar`, nên không thể sửa hoàn toàn từ source hiện có.
- Không commit một `app/src/main/assets/assist.apk` được build từ variant khác với master tương ứng.

## Ma trận kiểm thử trước khi phát hành

1. Cài mới và nâng cấp cả master/assistant trên Android 14, 15 và 16.
2. Cài/chạy app host 64-bit và 32-bit; xác nhận assistant tự triển khai và shared UID hợp lệ.
3. Activity, provider, receiver, background start, notification và FGS của nhiều guest phổ biến.
4. Google Play Services, split APK, app có native code và app dùng cả portrait/landscape.
5. Kill/restart/reboot, dữ liệu clone, file provider và storage/media permissions.
6. Thiết bị/emulator 16 KB page size; ghi rõ app nào chỉ chạy nhờ compatibility mode.

## Câu hỏi mở / backlog

- Có thể nhận binary `moon.jar`/`libmultiapp.so` được rebuild với 16 KB ELF alignment từ nhà cung cấp không?
- Có cần flavor `targetSdk 34` cho sideload trong thời gian target 36 chưa qua đủ device matrix không?
- Có thể thay shared UID và assistant sidecar bằng IPC/signing model hiện đại hơn không?
- Cần CI checkout submodule, build cả hai variant và so hash assistant tự động.

Chi tiết wrapper OpenSDK nằm tại `opensdk/docs/DEVELOPMENT.md`.
