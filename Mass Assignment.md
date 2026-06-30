# 1. Mass Assignment

일부 소프트웨어 프레임워크의 경우 사용자가 입력한 HTTP 요청 파라미터 값을 검증 없이 애플리케이션의 객체나 변수에 그대로 바인딩(자동 매핑)하는데, 이때 발생할 수 있는 취약점이다.


# 2. 예시

```java
public class User {
   private String userid;
   private String password;
   private String email;
   private boolean isAdmin;

   //Getters & Setters
}
```

```java
@RequestMapping(value = "/addUser", method = RequestMethod.POST)
public String submit(User user) {
   userService.add(user);
   return "successPage";
}
```

위와 같이 Getter, Setter가 포함된 User 클래스가 존재하고, 사용자가 입력한 파라미터 값을 제한 없이 User 클래스에 자동 바인딩하면 사용자는 아래와 같이 파라미터 값을 전달하여 개발자가 의도하지 않은 값까지 바인딩되도록 한다.

```
POST /addUser
...
userid=bobbytables&password=hashedpass&email=bobby@tables.com&isAdmin=true
```

이렇게 요청 값을 보내면 계정이 생성될 때 isAdmin 변수가 true가 되어 관리자 권한이 부여된다.

주의해야 할 점은, 이 취약점을 발생시킬 때 Setter 메소드 명에 주목해야 한다는 것이다.

Spring 프레임워크의 경우 아래와 같은 규칙을 지켜서 파라미터 값을 보내야 취약점이 발생한다.

```
예시) Setter 메소드 명이 SetIsAdmin 일때

파라미터 명은 Setter 메소드 이름에서 Set을 떼고, IsAdmin에서 첫 번째 글자 I 만 소문자로 바꾼 isAdmin이 되어야 한다.
```

이는 Spring 프레임워크에서 내부적으로 사용하는 JavaBeans 명명 규칙(Naming Convention) 때문이다.

# 3. 대응방안

1) DTO 내부에 민감한 필드(isAdmin, isAuth 등)가 포함되지 않도록 DTO를 작성한다.
```
public class UserRegistrationFormDTO {
 private String userid;
 private String password;
 private String email;

 //NOTE: isAdmin field is not present

 //Getters & Setters
}
```

2) 화이트리스트 바인딩 설정
```
@Controller
public class UserController
{
    @InitBinder
    public void initBinder(WebDataBinder binder, WebRequest request)
    {
        binder.setAllowedFields(["userid","password","email"]);
    }
...
}
```


# 4. 참고
- https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
