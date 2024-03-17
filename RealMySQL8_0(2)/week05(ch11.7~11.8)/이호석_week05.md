# 📌 11장 쿼리 작성 및 최적화

## ✅ 11.7 스키마 조작(DDL)

- DDL(Data Definitino Language) DBMS 서버의 모든 오브젝트를 생성하거나 변경하는 쿼리
    - 스토어드 프로시저, 함수, DB 테이블등을 생성 변경하는 대부분의 명령이 포함됨
- DDL문은 많은 개선이 이뤄졌지만 스키마 변경 작업중 상당히 오랜 시간이 걸리거나 부하를 발생시키는 작업이 있으므로 주의해야 합니다.

<br><br>

### 11.7.1 온라인 DDL

- 5.5 이전까지 mysql 서버에서 테이블 구조 변경 동안 다른 커넥션서 DML 실행 불가
- 8.0부터 대부분의 스키마 변경 작업은 mysql 서버에 내장된 온라인 DDL 기능으로 처리 가능해짐

<br>

**11.7.1.1 온라인 DDL 알고리즘**

- 온라인 DDL은 스키마를 변경하는 작업 도중에도 다른 커넥션에서 해당 테이블의 데이터를 변경하거나 조회하는 작업을 가능케함
- 온라인 DDL은 `ALGORITHM`, `LOCK` 옵션을 통해 어떤 모드로 스키마 변경을 실행할지 결정할 수 있음
- 온라인 DDL은 테이블의 구조를 변경, 인덱스 추가와 같은 대부분의 작업에 대해 작동함
- old_alter_table을 통해 ALTER TABLE이 온라인 DDL인지 예전 방식으로 처리할지를 결정하는데 8.0은 **기본값이 OFF이므로 자동으로 온라인 DDL이 활성화** 됩니다.
- ALTER TABLE 명령을 mysql 서버는 다음과 같은 순서로 스키마 변경에 적합한 알고리즘을 찾습니다.
    - `ALGORITHM` : 아래의 순서대로 스키마 변경이 가능한지 확인 후, 가능하다면 선택합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/2100f176-5e9a-4252-a9b7-626819d77a85)

    
    - 알고리즘의 우선순위가 낮을수록 mysql 서버는 스키마 변경을 위해 더 큰 잠금 + 많은 작업을 필요로 하기에 서버의 부하가 많아집니다.

- 온라인 DDL은 알고리즘 + 잠금 수준도 명시할 수 있습니다. (명시하지 않으면 mysql서버가 알아서 적절하게 선택함)
    
    ```sql
    ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL, 
    ALGORITHM=INPLACE, LOCK=NONE;
    ```
    
- `INSTANT` 알고리즘의 경우 매우 짧은 시간의 메타데이터 잠금이 필요하므로 별도의 LOCK옵션을 명시할 수 없습니다.
- 나머지 알고리즘의 경우 3가지 LOCK중 하나를 명시할 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c6e7752f-af9c-4a11-9d27-39ad31b988b1)

    
    - 알고리즘이 INPLACE인 경우 대부분은 NONE설정이 가능, 가끔 SHARED 수준까지 설정 할 필요가 있음
    - EXCLUSIVE는 과거 mysql서버의 전통적인 ALTER TABLE과 동일하므로 굳이 LOCK을 명시할 필요가 없습니다.
- 온라인 스키마 변경 작업이 INPLACE 알고리즘을 사용하더라도 내부적으로 테이블의 리빌드가 필요한 경우 및 필요하지 않은 경우
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/172e391f-35a2-42d1-aa30-1e7f9cba7eb7)

    
    - `ex) PK를 추가하는 작업`: 데이터 파일에서 레코드의 저장 위치가 바뀌어야 하므로(단순 칼럼 이름 변경은 리빌드 작업 필요 X)
- 테이블 레코드의 리빌드가 필요한 경우를 `Data Reorganizing`(데이터 재구성), `Table Rebuild`라고 명명합니다.

→ 온라인 DDL의 기능은 버전별 많은 차이가 있으므로 사용중인 mysql 버전의 특성을 살펴보고 실제 스키마 변경 실행 전에 내용을 숙지하는것이 좋습니다.

---

<br>

**11.7.1.2 온라인 처리 가능한 스키마 변경**

