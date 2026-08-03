# 1. 취약점 정의

임의 이용자의 권한으로 임의 주소에 HTTP 요청을 보낼 수 있는 취약점이다. 공격자는 이용자의 계정으로 임의 금액을 송금해 금전적인 이득을 취하거나 비밀번호를 변경해 계정을 탈취하고, 관리자 계정을 공격해 공지사항 작성 등으로 혼란을 야기할 수 있다.

# 2. 취약점 점검 방법

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

> **Tip:** XSS와 CSRF의 차이



# 3. 대응 방안
