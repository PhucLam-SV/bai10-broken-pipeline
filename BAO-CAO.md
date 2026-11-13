# Bài 10 — The broken pipeline

Repository: https://github.com/PhucLam-SV/bai10-broken-pipeline

Pipeline cuối cùng xanh: [run 43432641157](https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43432641157) — `Tests run: 3`, `BUILD SUCCESS`.

Mỗi lỗi được sửa riêng, push, rồi mới đọc log của lần chạy kế tiếp.

## Lỗi 1 — Workflow không checkout source

- File: `.github/workflows/ci.yml`, khối `steps` (trước khi sửa, bước đầu tiên là `Set up JDK 17`, không có `actions/checkout`). Bước `mvn package` khi đó nằm ở dòng 15–16.
- Run: https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43431734702

```text
[INFO] BUILD FAILURE
[ERROR] The goal you specified requires a project to execute but there is no POM in this directory (/home/runner/work/bai10-broken-pipeline/bai10-broken-pipeline). Please verify you invoked Maven from the correct directory. -> [Help 1]
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MissingProjectException
##[error]Process completed with exit code 1.
```

Runner của GitHub Actions bắt đầu với thư mục làm việc trống. `actions/setup-java` chỉ cài JDK. Không có `actions/checkout` thì `pom.xml` và `src/` không được tải về, nên Maven ném `MissingProjectException`.

Sửa: thêm bước `actions/checkout@v4` trước khi cài JDK và gọi Maven.

## Lỗi 2 — Phiên bản logback không tồn tại

- File: `pom.xml`, dòng 16 (`<version>9.9.9</version>` của `ch.qos.logback:logback-classic`).
- Run: https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43431837517

```text
[INFO] BUILD FAILURE
[ERROR] Failed to execute goal on project shipping-app: Could not resolve dependencies for project com.lab:shipping-app:jar:1.0-SNAPSHOT
[ERROR] dependency: ch.qos.logback:logback-classic:jar:9.9.9 (compile)
[ERROR] 	Could not find artifact ch.qos.logback:logback-classic:jar:9.9.9 in central (https://repo.maven.apache.org/maven2)
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/DependencyResolutionException
```

Maven Central không có `logback-classic` bản `9.9.9`. Dòng phiên bản thật của Logback là 1.x (1.2, 1.4, 1.5). Maven dừng ở pha resolve dependency, trước cả lúc biên dịch.

Sửa: đổi version thành `1.4.14`, bản đã có trên Central.

## Lỗi 3 — Surefire 2.12.4 không chạy JUnit 5

- File: `pom.xml`, dòng 32 (`maven-surefire-plugin` version `2.12.4`).
- Run sau khi hết lỗi 2: https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43431999003

Job này **xanh** (`BUILD SUCCESS`) nhưng log chứng minh test không chạy:

```text
Downloading .../surefire-junit3/2.12.4/surefire-junit3-2.12.4.jar
Running com.lab.ShippingCalculatorTest
Tests run: 0, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0 sec
Tests run: 0, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

Test dùng `org.junit.jupiter.api.Test` (JUnit 5). Surefire 2.12.4 chỉ kéo provider `surefire-junit3`, tức runner JUnit 3. Runner đó tìm test kiểu JUnit 3 (lớp kế thừa `junit.framework.TestCase`), không nhận annotation Jupiter, nên báo 0 test rồi vẫn exit 0. `failIfNoTests` mặc định là false, vì vậy pipeline không đỏ dù bộ test không bảo vệ được gì. JUnit 5 chỉ được Surefire hỗ trợ từ 2.22.0.

Sửa: nâng plugin lên `3.2.5`. Run kế tiếp https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43432249420 báo `Tests run: 3` và `BUILD SUCCESS`.

## Lỗi 4 — tự tạo: assertion express sai

Không trùng ba lỗi trên (không phải thiếu checkout, không phải artifact không tồn tại, không phải plugin không nhận JUnit 5).

- File: `src/test/java/com/lab/ShippingCalculatorTest.java`, dòng 17.
- Đổi kỳ vọng `testExpress` từ `45000.0` thành `40000.0`.
- Công thức trong `ShippingCalculator`: `weight * 5000 + 20000`. Với weight 5: `25000 + 20000 = 45000`.
- Run đỏ: https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43432407407
- Ảnh: `pipeline-do-loi-4.png` (Status: Failure, job `build`, exit code 1).

```text
[ERROR] Tests run: 3, Failures: 1, Errors: 0, Skipped: 0 <<< FAILURE! -- in com.lab.ShippingCalculatorTest
[ERROR] com.lab.ShippingCalculatorTest.testExpress
org.opentest4j.AssertionFailedError: expected: <40000.0> but was: <45000.0>
	at com.lab.ShippingCalculatorTest.testExpress(ShippingCalculatorTest.java:17)
[INFO] BUILD FAILURE
```

Surefire 3 đã thực sự chạy test, so sánh sai nên fail cả goal `test` và `mvn package` trả exit code 1. Annotation trên Actions: `Process completed with exit code 1.`

Sửa: trả expected về `45000.0`. Run https://github.com/PhucLam-SV/bai10-broken-pipeline/actions/runs/43432641157 xanh, `Tests run: 3`.
