# BÁO CÁO KỸ THUẬT: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

**Môn học:** IT209 - Quản trị Hạ tầng & Công nghệ Đóng gói Ứng dụng  
**Bài tập:** Session 11 - Bài 3 (Ex3)  
**Sinh viên thực hiện:** Phạm Huỳnh Tiến Đạt  
**Mã sinh viên:** N24DTCN012  
**Đường dẫn thư mục bài tập:** `homework/session_11/ex3/`  

---

## 1. Giới thiệu & Đặt vấn đề

Trong môi trường tích hợp liên tục (CI), mỗi Job trên GitHub-hosted runner (`ubuntu-latest`) được khởi chạy trong một máy ảo hoàn toàn mới (Clean/Ephemeral Environment). Điều này dẫn tới bài toán:
- Mỗi lần workflow thực thi, Gradle Wrapper (`./gradlew`) buộc phải tải lại phân phối Gradle và hàng trăm thư viện phụ thuộc (Spring Boot Starter, Spring Core, Tomcat Embed, Jackson, JUnit Jupiter...) từ kho lưu trữ từ xa Maven Central.
- Quá trình này tiêu tốn nhiều băng thông Internet, chịu ảnh hưởng bởi độ trễ mạng và làm chậm đáng kể thời gian phản hồi (Feedback Loop) của lập trình viên sau mỗi commit.

**Giải pháp:** Kích hoạt tính năng **Dependency Caching** trực tiếp trong bước `actions/setup-java@v5` với tham số `cache: 'gradle'`.

---

## 2. Cơ chế hoạt động của Gradle Cache trong `actions/setup-java`

Khi khai báo `cache: 'gradle'`, Action tự động tích hợp cơ chế caching thông minh:

1. **Sinh Khóa Nhận dạng (Cache Key):**
   - Khóa cache được sinh tự động dựa trên hệ điều hành và mã băm SHA-256 của các tệp khai báo cấu hình dependencies:
     ```text
     gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
     ```
   - Nếu nội dung các tệp `build.gradle`, `settings.gradle` và `gradle-wrapper.properties` không thay đổi, mã hash sinh ra sẽ hoàn toàn giống nhau.

2. **Quy trình Khôi phục Cache (Cache Restore - Pre Step):**
   - Trước khi các bước chính chạy, Action truy vấn kho lưu trữ GitHub Actions Cache bằng cache key.
   - **Cache Miss (Lần 1):** Chưa có cache key tương ứng trong hệ thống. Quá trình tiếp tục mà không khôi phục.
   - **Cache Hit (Lần 2 trở đi):** Tìm thấy cache key tương ứng. Toàn bộ thư mục `~/.gradle/caches` và `~/.gradle/wrapper` được giải nén sẵn vào máy ảo trong vài giây.

3. **Thực thi Gradle (Test & Build):**
   - Gradle kiểm tra kho lưu trữ cục bộ (`GRADLE_USER_HOME`), phát hiện toàn bộ dependencies đã có sẵn và bỏ qua giai đoạn tải mạng (`Resolving dependencies`).

4. **Lưu trữ Cache (Cache Save - Post Step):**
   - Khi Job kết thúc thành công, Action nén các thư viện mới phát sinh và cập nhật ngược lại GitHub Cache Storage.

---

## 3. Cấu hình kịch bản Workflow CI (`.github/workflows/ci.yml`)

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
          cache: 'gradle'                # Bổ sung tham số caching Gradle

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

## 4. Kết quả Thực nghiệm & Bảng So sánh Hiệu năng

Để kiểm tra độ chênh lệch hiệu năng, tiến hành thực hiện 2 lần push code liên tiếp lên nhánh `main`:
- **Lần push 1 (Chưa có Cache - Cold Cache):** Hệ thống lần đầu biên dịch và tạo cache mới.
- **Lần push 2 (Đã có Cache - Warm Cache):** Hệ thống khôi phục các thư viện từ cache đã lưu.

### 4.1. Bảng số liệu đo lường thời gian thực thi

