# 1. 취약점 정의

임의 이용자의 권한으로 임의 주소에 HTTP 요청을 보낼 수 있는 취약점이다. 공격자는 이용자의 계정으로 임의 금액을 송금해 금전적인 이득을 취하거나 비밀번호를 변경해 계정을 탈취하고, 관리자 계정을 공격해 공지사항 작성 등으로 혼란을 야기할 수 있다.

# 2. 취약점 점검 방법
CSRF는 기본적으로 XSS와 유사하다. XSS가 발생한다면 CSRF 취약점이 존재하는지도 의심할 수 있다. 취약점이 존재하는지 판별하는 방법은 스크립트나 HTML 태그를 입력해보고 동작하는지 확인하는 것이다.

아래의 코드에는 CSRF 취약점이 존재한다. /sendmoney 기능이 실행될 때, 이용자로부터 예금주와 금액을 입력받고 송금을 수행한다. 이때 계좌 비밀번호, OTP 등을 사용하지 않기 때문에 로그인한 이용자는 추가 인증 없이 해당 기능을 이용할 수 있다.
```python
# 이용자가 /sendmoney에 접속했을때 아래와 같은 송금 기능을 웹 서비스가 실행함.
@app.route('/sendmoney')
def sendmoney(name):
        # 송금을 받는 사람과 금액을 입력받음.
        to_user = request.args.get('to')
	amount = int(request.args.get('amount'))
	
	# 송금 기능 실행 후, 결과 반환	
	success_status = send_money(to_user, amount)
	
	# 송금이 성공했을 때,
	if success_status:
	    # 성공 메시지 출력
		return "Send success."
	# 송금이 실패했을 때,
	else:
	    # 실패 메시지 출력
		return "Send fail."
```

CSRF 공격에 성공하기 위해서는 공격자가 작성한 악성 스크립트를 이용자가 실행해야 한다. 이용자에게 스크립트가 노출되게 하는 방법은 메일을 보내거나 게시판에 글을 작성하는 방법이 있으며, CSRF 공격 스크립트는 HTML 또는 Javascript를 통해 작성될 수 있다.

/sendmoney와 같이 GET 메소드 요청을 받는 경우, img 태그의 src 속성을 통해 CSRF 공격을 시도할 수 있다.

```HTML
<img src='http://bank.dreamhack.io/sendmoney?to=Dreamhack&amount=1337' width=0px height=0px>
```

그리고 POST 요청을 받는 경우, 웹 페이지에 입력된 양식을 전송하는 form 태그를 사용하는 방법이 있다.

```HTML
<form action="https://test.dreamhack.io/users/1" method="post">
	<input name="user">
	<input name="pass">
	<input name="submit">
</form>
```

Javascript를 사용하는 방법도 있다.

```javascript
window.open('http://bank.dreamhack.io/sendmoney?to=Dreamhack&amount=1337');

location.href = 'http://bank.dreamhack.io/sendmoney?to=Dreamhack&amount=1337';
location.replace('http://bank.dreamhack.io/sendmoney?to=Dreamhack&amount=1337');
```

> [!TIP]
> XSS와 CSRF는 스크립트를 웹 페이지에 작성해 공격한다는 점에서 매우 유사하다.
> 공통점은 두 개의 취약점 모두 클라이언트를 대상으로 하는 공격이며, 이용자가 악성 스크립트가 포함된 페이지에 접속하도록 유도해야 한다는 것이다.
> 차이점은 공격 목적에 있다. XSS는 인증 정보인 세션 및 쿠키 탈취를 목적으로 하는 공격이며, 공격할 사이트의 오리진에서 스크립트를 실행한다. CSRF는 이용자가 임의 페이지에 HTTP 요청을 보내는 것을 목적으로 한다. 또한, 공격자는 악성 스크립트가 포함된 페이지에 접근한 이용자의 권한으로 웹 서비스의 임의 기능을 실행할 수 있다.

# 3. 대응 방안

## 3.1. Referer 헤더 확인

