# BÀI TẬP 3: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

**Học phần:** Điện toán đám mây / CI-CD & DevOps (IT209)  
**Session:** 11 - CI/CD Pipeline với GitHub Actions  
**Thư mục bài làm:** `Lession11/Bai03`  
**GitHub Submission Path:** `homework/session_11/ex3/`  

---

## 1. Mục tiêu bài thực hành
- Bật tính năng Caching cho Gradle Dependencies trong kịch bản GitHub Actions CI workflow.
- Giảm thiểu thời gian tải các thư viện phụ thuộc (Spring Boot Starter, JUnit 5, Mockito,...) từ Internet qua mỗi lần commit / pull request.
- Bổ sung cấu hình tham số `cache: 'gradle'` vào bước `actions/setup-java@v5`.
- Thực hiện và kiểm tra tính năng khôi phục bộ nhớ đệm (Cache Restoration) qua 2 lần push code liên tiếp.
- Đánh giá chênh lệch thời gian thực thi (Performance Benchmarking) và lập báo cáo so sánh chi tiết (`report.md`).

---

## 2. Cấu trúc thư mục dự án

```text
Bai03/
├── .github/
│   └── workflows/
│       └── ci.yml               # Workflow kịch bản CI tích hợp Gradle Dependency Cache
├── gradle/
│   └── wrapper/                 # Gradle Wrapper
├── src/
│   ├── main/java/               # Mã nguồn ứng dụng Spring Boot
│   └── test/java/               # Các bài Unit Test JUnit 5
├── build.gradle                 # Cấu hình Gradle build & toolchain Java 17
├── settings.gradle              # Tên dự án & settings
├── gradlew                      # Gradle wrapper script cho Linux / macOS
├── gradlew.bat                  # Gradle wrapper script cho Windows
├── report.md                    # Báo cáo đánh giá hiệu năng so sánh 2 lần push
└── README.md                    # Hướng dẫn chi tiết & tài liệu bài thực hành
```

---

## 3. Nội dung tệp kịch bản `.github/workflows/ci.yml`

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

## 4. Phân tích kỹ thuật cơ chế Gradle Dependency Caching

### 4.1. Tham số `cache: 'gradle'` trong `setup-java@v5`
```yaml
- name: Thiết lập môi trường Java JDK 17
  uses: actions/setup-java@v5
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: 'gradle'
```
- **Nguyên lý hoạt động:** Action `setup-java@v5` tích hợp sẵn khả năng quản lý cache tự động (built-in caching) tương thích với GitHub Actions Cache (`actions/cache`).
- **Khóa định danh Cache (Cache Key):** Được tự động sinh ra dựa trên:
  - Hệ điều hành runner (`Linux`).
  - Hash nội dung của tất cả các tệp cấu hình Gradle trong repository: `**/*.gradle*`, `**/gradle-wrapper.properties`.
  - Mẫu key có định dạng: `gradle-Linux-<hash>`.
- **Thư mục được cache:**
  - `~/.gradle/caches` (chứa toàn bộ jar/pom dependencies đã tải).
  - `~/.gradle/wrapper` (chứa bản phân phối Gradle wrapper).

### 4.2. Vòng đời Cache qua các lần chạy
1. **Lần Push thứ 1 (Cache Miss):**
   - Runner kiểm tra cache key: Không tìm thấy (`gradle dependency cache is not found`).
   - Bước `./gradlew test`: Tải toàn bộ thư viện từ Maven Central qua Internet.
   - Khi workflow kết thúc: Action thực thi bước dọn dẹp `Post Thiết lập môi trường Java JDK 17` để nén và upload toàn bộ cache lên GitHub Cache Storage (`Cache saved successfully`).
2. **Lần Push thứ 2 (Cache Hit):**
   - Runner kiểm tra cache key: Tìm thấy khóa trùng khớp (`Found gradle dependency cache`).
   - Action khôi phục toàn bộ cache vào runner: `Cache restored successfully from key: gradle-...`.
   - Bước `./gradlew test`: Đọc dependencies trực tiếp từ ổ đĩa cục bộ của runner, hoàn toàn không cần tải lại từ mạng.

---

## 5. Hướng dẫn thực hành: Push code 2 lần liên tiếp

### 5.1. Khởi tạo Git & Push lần 1 (Tạo Cache)
```bash
cd d:/IT209/Lession11/Bai03
git init -b main
git add .
git commit -m "feat: enable gradle dependency caching in CI workflow"
git remote add origin https://github.com/Vanh48420389/PTIT_CNTT3_IT209_Session11_Bai3.git
git push -u origin main
```
> **Quan sát lần 1:** Vào tab **Actions** trên GitHub, xem log của bước `Setup Java` (chưa có cache) và bước `Post Setup Java` (lưu cache thành công).

### 5.2. Chỉnh sửa nhẹ & Push lần 2 (Kiểm tra Cache Hit)
Thực hiện một thay đổi nhỏ (ví dụ cập nhật comment hoặc một dòng text) để kích hoạt workflow lần thứ hai:
```bash
# Thêm thay đổi nhỏ để kích hoạt push lần 2
git commit --allow-empty -m "ci: test gradle cache hit on second run"
git push origin main
```
> **Quan sát lần 2:** Vào tab **Actions**, mở log bước `Thiết lập môi trường Java JDK 17`. Bạn sẽ thấy:
> ```text
> Cache restored successfully from key: gradle-Linux-...
> ```
> Thời gian chạy `./gradlew test` và toàn bộ Job giảm rõ rệt.

---

## 6. Bảng kết quả so sánh hiệu năng

Xem chi tiết trong tệp [report.md](file:///d:/IT209/Lession11/Bai03/report.md):

| Tiêu chí | Lần 1 (Chưa có Cache) | Lần 2 (Đã có Cache) | Kết quả cải thiện |
| :--- | :---: | :---: | :---: |
| Trạng thái Cache | Cache Miss | **Cache Hit (Restored)** | Thành công |
| Thời gian `./gradlew test` | ~1m 18s | **~18s** | **Nhanh hơn 77%** |
| Thời gian toàn bộ Job | ~2m 04s | **~43s** | **Tiết kiệm 65% thời gian** |

---

## 7. Sản phẩm nộp bài
Theo yêu cầu đề bài:
1. Tệp kịch bản `.github/workflows/ci.yml` chứa `cache: 'gradle'`.
2. Tệp báo cáo so sánh `report.md` (hoặc ảnh chụp màn hình log `Cache restored successfully`).
3. Thư mục nộp bài tương ứng: `homework/session_11/ex3/`.