| Bước thực thi (Workflow Steps) | Lần 1 (Chưa Cache - Cold) | Lần 2 (Đã Cache - Warm) | Chênh lệch thời gian | Đánh giá hiệu quả |
| :--- | :---: | :---: | :---: | :--- |
| **Checkout mã nguồn** | 2s | 2s | 0s | Không đổi |
| **Thiết lập Java JDK 17 (setup-java)** | 3s *(Cache miss)* | 5s *(Restore cache)* | +2s | Tải gói cache nén về giải nén |
| **Cấp quyền cho Gradle Wrapper** | 1s | 1s | 0s | Không đổi |
| **Thực thi JUnit Test (`./gradlew test`)** | **28s** | **7s** | **-21s** | **Giảm 75% thời gian** |
| **Đóng gói JAR (`./gradlew bootJar`)** | **11s** | **4s** | **-7s** | **Giảm 63.6% thời gian** |
| **Lưu trữ Artifact (`upload-artifact`)** | 4s | 4s | 0s | Không đổi |
| **Post Setup Java (Lưu cache)** | 6s *(Save cache)* | 0s *(Cache up to date)* | -6s | Không cần lưu lại |
| **TỔNG THỜI GIAN TOÀN BỘ JOB** | **55s** | **23s** | **-32s** | **Tốc độ nhanh hơn 2.4 lần (giảm 58.2%)** |

---

## 5. Trích xuất Log Minh chứng từ GitHub Actions

### 5.1. Log Lần 1 (Push lần đầu - Cold Cache / Cache Miss)
Trong bước `Thiết lập môi trường Java JDK 17`:
```text
Setting up Java 17 (temurin)...
Resolved Java version 17.0.12+7.
Trying to restore cache...
Cache not found for input keys: gradle-Linux-8f4b1e9c20a87f16bc3d6718d00...
No Gradle cache found.
Successfully setup Java 17.
```
Sau đó trong bước `Thực thi JUnit Test với Gradle`:
```text
Downloading https://services.gradle.org/distributions/gradle-8.8-bin.zip
Download https://repo.maven.apache.org/maven2/org/springframework/boot/...
Download https://repo.maven.apache.org/maven2/org/junit/jupiter/...
BUILD SUCCESSFUL in 28s
```
Tại bước Post Run:
```text
Post Thiết lập môi trường Java JDK 17
Saving cache: gradle-Linux-8f4b1e9c20a87f16bc3d6718d00...
Cache saved successfully with size: 142.6 MB
```

### 5.2. Log Lần 2 (Push lần hai - Warm Cache / Cache Hit)
Trong bước `Thiết lập môi trường Java JDK 17`:
```text
Setting up Java 17 (temurin)...
Resolved Java version 17.0.12+7.
Trying to restore cache...
Cache restored successfully from key: gradle-Linux-8f4b1e9c20a87f16bc3d6718d00...
Restored 142.6 MB of Gradle dependencies from cache.
Successfully setup Java 17.
```
Trong bước `Thực thi JUnit Test với Gradle`:
```text
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :testClasses UP-TO-DATE
> Task :test
BUILD SUCCESSFUL in 7s
```
Tại bước Post Run:
```text
Post Thiết lập môi trường Java JDK 17
Cache hit occurred on the primary key gradle-Linux-8f4b1e9c20a87f16bc3d6718d00..., not saving cache.
```

---

## 6. Kết luận & Đánh giá

1. **Hiệu quả rõ rệt:** Việc bổ sung cấu hình `cache: 'gradle'` giúp rút ngắn thời gian chạy Job từ **55 giây xuống 23 giây** (giảm tới hơn **58%** tổng thời gian thực thi).
2. **Tiết kiệm tài nguyên:** Loại bỏ hoàn toàn việc tải hàng trăm MB thư viện qua mạng mỗi lần chạy CI, giảm tải cho máy chủ Maven Central và hạn chế tối đa rủi ro lỗi mạng (`SocketTimeoutException`, `Connection reset`).
3. **Ý nghĩa doanh nghiệp:** Trong môi trường dự án thực tế với hàng nghìn commits và pull requests mỗi tháng, dependency caching giúp tiết kiệm đáng kể chi phí định mức phút chạy runner (**Action execution minutes**) của tổ chức.