- mysql 서버의 모든 스키마 변경작업이 온라인으로 가능한 것이 아니므로 필요한 스키마 변경 작업에 대해 온라인 처리 가능 여부 및 테이블의 읽고 쓰기가 대기 하는지에 대한 정보는 확인하는 것이 좋습니다.
- 책에서는 8.0.21기준으로 메뉴얼을 정리합니다.
- https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html
- ALTER TABLE 문장에 LOCK, ALGORITHM절을 명시해 처리 알고리즘 및 락을 강제할 수 있지만, 반드시 해당 알고리즘으로 처리되지 않습니다. (불가능하다면 에러를 뱉음)
    
    ```sql
    # 1. ALGORITHM=INSTANT 옵션으로 스키마 변경을 시도 (성공)
    ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=INSTANT;
    ERROR 1846 (0A000): ALGORITHM=INSTANT is not supported. Reason: Dropping a primary key is not allowed without also adding a new primary key. Try ALGORITHM=COPY/INPLACE.
    
    # 2. 실패하면 ALGORITHM=INPLACE, LOCK=NONE 옵션으로 스키마 변경을 시도 (성공)
    ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=NONE;
    ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Dropping a primary key is not allowed without also adding a new primary key. Try ALGORITHM=COPY.
    
    # 3. 실패하면 ALGORITHM=INPLACE, LOCK=SHARED 옵션으로 스키마 변경을 시도 (성공)
    mysql> ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=SHARED;
    ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Dropping a primary key is not allowed without also adding a new primary key. Try ALGORITHM=COPY.
    
    # 4. 실패하면 ALGORITHM=COPY, LOCK=SHARED 옵션으로 스키마 변경을 시도 (성공)
    ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
    Query OK, 300024 rows affected (6.24 sec)
    Records: 300024  Duplicates: 0  Warnings: 0
    
    # 5. 실패하면 ALGORITHM=COPY, LOCK=EXCLUSIVE 옵션으로 스키마 변경을 시도 (성공)
    ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=EXCLUSIVE;
    Query OK, 300024 rows affected (2.38 sec)
    Records: 300024  Duplicates: 0  Warnings: 0
    
    # 삭제한 PK를 다시 추가함
    ALTER TABLE employees ADD PRIMARY KEY (emp_no), ALGORITHM=INPLACE, LOCK=NONE;
    Query OK, 0 rows affected (1.48 sec)Records: 0  Duplicates: 0  Warnings: 0
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ef8c6980-a693-4af2-95c4-656899e14808)

    
    - 스키마 변경 작업에 의해 DML이 멈추면 안된다? → 1, 2만 시도
    - 그래도 안되면 서비스를 멈추고 DML을 멈춘 다음 스키마 변경을 해야 합니다.
    
    **→ 1, 2번으로 해결이 가능해 타 커넥션을 대기하게 만들지 않더라도 해당 작업에 의해 부하가 발생할 수 있으므로 이를 주의해서 사용해야 합니다.**
    

---

<br>

**11.7.1.3 INPLACE 알고리즘**

- 임시 테이블로 레코드를 복사하진 않아도 내부적으로 테이블의 모든 레코드를 리빌드해야 하는 경우가 많음
- 리빌드를 해야 하는 경우라면 mysql서버는 다음과 같은 과정을 거칩니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6bfd518f-2d11-4245-9e69-b67380a05184)


    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/64802277-77c4-4940-959b-20bdde2ece7c)

    
    - `INPLACE더라도 2, 4 단계에서 잠깐의 배타적 잠금이 필요`합니다.(다른 커넥션의 DML들이 잠깐 대기함)
    - 작업 실행 및 많은 시간이 소요되는 3번 단계는 타 커넥션의 DML대기 없이 즉시 처리 됨(별도의 로그에 저장)
    - 스키마 변경동안 온라인 변경 로그에 쌓인 DML 쿼리를 실제 테이블에 일괄 적용함
- 온라인 변경 로그는 디스크가 아닌 메모리에만 생성됨 (`innodb_online_alter_log_max_size` 시스템 설정 변수)
    - 유입되는 DML이 많다면 크게 설정하는 적절히 조절할 필요가 있음 (기본값은 128MB)
    - 세션 단위의 동적 변수이므로 언제든지 변경 가능

---

<br>

**11.7.1.4 온라인 DDL의 실패 케이스**

- 온라인 DDL이 INSTANT 알고리즘을 사용하는 경우는 시작과 동시에 작업이 완료되기에 작업 도중 실패 가능성은 거의 없음
- INPLACE의 경우 내부적인 테이블 리빌드 → 최종 로그 적용이 필요 합니다. 따라서 중간 과정에서 실패할 가능성이 상대적으로 높습니다.
- 따라서 여러 실패 케이스를 살펴보고 최대한 온라인 DDL의 실패 가능성을 낮춰야 합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3a9b1ca8-4309-4c9a-a6dc-56c5e0f9ac17)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/0ad3fda9-8436-4b79-a56c-2f49860e2ddc)

    
- **주의**
    - 메타데이터에 대한 잠금을 확인하지 못할때 타임 아웃의 기준으로 사용되는 시스템 변수는 `lock_wait_timeout` 시스템 변수에 의해 결정됩니다. 
    (글로벌 + 세션 변수이므로 세션에서 적정 값으로 조정하여 사용하는편이 좋습니다.)

---

<br>

**11.7.1.5 온라인 DDL 진행 상황 모니터링**

- `온라인 DDL 포함 모든 ALTER TABLE 명령`은 mysql 서버의 `performance_schema`를 통해 진행 상황을 모니터링 할 수 있습니다.

1. 모니터링을 위해 performance_schema 옵션(Instrument, Consumer 옵션)이 활성화 돼야 함
    
    ```sql
    # performance_schema 시스템 변수 활성화(MySQL 서버 재시작 필요)
    SET GLOBAL performance_schema=ON;
    
    # "stage/innodb/alter%" instrument 활성화
    UPDATE performance_schema.setup_instruments
    SET ENABLED = 'YES', TIMED = 'YES'
    WHERE NAME LIKE 'stage/innodb/alter%';
    
    # "%stages%" consumer 활성화
    UPDATE performance_schema.setup_consumers
    SET ENABLED = 'YES'
    WHERE NAME LIKE '%stages%';
    ```
    
2. 스키마 변경 작업의 진행 상황은 performance_schema.`events_stages_current` 테이블을 통해 확인할 수 있습니다. 실행 중인 스키마 변경 종류에 따라 기록 내용이 조금씩 다릅니다.
    
    ```sql
    # COPY 알고리즘의 스키마 변경
    mysql_session1> ALTER TABLE salaries DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
    
    # performance_schema의 진행 상황
    mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                    FROM performance_schema.events_stages_current;
    ```
    
3. 스키마 변경 작업이 온라인 DDL로 실행되는 경우 **온라인 DDL이 단계(Stage)별로 EVENT_NAME 칼럼의 값을 달리해서 보여주기에 다양한 상태를 보여줍니다.**
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/7388122d-75f6-45e3-a4ff-b7eb281753dd)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/582a3584-4152-4f47-a746-c6a9c35fb8f7)

    
    - WORK_COMPLETED, WORK_ESTIMATED 칼럼의 값을 통해 ALTER TABLE 진행 상황을 예측할 수 있습니다.
    - WORK_ESTIMATED 칼럼의 값은 예측치 이므로 조금씩 변경됩니다.
    - 대략적인 진행률은 `(WORK_COMPLETED * 100 / WORK_ESTIMATED)` 값
    - EVENT_NAME은 훨씬 많지만 너무 빨리 완료되므로 포착되지 않을 수 있습니다.

---

<br><br>

### 11.7.2 데이터베이스 변경

- MySQL에서 하나의 인스턴스는 1개 이상의 데이터베이스를 가질 수 있습니다.
- 다른 DBMS는 스키마와 데이터베이스를 구분해서 관리하지만 MySQL 서버에서는 동격의 개념입니다.
- **MySQL의 데이터베이스는 디스크의 물리적인 저장소를 구분하기도 하지만, 여러 데이터베이스의 테이블을 묶어서 조인 쿼리를 사용할 수 있으므로 단순히 논리적인 개념이기도 합니다.**
- 데이터베이스는 객체에 대한 권한을 구분하는 용도록 사용되기도 하지만, 그 이상의 큰 의미를 가지지는 않음
    - 따라서 DB 단위로 변경, 설정하는 DDL 명령은 많지 않습니다. (기본 문자 집합, 콜레이션 설정 정도)

<br>

**11.7.2.1 데이터베이스 생성**

```sql
# 기본 문자 집합과 콜레이션으로 생성 (character_set_server 시스템 변수에 정의된)
CREATE DATABASE [IF NOT EXISTS] employees;

# 별도의 문자집합 및 기본 콜레이션
CREATE DATABASE [IF NOT EXISTS] employees CHARACTER SET utf8mb4;

# 별도의 문자집합, 콜레이션
CREATE DATABASE [IF NOT EXISTS] employees
		CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

- IF NOT EXISTS는 해당 데이터베이스가 없을때만 DDL을 실행하도록 합니다.

---

<br>

**11.7.2.2 데이터베이스 목록**

```sql
SHOW DATABASES;
SHOW DATABASES LIKE '%emp%';
```

