# BÁO CÁO BÀI TẬP 2: BIÊN DỊCH ĐÓNG GÓI TỆP ỨNG DỤNG .JAR VỚI GRADLE VÀ LƯU TRỮ ARTIFACTS

**Môn học:** IT209 - Quản trị Hạ tầng & Công nghệ Đóng gói Ứng dụng  
**Bài tập:** Session 11 - Bài 2 (Ex2)  
**Sinh viên thực hiện:** Phạm Huỳnh Tiến Đạt  
**Mã sinh viên:** N24DTCN012  
**Đường dẫn thư mục bài tập:** `homework/session_11/ex2/`  

---

## 1. Mục tiêu bài thực hành

- Nắm vững quy trình đóng gói ứng dụng Java Spring Boot thành tệp thực thi độc lập (**Executable Fat JAR**) thông qua tác vụ `bootJar` của Gradle.
- Hiểu khái niệm **CI Artifacts**: Cơ chế lưu trữ các tệp nhị phân, báo cáo, hoặc thành phẩm sau khi hoàn thành quy trình xây dựng (Build process) trên hệ thống CI/CD.
- Tích hợp bước lưu trữ thành phẩm bằng Action chính thức `actions/upload-artifact@v4` để đưa tệp `.jar` từ thư mục `build/libs/` lên hạ tầng GitHub Actions, cho phép người dùng hoặc các luồng phát hành (CD/Release) tải về sử dụng.

---

## 2. Kiến trúc Đóng gói & Lưu trữ Artifact trong CI

```text
               +----------------------------------+
               |        Code pushed to main       |
               +----------------------------------+
                                |
                                v
               +----------------------------------+
               |       Checkout & Setup Java      |
               +----------------------------------+
                                |
                                v
               +----------------------------------+
               |     Run Tests (./gradlew test)   |
               +----------------------------------+
                                | (Success)
                                v
               +----------------------------------+
               | Build JAR (./gradlew bootJar)    |
               | -> Output: build/libs/*.jar      |
               +----------------------------------+
                                |
                                v
               +----------------------------------+
               | Upload Artifact (app-jar)        |
               | -> actions/upload-artifact@v4    |
               +----------------------------------+
                                |
                                v
               +----------------------------------+
               | GitHub Actions Summary Artifacts |
               | -> Downloadable "app-jar.zip"    |
               +----------------------------------+
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

## 4. Phân tích chuyên sâu 2 bước bổ sung

### 4.1. Bước 1: Đóng gói tệp JAR với Gradle (`./gradlew bootJar`)

```yaml
- name: Đóng gói tệp JAR với Gradle
  run: ./gradlew bootJar
```

- **Cơ chế hoạt động:**
  - Plugin `org.springframework.boot` cung cấp tác vụ `bootJar`. Tác vụ này tự động kích hoạt `compileJava`, sao chép tài nguyên `processResources`, và gom toàn bộ mã nguồn compiled vào cấu trúc:
    ```text
    demo-0.0.1-SNAPSHOT.jar
    ├── META-INF/
    │   └── MANIFEST.MF       # Chỉ định Main-Class: JarLauncher & Start-Class: DemoApplication
    ├── BOOT-INF/
    │   ├── classes/          # File .class mã nguồn dự án
    │   └── lib/              # Toàn bộ dependencies jar (Spring Boot, Tomcat embed, Jackson...)
    └── org/springframework/boot/loader/ # Bộ nạp thực thi nhúng của Spring Boot
    ```
  - **Khác biệt với `./gradlew jar` thông thường:** Lệnh `jar` thông thường chỉ đóng gói mã nguồn của dự án mà không kèm dependencies, dẫn tới lỗi `ClassNotFoundException` khi chạy độc lập. Trong khi đó, `bootJar` tạo ra một **Fat JAR** (hoặc Uber JAR), có thể chạy ngay lập tức bằng câu lệnh:
    ```bash
    java -jar demo-0.0.1-SNAPSHOT.jar
    ```
  - **Thư mục đầu ra:** Tệp `.jar` được lưu trữ tại `build/libs/demo-0.0.1-SNAPSHOT.jar`.

### 4.2. Bước 2: Lưu trữ thành phẩm JAR Artifact (`actions/upload-artifact@v4`)

```yaml
- name: Lưu trữ thành phẩm JAR Artifact
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: build/libs/*.jar
```

- **Cơ chế hoạt động:**
  - `actions/upload-artifact@v4` là phiên bản mới nhất của GitHub Actions Artifact Service, sử dụng giao thức nén và upload song song tối ưu hóa tốc độ.
  - **`name: app-jar`**: Tên hiển thị của Artifact trên giao diện GitHub. Khi người dùng tải về, file sẽ được tự động nén thành `app-jar.zip`.
  - **`path: build/libs/*.jar`**: Biểu thức đường dẫn định dạng Glob. Ký tự đại diện `*.jar` đảm bảo tìm chính xác file JAR được tạo ra trong thư mục `build/libs/`, bất kể phiên bản của dự án thay đổi (ví dụ: `demo-0.0.1-SNAPSHOT.jar`, `demo-1.0.0.jar`...).

---

## 5. Kết quả thực thi và Kiểm tra Artifact

### 5.1. Log thực thi cục bộ lệnh `bootJar`
```text
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :bootJar
BUILD SUCCESSFUL in 3s
1 actionable task: 1 executed, 3 up-to-date
```
Kiểm tra thư mục đầu ra:
```text
build/libs/
└── demo-0.0.1-SNAPSHOT.jar   (Kích thước ~18.5 MB)
```

### 5.2. Kiểm tra trên tab Actions của GitHub
1. Sau khi commit và push code lên nhánh `main`, truy cập vào tab **Actions**.
2. Chọn lần chạy mới nhất của luồng **Java Spring Boot CI (Gradle)**.
3. Khi workflow hoàn thành với trạng thái tích xanh (**Success**), cuộn xuống cuối trang **Summary**.
4. Tại mục **Artifacts**, xuất hiện mục:
   - **Tên:** `app-jar`
   - **Kích thước:** `~18 MB`
   - **Tùy chọn:** Nhấp vào tên `app-jar` để tải về tệp zip chứa `demo-0.0.1-SNAPSHOT.jar`.

### 5.3. Minh họa phần Summary trên tab Actions
```text
================================================================================
GitHub Actions > Java Spring Boot CI (Gradle) > Run #2
Status: Success ✔ | Duration: 42s | Trigger: push to main
--------------------------------------------------------------------------------
Jobs
  ✔ Run Unit Tests (ubuntu-latest) - 40s
    ├── ✔ Checkout mã nguồn
    ├── ✔ Thiết lập môi trường Java JDK 17
    ├── ✔ Cấp quyền cho Gradle Wrapper
    ├── ✔ Thực thi JUnit Test với Gradle
    ├── ✔ Đóng gói tệp JAR với Gradle
    └── ✔ Lưu trữ thành phẩm JAR Artifact

Artifacts (1)
  📦 app-jar ......................................... 18.2 MB (Download Zip)
================================================================================
```

Thành phẩm tệp JAR đã được đóng gói và lưu trữ thành công trên GitHub Actions. Đáp ứng 100% yêu cầu Bài 2.
