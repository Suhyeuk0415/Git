# [Rocky Linux 9] SQL Injection 취약점 분석 및 대응(시큐어 코딩) 실습 포트폴리오

## 📋 프로젝트 개요
* **목적**: Rocky Linux 9 환경에서 웹 애플리케이션의 SQL Injection 취약점을 직접 실습하고, 원인 분석 및 선처리 질의문(Prepared Statement)을 통한 시큐어 코딩 방어 기제를 적용하는 포트폴리오입니다.
* **실습 환경**:
  * **서버 (Blue Team / Target)**: Rocky Linux 9 (`192.168.62.130`)
  * **공격자 (Red Team)**: Kali Linux
  * **호스트**: Windows

---

## 🛠️ 실습 단계별 구현 및 캡처

### 1. 데이터베이스 구축 및 관리자 계정 생성
* **내용**: MariaDB를 설치하고 `sqli_lab` 데이터베이스와 `users` 테이블을 생성한 뒤, 관리자(`admin`) 계정을 등록하였습니다.
* **증명 사진**: <img width="619" height="231" alt="image" src="https://github.com/user-attachments/assets/a23ee97d-239b-4519-b2b9-6a7e59c06400" />

```sql
CREATE DATABASE sqli_lab;
USE sqli_lab;
CREATE TABLE users (id VARCHAR(50), password VARCHAR(50));
INSERT INTO users (id, password) VALUES ('admin', 'SuperSecretPassword123!');
2. 취약한 로그인 페이지(login.php) 구현
내용: 사용자 입력값에 대한 검증이나 이스케이프 처리가 전혀 없어 SQL Injection에 취약한 로그인 웹 페이지를 Apache 웹 서버 경로(/var/www/html/login.php)에 생성하였습니다.

* **증명 사진**:<img width="546" height="64" alt="image" src="https://github.com/user-attachments/assets/09e163db-d4aa-47ef-8af4-4b7f58f7dc26" />


3. 웹 서비스 접속 확인
내용: 칼리 리눅스(공격자)의 웹 브라우저를 통해 방어자 서버의 로그인 페이지에 정상적으로 접속되는지 확인하였습니다.

* **증명 사진**:<img width="1030" height="337" alt="image" src="https://github.com/user-attachments/assets/66ff5c43-2bae-411c-8f96-26d10598a826" />


4. SQL Injection 공격 수행 (인증 우회)
내용: 패스워드를 모르는 상태에서 인증을 우회하기 위해 ID 입력란에 주석 페이로드(admin' #)를 입력하여 뒤쪽의 패스워드 검증 쿼리를 무력화하고 로그인을 시도했습니다.

* **증명 사진**:<img width="980" height="251" alt="image" src="https://github.com/user-attachments/assets/9c533a72-72fc-4619-8367-524f85c04564" />


💻 소스코드 비교 및 보안 분석
1. 취약한 코드 (Vulnerable Code)
사용자의 입력값($userid, $password)이 SQL 쿼리 문자열에 직접 결합되어 있어, 공격자가 특수문자나 주석(#)을 이용해 쿼리 구조를 임의로 조작(SQL Injection)할 수 있습니다.

PHP
<?php
$conn = new mysqli("localhost", "webuser", "1234", "sqli_lab");

if (isset($_POST['userid'])) {
    $userid = $_POST['userid'];
    $password = $_POST['password'];

    // ⚠️ 취약점: 사용자 입력값이 쿼리에 직접 문자열로 결합됨
    $query = "SELECT * FROM users WHERE id = '$userid' AND password = '$password'";
    $result = $conn->query($query);

    if ($result->num_rows > 0) {
        echo "<h1>로그인 성공! 관리자 시스템에 오신 것을 환영합니다.</h1>";
    } else {
        echo "<h1>로그인 실패: 계정 정보가 틀렸습니다.</h1>";
    }
}
?>
<form method="POST">
    ID: <input type="text" name="userid"><br>
    PW: <input type="password" name="password"><br>
    <input type="submit" value="Login">
</form>
2. 선처리 질의문 적용 코드 (Secure Code - Prepared Statement)
쿼리의 뼈대를 미리 정의(? 바인딩)한 후, 사용자의 입력값은 실행 단계에서 순수한 데이터(문자열)로만 취급되도록 분리하여 SQL Injection을 원천 차단합니다.

PHP
<?php
$conn = new mysqli("localhost", "webuser", "1234", "sqli_lab");

if (isset($_POST['userid'])) {
    $userid = $_POST['userid'];
    $password = $_POST['password'];

    // ✔️ 방어: 쿼리 구조를 고정하고 ? 기호로 입력값 영역을 분리
    $stmt = $conn->prepare("SELECT * FROM users WHERE id = ? AND password = ?");
    
    // ✔️ 바인딩: 입력값을 문자열(string, "ss") 파라미터로 안전하게 전달
    $stmt->bind_param("ss", $userid, $password);
    
    $stmt->execute();
    $result = $stmt->get_result();

    if ($result->num_rows > 0) {
        echo "<h1>로그인 성공! 관리자 시스템에 오신 것을 환영합니다.</h1>";
    } else {
        echo "<h1>로그인 실패: 계정 정보가 틀렸습니다.</h1>";
    }
    
    $stmt->close();
}
$conn->close();
?>
<form method="POST">
    ID: <input type="text" name="userid"><br>
    PW: <input type="password" name="password"><br>
    <input type="submit" value="Login">
</form>
🛡️ 결론 및 대응 방안
취약점 원인: 사용자 입력값이 SQL 쿼리 문자열에 직접 결합되어 공격자가 입력한 특수문자나 주석(' #)이 SQL 구문의 일부로 해석됨.

해결 기법: Prepared Statement(선처리 질의문)를 도입하여 쿼리의 구조를 고정하고 입력값을 매개변수화함으로써, 동일한 인젝션 페이로드가 입력되더라도 공격 코드가 아닌 단순 문자열 데이터로 취급되어 SQL Injection 공격을 원천 차단함.
