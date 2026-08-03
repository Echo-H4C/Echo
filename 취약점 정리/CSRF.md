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

# 4. 참고

- https://junhyunny.github.io/information/security/spring-boot/spring-security/cross-site-reqeust-forgery/

- 
