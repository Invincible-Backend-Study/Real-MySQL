<h1>ch.02, ch.03 설치와 설정, 사용자 및 권한</h1>

## 설치

msi인스톨러 책에서 알려준 사이트를 통해 진행

https://dev.mysql.com/downloads/mysql/

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/b0844e64-026b-45a5-9cf9-5b49d00b364b)


해당 버전으로 진행

msi파일로 설치 진행 시에는 c/Program Files/MySQL 폴더에 저장된다.

<br><br>

### 서버 설정 my.cnf(my.ini)

---

일반적으로 MySQL의 서버는 하나의 설정파일만 사용한다.

리눅스를 포함한 유닉스 계열에서는 my.cnf 파일명을 사용하고,

윈도우 계열에서는 my.ini라는 이름을 사용한다.

**`my.ini`**의 위치는 보통의 경우 MySQL의 basedir 하위에 들어가 있다.

하지만 msi파일로 설치를 했을 경우는 다른 위치에 설치되는데

msi상의 경로로는

`C:\ProgramData\MySQL\MySQL Server 8.0`

해당 위치고 만약 여기도 없다면 아래의 명령어를 통해서 지정되어있는 위치를 모두 찾아봐야 한다.

`show variables like '%dir';`

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/201b889e-27f9-422d-b8e6-e677ce2d280b)


또는

services.msc에 있는 실행설정에 있는 경로를 확인해서도 확인이 가능하다

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/9f6ac506-4f12-44e6-97ba-6cd8c6d1d457)


"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqld.exe" --defaults-file="C:\ProgramData\MySQL\MySQL Server 8.0\my.ini" MySQL80

<br><br>

## First Login

---

**MySQL shell**

명령어 : \connect —mysql root@localhost:3306

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/f32cffe5-9cc8-481a-b159-41a66b192d15)


\sql을 통해 sql명령어로 변경이 가능

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/c420237f-f614-4d8c-b1d0-c9f13aa4f455)


MySQL CommandLine Client

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/dac8dbaf-f52b-4e58-b31f-325c1979c0b3)


root계정으로 즉시접속됨

<br><br>

### MySQL 시스템 변수의 특징

---

시스템 변수는 상당히 많아서 다 나열할 수는 없지만 변경이 필요할 때 어느 부분에서 속성 변경을 해야하는지 정확히 알아두고 진행해야 할 필요가 있기에 기억을 해둬야한다.

해당 refence DOC : https://dev.mysql.com/doc/refman/8.0/en/server-system-variable-reference.html

또, 해당 표에 적혀있는 5가지 속성의 의미는 다음과 같다.

- Cmd-Line : MySQL 서버의 명령행 인자로 설정될 수 있는지의 여부
- Option file : my.cnf(my.ini)로 제어할 수 있는지의 여부
- System Var : 시스템 변수인지 아닌지(언더바 _ 의 여부)
- Var Scope : 시스템 변수의 적용범위
- Dynamic : 동적인지 정적인지 구분

<br><br>

### 글로벌 변수와 세션 변수

---

MySQL의 시스템 변수의 종류

### Global

하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수를 의미 주로 MySQL 서버 자체에 대한 설정의 경우가 많다.

대표적인 글로벌 변수로는 innodb_buffer_pool_size(MySQL db엔진 버퍼크기)

### Session

각 클라이언트가 MySQL서버에 접속할 때 기본으로 부여하는 옵션의 기본값을 제어하는 데 사용된다.

세션 변수를 이렇게 지정해서 사용하는 이유는 클라이언트별로 원하는 설정방식이 있을 때 정하는데, 대표적인 예시로 autocommit 변수가 있다.

이 변수를 ON으로 설정해두면 해당 서버에 접속하는 모든 커넥션은 기본으로 autocommit으로 되어있지만, 개별 클라이언트 커넥션의 설정에 따라 해당 변수를 OFF설정해 자동 커밋을 비활성화 할 수 있다. 또 이렇게 전역에도 되어있고, 세션에도 개별설정이 가능한 경우

VarScope의 범위가 Both인 것이다.

<br><br>

### 정적 변수와 동적 변수

---


MySQL 서버의 시스템 변수는 MySQL 서버가 기동중인 상태에서 변경 가능한지에 따라 동적 변수와 정적 변수로 구분된다.

디스크에 저장돼 있는 설정파일(my.cnf or my.ini)을 변경하는 경우(정적변수)

이미 기동 중인 MySQL 서버의 메모리에 있는 MySQL서버의 시스템 변수를 변경하는 경우로 구분할 수 있다.

이 때 set을 통해서 변수명을 변경할 수도 있고, 변수명을 정확히 모른다면 SQL문장형식으로 패턴검색을 할 수도 있다.

검색

`SHOW GLOBAL VARIABLES LIKE ‘%max_connections%’;`

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/30469ca2-032a-4e1f-8aa9-4030c1a717a6)


변경

`SET GLOBAL max_connections=152;`

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/e305ace9-2dbc-490d-92b1-b9762cbd0031)


**MySQL 8.0 이전**

SET을 통해 시스템 변수값을 변경해도 해당 mysql인스턴스에만 적용되기 때문에 영구적용을 하려면 my.ini파일의 직접변경이 필요하다

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/75903442/fa75599e-c2ec-4563-a82d-dc5800751c5d)


**MySQL 8.0 이상**

SET PERSIST명령을 이용하면 인스턴스와 my.시스템파일도 같이 변경된다.

시스템 변수의 범위가 Both라면 글로벌 시스템 변수의 값을 변경해도 값이 변하지 않는다

<br><br>

### SET PERSIST

---

MySQL 동적변수의 경우 SET GLOBAL로 시스템변수를 즉시 적용할 수 있다.

하지만 이는 MySQL이 실행되어있는 이번 인스턴스에서만 적용이 되기 때문에 SQL을 종료했다가 재실행 하게되면 SET으로 변경된 값이 원래대로 돌아온다.

이 이유는 my.ini의 초기설정이 안되어있기 때문이고, 이 값은 SET으로 변경되지 않았기 때문이다.

이를 해결하기 위해 MySQL 8.0부터 SET PERSIST명령으로 시스템 변수를 변경했을 때 시스템설정파일도 변경한 것과 동일한 효과를 낼 수 있게 되었는데, 이 이유는 my.cnf파일이 바뀌지 않고, mysqld-auto.cnf에 변경된 내용을 추가기록하고 MySQL이 실행될 때 my.cnf 포함 mysqld-auto.cnf파일도 읽어들이기 때문이다.

<br><br>

## 3. 사용자 및 권한

---

### 사용자 식별
---

MySQL은 접속 지점과 아이디가 같이 언급이된다.

ex) svd_id@127.0.0.1

만약 해당 컴퓨터 뿐만이 아니라 다른 컴퓨터에서도 접속이 가능하게 생성하려면

호스트부분을 우측과 같이 해줘야한다. `xxx@%`

또 만약, 하나의 dbms에 xxx@%와 xxx@127.0.0.1이 같이 있다면 좁은 범위인 127.0.0.1이 먼저 선택되어 접속이 된다.

<br><br>

### 시스템 계정과 일반 계정
---

일반계정을 관리, 데이터베이스 서버 관리 등과 같은 중요작업(DBA가 관리하는 것)을 하는 **시스템계정**과 일반 응용프로그램을 확인하거나 개발자들이 활용하는 **일반 계정**으로 나뉘어져 있다.

구체적인 시스템계정의 역할

- 계정 관리(계정 생성 및 삭제, 계정의 권한 부여 및 제거)
- 다른 세션(Connection)또는 그 세션에서 실행 중인 쿼리를 강제 종료
- **스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정**

<br><br>

### 계정 생성
---

MySQL 8.0버전부터는 계정생성을 GRANT가 아닌, CREATE USER명령을 통해, 권한 부여는 GRANT 명령을 통해 실행되도록 구분했다.

첫 계정 생성시에 설정되는 옵션으로는

- 계정의 인증 방식과 비밀번호
- 비밀번호 관련 옵션(유효 기간, 이력 개수, 비밀번호 재사용 불가기간)
- 기본 역할
- SSL 옵션
- 계정 잠금 여부

등이 있으며, 작성예시는 다음과 같다

```sql
CREATE USER 'user'@'%'
IDENTIFIED WITH 'mysql_native_password' BY ' 'password'
REQUIRE NONE
PASSWORD EXPIRE INTERVAL 30 DAY
ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT;
```

위 옵션을 하나하나 보자면

### IDENTIFIED WITH

사용자 인증 방식과 비밀번호를 설정 방식에는 4가지가 대표적

