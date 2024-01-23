![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/28b572e1-a977-4c01-9df6-dcc64deed6ee)# 1주차 2, 3장

# 📌 02장: 설치와 설정

***mysqld***는 백그라운드에서 돌아가고 있는 프로세스, MYSQL 서버이고

***mysql***은 우분투의 터미널처럼 *sql*문을 실행시켜주는 command-line client입니다.

brew로 설치한 MySQL의 `my.cnf` 파일 위치 : /opt/homebrew/etc/my.cnf

- 최소 패치 버전이 15 ~ 20번 이상 릴리스 된 버전을 선택하는것이 좋다.
- 삭제하면 안되는 디렉터리(mac os)
    - bin: MySQL 서버와 클라이언트 프로그램, 유틸리티를 위한 디렉터리
    - data: 로그 파일, 데이터 파일들이 저장되는 디렉터리
    - include: C/C++ 헤더 파일들이 저장된 디렉터리
    - llib: 라이브러리 파일들이 저장도니 디렉터리
    - share: 다양한 지원 파일들이 저장돼 있음, 에러 메세지, 샘플 설정 파일(my.cnf)가 있는 디렉터리

<br><br>

## ✅ 2.2 MySQL 서버의 시작과 종료

리눅스 환경에서 MySQL 서버를 실행하는데 필요한 초기 데이터 파일(시스템 테이블 저장되는 데이터 파일), 트랜잭션 로그(리두 로그) 파일을 생성할 수 있음

```bash
# 필요한 초기 데이터 파일과 로그 파일을 생성하고 비밀번호가 없는 관리자 계정인 root 유저 생성
mysqld --defaults-file/etc/my.cnf --initialize-insecure

# 필요한 초기 데이터 파일과 로그 파일을 생성하고 비밀번호가 있는 관리자 계정을 생성함 -> 비밀번호를 에러 로그에 기록됨
mysqld --defaults-file/etx/my.conf -initialize
```

**클린 셧다운**

MySQL 서버는 실제 트랜잭션이 정상 커밋되어도 데이터 파일에 변경된  내용이 기록되지 않고 로그 파일(리두 로그)에만 기록돼 있을 수 있습니다.

MySQL서버가 종료되고 다시 시작해도 로그 파일에만 기록되어있을 수 있는데 서버가 종료될때 모든 커밋 내용을 데이터 파일에 기록하고 종료할 수 있습니다.

```sql
SET GLOBAL innodb_fast_shutdown=0;

bash-with-homebrew> mysql.server stop
```

<br>

### 2.2.3 서버 연결 테스트

**mysql -uroot -p --host=localhost --port=3306 혹은 mysql -uroot -p는 Unix domain Socket 이용**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cd58655e-4925-4418-ac61-5eac561be4bc)

실제 loopback에 3306포트의 통신을 확인했을때 Unix domain Socket으로 mysql에 접속하면 어떤 네트워크 패킷도 잡히지 않음

lsof 명령어를 -U 옵션을 주고 실행하면 시스템의 전체 유닉스 도메인 소켓 목록을 확인할 수 있는데 mysqld를 확인할 수 있다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5281ec1c-cc43-458c-a891-f1484fd1f27b)

**mysql -uroot -p --host=127.0.0.1 --port=3306 → loopback 주소 이용 (TCP/IP)**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/95f9e66d-c949-456b-bbda-78a9ca7f7152)

loopback 주소를 사용하면 일반 TCP/IP 통신 방식을 이용하므로 네트워크 패킷이 잡히게 됩니다.

**접속하지 않고 원격 서버에서 서버의 접속 가능 여부 파악하기**

Telnet 혹은 Netcat을 활용할 수 있음

과거 MySQL을 원격접속하기 위해 했던 시도들이 일치하네..?

