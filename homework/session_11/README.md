# TỔNG HỢP BÀI TẬP SESSION 11: CI/CD WORKFLOW VỚI GITHUB ACTIONS VÀ GRADLE

**Môn học:** IT209 - Quản trị Hạ tầng & Công nghệ Đóng gói Ứng dụng  
**Khóa học:** IT209  
**Sinh viên thực hiện:** Phạm Huỳnh Tiến Đạt  
**Mã sinh viên:** N24DTCN012  
**Thư mục làm việc:** `homework/session_11/`  

---

## 1. Cấu trúc nộp bài Session 11

```text
homework/session_11/
├── README.md                                # Tài liệu tổng quan Session 11
├── ex1/                                     # Bài 1: Workflow Unit Test tự động
│   ├── .github/
│   │   └── workflows/
│   │       └── ci.yml
│   └── README.md
├── ex2/                                     # Bài 2: Đóng gói bootJar & Upload Artifact
│   ├── .github/
│   │   └── workflows/
│   │       └── ci.yml
│   └── README.md
├── ex3/                                     # Bài 3: Dependency Caching cho Gradle
│   ├── .github/
│   │   └── workflows/
│   │       └── ci.yml
│   └── report.md
└── ex4/                                     # Bài 4: GitHub Secrets & GHCR Login
    ├── .github/
    │   └── workflows/
    │       └── ci.yml
    └── README.md
```

Ngoài ra, tại thư mục gốc của dự án `ss11/` chứa toàn bộ mã nguồn Java Spring Boot hoàn chỉnh, tệp thực thi Gradle Wrapper và tệp kịch bản `.github/workflows/ci.yml` chuẩn để có thể đẩy trực tiếp lên GitHub và chạy CI tức thì.

---

## 2. Tóm tắt nội dung 4 bài tập

| Bài tập | Mục tiêu chính | Công nghệ / Action sử dụng | Sản phẩm bàn giao |
| :--- | :--- | :--- | :--- |
| **Bài 1 (Ex1)** | Khởi tạo CI workflow kích hoạt khi push lên `main`, cấu hình Java 17 Temurin, cấp quyền `gradlew` và chạy `./gradlew test`. | `actions/checkout@v5`<br>`actions/setup-java@v5` | `homework/session_11/ex1/.github/workflows/ci.yml`<br>`homework/session_11/ex1/README.md` |
| **Bài 2 (Ex2)** | Biên dịch và đóng gói Spring Boot Executable JAR bằng `./gradlew bootJar`, đính kèm Artifact lưu trữ trên GitHub Actions. | `./gradlew bootJar`<br>`actions/upload-artifact@v4` | `homework/session_11/ex2/.github/workflows/ci.yml`<br>`homework/session_11/ex2/README.md` |
| **Bài 3 (Ex3)** | Tối ưu hóa thời gian chạy CI bằng cơ chế Gradle Dependency Caching (`cache: 'gradle'`), lập báo cáo phân tích hiệu năng Lần 1 vs Lần 2. | `cache: 'gradle'` trong `setup-java` | `homework/session_11/ex3/.github/workflows/ci.yml`<br>`homework/session_11/ex3/report.md` |
| **Bài 4 (Ex4)** | Khởi tạo GitHub Personal Access Token (`write:packages`), cấu hình Repository Secret `REGISTRY_TOKEN` và đăng nhập an toàn vào GHCR. | `docker/login-action@v3`<br>`secrets.REGISTRY_TOKEN` | `homework/session_11/ex4/.github/workflows/ci.yml`<br>`homework/session_11/ex4/README.md` |

---

## 3. Tệp kịch bản Workflow CI đầy đủ nhất (`.github/workflows/ci.yml`)

Dưới đây là kịch bản hoàn chỉnh tích hợp toàn bộ các tính năng từ Bài 1 đến Bài 4:

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
      # Bước 1: Kéo mã nguồn dự án về máy ảo runner
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      # Bước 2: Cài đặt JDK 17 Temurin và kích hoạt Caching cho Gradle Dependencies (Bài 1 & Bài 3)
      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      # Bước 3: Cấp quyền thực thi cho Gradle Wrapper script trên Linux (Bài 1)
      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      # Bước 4: Thực thi kiểm thử tự động với JUnit Platform (Bài 1)
      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      # Bước 5: Biên dịch và đóng gói thành tệp thực thi Fat JAR (Bài 2)
      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      # Bước 6: Lưu trữ tệp JAR lên kho lưu trữ Artifacts của GitHub Actions (Bài 2)
      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

      # Bước 7: Xác thực đăng nhập vào GitHub Container Registry bằng Repository Secret (Bài 4)
      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}
```

---

## 4. Hướng dẫn kiểm tra và nộp bài trên GitHub

1. **Kiểm tra cục bộ:**
   ```bash
   # Chạy Unit Test
   ./gradlew test
   
   # Đóng gói JAR
   ./gradlew bootJar
   ```
2. **Đẩy mã nguồn lên GitHub:**
   ```bash
   git add .
   git commit -m "feat(ci): complete session 11 exercises 1-4"
   git push origin main
   ```
3. **Cấu hình Secret trước khi chạy Bài 4:**
   - Vào Repository **Settings** -> **Secrets and variables** -> **Actions**.
   - Tạo secret `REGISTRY_TOKEN` với giá trị là Personal Access Token (classic) có quyền `write:packages`.
4. **Kiểm tra trên tab Actions:**
   - Mở tab **Actions** trên GitHub.
   - Xác nhận Job `Run Unit Tests` chạy thành công (Tích xanh :heavy_check_mark:).
   - Kiểm tra phần **Artifacts** có tệp `app-jar`.
   - Kiểm tra log bước `setup-java` có thông báo khôi phục cache.
   - Kiểm tra log bước `docker/login-action` có thông báo `Login Succeeded`.
