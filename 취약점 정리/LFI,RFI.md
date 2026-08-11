# 1. 취약점 정의

웹 애플리케이션이 사용자 입력 값을 적절히 검증하지 않고 서버 내부의 파일 경로를 참조할 때 발생하는 취약점이다. 서버 내부의 파일 경로를 참조하는 것은 LFI(Local File Inclusion), 원격으로 외부 서버에 있는 파일을 참조하는 것은 RFI(Remote File Inclusion)이라고 한다.

이 취약점은 각 언어의 특정 함수를 사용했을 때 발생할 가능성이 생긴다.

- PHP : include, require, file_get_contents
- Node.js : fs.readfile, require
- Python : open
- Java(JSP) : <jsp:include page="..." /> 태그 또는 RequestDispatcher.forward

# 2. 점검 방법

# 3. 대응 방안

# 4. 참고

- https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion
