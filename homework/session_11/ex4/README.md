# BÁO CÁO BÀI TẬP 4: KHAI BÁO GITHUB SECRETS VÀ CẤU HÌNH ĐĂNG NHẬP CONTAINER REGISTRY

**Môn học:** IT209 - Quản trị Hạ tầng & Công nghệ Đóng gói Ứng dụng  
**Bài tập:** Session 11 - Bài 4 (Ex4)  
**Sinh viên thực hiện:** Phạm Huỳnh Tiến Đạt  
**Mã sinh viên:** N24DTCN012  
**Đường dẫn thư mục bài tập:** `homework/session_11/ex4/`  

---

## 1. Mục tiêu bài thực hành

- Hiểu rõ tầm quan trọng của việc bảo mật thông tin xác thực (Credentials Security) trong quy trình CI/CD.
- Khởi tạo và quản lý **GitHub Personal Access Token (PAT)** với phạm vi quyền hạn (Scopes) tối thiểu cần thiết để thao tác với Registry.
- Cấu hình tệp mã hóa an toàn (**Repository Secret**) trên GitHub Repository để lưu trữ Token bảo mật.
- Cấu hình bước đăng nhập tự động vào **GitHub Container Registry (GHCR - `ghcr.io`)** sử dụng Action chính thức `docker/login-action@v3`.
- Xác nhận đăng nhập thành công (`Login Succeeded`) và đảm bảo giá trị mật khẩu được che giấu tự động (**Masked Secret**) trong nhật ký thực thi (Execution Log).

---

## 2. Tổng quan về GitHub Container Registry (GHCR) & GitHub Secrets

```text
+-----------------------+              +-----------------------+
|  GitHub Account (PAT) |              |  GitHub Repository    |
|  Scope: write:packages|              |  Settings > Secrets   |
+-----------+-----------+              +-----------+-----------+
            |                                      |
            +------------> [ REGISTRY_TOKEN ] <----+
                                   |
                                   v (Encrypted Secret)
+--------------------------------------------------------------+
| GitHub Actions Runner (ubuntu-latest)                        |
|                                                              |
|   Step: docker/login-action@v3                               |
|     registry: ghcr.io                                        |
|     username: ${{ github.actor }}                            |
|     password: ${{ secrets.REGISTRY_TOKEN }}                  |
|                                                              |
|   Log: Password: *** -> "Login Succeeded"                   |
+------------------------------+-------------------------------+
                               |
                               v (Authenticated)
+--------------------------------------------------------------+
| GitHub Container Registry (ghcr.io)                          |
| Ready for docker build & push image: ghcr.io/<owner>/<app>   |
+--------------------------------------------------------------+
```

### Tại sao tuyệt đối KHÔNG hard-code Token vào tệp kịch bản YAML?
- Các tệp `.github/workflows/*.yml` được lưu trữ trực tiếp trong kho mã nguồn Git. Nếu kho mã nguồn ở chế độ công khai (Public) hoặc chia sẻ nội bộ, bất kỳ ai có quyền truy cập mã nguồn đều có thể lấy cắp token, chiếm quyền kiểm soát Registry và tải lên các image chứa mã độc.
- **GitHub Secrets** sử dụng thuật toán mã hóa công khai bất đối xứng **NaCl Box Public-Key Cryptography**. Dữ liệu được mã hóa ngay trên trình duyệt trước khi gửi về máy chủ GitHub. Khi workflow thực thi, giá trị chỉ được giải mã tạm thời trong bộ nhớ của Runner và tự động được hệ thống mask thành `***` nếu in ra log.

---

## 3. Hướng dẫn từng bước thực hiện

### Bước 1: Khởi tạo GitHub Personal Access Token (PAT classic)
1. Đăng nhập GitHub, nhấp vào ảnh đại diện (góc trên bên phải) -> chọn **Settings**.
2. Cuộn menu bên trái xuống dưới cùng, chọn **Developer settings**.
3. Chọn **Personal access tokens** -> nhấp vào **Tokens (classic)**.
4. Chọn **Generate new token** -> chọn **Generate new token (classic)**.
5. Thiết lập các thông số:
   - **Note (Mô tả):** `GHCR Token for Session 11 CI/CD`
   - **Expiration (Thời hạn):** Chọn `30 days` hoặc `90 days` tùy nhu cầu.
   - **Select scopes (Chọn quyền):**
     - Đánh dấu chọn: `write:packages` (Tự động kích hoạt quyền phụ thuộc `read:packages`).
     - Tùy chọn: Đánh dấu `delete:packages` nếu cần quản lý dọn dẹp image cũ.