- 권한을 가지고 있는 데이터베이스의 목록을 나열 합니다. (`show databases` 명령에 대한 권한이 있어야함)
- 특정 이름을 포함한 데이터베이스 목록을 검색할수도 있습니다.

---

<br>

**11.7.2.3 데이터베이스 선택**

```sql
USE employees;
```

- 기본 데이터베이스를 선택함
- 기본 데이터베이스가 아닌 타 데이터베이스를 사용하려면 테이블, 프로시저 앞에 데이터베이스 이름을 반드시 명시해야 함

---

<br>

**11.7.2.4 데이터베이스 속성 변경**

```sql
ALTER DATABASE employees CHARACTER SET=euckr;
ALTER DATABASE employees CHARACTER SET=euckr COLLATE= euckr_korean_ci;
```

- DB 생성시 지정한 문자집합, 콜레이션 변경

---

<br>

**11.7.2.5 데이터베이스 삭제**

```sql
DROP DATABASE [IF EXISTS] employees;
```

---

<br><br>

### 11.7.3 테이블 스페이스 변경

- MySQL 서버에는 전통적으로 테이블별로 전용의 테이블스페이스를 사용해옴
- InnoDB 스토리지 엔진의 `시스템 테이블 스페이스`(ibdata1 파일)만 `제너럴 테이블스페이스`를 사용했는데, 제너럴 테이블스페이스는 여러 테이블의 데이터를 한꺼번에 저장하는 테이블스페이스를 의미합니다.
- 8.0버전이 되며 사용자 테이블을 제너럴 테이블스페이스로 저장하는 기능이 추가됐고, 테이블스페이스를 관리하는 DDL 명령들이 추가됐습니다.
- 하지만 8.0에서도 제너럴 테이블스페이스는 여러가지 제약사항을 가집니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/0349b378-71c3-471b-8a95-81b2a8ba1506)

    
- 그럼에도 사용자 테이블이 제너럴 테이블스페이스를 사용할 수 있게 개선된건 다음과 같은 장점이 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d066609f-e910-47e4-9a94-64452a91c2bf)

    
- 다만 위 장점은 테이블의 개수가 매우 많은 경우에 좀 더 도드라집니다. 제너럴 스페이스 사용여부는 innodb_file_per_table 시스템 변수의 값을 통해 결정할 수 있습니다.

- `DOMORE` 우리가 생성하는 일반적인 테이블은 File-Per Tablespace에 저장됩니다.
    - innodb_file_per_table 시스템 변수가 ON이라면 해당 테이블스페이스를 사용함(디폴트가 on)
    - 테이블당 하나의 파일 테이블스페이스 데이터 파일을 만듭니다.
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/936ddc55-ad0f-4aaa-a51f-d970144d7b4d)

        https://dev.mysql.com/doc/refman/8.0/en/innodb-file-per-table-tablespaces.html
        
- 반면에 제너럴 테이블스페이스는 여러개의 테이블을 하나의 제너럴 테이블스페이스 파일에 저장하므로 FD(File Descriptor)사용을 File-Per Tablespace보다 적게 사용하며, 테이블스페이스 관리에 필요한 메모리 공간 또한 최소화할 수 있습니다.
    - File Descriptor: 유닉스 계열에서 파일을 다룰때 사용하는 개념, 파일에 접근하기 위한 추상적인 값

---

<br><br>

### 11.7.4 테이블 변경

- 테이블은 사용자의 데이터를 가지는 주체
- mysql 서버의 많은 옵션, 인덱스 등의 기능이 테이블에 종속되어 사용됨

<br>

**11.7.4.1 테이블 생성**

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tb_test (
		member_id BIGINT [UNSIGNED] [AUTO_INCREMENT],
		nickname CHAR(20) [CHARACTER SET 'utf8'] [COLLATE 'utf8_general_ci'] [NOT NULL],
		home_url VARCHAR(200) [COLLATE 'latin1_general_cs'],
	  birth_year SMALLINT [(4)] [UNSIGNED] [ZEROFILL],
		member_point INT [NOT NULL] [DEFAULT 0],
	  registered_dttm DATETIME [NOT NULL],
		modified_ts TIMESTAMP [NOT NULL] [DEFAULT CURRENT_TIMESTAMP],
		gender ENUM('Female','Male') [NOT NULL],
		hobby SET('Reading','Game','Sports'),
	  profile TEXT [NOT NULL],
	  session_data BLOB,
	  PRIMARY KEY (member_id),
		UNIQUE INDEX ux_nickname (nickname),
	  INDEX ix_registereddttm (registered_dttm)
) ENGINE=INNODB;
```

- `TEMPORARY 키워드`: 해당 DB 커넥션(세션)에서만 사용 가능한 임시 테이블을 생성함
- 테이블을 정의한 스크립트 마지막에 스토리지 엔진을 결정하기 위한 ENGINE 키워드를 사용할 수 있음(기본값은 InnoDB)
- 각 칼럼명은 `칼럼명 + 칼럼타입 + [타입별 옵션] + [NULL 여부] + [기본값]`의 순서로 명시합니다.
- 또한 타입별로 다음과 같은 옵션을 추가로 사용할 수 있습니다.
    - 모든 칼럼은 기본값 설정인 `DEFAULT`, `NULL`여부인 `NULL` or `NOT NULL` 제약 명시 가능
    - `문자열`
        - 문자열 타입은 칼럼에 최대한 저장할 수 있는 문자 수를 명시해야 함
        - CHARACTER SET절은 칼럼에 저장되는 문자열이 사용할 문자집합을 결정(이것만 쓰면 문자집합의 기본 콜레이션 사용)
        - COLLATE로 문자열 비교나 정렬 규칙을 나타내기 위한 콜레이션 설정 가능
    - `숫자`
        - 칼럼에 저장될 값의 길이가 아니라 단순히 값을 표시할 때 보여줄 길이를 지정함
        (`TINYINT`, `SMALLINT`, `INT`, `BIGINT`타입에 사용되는 길이는 8.0버전부터는 효력이 없음`(deprecated)`)
        - 양수, 음수를 선택할 수 있게 UNSIGNED 키워드를 붙일 수 있습니다.(기본은 SIGNED)
        - ZEROFILL을 통해 숫자 값의 왼쪽에 ‘0’패딩을 할지 결정함
    - `날짜`
        - 5.6부터 DATE, DATETIME, TIMESTAMP 타입 모두 값이 자동으로 현재 시간으로 업데이트되도록 기본 값을 명시할 수 있음
    - `ENUM`, `SET` 타입은 타입의 이름 뒤에 해당 칼럼이 가질 수 있는 값을 괄호로 정의해야 합니다.

---

<br>

**11.7.4.2 테이블 구조 조회**

- MySQL에서 테이블의 구조를 확인하는 방법은 SHOW CREATE TABLE 명령과 DESC 명령으로 두 가지가 존재
- `SHOW CREATE TABLE`
    - CREATE TABLE 문장을 표시해줌
    - 최초 테이블 생성할때 실행 내용을 그대로 보여주진 않고, mysql서버가 테이블의 메타 정보를 읽어서 이를 재작성해 보여줌
    - 특별한 수정 없이 바로 사용할 수 있는 create table 명령어를 만들어주므로 유용함
    - 칼럼, 인덱스, 외래키 정보를 동시에 보여주므로 SQL 튜닝 혹은 구조 확인시 주로 사용
- `DESC`
    - `DESCRIBE`의 약어 형태의 명령, 둘 모두 같은 결과를 보여줌
    - 테이블의 칼럼 정보를 보기 편한 표 형태로 표시
    - 인덱스 칼럼의 순서, 외래키, 테이블 자체의 속성을 보여주진 않으므로 전체적인 구조 파악에 어려움이 있음

---

<br>

**11.7.4.3 테이블 구조 변경**

- ALTER TABLE 명령을 사용한다.
- 테이블 자체의 속성 변경, 인덱스의 추가 삭제, 칼럼의 추가 삭제 등 다양하게 사용됨
- 자세한 내용은 메뉴얼에 존재 https://dev.mysql.com/doc/refman/8.0/en/alter-table.html

```sql
# 테이블의 기본 문자 집합과 콜레이션 변경. 테이블의 모든 칼럼과 기존 데이터까지 모두 변경함
ALTER TABLE employees CONVERT TO CHARACTER SET UTF8MB4 COLLATE UTF8MB4_GENERAL_CI,
ALGORITHM=INPLACE, LOCK=NONE;

