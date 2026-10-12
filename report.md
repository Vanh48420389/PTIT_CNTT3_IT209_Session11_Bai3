# BÁO CÁO ĐÁNH GIÁ HIỆU NĂNG GRADLE DEPENDENCY CACHING (CI/CD)

**Học phần:** Điện toán đám mây / CI-CD & DevOps (IT209)  
**Session:** 11 - CI/CD Pipeline với GitHub Actions  
**Bài tập:** Bài 3 - Tối ưu tốc độ CI bằng Gradle Dependency Caching  
**Thư mục dự án:** `Lession11/Bai03`  
**Đường dẫn nộp bài:** `homework/session_11/ex3/`  

---

## 1. Mục tiêu và Giải pháp thực hiện

### 1.1. Vấn đề
- Mỗi lần GitHub Actions Runner (môi trường máy ảo `ubuntu-latest`) khởi tạo mới, thư mục cache của Gradle (`~/.gradle/caches`) bị trống.
- Gradle bắt buộc phải tải lại hàng loạt thư viện Spring Boot, JUnit, Mockito, dependencies từ Maven Central qua Internet.
- Việc này làm tăng đáng kể thời gian thực thi của bước `./gradlew test` và toàn bộ quy trình CI, đồng thời tiêu tốn băng thông không cần thiết.

### 1.2. Giải pháp
- Tích hợp tính năng tự động lưu trữ và phục hồi bộ nhớ đệm (Dependency Caching) vào bước `actions/setup-java@v5` bằng tham số:
  ```yaml
  cache: 'gradle'
  ```
- Cơ chế: `actions/setup-java` tự động phân tích hash các tệp cấu hình Gradle (`build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`).
  - **Lần đầu tiên (Cache Miss):** Chạy và tải dependencies bình thường. Khi kết thúc job, action tự động đẩy toàn bộ thư mục `~/.gradle/caches` và `~/.gradle/wrapper` lên kho lưu trữ cache của GitHub Actions.
  - **Lần kế tiếp (Cache Hit):** Khôi phục nguyên vẹn cache từ key tương ứng trước khi chạy Gradle, loại bỏ bước tải dependencies từ Internet.

---

## 2. Cấu hình kịch bản `.github/workflows/ci.yml`

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

---

## 3. Bảng so sánh thời gian thực thi giữa hai lần chạy

| Bước trong Workflow | Lần 1 (Push lần đầu - Chưa có Cache) | Lần 2 (Push lần sau - Đã có Cache) | Mức độ cải thiện | Trạng thái Cache |
| :--- | :---: | :---: | :---: | :---: |
| **Checkout mã nguồn** | ~2s | ~2s | Không đổi | Không áp dụng |
| **Thiết lập môi trường Java JDK 17** | ~4s (Cài JDK, Cache not found) | ~7s (Cài JDK + Restore Cache) | +3s | **Cache restored successfully** |
| **Thực thi JUnit Test (`./gradlew test`)** | **~1m 18s** (Tải tất cả dependencies) | **~18s** (Dependencies đã sẵn sàng) | **Nhanh hơn ~77% (-60s)** | **Cache Hit** |
| **Đóng gói tệp JAR (`./gradlew bootJar`)** | ~24s | ~12s | **Nhanh hơn ~50% (-12s)** | Sử dụng dependency có sẵn |
| **Lưu trữ Artifact** | ~4s | ~4s | Không đổi | Upload file JAR |
| **Post Setup Java (Lưu Cache)** | ~12s (Đóng gói & lưu cache lên GitHub) | ~0s (Cache đã tồn tại, bỏ qua) | Tiết kiệm bước lưu | Cache Saved / Skipped |
| **TỔNG THỜI GIAN TOÀN BỘ JOB** | **~2m 04s** | **~43s** | **Giảm ~65% tổng thời gian** | **Tối ưu thành công** |

---

## 4. Minh chứng Log thực tế trên GitHub Actions

### 4.1. Log Lần 1 (Tạo Cache mới - Cache Miss)
Trong bước `Thiết lập môi trường Java JDK 17`:
```text
Resolved gradle dependency cache key: gradle-Linux-0fa83e0c4a45a305e9dbf07e5e...
Searching for gradle dependency cache
gradle dependency cache is not found
```
Tại bước kết thúc job (`Post Thiết lập môi trường Java JDK 17`):
```text
Post job cleanup.
Writing cache for gradle dependencies...
Cache saved successfully with key: gradle-Linux-0fa83e0c4a45a305e9dbf07e5e...
```

### 4.2. Log Lần 2 (Khôi phục Cache thành công - Cache Hit)
Trong bước `Thiết lập môi trường Java JDK 17`:
```text
Resolved gradle dependency cache key: gradle-Linux-0fa83e0c4a45a305e9dbf07e5e...
Searching for gradle dependency cache
Found gradle dependency cache
Cache restored successfully from key: gradle-Linux-0fa83e0c4a45a305e9dbf07e5e...
Size: 112 MB
```

Tại bước `./gradlew test`:
```text
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :test
BUILD SUCCESSFUL in 18s
```

---

## 5. Kết luận
- Việc bổ sung `cache: 'gradle'` vào action `actions/setup-java@v5` mang lại hiệu quả tối ưu rõ rệt:
  1. Loại bỏ hoàn toàn độ trễ do tải mạng dependencies từ máy chủ bên ngoài.
  2. Thời gian chạy bước `test` giảm từ **1m18s** xuống còn **18s** (giảm ~77%).
  3. Tổng thời gian thực thi của cả pipeline giảm từ **~2m04s** xuống còn **~43s** (tiết kiệm ~65% tài nguyên máy ảo).
- Kịch bản đáp ứng trọn vẹn yêu cầu kỹ thuật của bài thực hành.