- Native Pluggable Authentication : 비밀번호 해시 SHA-1알고리즘을 통한 값 저장
- Caching SHA-2 Pluggable Authentication : SHA-2(256비트)알고리즘을 통한 값 저장
- PAM Pluggable Authentication : 유닉스나 리눅스 패스워드 또는 LDAP같은 외부 인증을 사용할 수 있게 해주는 인증 방식으로, MySQL 엔터프라이즈 에디션에서만 사용 가능
- LDAP Pluggable Authentication : LDAP를 이용한 외부인증방식 이또한 MySQL 엔터프라이즈에서만 사용 가능

기본 인증방식은 Caching SHA-2 Authentication이며, 이는 SSL/TLS 또는 RSA 키페어를 필요로 하기 때문에 혹시나 Native Pluggable 인증방식으로 바꿀 때는 추가 명령어가 필요하다.

선택적으로 바꾸는 이유 : 암호화 및 보안에는 SHA-2가 더 좋지만 , 비밀번호 인증 과정에서 해시함수 확인을 위해 CPU의 자원을 많이 소모하기 때문

### REQUIRE

MySQL 서버에 접속할 때 SSL/TLS의 사용 여부

### PASSWORD EXPIRE

비밀번호의 유효기간 설정 PASSWORD EXPIRE 절에 설정 가능한 옵션은 다음과 같다.

- EXPIRE : 계정 생성과 동시에 비밀번호의 만료 처리
- NEVER : 비밀번호 만료기간 없음
- DEFAULT : default_password_lifetime : 시스템 변수에 저장된 기간
- INTERVAL n DAY : 비밀번호의 유효기간을 오늘부터 n일자로 설정

### PASSWORD HISTORY

한번 사용했던 비밀번호를 재사용 못하게 하는 옵션

- PASSWORD HISTORY DEFAULT : 시스템변수에 저장된 개수만큼
- n : 최근 n개까지만 비밀번호의 이력을 저장해 이력에 남아있는 비밀번호만 불가능

### PASSWORD REUSE INTERVAL

한번 사용했던 비밀번호의 재사용 금지**기간을 설정하는 옵션**

### PASSWORD REQUIRE

비밀번호가 만료되었을 때 현재 비밀번호를 필요로 할지 말지를 결정

### ACCOUNT LOCK / UNLOCK

계정 생성 시, ALTER USER 명령을 사용해 계정 정보를 변경할 때 계정을 잠글지 사용가능하게할지 여부 설정

<br><br>

### 비밀번호 관리
---

암호설정을 고수준으로 하도록 변경할 수도 있다.

고수준 암호 옵션은 세가지로 나뉜다.

- LOW : 비밀번호의 길이만 검증
- MEDIUM : 비밀번호의 길이를 검증, 숫자 대소문자, 특수문자 배합 검증
- STRONG : MEDIUM의 검증 포함하여 금칙어 검증

등이 있다.

<br><br>

### 권한
---

데이터베이스나 테이블과 객체에 적용되는 권한을 글로벌 권한이라고 하며, 데이터베이스나 테이블을 제어하는 데 필요한 권한을 객체 권한이라고 한다.

객체 권한은 GRANT 명령으로 부여할 때 특정 객체를 명시해야 하며, 글로벌 권한은 명시하지 말아야 한다.

예외적인 권한 부여 방법으로는 ALL 또는 ALL PRIVILEGES가 있는데, 이는 글로벌과 객체 두가지 용도 모두 사용될 수 있는데, 객체에 ALL이 부며되면 해당 객체에 적용될 수 있는 모든 권한을 부여받는 것이며, 글로벌로 ALL이 사용되면 글로벌 수준에서 가능한 모든 권한을 부여받는다.

<br><br>

### 역할
---

MySQL 8.0 버전부터는 권한을 묶어서 역할(ROLE)을 사용할 수 있게 되었다.

```sql
CREATE ROLE
role_emp_read,
role_emp_write;
```

처럼 역할을 정의할 수 있고 이러한 역할에 GRANT를 통해 실제 권한들을 부여해 관리할 수 있다.

```sql
GRANT SELECT ON employees.* TO role_emp_read;
GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
```

role_emp_read 객체에는 employees DB의 모든 객체에 SELECT만 가능

관리하는 방법에 따라서 편하게 역할과 권한으로 지정해서 사용하면 될 것 같다.