# 테이블의 스토리지 엔진을 변경하는 명령
# -> 테이블 저장소의 변경이므로 항상 테이블의 모든 레코드를 복사하는 작업이 필요함
# 테이블 데이터를 리빌드해 테이블에서 데이터가 저장되지 않은 빈 공간(Fragmentation)을 제거해 디스크 사용 공간을 줄이는 역할을 함
ALTER TABLE employees ENGINE=InnoDB, 
ALGORITHM=INPLACE, LOCK=NONE;
```

- 디스크 공간의 fragmentation을 최소화하기 위해 OPTIMIZE TABLE이란 명령도 있지만 이는 내부적으로 위 쿼리와 동일한 작업을 수행합니다.

---

<br>

**11.7.4.4 테이블 명 변경**

- RENAME TABLE 명령을 이용합니다.
- 단순히 테이블의 이름 변경뿐 아니라 다른 데이터베이스로 테이블을 이동할 때도 사용할 수 있습니다.

```sql
RENAME TABLE table1 TO table2; # 단순 이름 변경
RENAME TABLE db1.table1 TO db2.table2; # 데이터베이스 이동
```

- 단순 이름 변경은 메타데이터만 변경하므로 빠르게 처리됨
- DB를 변경하면 테이블이 저장된 파일까지 다른 디렉터리로 이동해야 함
    - db1, 2가 서로 다른 파티션이라면 데이터 파일을 먼저 복사하고 원본 파티션의 파일을 삭제합니다. mysql 서버의 RENAME TABLE에서도 똑같이 동작합니다.
    - 서로 다른 운영체제의 파일 시스템을 사용하고 있다면 데이터 파일의 복사 작업이 필요하므로 파일 크기에 비례하여 시간이 소요됨
- 일정 주기로 테이블을 교체(Swap)하는 경우도 있습니다.
    
    ```sql
    -- // 새로운 테이블 및 데이터 생성
    CREATE TABLE batch_new (...);
    INSERT INTO batch_new SELECT ...;
    
    -- // 기존 테이블과 교체
    RENAME TABLE batch TO batch_old;
    RENAME TABLE batch_new TO batch;
    ```
    
    - 위와 같이 배치 프로그램이 작성되면 마지막 기존 테이블과 신규 테이블을 교체하는 동안 일시적으로 batch 테이블이 없어지는 시점이 발생함
    - 응용 프로그램은 batch 테이블을 찾지 못해 에러를 발생시킵니다.
    - 이를 위해 여러 테이블의 RENAME 명령어를 하나의 문장으로 묶어서 실행할 수 있습니다.
        
        ```sql
        RENAME TABLE batch TO batch_old, batch_new TO batch;
        ```
        
        - mysql 서버는 명시된 모든 테이블에 대해 잠금을 걸고 테이블의 이름 변경 작업을 실행함
        - 응용프로그램 입장에선 batch 테이블을 조회하려 할때 잠금으로 대기합니다.
        (잠깐의 잠금 대기, not error)

---

<br>

**11.7.4.5 테이블 상태 조회**

- mysql의 모든 테이블은 만들어진 시간, 대략의 레코드 건수, 데이터 파일의 크기 등의 정보를 가지고 있음
- 데이터 파일 버전, 레코드 포맷과 같이 자주 사용되진 않아도 중요한 정보들을 가지고 있음
- 이를 조회하려면 `SHOW TABLE STATUS … (LIKE 패턴 등)`를 통해 조회할 수 있습니다.
    
    ```sql
    SHOW TABLE STATUS LIKE 'employees' \G # \G 옵션 좋네용
    *************************** 1. row ***************************
               Name: employees
             Engine: InnoDB
            Version: 10
         Row_format: Dynamic
               Rows: 299238
     Avg_row_length: 57
        Data_length: 17317888
    Max_data_length: 0
       Index_length: 44138496
          Data_free: 6291456
     Auto_increment: NULL
        Create_time: 2024-03-03 23:14:56
        Update_time: NULL
         Check_time: NULL
          Collation: utf8mb4_general_ci
           Checksum: NULL
     Create_options: stats_persistent=1
            Comment:
    1 row in set (0.00 sec)
    ```
    
- 테이블이 사용하는 스토리지 엔진, 데이터 파일 포맷, 전체 레코드 건수, 레코드 건당 바이트 크기 등 다양한 정보를 일괄적으로 확인할 수 있습니다.
    - 레코드 건수, 레코드 평균 크기는 서버 예측값으로 테이블이 너무 작거나 크면 오차가 커질 수 있습니다.
- 또한 SELECT 쿼리를 이용해서도 조회가 가능합니다.
    
    ```sql
    SELECT * FROM information_schema.TABLES
    WHERE TABLE_SCHEMA='employees' AND TABLE_NAME='employees' \G
    *************************** 1. row ***************************
      TABLE_CATALOG: def
       TABLE_SCHEMA: employees
         TABLE_NAME: employees
         TABLE_TYPE: BASE TABLE
             ENGINE: InnoDB
            VERSION: 10
         ROW_FORMAT: Dynamic
         TABLE_ROWS: 299238
     AVG_ROW_LENGTH: 57
        DATA_LENGTH: 17317888
    MAX_DATA_LENGTH: 0
       INDEX_LENGTH: 44138496
          DATA_FREE: 6291456
     AUTO_INCREMENT: NULL
        CREATE_TIME: 2024-03-03 23:14:56
        UPDATE_TIME: NULL
         CHECK_TIME: NULL
    TABLE_COLLATION: utf8mb4_general_ci
           CHECKSUM: NULL
     CREATE_OPTIONS: stats_persistent=1
      TABLE_COMMENT:
    ```
    
    - information_schema DB는 mysql 서버가 가진 스키마들에 대한 메타 정보를 가진 딕셔너리 테이블이 관리됩니다.
    - mysql 서버가 시작되며 DB와 테이블에 등에 대한 다양한 메타 정보를 모아서 메모리에 모아두고 사용자가 참조할 수 있는 테이블 입니다.
    - TABLES or COLUMNS 뷰를 이용해 많은 정보를 얻을 수 있음
        
        ```sql
        # MySQL 서버에 존재하는 테이블들이 사용하는 디스크 공간 정보 조회
        SELECT TABLE_SCHEMA,
        	SUM(DATA_LENGTH)/1024/1024 as data_size_mb,
        	SUM(INDEX_LENGTH)/1024/1024 as index_size_mb       
        FROM information_schema.TABLES
        GROUP BY TABLE_SCHEMA;
        
        +---------------------+---------------+---------------+
        | TABLE_SCHEMA        | data_size_mb  | index_size_mb |
        +---------------------+---------------+---------------+
        | mysql               |    2.35937500 |    0.32812500 |
        | information_schema  |    0.00000000 |    0.00000000 |
        | performance_schema  |    0.00000000 |    0.00000000 |
        | sys                 |    0.01562500 |    0.00000000 |
        | java2-jdbc          |    0.01562500 |    0.00000000 |
        | jscode-spring-class |    1.56250000 |    0.07812500 |
        | jscode-springboot   |    0.06250000 |    0.01562500 |
        | mlb                 |    0.07812500 |    0.15625000 |
        | theater_db          |    0.09375000 |    0.09375000 |
        | tonton              |    0.12500000 |    0.17187500 |
        | teset               |    0.01564503 |    0.00195313 |
        | employees           | 4228.03516293 |  167.68847656 |
        | spring-lab-security |    0.03125000 |    0.00000000 |
        | test                |    0.01562500 |    0.01562500 |
        +---------------------+---------------+---------------+
        ```
        
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/375f6461-6591-420b-8fc1-a60f9448ee93)

    
    https://dev.mysql.com/doc/refman/8.0/en/information-schema.html
    

---

<br>

**11.7.4.6 테이블 구조 복사**

- 이름만 다른 테이블은 SHOW CREATE TABLE 명령을 이용해 테이블의 DDL을 조회하고 조금 변경하여 만들 수 있음
    - 혹은 CREATE TABLE … AS SELECT … LIMIT 0과 같이 만들 수도 있음(단, 인덱스는 생성 안됨)
- 데이터 복사 없이 테이블의 구조만 동일하게 복사하는 명령은 `CREATE TABLE … LIKE`를 사용하면 됩니다.
    
    ```sql
    # 테이블 구조 복사
    CREATE TABLE temp_employees LIKE employees;
    
    # 데이터 복사
    INSERT INTO temp_employees SELECT * FROM employees;
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/90c9815e-ed16-463d-b1d4-3a03e95ca93d)
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3a88a575-b6b9-41b0-bdea-bc7985dc40a8)


    

