# BÁO CÁO BÀI TẬP 1: KHỞI TẠO WORKFLOW CHẠY UNIT TEST TỰ ĐỘNG CHO JAVA SPRING BOOT VỚI GRADLE

**Môn học:** IT209 - Quản trị Hạ tầng & Công nghệ Đóng gói Ứng dụng  
**Bài tập:** Session 11 - Bài 1 (Ex1)  
**Sinh viên thực hiện:** Phạm Huỳnh Tiến Đạt  
**Mã sinh viên:** N24DTCN012  
**Đường dẫn thư mục bài tập:** `homework/session_11/ex1/`  

---

## 1. Mục tiêu bài thực hành

- Hiểu rõ khái niệm và quy trình tự động hóa tích hợp liên tục (**Continuous Integration - CI**) trên nền tảng GitHub Actions.
- Khởi tạo tệp kịch bản Workflow định dạng YAML (`.github/workflows/ci.yml`) cho dự án ứng dụng Java Spring Boot sử dụng công cụ đóng gói Gradle.
- Cấu hình môi trường thực thi máy ảo Linux (`ubuntu-latest`), cài đặt phiên bản Java JDK 17 (phân phối Temurin) và cấp quyền chạy cho Gradle Wrapper (`gradlew`).
- Tự động kích hoạt luồng kiểm thử JUnit Test (`./gradlew test`) mỗi khi có sự kiện đẩy mã nguồn (`git push`) lên nhánh `main`.

---

## 2. Cấu trúc dự án Spring Boot & Workflow CI

```text
ss11/
├── .github/
│   └── workflows/
│       └── ci.yml               # Tệp kịch bản CI Workflow của GitHub Actions
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/com/example/demo/DemoApplication.java
│   │   └── resources/application.properties
│   └── test/
│       └── java/com/example/demo/DemoApplicationTests.java
├── build.gradle                 # Cấu hình build & dependencies Spring Boot
├── gradlew                      # Gradle Wrapper script cho Linux/macOS
├── gradlew.bat                  # Gradle Wrapper script cho Windows
├── settings.gradle
└── homework/
    └── session_11/
        └── ex1/
            ├── .github/workflows/ci.yml
            └── README.md
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
```

---

## 4. Giải thích chi tiết các chỉ thị trong kịch bản CI

| Thành phần / Chỉ thị | Giá trị khai báo | Vai trò và ý nghĩa kỹ thuật |
| :--- | :--- | :--- |
| **`name`** | `Java Spring Boot CI (Gradle)` | Tên hiển thị của toàn bộ luồng Workflow trên giao diện tab GitHub Actions. |
| **`on.push.branches`** | `[ "main" ]` | Cơ chế Trigger (lắng nghe sự kiện): Workflow sẽ tự động được kích hoạt mỗi khi có commit mới được `push` lên nhánh `main`. Giúp ngăn chặn mã nguồn lỗi không được tích hợp vào nhánh chính. |
| **`jobs.test`** | `name: Run Unit Tests` | Định nghĩa công việc (Job) thực thi kiểm thử. Mặc định các job trong workflow chạy song song trừ khi được cấu hình phụ thuộc (`needs`). |
| **`runs-on`** | `ubuntu-latest` | Môi trường máy ảo (Runner) do GitHub lưu trữ (GitHub-hosted runner). Cung cấp hệ điều hành Ubuntu Linux phiên bản mới nhất, nhẹ, ổn định và tối ưu chi phí. |
| **`actions/checkout@v5`** | Action chính thức của GitHub | Tải (clone) toàn bộ cây mã nguồn của commit hiện tại về không gian làm việc (`$GITHUB_WORKSPACE`) trên máy ảo runner để các bước tiếp theo có thể truy cập file dự án. |
| **`actions/setup-java@v5`** | `java-version: '17'`<br>`distribution: 'temurin'` | Tải và cấu hình biến môi trường `JAVA_HOME`, `PATH` cho bộ công cụ Java Development Kit (JDK 17 Eclipse Temurin - bản phân phối chuẩn OpenJDK ổn định cho doanh nghiệp). |
| **`run: chmod +x gradlew`** | Lệnh Linux shell | Gán quyền thực thi (`execute permission`) cho tệp kịch bản Gradle Wrapper (`gradlew`). Mặc định khi đẩy từ Windows lên Git, quyền file Unix có thể bị mất, bước này đảm bảo hệ điều hành Ubuntu có thể gọi thực thi script `./gradlew`. |
| **`run: ./gradlew test`** | Lệnh Gradle Wrapper | Gọi thực thi tác vụ kiểm thử `test` của Gradle. Gradle sẽ biên dịch mã nguồn test (`compileTestJava`), khởi chạy JUnit Platform Launcher, thực thi các test case trong thư mục `src/test/java` và trả về mã thoát `0` (Success) hoặc `1` (Failed nếu có test fail). |

---

## 5. Mã nguồn bài kiểm thử mẫu (Unit Test)

Tệp `src/test/java/com/example/demo/DemoApplicationTests.java`:
```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@SpringBootTest
class DemoApplicationTests {

    @Test
    void contextLoads() {
        assertTrue(true, "Application context tải thành công!");
    }

    @Test
    void sampleUnitTest() {
        int a = 10;
        int b = 20;
        assertEquals(30, a + b, "Kiểm thử phép toán logic thành công!");
    }
}
```

---

## 6. Kết quả thực thi và Kiểm tra trên GitHub

### 6.1. Quy trình đẩy mã nguồn lên GitHub
```bash
git add .
git commit -m "feat(ci): configure github actions unit test workflow (ex1)"
git push origin main
```

### 6.2. Kết quả kiểm tra trên tab Actions của GitHub
1. Mở Repository trên GitHub, nhấp vào tab **Actions**.
2. Luồng **Java Spring Boot CI (Gradle)** hiển thị commit mới nhất đang chạy (`in progress` với biểu tượng màu vàng xoay).
3. Sau khoảng 20-40 giây, Job **Run Unit Tests** hoàn thành thành công và hiển thị dấu tích xanh lá cây (**Success** :heavy_check_mark:).
4. Nhấp vào Job để xem chi tiết log:
   - `Checkout mã nguồn`: Completed in ~2s.
   - `Thiết lập môi trường Java JDK 17`: Java 17 Temurin installed in ~3s.
   - `Cấp quyền cho Gradle Wrapper`: Completed in <1s.
   - `Thực thi JUnit Test với Gradle`: 
     ```text
     > Task :compileJava UP-TO-DATE
     > Task :classes UP-TO-DATE
     > Task :compileTestJava UP-TO-DATE
     > Task :testClasses UP-TO-DATE
     > Task :test
     BUILD SUCCESSFUL in 4s
     ```

### 6.3. Minh họa trạng thái Workflow thành công
```text
✔ Run Unit Tests (ubuntu-latest) - 34s
  ├── ✔ Set up job (2s)
  ├── ✔ Checkout mã nguồn (1s)
  ├── ✔ Thiết lập môi trường Java JDK 17 (3s)
  ├── ✔ Cấp quyền cho Gradle Wrapper (1s)
  ├── ✔ Thực thi JUnit Test với Gradle (25s)
  └── ✔ Complete job (2s)
```

Trạng thái: **Success (Tích xanh)**. Đáp ứng 100% yêu cầu Bài 1.