서버에서 사용자의 요청 내용에서 Referer 헤더를 확인하는 방법이 있다. Referer 헤더에는 요청을 보낸 직전 페이지의 URL이 담긴다. 예를 들면, https://test.com 과 https://qwer.com 이 존재하고 https://test.com 에서 CSRF 스크립트가 실행되어 https://qwer.com 쪽으로 요청을 보내게 된다고 하면, 요청 내용에서 Referer 헤더에는 요청이 시작된 https://test.com URL이 담기는 것이다. Referer 헤더의 이러한 특성을 이용해 Referer 헤더를 검증하여 CSRF를 방어한다. 하지만 이 방법은 Same Origin에서 발생하는 CSRF는 방어하기 어렵다는 단점이 존재한다.

## 3.2. CSRF Token

서버 측에서 사용자의 세션 또는 임의의 난수를 생성하고, 이를 사용자에 대한 응답 HTML 문서에 hidden 타입의 input 태그에 심어둔다. 그리고 서버 측으로 요청이 전달될 때, 파라미터 값으로 Token 값을 받아 서버 측에서 검증하고, 검증에 통과하면 해당 기능을 실행하도록 한다. 이 Token은 서버 측에서 생성되는 임의의 값으로 공격자가 알아내기 어렵기 때문에 공격자가 작성한 CSRF 스크립트에도 서버 측으로 보내는 요청에 Token 값을 포함한 요청을 보내도록 하는 스크립트를 작성할 수 없다. 그렇기 때문에 CSRF Token을 적용하면 공격자의 스크립트에 노출되더라도 기능이 실행되지 않고 검증에 의해 차단된다.

## 3.3. SameSite 쿠키 옵션 적용

SameSite는 크로스 사이트(Cross-Site) 요청 시 쿠키의 전송 여부를 제어하여 CSRF(Cross-Site Request Forgery) 공격을 방어하기 위해 사용하는 HTTP 응답 헤더(Set-Cookie)의 보안 속성이다. SameSite의 속성에 따라 3가지의 동작 모드로 구분된다.

SameSite 옵션이 CSRF를 방어하는 원리는 웹페이지A와 웹페이지B가 있다고 가정했을 때, 다음과 같다.

1. 웹페이지A에서 로그인을 하면 사용자의 브라우저에 쿠키가 저장된다. 이 때, Set-Cookie 헤더에서 쿠키를 SameSite=Strict 옵션을 준다.

2. 이 상태로 웹페이지B에 방문한다. 웹페이지B에는 공격자가 심은 웹페이지A의 비밀번호 변경 기능으로 요청을 보내도록 하는 스크립트가 심어져 있다.

3. 사용자가 웹페이지B에 심어진 스크립트에 노출되고, 웹페이지A로 비밀번호 변경 요청을 보내게 된다. 이 때, SameSite=Strict 로 지정되어 있어 동일 사이트/동일 출처가 아니라면 이 옵션이 지정된 쿠키는 포함하지 않은 상태로 요청이 전달되므로 비밀번호 변경 요청은 불가하게 된다.

### SameSite=Strict

- 동작 방식 : 동일 출처/동일 사이트(First-Party) 요청에만 쿠키를 전송한다.

- 주요 특징 및 활용 : 외부 링크 클릭, 외부 사이트의 폼 전송 등 어떠한 크로스 사이트 요청에도 쿠키가 제외된다. 보안성이 가장 높지만, 외부 링크를 타고 들어올 때 로그인이 풀려 보이는 UX 저하가 발생할 수 있다.

### SameSite=Lax (기본값)

- 동작 방식 : 기본적으로 크로스 사이트 쿠키 전송을 차단하지만, 안전한 탐색(Top-level Navigation)에는 허용한다.

- 주요 특징 및 활용 : 외부 사이트에서 단순 링크(<a> 태그), GET 폼 요청 등을 통해 접속할 때는 쿠키가 전송된다. 반면, POST, PUT, iframe, AJAX(fetch/XHR) 요청 시에는 쿠키가 제외된다.


### SameSite=None

- 동작 방식 : 모든 크로스 사이트 요청에 대해 쿠키를 전송한다.

- 주요 특징 및 활용 : 타사 위젯, 임베디드 서비스, SSO 연동 등에 필요하다. 반드시 Secure 속성이 함께 지정되어야 하며(HTTPS 필수), 그렇지 않으면 브라우저에 쿠키 설정을 거부한다.



# 4. 참고

- https://junhyunny.github.io/information/security/spring-boot/spring-security/cross-site-reqeust-forgery/

- https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Referer