---

<br>

**11.7.4.7 테이블 삭제**

- 레코드가 많지 않은 테이블을 삭제하는 작업은 서비스 도중이어도 문제가 되지 않습니다.
- 8.0 버전은 특정 테이블을 삭제하는 작업이 다른 테이블의 DML, 쿼리를 직접 방해하지 않습니다.

```sql
DROP TABLE [ IF EXISTS ] table1;
```

- `다만 용량이 큰 테이블의 삭제는 부하가 큰 작업에 속합니다.`
    - 테이블이 사용하는 데이터 파일의 크기가 너무커 분산 저장돼 있을때 많은 Disk I/O발생
    - Disk I/O의 증가는 다른 커넥션의 쿼리 처리 성능을 떨어트릴 수 있음
    - 이런 간접적인 영향 때문에 서비스 도중에 DROP TABLE은 수행하지 않는것이 좋음
    - 데이터 파일이 리눅스의 ext3 파일 시스템을 사용하면 디스크의 이곳저곳에 분산되어 저장됨 (주의)
- `또한 InnoDB 스토리지 엔진의 어댑티브 해시 인덱스를 주의해야 합니다.`
    - 어댑티브 해시 인덱스는 InnoDB 버퍼 풀의 각 페이지가 가진 레코드에 대한 해시 인덱스 기능을 제공함
    - 활성화되어 있다면 테이블이 삭제되면 같이 모두 삭제해야 함
    - 정보를 많이 들고 있다면 어댑티브 해시 인덱스 삭제로 인해 mysql서버의 부하가 높아지고 간접적으로 다른 쿼리에 영향을 줌
    - 어댑티브 해시 인덱스는 테이블 삭제 뿐만 아니라 테이블의 스키마 변경에도 영향을 미칠 수 있음

---

<br><br>

### 11.7.5 칼럼 변경

- 테이블 구조 변경 작업은 주로 칼럼 추가, 타입 변경 등의 작업입니다.

<br>

**11.7.5.1 칼럼 추가**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/74447f29-1d11-47ee-a965-b7acd5cf9e8b)


- 8.0버전부터 테이블의 칼럼 추가 작업은 대부분 INPLACE 알고리즘을 사용하는 온라인 DDL로 처리가 가능합니다.
- 만약 칼럼을 테이블 가장 마지막 칼럼으로 추가한다면 INSTANT 알고리즘으로 즉시 추가 됩니다.
    - 따라서 테이블 큰 경우라면 테이블의 가장 마지막 칼럼에 추가하는 것이 좋습니다.
- 스키마 변경 작업시 어떤 종류의 알고리즘, 락을 사용할지 모르겠다면 TOP DOWN으로 진행해보는것을 권장합니다. (ALGORITHM이 INSTANT면 lock절을 사용할 수 없습니다.)

---

<br>

**11.7.5.2 칼럼 삭제**

- 칼럼 삭제는 항상 테이블의 리빌드를 필요로함 (INSTANT 알고리즘 사용 불가)
- 항상 INPLACE 알고리즘으로만 칼럼 삭제가 가능합니다.
- ALTER TABLE 명령에서 COLUMN 키워드는 입력하지 않아도 됩니다.

```sql
ALTER TABLE employees DROP COLUMN emp_telno,
ALGORITHM=INPLACE, LOCK=NONE;
```

---

<br>

**11.7.5.3 칼럼 이름 및 칼럼 타입 변경**