6. Nhấp nút xanh **Generate token** ở cuối trang.
7. **Lưu ý đặc biệt:** Sao chép ngay mã token hiển thị (dạng `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`) và lưu trữ an toàn, vì GitHub chỉ hiển thị token này một lần duy nhất.

---

### Bước 2: Khai báo Repository Secret trên GitHub
1. Điều hướng về kho lưu trữ dự án trên GitHub.
2. Nhấp vào tab **Settings** của Repository (lưu ý: Settings của Repo, không phải Settings tài khoản cá nhân).
3. Tại menu bên trái, tìm mục **Security** -> chọn **Secrets and variables** -> chọn **Actions**.
4. Tại mục **Repository secrets**, nhấp vào nút **New repository secret**.
5. Điền thông tin:
   - **Name:** `REGISTRY_TOKEN` *(bắt buộc viết hoa chính xác từng ký tự)*.
   - **Secret:** Dán mã Personal Access Token đã sao chép từ Bước 1 vào ô này.
6. Nhấp nút **Add secret**.
7. Màn hình hiển thị `REGISTRY_TOKEN` nằm trong danh sách Repository secrets với nhãn `Updated by [tên tài khoản] just now`.

---

### Bước 3: Cấu hình tệp kịch bản `.github/workflows/ci.yml`

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

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}
```

---

## 4. Phân tích tham số của Action `docker/login-action@v3`

| Tham số cấu hình | Giá trị thiết lập | Giải thích kỹ thuật |
| :--- | :--- | :--- |
| **`uses`** | `docker/login-action@v3` | Action chính thức do Docker Inc phát hành, chịu trách nhiệm xác thực Docker CLI với các Container Registry (Docker Hub, GHCR, AWS ECR, Google Artifact Registry...). |
| **`registry`** | `ghcr.io` | Địa chỉ tên miền máy chủ của **GitHub Container Registry**. |
| **`username`** | `${{ github.actor }}` | Biến ngữ cảnh có sẵn của GitHub Actions. Tự động lấy username của người thực hiện commit hoặc kích hoạt workflow (ví dụ: `phamhuynhtiendat` hoặc `IT-209`). |
| **`password`** | `${{ secrets.REGISTRY_TOKEN }}` | Truy xuất giá trị bí mật `REGISTRY_TOKEN` đã được lưu trữ an toàn trong Repository Secrets. Runner giải mã và truyền an toàn vào câu lệnh đăng nhập ngầm `docker login`. |

---

## 5. Kết quả kiểm tra trên GitHub Actions Log

Sau khi commit và push code lên GitHub:
1. Mở tab **Actions** -> chọn lần chạy mới nhất.
2. Nhấp vào Job **Run Unit Tests**.
3. Mở bước **Đăng nhập vào GitHub Container Registry** để quan sát log chi tiết:

```text
Logging into ghcr.io...
Username: IT-209
Password: ***
Login Succeeded!
```

### Đánh giá an toàn thông tin:
- Mật khẩu/Token hiển thị là `***`, hoàn toàn không bị rò rỉ ra bên ngoài.
- Dòng chữ `Login Succeeded!` xác nhận Token hợp lệ, có đầy đủ quyền hạn và máy ảo runner đã được cấp quyền đẩy/kéo images lên namespace của tài khoản.

---

## 6. Hướng dẫn minh chứng nộp bài

Sản phẩm nộp bài gồm:
1. Tệp kịch bản `.github/workflows/ci.yml` (đã có bước `docker/login-action@v3`).
2. **Ảnh chụp màn hình 1:** Mục **Repository Secrets** trong Settings repo hiển thị biến `REGISTRY_TOKEN`.
3. **Ảnh chụp màn hình 2:** Log chi tiết trên tab **Actions** tại bước Đăng nhập Registry hiển thị dòng chữ `Login Succeeded`.
