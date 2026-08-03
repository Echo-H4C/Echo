# 1. 취약점 정의

임의 이용자의 권한으로 임의 주소에 HTTP 요청을 보낼 수 있는 취약점이다. 공격자는 이용자의 계정으로 임의 금액을 송금해 금전적인 이득을 취하거나 비밀번호를 변경해 계정을 탈취하고, 관리자 계정을 공격해 공지사항 작성 등으로 혼란을 야기할 수 있다.

# 2. 취약점 점검 방법

아래의 코드에는 CSRF 취약점이 존재한다. /sendmoney 기능이 실행될 때, 이용자로부터 예금주와 금액을 입력받고 송금을 수행한다. 이때 계좌 비밀번호, OTP 등을 사용하지 않기 때문에 로그인한 이용자는 추가 인증 없이 해당 기능을 이용할 수 있다.
```
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

# 3. 대응 방안