```sql
# 칼럼의 이름 변경(데이터 리빌드 없는 INPLACE 알고리즘)
ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL,
ALGORITHM=INPLACE, LOCK=NONE;

# INT 칼럼을 VARCHAR 타입으로 변경(데이터 타입 변경시 COPY, 테이블 읽기만 가능)
ALTER TABLE salaries MODIFY salary VARCHAR(20),
ALGORITHM=COPY, LOCK=SHARED;

# VARCHAR 타입의 길이 확장(테이블 리빌드 필요하거나 필요하지 않을 수 있음)
ALTER TABLE employees MODIFY last_name VARCHAR(30) NOT NULL,
ALGORITHM=INPLACE, LOCK=NONE;

# VARCHAR 타입의 길이 축소(타입 길이 축소시 COPY 알고리즘 사용, 변경 막기 위해 SHARED)
ALTER TABLE employees MODIFY last_name varchar(10) NOT NULL,
ALGORITHM=COPY, LOCK=SHARED;
```

- 주로 가장 빈번한 경우는 VARCHAR, VARBINARY 타입 길이 확장
- VARCHAR, VARBINARY의 최대 허용 사이즈는 메타데이터에 저장, 실제 칼럼이 가지는 값의 길이는 데이터 레코드의 칼럼 헤더에 저장됨
- 값의 길이를 위해서 사용하는 공간의 크기는 VARCHAR 칼럼이 최대 가질 수 있는 바이트 수만큼 필요함
    - 칼럼값의 바이트수가 255이하면 1바이트만 사용(8개 비트)
    - 256바이트 이상이면  2바이트 사용(이때부터 9개비트가 필요하므로)
    - 동일 값이라도 VARCHAR(10)보다 VARCHAR(1000)칼럼에 저장될때 1바이트를 더 씀
- INPLACE 알고리즘에서 VARCHAR(10) → VARCHAR(20)은 둘다 1바이트만 사용하므로 테이블 리빌드가 필요없지만,
- 한 글자가 최대 4바이트를 사용하는 UTF8MB4 문자 셋을 사용하는 경우 `VARCHAR(10) → VARCHAR(64)` 변경시 테이블 리빌드 필요(256바이트를 넘기므로)

---

<br><br>

### 11.7.6 인덱스 변경

- 8.0 버전부터는 대부분의 인덱스 변경 작업이 온라인 DDL로 처리 가능하도록 개선

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/9933c0d8-bf34-4889-a80d-5cf5c3946040)

<br>

**11.7.6.1 인덱스 추가**

- 인덱스의 종류, 인덱싱 알고리즘 별 사용 가능한 ALTER TABLE ADD INDEX 문장
    
    ```sql
    ALTER TABLE employees ADD PRIMARY KEY (emp_no),
    ALGORITHM=INPLACE, LOCK=NONE;
    
    ALTER TABLE employees ADD UNIQUE INDEX ux_empno (emp_no),
    ALGORITHM=INPLACE, LOCK=NONE;
    
    ALTER TABLE employees ADD INDEX ix_lastname (last_name),
    ALGORITHM=INPLACE, LOCK=NONE;
    
    # 전문 검색
    ALTER TABLE employees ADD FULLTEXT INDEX fx_firstname_lastname (first_name, last_name),
    ALGORITHM=INPLACE, LOCK=SHARED;
    
    # 공간 검색
    ALTER TABLE employees ADD SPATIAL INDEX fx_loc (last_location),
    ALGORITHM=INPLACE, LOCK=SHARED;
    ```
    
    - 전문 검색, 공간 검색은 INPLACE + SHARED LOCK이 필요
    - 나머지 B-Tree 자료 구조를 사용하는 인덱스의 추가는 PK여도 INPLACE + NONE LOCK으로 온라인 인덱스 생성이 가능합니다.

---

<br>

**11.7.6.2 인덱스 조회**

- SHOW INDEXES, SHOW CREATE TABLE 명령으로 인덱스를 조회할 수 있음
(후자는 테이블 생성 명령을 참조)

```sql
show index from employees;
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name              | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          1 | ix_hiredate           |            1 | hire_date   | A         |        5236 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_gender_birthdate   |            1 | gender      | A         |           1 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_gender_birthdate   |            2 | birth_date  | A         |        9514 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_firstname          |            1 | first_name  | A         |        1310 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_lastname_firstname |            1 | last_name   | A         |        1666 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_lastname_firstname |            2 | first_name  | A         |      277255 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
```

- `Seq_in_index`: 인덱스에서 해당 칼럼의 위치
- `Cardinality`: 인덱스에서 해당 칼럼까지의 유니크한 값의 개수를 보여줍니다.