[우여곡절 MySql 원격접속 하기ㅠ.ㅠ](https://velog.io/@wpdlzhf159/MySql-원격접속-하기)

<br><br>

## ✅ 2.3 MySQL 서버 업그레이드

1. `In Place Upgrade`: MySQL 서버의 데이터 파일을 그대로 두고 업그레이드하는 방법
2. `Logical Upgrade`: mysqldump 도구 등을 이용해 덤프 및 새로운 버전에서 덤프된 데이터 적재 하는 방법

<br>

### 2.3.1 인플레이스 업그레이드 제약 사항

**동일한 메이저 버전, 다른 마이너 버전일때**

대부분 데이터 파일의 변경 없이 진행됨, 단순히 MySQL 서버 프로그램만 재설치하면 된다.

**메이저 버전간 업그레이드**

메이저 버전간에는 크고 작은 데이터 파일의 변경이 필요하므로 직전 버전에서만 업그레이드가 허용됨
(애초에 업그레이드 하려는 서버 프로그램이 직전 메이저 버전에서 사용하던 데이터 파일과 로그 포맷만 인식하도록 구현함)

혹은 특정 마이너 버전에서만 가능한 경우도 있음 (General Availability) 버전

<br>

### 2.3.2 MySQL 8.0 업그레이드 시 고려 사항

- 사용자 인증 방식 변경: Caching SHA-2 Authentication 인증 방식이 기본 인증 방식으로 바뀜
- MySQL 8.0과의 호환성 체크 필요
- 외래키 이름의 길이 → 외래키 이름이 64글자로 제한됨
- 인덱스 힌트 → MySQL 5.x 버전에서 사용하던 인덱스 힌트가 있다면 성능 테스트를 진행해보길 권장(성능 저하를 유발할 수 있다.)
- GROUP BY에 사용된 정렬 옵션
    
    [MySQL :: WL#8693: Remove the syntax for GROUP BY ASC and DESC](https://dev.mysql.com/worklog/task/?id=8693)
    
- 파티션을 위한 공용 테이블스페이스 → 파티션의 각 테이블스페이스를 공용 테이블스페이스에 저장할 수 없음

<br>

### 2.3.3 MySQL 8.0 업그레이드

MySQL 8.0 버전 부터는 시스템 테이블의 정보와 데이터 딕셔너리 정보의 포맷이 완전히 바뀜

**5.7 → 8.0으로 업그레이드가 처리되는 두 가지 단계**

1. 데이터 딕셔너리 업그레이드
    
    5.7까지는 데이터 딕셔너리 정보가 FRM 확장자를 가진 파일로 별도 보관 됐었지만 8.0부턴 트랜잭션이 지원되는 InnoDB 시스템 테이블로 저장합니다.
    
2. 서버 업그레이드
    
    MySQL 서버의 시스템 데이터 베이스의 테이블 구조를 MySQL 8.0 버전에 맞게 변경함
    

<br><br>

## ✅ 2.4 서버 설정

- MySQL 서버는 my.cnf라는 단 하나의 설정 파일을 사용함 또한 서버가 시작될대만 해당 설정 파일을 참조한다.
- my.cnf의 경로는 고정되어 있지않고 순차 탐색하여 처음 발견된 my.cnf 파일을 사용함

my.cnf 탐색 디렉터리 확인은 다음과 같이 할 수 있다.

```bash
# MySQL 서버의 실행 프로그램으로 서비스용으로 사용되는 서버에서 이미 실행중인데 또 호출하지 않도록 조심
mysqld --verbose --help

# 커맨드 라인 클라이언트 사용
mysql --help
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a1584b17-4a81-4f4f-947f-a8c22288360a)

1, 2, 4번은 어떤 MySQL이나 동일하게 검색하는 경로

3번은 컴파일 될 때 MySQL 프로그램에 내장된 경로 입니다. → 현재는 homebrew로 깔았기에 해당 경로가 됨

MySQL 서버용 설정 파일은 주로 1번 혹은 2번을 사용함

(만약 한개의 서버에서 두 개의 인스턴스를 실행한다면 별도 디렉터리에 설정 파일을 준비하고 MySQL 시작 스크립트 내용을 변경하는 방법이 좋다.)

<br>

### 2.4.1 설정 파일의 구성

MySQL 설정 파일은 하나의 my.cnf파일에 여러 개의 설정 그룹을 담을 수 있습니다.

주로 실행 프로그램 이름을 그룹명으로 사용합니다.

<br>

### 2.4.2 MySQL 시스템 변수의 특징

시스템 변수는 `SHOW VARIABLES` or `SHOW GLOBAL VARIABLES` 라는 명령으로 확인할 수 있습니다.

시스템 변수 값이 MySQL 서버 혹은 클라이언트에 어떤 영향을 미치는지 알려면 각 변수가 `글로벌 변수`인지 `세션 변수`인지 구분할 수 있어야 합니다.-

[MySQL :: MySQL 8.0 Reference Manual :: 5.1.5 Server System Variable Reference](https://dev.mysql.com/doc/refman/8.0/en/server-system-variable-reference.html)

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/8d288ee2-d5a7-452e-9f7f-94fb33dfd89e)

- Cmd-Line: MySQL 서버의 명령행 인자로 설정될 수 있는지 여부
- Option File: MySQL의 설정파일인 my.cnf로 제어할 수 있는지 여부
- System Var: 시스템 변수인지 아닌지를 나타냅니다.
- Var Scope: 시스템 변수의 적용 범위 (글로벌 혹은 세션 혹은 전부)
- Dynamic: 시스템 변수가 동적인지 정적인지 구분하는 변수

<br>

### 2.4.3 글로벌 변수와 세션 변수

변수 **종류에 따른 역할**

- `글로벌 변수`: 하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수 (주로 MySQL 서버 자체 관련 설정)
- `세션 변수`: MySQL 클라이언트가 MySQL 서버에 접속할 때 기본으로 부여하는 옵션의 기본값을 제어하는데 사용됨, 클라이언트 필요에 따라 개별  커넥션 단위로 다른 값으로 변경할 수 있다.

**Var Scope에 따른 차이**

- `Both 변수`: my.cnf와 같은 MySQL 서버의 설정 파일에 명시해 초기화할 수 있는 변수들의 대부분 범위이며 MySQL 서버가 기억만 하고 있다가 실제 클라이언트와의 커넥션이 생성되는 순간 해당 커넥션의 기본값으로 사용되는 값입니다.
- `Session 변수`: 순수하게 범위가 세션으로 MySQL 서버의 설정 파일에 초깃값을 명시할 수 없으며 커넥션이 만들어지는 순간부터 해당 커넥션에서만 유효한 설정 변수를 의미 합니다.

<br>

### 2.4.4 시스템 변수: 정적 변수와 동적 변수

MySQL 서버의 `시스템 변수`는 디스크에 저장돼 있는 설정 파일(my.cnf or my.ini)을 변경하는 경우와 이미 기동 중인 MySQL 서버의 메모리에 있는 MySQL 서버의 시스템 변수를 변경하는 경우로 구분됨

**정적 변수는 서버가 기동중인 상태에서 변경할 수 없고, 동적 변수는 가능하다.**

**SET 명령어(동적 변수만 가능): 현재 기동 중인 MySQL 인스턴스에서 유효한 명령**

- **동적 변수만 가능함**
- 시스템 변수의 범위: Global, Session 모두 가능
    - global 키워드를 붙이면 글로벌 시스템 변수의 목록 내용 읽고 변경, 빼면 자동으로 세션 변수를 조회 및 변경함
    - global 변수를 변경했다면 mysql 서버가 켜져있을때까지 유효하며, **서버를 껐다 키면 다시 원래 변수로 초기화** 됩니다.
- 시스템 변수의 범위가 Both 일때
    - global 키워드를 붙여서 글로벌 시스템 변수의 값을 변경해도 이미 존재하는 커넥션의 세션 변숫값은 변경되지 않고 그대로 유지됨

**SET PERSIST 명령어(동적 변수, 글로벌 변수만 가능): 실행중인 MySQL 인스턴스 및 설정 파일로도 기록함**

- **동적 변수만 가능함**
- 시스템 변수의 범위: Global만 가능
    - MySQL 서버의 글로벌 변수에 즉시 반영하는 동시에 별도의 설정 파일에도 기록하여 **MySQL 서버를 껐다 켜도 반영되도록 합니다.**(mysql.server stop → start)
    - 별도의 설정파일은 `mysqld-auto.cnf`라는 파일에 기록하게 됩니다.
    - 변경 내용을 수동으로 기록하지 않아도 자동으로 영구 변경이 됩니다.

**SET PERSIST_ONLY 명령어(정적 및 동적 변수, 글로벌 변수만 가능): 설정 파일에만 기록함**

- 글로벌 변수만 가능하며 정적, 동적 변수는 상관이 없습니다.
- 정적 변수는 실행중인 MySQL 서버에서 변경할 수 없으므로 SET PERSIST_ONLY를 활용합니다.
- 이 명령어를 사용하면 mysql.server stop → start시 적용이 됩니다.(서버를 껐다가 켰을때)

**시스템 변수 삭제하기**

- set persist 또는 set persist_only 명령으로 변경된 시스템 변수의 메타데이터는 performance_schema.variables_info 뷰와 performance_schema.persisted_variables 테이블을 통해 참조할 수 있습니다.
    
    ```sql
    select a.variable_name, b.variable_value, a.set_time, a.set_user, a.set_host 
    from performance_schema.variables_info a 
    inner join performance_schema.persisted_variables b 
    on a.variable_name=b.variable_name 
    where b.variable_name like '이름';
    ```
    

- 시스템 변수 삭제
    
    ```sql
    ## 특정 시스템 변수만 삭제
    RESET PERSIST 시스템변수명;
    RESET PERSIST IF EXISTS 시스템변수명;
    
    ## mysqld-auto.cnf 파일의 모든 시스템 변수를 삭제
    RESET PERSIST;
    ```
    
<br><br><br>

# 📌 3장 사용자 및 권한

- MySQL의 사용자 계정은 단순히 사용자의 아이디뿐 아니라 해당 사용자가 어느 IP에서 접속하고 있는지도 확인함
- MySQL 8.0부터는 역할(Role)이 도입 됐는데 이는 권한들의 집합이다. 따라서 사용자에게 미리 준비한 권한 세트를 부여할 수 있다.

<br><br>

## ✅ 3.1 사용자 식별

**MySQL의 계정의 의미**

- MySQL의 계정 = 사용자 계정 + 사용자의 접속 지점(호스트 명이나 도메인 또는 IP 주소)
- MySQL에서는 계정을 언급할때 항상 아이디 + 호스트를 명시해야 합니다.
    
    ```sql
    'hoseok'@'192.168.0.10'
    'hoseok'@'%'
    ```
    
    위와 같이 동일한 계정이 있을경우 호스트의 범위가 더 구체적인것(좁은것)을 이용해서 사용자를 인증하게 됩니다. 여기서는 192.168.0.10의 IP를 가진 hoseok 계정이 선택됩니다.
    
<br><br>

## ✅ 3.2 사용자 계정 관리

<br>

### 3.2.1 시스템 계정과 일반 계정

**MySQL 8.0부터 `SYSTEM_USER` 권한 여부에 따라 `시스템 계정` 및 `일반 계정`을 구분하게 됩니다.**

- `시스템 계정`: DB 서버 관리자를 위한 계정
    - 시스템 계정은 시스템 계정, 일반 계정을 관리(생성, 삭제, 변경, 계정의 권한 부여 및 제거) 할 수 있습니다.
    - 추가적으로 다른 세션 또는 해당 세션에서 실행 중인 쿼리를 강제 종료 하거나 스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정할 수 있습니다.
- `일반 계정`: 응용 프로그램이나 개발자를 위한 계정
    - 일반 계정은 시스템 계정을 관리할 수 없습니다.

**MySQL 서버 내장 계정**

- `‘mysql.sys’@’localhost’`: MySQL 8.0부터 기본으로 내장된 sys 스키마의 객체(뷰, 함수, 프로시저)들의 DEFINER로 사용되는 계정
- `‘mysql.session’@’localhost’`: MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- `‘mysql.infoschema’@’localhost’`: information_schema에 정의된 뷰의 DEFINER로 사용되는 계정

‘root’@’localhost’를 제외한 위 3개의 게정은 처음부터 잠겨있는(account_locked 칼럼) 상태이므로 보안 및 삭제에 대한 걱정을 하지 않아도 된다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/8bc55e97-d73b-4c44-b05d-de6853640f1e)

<br>

### 3.2.2 계정 생성

MySQL 8.0 버전부터는 계정 생성은 CREATE USER 명령으로, 권한 부여는 GRANT 명령으로 구분해서 실행하도록 합니다.

계정 생성시 다음과 같은 다양한 옵션이 존재합니다.

- 계정의 인증 방식과 비밀번호
- 비밀번호 관련 옵션(유효 기간, 비밀번호 이력 개수, 비밀번호 재사용 불가 기간)
- 기본 역할(Role)
- SSL 옵션
- 계정 잠금 여부

**계정 생성 예시**

```sql
CREATE USER 'user'@'%'
		IDENTIFIED WITH 'mysql_native_password' BY 'password'
		REQUIRE NONE
		PASSWORD EXPIRE INTERVAL 30 DAY
		ACCOUNT UNLOCK
		PASSWORD HISTORY DEFAULT
		PASSWORD REUSE INTERVAL DEFAULT
		PASSWROD REQUIRE CURRENT DEFAULT;
```

- `IDENTIFIED WITH`
    
    사용자의 인증 방식과 비밀번호를 설정함, 기본 인증 방식을 사용하려면 IDENTIFIED BY ‘password’와 같은 형식 사용 
    (Native Pluggable Authenticaion(5.7 기본 인증), Caching SHA-2 Pluggable Authentication (8.0에서부터의 기본 인증))
    
- `REQUIRE`
    
    MySQL 서버에 접속할 때 압호화된 SSL/TLS 채널을 사용할 여부 결정
    
- `PASSWORD EXPIRE`
    
    비밀번호의 유효기간 설정 옵션
    
- `PASSWORD HISTORY`
    
    한 번 사용했던 비밀번호에 대한 재사용 여부 결정 옵션
    
- `PASSWORD REUSE INTERVAL`
    
    한 번 사용했던 비밀번호의 재사용 금지 기간 설정 옵션
    
- `PASSWORD REQUIRE`
    
    비밀번호 만료시 새로운 비밀번호 입력할때 만료된 비밀번호 필요 여부 결정 옵션
    
- `ACCOUND LOCK / UNLOCK`
    
    계정 생성 시 혹은 ALTER USER 명령을 사용해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부 결정
    
<br><br>

## ✅ 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

MySQL 서버의 비밀번호는 유효기간이나 이력 관리를 통한 재사용 금지 및 비밀번호를 쉽게 유추할 수 있는 단어들이 사용되지 않게 글자의 조합을 강제하거나 금칙어를 설정하는 기능도 제공합니다.

다만 이를 사용하기 위해선 validate_password 컴포넌트를 이용해야 합니다. (자세한 내용은 책 참조)

<br>

### 3.3.2 이중 비밀번호

DB는 여러 응용 프로그램에서 공용으로 사용하고 있으므로 DB 서버의 계정 정보를 변경하기란 쉬운일은 아닙니다.

이런 문제 해결을 위해 MySQL 8.0부턴 계정의 비밀번호로 2개의 값을 동시에 사용할 수 있는 Dual Password 기능을 제공합니다.

- 2개의 비밀번호는 Primary와 Secondary로 나뉩니다.
- 최근에 설정된 비밀번호가 Primary 비밀번호, 이전 비밀번호는 Secondary가 됩니다.

```sql
// 비밀번호를 old_password로 설정 old_password가 primary 비밀번호
ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password'

// 비밀번호를 new_password로 변경하며 기존 비밀번호를 Secondary 비밀번호로 변경
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;
```

이중 비밀번호의 사용은 RETAIN CURRENT PASSWORD 옵션만 추가해주면 됩니다.

<br><br>

## ✅ 3.4 권한(Privilege)

**권한의 종류는 글로벌 권한과 객체 단위의 권한으로 나뉩니다. (정적 권한)**

- 글로벌 권한
    - 데이터베이스나 테이블 이외의 객체에 적용되는 권한
    - GRANT 명령에서 특정 객체를 명시하지 말아야 함
- 객체 권한
    - 데이터베이스나 테이블을 제어하는데 필요한 구너한
    - GRANT 명령에서 반드시 특정 객체를 명시해야 함(특정 객체는 전체 객체나, 일부 객체 등 여러개가 존재)

> 예외적으로 ALL(or ALL PRIVILEGES)는 글로벌과 객체 권한 두 가지 용도로 사용될 수 있으며 특정 객체에 부여하면 해당 객체에 적용될 수 있는 모든 객체 권한을 부여하고, 글로벌로 사용되면 글로벌 수준에서 가능한 모든 권한을 부여합니다.
> 

`권한 표는 책 65page 참조`

**MySQL 8.0에서 추가된 동적 권한(기존 권한들은 정적 권한으로 분류됨)**

- MySQL 서버가 시작되면서 동적으로 권한이 생성되기에 동적 권한이라고 부릅니다. 
(플러그인, 컴포넌트가 설치되면 그때 등록되는 권한들을 동적 권한이라 함)
- 5.7까지 사용되는 SUPER라는 권한은 DB관리를 위해 꼭 필요했습니다. 8.0부터는 이를 잘개 쪼개어서 표3.2(67page)의 동적 권한으로 분산 했습니다.

**권한 부여 명령**

GRNAT 명령어를 사용함

```sql
GRANT privilge_list ON db.table_name TO 'user'@'host';
```

- 위 명령어는 반드시 user사용자를 생성하고 GRANT를 통해 권한을 부여해야 함
- GRANT OPTION(다른 사용자에게 에게 권한을 부여할 수 있는지)권한은 마지막에 WITH GRANT OPTION을 명시해 부여
- privilege_list에는 구분자(,)를 써서 복수개의 권한 명시 가능
- TO 키워드 뒤에는 권한 부여 대상 사용자를 명시
- ON 키워드 뒤에는 어떤 DB의 어떤 오브젝트 (혹은 모든 범위(*.*))에 부여할지 결정

**권한 부여 명령 - 글로벌 권한**

```sql
GRANT SUPER ON *.* TO 'user'@'localhost';
```

- 글로벌 권한은 특정 DB, 테이블에 부여할 수 없으므로 ON 뒤에는 항상 `*.*` 을 사용해야 합니다.
- `*.*` DB의 모든 오브젝트(테이블, 스토어드 프로시저, 함수)를 포함해서 MySQL 서버 전체를 의미 함

**권한 부여 명령 - DB 권한**

```sql
GRANT EVENT ON *.* TO 'user'@'localhost';
GRANT EVENT ON employees.* TO 'user'@'localhost';
```

- 데이터벵스 권한은 특정 DB 및 서버의 모든 DB에 권한 부여가 가능함, 다만 특정 테이블까지 부여할 수 없음
- 따라서 ON절에 `*.*` 이나 `employees.*` 모두 사용 가능

**권한 부여 명령 - 테이블 권한**

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.department TO 'user'@'localhost';
```

- 서버의 모든 DB에 대해 권한 부여 가능
- 특정 DB의 오브젝트에 대해서만 권한을 부여하는 것도 가능
- 특정 DB의 특정 테이블에 대해서만 권한을 부여하는 것도 가능

**권한 부여 명령 - 특정 칼럼**

특정 칼럼에 대해서만 권한을 부여하는 경우는 GRANT 명령의 문법이 조금 바뀌어야 함

또한 칼럼에 부여할 수 있는 권한은 DELEETE 제외 `INSERT`, `UPDATE`, `SELECT` 3가지임

```sql
GRANT SELECT, INSERT, UPDATE(dept_name) ON employees.department TO 'user'@'localhost';
```

다만 칼럼 단위의 접근 권한은 GRANT 명령으로 해결하다보면 모든 테이블의 모든 칼럼을 체크하므로 성능에 영향이 있을 수 있습니다.

따라서 테이블 자체에서 권한을 허용하고자 하는 칼럼만으로 별도의 뷰(View)를 만들어 사용하는 방법도 있습니다.

**뷰 또한 하나의 테이블로 인식하므로 뷰의 칼럼에 대해 권한을 체크하지 않고 뷰 자체에 대한 권한만 체크하게 됩니다.**

`mysql DB의 권한 관련 테이블을 참조하면 표 형태로 각 계정이나 역할에 부여된 권한 혹은 역할을 확인할 수 있습니다. (책 70page 참조)`

<br><br>

## ✅ 3.5 역할(Role)

8.0부터 권한을 묶어서 역할(Role)을 사용할 수 있게 됐습니다. 또한 역할은 계정과 똑같은 모습을 하고 있습니다.

<br>

### 역할 만들기

1. 역할 껍데기 생성
    
    ```sql
    CREATE ROLE role_emp_read, role_emp_write;
    ```
    
2. 역할에 이름에 맞는 권한 부여
    
    ```sql
    # 읽기 역할에 읽기만 부여
    GRANT SELECT ON employees.* TO role_emp_read;
    
    # 쓰기 역할에 생성, 변경, 삭제 부여
    GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
    ```
    
3. 사용자 만들고 역할 부여
    
    ```sql
    # 사용자 생성
    CREATE USER reader@'127.0.0.1' IDENTIFIED BY '1234';
    CREATE USER writer@'127.0.0.1' IDENTIFIED BY '1234';
    
    # GRANT 명령 통한 역할 부여
    GRANT role_emp_read TO reader@'127.0.0.1';
    GRANT role_emp_write TO writer@'127.0.0.1';
    ```
    
4. 계정 접속 및 역할 확인
    
    ```sql
    linux> mysql -h127.0.0.1 -ureader -p
    
    # 현재 계정의 권한 확인
    SHOW GRANTS;
    
    # 결과
    +---------------------------------------------------+
    | Grants for reader@127.0.0.1                       |
    +---------------------------------------------------+
    | GRANT USAGE ON *.* TO `reader`@`127.0.0.1`        |
    | GRANT `role_emp_read`@`%` TO `reader`@`127.0.0.1` |
    +---------------------------------------------------+
    ```
    
5. 권한 에러 및 역할 활성화 명령어
    
    ```sql
    # SELECT 조회를 했지만 권한 에러 발생
    SELECT * FROM employees.employees LIMIT 10;
    
    # ERROR 1142 (42000): SELECT command denied to user 'reader'@'localhost' for table 'employees'
    
    # 역할이 없음
    SELECT current_role();
    +----------------+
    | current_role() |
    +----------------+
    | NONE           |
    +----------------+
    
    ```
    
    실제 역할 부여 이후 DB 데이터를 조회 및 변경 시도시 권한이 없다는 에러를 마주하게 됩니다.
    
    role_emp_read 역할을 사용하기 위해서는 `SET ROLE` 명령어를 통해 실행에 해당되는 역할을 활성화 해야 합니다.
    
    (다만 계정 로그아웃 이후 재 로그인을 하면 역할이 활성화되지 않은 상태로 초기화 됩니다.)
    
    ```sql
    SET ROLE 'role_emp_read';
    
    SELECT current_role();
    +---------------------+
    | current_role()      |
    +---------------------+
    | `role_emp_read`@`%` |
    +---------------------+
    ```
    
6. activate_all_roles_on_login 시스템 변수를 ON으로 두어 매번 SET ROLE 명령어 사용하지 않아도 활성화 시키기
    
    ```sql
    SET GLOBAL activate_all_roles_on_login=ON;
    ```
    
    위 설정을 통해 재로그인을 해도 역할이 활성화 된 상태가 됩니다.
    

<br>

### 역할의 비밀 - 사용자 계정과 같은 모습을 함

- **MySQL 서버의 역할은 사용자 계정과 거의 같은 모습을 합니다. 실제로 MySQL 서버 내부적으로 역할과 계정은 동일한 객체로 취급됩니다.**
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ec8866d2-ca2e-409a-918f-dfdbb71e6aeb)

    
- 위 결과를 보면 사용자와 역할의 구분이 우리가 지은 이름을 제외하고 모호하다는 것을 알 수 있습니다.
- **사실 하나의 계정에 다른 계정의 권한을 병합하기만 하면 되므로 MySQL 서버는 역할과 계정을 구분할 필요가 없게 됩니다.**
- 역할을 만들때는 CREATE ROLE을 통해 만들고 생성시 호스트도 생략하는데 차이가 있는것이 아니냐고 생각될 수 있지만, 호스트 부분을 명시하지 않는다면 자동으로 모든 호스트(%)가 추가됩니다.
    
    ```sql
    CREATE ROLE role_emp_read, role_emp_write;
    
    CREATE ROLE role_emp_read@'%', role_emp_write@'%'; # 위와 동일한 결과
    ```
    
- 따라서 역할과 계정의 명확한 구분이 필요하다면 DB 관리자가 식별할 수 있는 프리픽스나 키워드를 추가하는 방식을 권장합니다.

**호스트를 가진 역할에 대한 고민**

```sql
CREATE ROLE role_emp_local_read@localhost;
CREATE USER reader@'127.0.0.1' IDENTIFIED BY '1234';

GRANT SELECT ON employees.* TO role_emp_local_read@'localhost';
GRANT role_emp_local_read@'localhost' TO reader@'127.0.0.1';
```

위 예제는 localhost의 역할을 127.0.0.1 계정에 부여하는 예제입니다. 호스트가 달라 호환되지 않는 상태이지만 **`역할의 호스트 부분은 역할 부여와 아무런 영향이 없습니다.`**

만약 역할을 사용해 직접 로그인 하는 용도로 사용된다면 그때 역할의 호스트 부분이 중요해집니다.

**그렇다면 왜 역할을 나눴을까? (CREATE ROLE, CREATE USER)**

이는 데이터베이스 관리의 직무를 분리할 수 있게 하여 보안을 강화하는 용도로 사용하기 위함입니다.

- CREATE USER 명령 권한은 없고, CREATE ROLE 명령만 실행 가능한 사용자는 역할을 생성할 수 있습니다.
- 해당 사용자가 생성한 역할은 계정과 동일한 객체를 생성하지만 account_locked 칼럼의 값이 ‘Y’로 설정되므로 로그인 용도로 사용할 수 없게 됩니다.

계정의 기본 역할 혹은 역할에 부여된 역할 그래프 관계는 SHOW GRANTS 명령뿐만 아니라 mysql DB의 역할 관련 테이블을 참조할 수 있습니다.

- mysql.default_roles: 계정별 기본 역할
- mysql.role_edges: 역할에 부여된 역할 관계 그래프