```sql
show create table employees;

CREATE TABLE `employees` (
  `emp_no` int NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `last_name` varchar(16) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `gender` enum('M','F') CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `hire_date` date NOT NULL,
  KEY `ix_hiredate` (`hire_date`),
  KEY `ix_gender_birthdate` (`gender`,`birth_date`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_lastname_firstname` (`last_name`,`first_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci STATS_PERSISTENT=1
```

- 테이블의 생성 구문을 통해 인덱스와 함께 테이블의 모든 칼럼까지 표시합니다.

---

<br>

**11.7.6.3 인덱스 이름 변경**

- 기존 쿼리 문장에서 인덱스의 이름을 힌트로 사용했을때 인덱스의 변경, 삭제가 발생하면 응용 프로그램의 코드 변경 필요했음
- 5.7부터는 인덱스의 이름을 변경할 수 있게 되어 위 문제를 해결할 수 있음

```sql
ALTER TABLE salaries RENAME INDEX ix_salary TO ix_salary2,
ALGORITHM=INPLACE, LOCK=NONE;
```

- INPLACE 알고리즘을 사용 하지만 테이블 리빌드가 필요하진 않습니다.

```sql
# ix_firstname(first_name) -> ix_firstname(first_name, last_name) 변경 작업

-- // 1. index_new라는 이름으로 새로운 인덱스를 생성
ALTER TABLE employees
ADD INDEX index_new (first_name, last_name),
ALGORITHM=INPLACE, LOCK=NONE;

-- // 2. 기존 인덱스(ix_firstname)를 삭제하고,
-- //    동시에 새로운 인덱스(index_new)의 이름을 ix_firstname으로 변경
ALTER TABLE employees
DROP INDEX ix_firstname,
RENAME INDEX index_new TO ix_firstname,
ALGORITHM=INPLACE, LOCK=NONE;
```

---

<br>

**11.7.6.4 인덱스 가시성 변경**

- 인덱스를 삭제하려하는데 이미 사용중이라면 부작용이 있을 수 있습니다. 따라서 8.0부터는 인덱스의 가시성을 제어할 수 있는 기능이 도입됐습니다.
- 인덱스의 가시성
    - mysql 서버가 쿼리 실행할 때 해당 인덱스를 사용할 수 있게 할지 말지를 결정하는 것입니다.

```sql
# 응용 프로그램의 쿼리에서 특정 인덱스가 사용되지 못하게 함
ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;

# 사용하게 하려면 VISIBLE 옵션을 명시하면 됨
```

- 인덱스가 INVISIBLE 상태로 변경되면 옵티마이저는 해당 상태 인덱스를 없는것으로 간주하고 실행 계획을 수립합니다.
- 위 명령은 메타데이터만 변경하므로 온라인 DDL로 실행되는지 여부를 고려하지 않아도 됩니다.
- 따라서 8.0부터는 인덱스 삭제 이전에 비활성화를 하여 미리 모니터링하고 인덱스를 삭제할 수 있습니다.
- 추가로 최초 인덱스 생성시에도 가시성을 설정할 수 있습니다.
    
    ```sql
    ALTER TABLE employees ADD INDEX ix_firstname_lastname (first_name, last_name) INVISIBLE;
    
    SHOW CREATE TABLE employees \G
    
    CREATE TABLE employees (
      emp_no int NOT NULL,
      ...
      PRIMARY KEY (emp_no),
    	KEY ix_hiredate (hire_date),
      KEY ix_gender_birthdate (gender,birth_date),
      KEY ix_firstname (first_name),
      KEY ix_firstname_lastname (first_name,last_name) /*!80000 INVISIBLE */
    ) ENGINE=InnoDB
    ```
    
    - 이렇게 새로 추가할 인덱스를 invisible로 두고, 부하가 적은 시점에 적용할 수 있습니다.
    - optimizer_switch 시스템 변수에 use_invisible_indexes 옵션이 ON으로 설정되면 옵티마이저는 invisible 상태의 인덱스도 사용할 수 있습니다.

---

<br>

**11.7.6.4 인덱스 삭제**

- ALTER TABLE … DROP INDEX … 명령으로 삭제할 수 있고, 일반적으로 삭제는 빨리 처리됩니다.
- INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 필요로 하지 않습니다.
- PK 삭제는 모든 세컨더리 인덱스의 리프 노드에 저장된 PK값을 삭제해야 하므로 임시 테이블로 레코드를 복사해서 테이블을 재구축 해야 합니다.
    - PK 삭제는 COPY 알고리즘을 사용함

```sql
# PK는 COPY, SHARED 잠금 필요
ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
ALTER TABLE employees DROP INDEX ux_empno, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE employees DROP INDEX fx_loc, ALGORITHM=INPLACE, LOCK=NONE;
```

---

<br><br>

### 11.7.7 테이블 변경 묶음 실행

- 하나의 테이블에 대해 여러가지 스키마 변경을 해야 하는 경우 온라인 DDL로 빠르게 변경하는 경우가 아니라면 모아서 실행하는게 효율적임

```sql
# 하나의 인덱스 생성마다 테이블 레코드 풀 스캔
ALTER TABLE employees ADD INDEX ix_lastname (last_name, first_name),
ALGROITHM=INPLACE, LOCK=NONE;

ALTER TABLE employees ADD INDEX ix_birthdate (birth_date)
ALGROITHM=INPLACE, LOCK=NONE;

# 테이블 레코드 한번만 풀스캔해 2개의 인덱스를 한꺼번에 생성할 수 있음
ALTER TABLE employees
ADD INDEX ix_lastname (last_name, first_name),
ADD INDEX ix_birthdate (birth_date),
ALGROITHM=INPLACE, LOCK=NONE;
```

- 또한 가능하면 같은 알고리즘을 사용하는 스키마 변경 작업을 모아서 실행하는게 효율적입니다. (INPLACE여도 테이블 리빌드 여부를 고려해서 모아 실행하면 더 효율 UP)

---

<br><br>

### 11.7.8 프로세스 조회 및 강제 종료

- SHOW PROCESSLIST로 접속된 사용자 목록, 클라이언트 사용자가 실행하는 쿼리를 확인할 수 있음
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ee2ab179-fffa-4e7e-9676-c591b454596b)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/7dc6fa08-1e3e-4b52-a0ae-88ac39ec7ed0)

    
    스레드의 상태 모니터링 - https://dev.mysql.com/doc/refman/8.0/en/thread-information.html
    
- SHOW PROCESSLIST는 mysql 서버가 어떤 상태인지 판단하는데도 도움이 됩니다. (Query인데 장시간 실행중인 경우 등)
- 특별히 관심을 두어야 할 부분은 `State 칼럼`의 내용입니다.
(대표적으로 Copying…, Sorting…로 시작하면 주의 깊게 봐야 함)
- Id 칼럼 값을 통해 특정 스레드에서 실행 중인 쿼리, 커넥션 자체를 강제 종료할 수 있습니다.
    
    ```sql
    # 4228 쿼리는 강제 종료, 커넥션은 유지
    KILL QUERY 4228;
    # 쿼리 및 커넥션까지 강제 종료, 커넥션에서 처리하던 트랜잭션은 자동 롤백
    KILL 4228;
    ```
    

---

<br><br>

### 11.7.9 활성 트랜잭션 조회

- 트랜잭션이 오랜 시간 완료되지 않고 활성 상태로 남아있는 것도 mysql 서버 성능에 영향을 미칩니다.
- `information_schema.innodb_trx` 테이블을 통해 트랜잭션 목록을 확인할 수 있습니다.
    
    ```sql
    # 5초 이상 활성 상태로 남아있는 프로세스만 조사하는 쿼리
    SELECT trx_id,
    		(SELECT CONCAT(user,'@',host)
    			FROM information_schema.processlist
    			WHERE id=trx_mysql_thread_id) AS source_info,
    		trx_state,
    		trx_started,
    		now(), 
    		(unix_timestamp(now()) - unix_timestamp(trx_started)) AS lasting_sec,
    		trx_requested_lock_id,
    		trx_wait_started,
    		trx_mysql_thread_id,
    		trx_tables_in_use,  
    		trx_tables_locked 
    FROM information_schema.innodb_trx
    WHERE (unix_timestamp(now()) - unix_timestamp(trx_started))>5 \G
    
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/548dfaa3-f885-49b0-952e-720659e88bfa)

    
    - 170초 동안 활성 트랜잭션 상태를 유지하고 있음
- 트랜잭션이 오랜시간 활성 상태를 유지하고 있다면 `information_schema.innodb_trx` 테이블에서 모든 정보를 조회해 트랜잭션이 변경한 레코드 수와, 잠그고 있는 레코드 수를 확인할 수 있습니다.
    
    ```sql
    SELECT * FROM information_schema.innodb_trx WHERE trx_id=23000 \G
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/126e3fd5-368e-448a-8670-e1ce1269dd7a)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3d15d3a7-b179-41d8-8bbf-be9f04b3bbf7)


    
    - trx_rows_modified, trx_rows_locked를 보면 1개의 레코드를 변경 및 잠금함을 알 수 있습니다.
- 잠그고 있는 레코드는 `performance_schema.data_locks` 테이블을 참조하면 됩니다.
    
    ```sql
    select * from performance_schema.data_locks;
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f8db1528-a0de-4893-a291-6b37fe6226d9)

    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/7688a4fb-09b7-4134-9b1e-4c0339bfacc5)

    
    - employees 테이블에 대해 IX 잠금(Intention Exclusive Lock)
    - employees PK record에 배타적 잠금을 가지고 있습니다.
    (REC_NOT_GAP은 갭락이 아니라 단순한 레코드 락을 의미)
- **5.3.3절에서도 레코드 수준의 잠금 확인 및 해제에서도 다루니 참고**
- 쿼리만 강제 종료하면 커넥션, 트랜잭션은 활성상태입니다. (응용 프로그램에서 쿼리의 에러를 감지해 트랜잭션을 롤백한다면 사용해도 됨)
    - `KILL QUERY {number}`
- 응용 프로그램에서 에러 핸들링이 확실치 않다면 커넥션 자체를 종료시키는 방법이 더 안정적입니다.
    - `KILL {number}`

---

<br><br><br>

## ✅ 11.8 쿼리 성능 테스트

- 쿼리의 효율성을 따지려면 실행 계획을 보고 문제가 될만한 부분이 있는지 검토합니다.
- 문제 없다면 쿼리를 실행해봄
- 다만 주변 방해 요소들을 간과하는건 위험한 일입니다. 따라서 성능 판단을 하려면 어떠한 부분을 고려하고 어떤 영향 요소가 있는지 살펴봐야 합니다.
- 이는 테스트 및 실제 쿼리가 실행될때의 과정 이해에 도움이 됩니다.

<br><br>

### 11.8.1 쿼리의 성능에 영향을 미치는 요소

- 성능 판단시 가장 큰 영향을 주는 변수는 mysql 서버가 가지고 있는 여러 종류의 버퍼, 캐시입니다.
- 이를 최소화 하기 위한 방법들을 알아봄

<br>

**11.8.1.1 OS의 캐시**

- mysql 서버는 OS의 파일 시스템 관련 기능(시스템 콜)을 이용해 데이터 파일을 읽어옴
- 대부분의 OS는 한 번 읽은 데이터는 OS가 관리하는 별도의 캐시 영역에 보관하고, 데이터가 요청되는 디스크 I/O없이 캐시의 내용을 바로 mysql 서버로 반환합니다.
- InnoDB는 일반적으로 파일 시스템의 캐시, 버퍼를 거치지 않는 `Direct I/O`를 사용하므로 OS의 캐시가 큰 영향을 미치지 않음 
(MyISAM은 OS 캐시 의존이 높아서 OS 캐시에 따라 성능 차이가 큼)
- OS가 관리하는 캐시, 버퍼는 공용 공간임
    - mysql 서버같은 응용 프로그램이 종료되어도 여전히 남아 있을 수 있음
    - 그래서 캐시나 버퍼가 전혀 없는 상태에서 성능을 테스트 하기 위해 OS의 캐시 삭제 명령을 실행하고 테스트하는것이 좋습니다. (OS마다 다를 수 있음)
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/71349d07-1ed7-4ba8-a6fe-20aec842fdfd)

        

---

<br>

**11.8.1.2 MySQL 서버의 버퍼 풀(InnoDB 버퍼 풀과 MyISAM의 키 캐시)**

- InnoDB 스토리지 엔진이 관리하는 캐시를 `버퍼 풀` 이라고 함
(MyISAM은 `키 캐시`라고 함
- `InnoDB 버퍼 풀`
    - 인덱스 페이지, 데이터 페이지 캐시 및 쓰기 작업을 위한 버퍼링 작업 처리
- `MyISAM 키 캐시`
    - 인덱스 데이터에 대해서만 캐시 기능을 제공함
    - 주로 읽기를 위한 캐시 역할을 수행하고 제한적으로 인덱스 변경만을 위한 버퍼 역할
    - 결국 인덱스 제외 테이블 데이터는 OS의 캐시에 의존할 수 밖에 없음
- 키 캐시, 버퍼 풀 초기화
    - MySQL 재시작 밖에 없음
    - `innodb_buffer_pool_load_at_startup`, `innodb_buffer_pool_load_at_shutdown` 변수를 OFF로 두면 mysql 서버가 종료될때 버퍼풀의 내용을 dump하지 않고 켤때도 자동으로 적재하지 않도록 할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5684f83a-da2a-40e2-ac81-cb9f347e366b)



---

<br>

**11.8.1.3 독립된 MySQL 서버**

- mysql 서버가 가동 중인 장비에 웹 서버나 다른 배치용 프로그램이 실행되고 있다면 테스트 하려는 쿼리의 성능이 영향을 받습니다.
- 테스트 쿼리를 실행하는 클라이언트 프로그램인 `네트워크의 영향` 요소도 고려해야 합니다.
- mysql 서버가 설치된 서버에 직접 로그인해 테스트해볼 수 있다면 위와 같은 요소를 쉽게 배제할 수 있습니다.

---

<br>

**11.8.1.4 쿼리 테스트 횟수**

- 성능 테스트를 mysql 서버가 `웜업 상태`(캐시, 버퍼 필요 데이터로 준비완료) 혹은 `콜드 상태`(캐시, 버퍼 초기화 상태)일때 진행할지도 고려해야 함
- 일반적인 쿼리 테스트는 `웜엄 상태를 가정하고 테스트` 합니다.
(특별히 영향을 미칠만할 프로세스, 다른 쿼리가 없다면 테스트 진행해도 충분함)
- 운영체제의 캐시, mysql 버퍼 풀 or 키 캐시는 크기가 `제한적`입니다.
    - 쿼리에서 필요로 하는 데이터나 인덱스 페이지보다 크기가 작으면 플러시 작업과 캐시 작업이 반복하여 발생하므로 한 번 실행해서 나온 결과를 그대로 신뢰하면 안됩니다.
    - 6 ~ 7번정도 실행하고 초기 결과는 버리고 나머지 결과의 평균값을 기준으로 비교하는것이 좋습니다.
    (처음에는 웜엄 되지 않았을 수 있기 때문에 편차 있을 수 있음)

위와 같은 사항을 고려해 쿼리의 성능을 비교하더라도 상대적인 비교지 절대적인 성능이 아닙니다. 따라서 해당 쿼리가 어떤 서버에서도 테스트 시간안에 처리된다고 보장할 수 없습니다.

운영서버에선 쿼리가 동시에 4~50개는 실행 중인 상태일 것(그랬으면 좋겠다..)이므로 보다 많은 경합이 발생하여 테스트때보다는 느린 처리 성능을 보이는게 일반적입니다.
