# 📌 11장 쿼리 작성 및 최적화

## ✅ 11.5 INSERT

- 소량의 레코드 INSERT는 성능을 고려할 부분이 많지 않음, 많은 INSERT 문장의 실행은 테이블의 구조가 성능에 더 큰 영향을 미칩니다.
- 하지만 INSERT, SELECT의 성능을 동시에 빠르게 만들 수 있는 테이블 구조는 없음
(어느정도 타협해야 함)

<br><br>

### 11.5.1 고급 옵션

- 유용한 기능
    - `INSERT IGNORE`, `INSERT … ON DUPLICATE KEY UPDATE`
    - 유니크 인덱스, PK에 대해 중복레코드를 어떻게 처리할지를 결정합니다.
    - INSERT IGNORE는 INSERT 문장의 에러 핸들링에 대한 기능도 포함합니다.

<br>

**11.5.1.1 INSERT IGNORE**

- **아래 경우를 모두 무시하고 다음 레코드를 처리할 수 있게 합니다.**
    - 레코드의 PK나 유니크 인덱스 칼럼의 값이 기존 레코드와 중복되는 경우
    - 저장하는 레코드의 칼럼이 테이블의 칼럼과 호환되지 않는 경우
- `주로 여러 레코드를 하나의 INSERT 문장으로 처리하는 경우 유용합니다.`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/9691e503-db7d-4d8a-b6fb-12d859b8d15c)

    
    - `PK는 (emp_no, from_date)` 만약 동일 조합의 중복 레코드가 있다면 MySQL서버는 에러가 아니라 경고 메시지로 변경하고 **해당 레코드를 무시하고 나머지 레코드의 INSERT**는 계속 진행합니다.
    - INSERT하고자 하는 레코드가 정교하지 않아도 되는 경우 자주 사용됩니다.
    - INSERT 테이블이 PK, UNIQUE INDEX 동시에 가지고 있다면 INSERT IGNORE는 두 인덱스 중 하나라도 중복이 발생하는 레코드에 대해서는 INSERT를 무시합니다.
- `데이터 타입이 일치하지 않아서 INSERT할 수 업슨 경우 칼럼의 기본값으로 INSERT를 함`
    
    ```sql
    insert into salaries values(null, null, null, null);
    ERROR 1048 (23000): Column 'emp_no' cannot be null
    
    insert ignore into salaries values (null, null, null, null);
    Query OK, 1 row affected, 4 warnings (0.01 sec)
    
    select * from salaries where emp_no = 0;
    +--------+--------+------------+------------+
    | emp_no | salary | from_date  | to_date    |
    +--------+--------+------------+------------+
    |      0 |      0 | 0000-00-00 | 0000-00-00 |
    +--------+--------+------------+------------+
    ```
    
- `제대로 검증되지 않은 INSERT IGNORE는 의도하지 않은 에러까지 모두 무시하므로 데이터 중복 이외의 에러가 발생할 여지가 없는지 면밀히 확인해야 합니다.`

---

<br>

**11.5.1.2 INSERT … ON DUPLICATE KEY UPDATE**

- PK나 UNIQUE INDEX 중복 발생시 UPDATE 문장과 비슷한 역할을 수행하게 합니다.
    - REPLACE 문장도 INSERT … ON DUPLICATE KEY UPDATE와 비슷한 역할을 하지만 내부적으로 REPLACE 문장은 DELETE와 INSERT의 조합으로 작동합니다. (성능상 큰 장점도 없음)
- INSERT … ON DUPLICATE KEY UPDATE는 중복된 레코드가 있다면 기존 레코드를 삭제하지 않고 기존 레코드의 칼럼을 UPDATE하는 방식으로 작동합니다.
- `일별로 집계되는 값 관리시 편리하게 사용할 수 있음`
    
    ```sql
    CREATE TABLE daily_statistic (
    		target_date DATE NOT NULL,
    		stat_name VARCHAR(10) NOT NULL,
    		stat_value BIGINT NOT NULL DEFAULT 0,
    		PRIMARY KEY(target_date, stat_name)
    );
    
    INSERT INTO daily_statistic
    		(target_date, stat_name, stat_value)
    		VALUES (DATE(NOW()), 'VISIT', 1)
    ON DUPLICATE KEY UPDATE stat_value=stat_value+1;
    
    ```
    
    - PK (target_date, stat_name)조합이므로 stat_name은 하나만 존재할 수 있습니다.
    - **stat_name이 최초로 저장된다면 INSERT 문장만 실행 ON DUP.. 문장은 무시됨**
    - **레코드가 존재한다면 INSERT대신 ON DUP…이 실행되어 기존 레코드 값 + 1의 값을 UPDATE함**
- `배치 형태로 GROUP BY결과를 daily_statistic 테이블에 한 번에 INSERT하는 예제`
    
    ```sql
    # VALUES()는 INSERT 하고자 했던 값을 다시 가져오지만
    # 8.0.20이후 부터 Deprecated 될 에정이므로 다른 문법 사용 권장
    INSERT INTO daily_statistic
    		SELECT DATE(visited_at), 'VISIT', COUNT(*)
    		FROM access_log
    		GROUP BY DATE(visited_at)
    ON DUPLICATE KEY UPDATE stat_value=stat_value + VALUES(stat_value);
    
    INSERT INTO daily_statistic
    	SELECT target_date, stat_name, stat_value
    	FROM (
    		SELECT DATE(visited_at) target_date, 'VISIT' stat_name, COUNT(*) stat_value
    		FROM access_log
    		GROUP BY DATE(visited_at)
    	) stat
    ON DUPLICATE KEY UPDATE
    daily_statistic.stat_value=daily_statistic.stat_value + stat.stat_value;
    
    # INSERT SELECT 형태의 문법이 아닌 경우 레코드에 별칭 부여
    INSERT INTO daily_statistic (target_date, stat_name, stat_value)
    		VALUES ('2020-09-01', 'VISIT', 1),
    						('2020-09-02', 'VISIT', 1)
    		AS new /* "new"라는 이름으로 별칭을 부여 */
    ON DUPLICATE KEY
    UPDATE daily_statistic.stat_value=daily_statistic.stat_value+new.stat_value;
    
    # INSERT INTO SET 문법에 적용
    INSERT INTO daily_statistic
    		/* "new"라는 이름으로 별칭을 부여 */
    		SET target_date='2020-09-01', stat_name='VISIT', stat_value=1 AS new
    ON DUPLICATE KEY
    UPDATE daily_statistic.stat_value=daily_statistic.stat_value+new.stat_value;
    
    INSERT INTO daily_statistic
    		/* "new"라는 이름으로 별칭을 부여하면서 칼럼의 별칭까지 부여 */
    		SET target_date='2020-09-01', stat_name='VISIT', stat_value=1 AS new(fd1, fd2, fd3)
    ON DUPLICATE KEY
    UPDATE daily_statistic.stat_value=daily_statistic.stat_value+new.fd3;
    
    ```
    

---

<br><br>

### 11.5.2 LOAD DATA 명령 주의 사항

- RDBMS에서 데이터를 빠르게 적재할 수 있는 방법으로 소개되는 명령어
- MySQL 서버도 내부적으로 MySQL 엔진, 서버 엔진 호출 횟수를 최소화하고, 스토리지 엔진이 직접 데이터를 적재하므로 일바넞ㄱ인 INSERT 명령과 비교했을때 빠름
- 다만 단점이 존재
    - `단일 스레드로 실행`
    - `단일 트랜잭션으로 실행`

- **데이터가 매우 커서 실행 시간이 길어진다면 다른 온라인 트랜잭션 쿼리들의 성능이 영향을 받을 수 있습니다.**
    - LOAD DATA는 단일 스레드 실행 → 데이터 파일이 매우크면 → 시간도 길어짐
    - 테이블에 여러 인덱스 있을때 → 레코드 삽입 이후 → 인덱스에도 키 값을 INSERT해야 함 → INSERT양이 많아질수록 크기도 비례하기 때문에 속도가 현저히 느려짐 (단일 스레드이기에)
- **하나의 트랜잭션으로 처리되기에 LOAD DATA 문장 시작 시점부터 언두로그가 삭제되지 못하고 유지됩니다.**
    - 언두 로그를 디스크로 기록해야 하는 부하 생성
    - 언두로그가 많이 쌓이면 레코드를 읽는 쿼리들이 필요한 레코드를 찾는 데 더 많은 오버헤드를 만듦

- 따라서 가능하면 적재할 데이터 파일을 여러 개의 파일로 준비해 LOAD DATA를 동시에 여러 트랜잭션으로 나뉘어 실행하게 하는것이 좋습니다.
- 테이블간 데이터 복사 작업이면 `INSERT … SELECT …` 문장으로 데이터를 부분적으로 자르는 것이 효율적임

---

<br><br>

### 11.5.3 성능을 위한 테이블 구조

- INSERT 문장은 테이블의 구조에 의해 성능이 많이 결정됨
- 소량의 레코드를 INSERT하는 문장은 쿼리 튜닝에서 무시되는 경우가 많음

<br>

**11.5.3.1 대량 INSERT 성능**

- 하나의 INSERT 문장으로 수백 건, 수천 건 레코드를 INSERT 한다면 **INSERT될 레코드들을 PK 값 기준으로 미리 정렬해서 INSERT 문장을 구성하는것이 성능에 도움이 될 수 있습니다.**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d2dae1b2-95cd-46be-92b6-5683175877c6)


- **위와 같은 파일이 있을때 책에서는 약 2배 이상의 시간차이가 발생함을 알 수 있습니다. 이는 PK로 정렬돼 있지 않기 때문입니다.**
- **PK로 정렬돼 있지 않다면**
    1. LOAD DATA 문장의 INSERT마다 InnoDB 스토리지 엔진은 PK를 검색해서 레코드가 저장될 위치를 찾음
    2. 각 레코드의 PK가 너무 다른 값을 가지면 InnoDB 스토리지 엔진이 레코드 저장때마다 B-Tree 랜덤 페이지를 메모리로 읽음 (랜덤 액세스)
    3. 이로인해 속도 저하됨 (버퍼풀이 엄청 크다면 차이가 조금 줄어들수도)
- **PK로 정렬돼 있다면**
    1. INSERT할 레코드의 PK값이 직전에 INSERT된 값보다 항상 큼
    2. 메모리에는 PK의 마지막 페이지만 적재돼있다면 새로운 페이지를 메모리로 가져오지 않아도 레코드를 저장할 위치를 찾을 수 있음
- **PK정렬 여부 말고도 세컨더리 인덱스도 INSERT 성능을 떨어트립니다.
(SELECT 성능은 높이지만)**
    - 따라서 세컨더리 인덱스를 너무 남용하는 것은 성능상 좋지 못합니다.
    - InnoDB 스토리지 엔진에서 세컨더리 인덱스의 변경이 일시적으로 체인지 버퍼에 버퍼링됐다가 백그라운드 스레드에 의해 일괄 처리할 수 있습니다.
    - 다만 너무 많은 세컨더리 인덱스는 백그라운드의 작업 부하를 유발하므로 전체적인 성능저하가 발생합니다.

---

<br>

**11.5.3.2 프라이머리 키 선정**

- **PK의 선정은 INSERT 성능, SELECT 성능의 대립되는 두 가지 요소 중에서 하나를 선택해야 합니다.**
- **INSERT가 더 많이 실행되는 테이블이라면 PK를 단조 증가 또는 단조 감소하는 패턴의 값을 선택하는것이 좋습니다.**
    - INSERT가 빨리 되려면 B-Tree 전체가 메모리에 적재돼 있어야 함
    - 단조 증가, 단조 감소라면 이전의 값보다 현재 값이 더 크거나 같음을 보장하므로 이전에 insert한 값이 메모리에 존재한다면 새로운 페이지를 메모리로 가져오지 않아도 레코드를 저장할 수 있음
    - 또한 인덱스의 개수를 최소화 하는것이 좋음
- **SELECT가 더 많이 실행되는 테이블이라면 빈번하게 실행되는 SELECT 쿼리의 조건을 기준으로 PK를 선택하는 것이 좋습니다.**
    - 쿼리에 맞게 필요한 세컨더리 인덱스들을 추가해도 시스템 전반적으로 영향도가 크지 않습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/0156a9b4-9a9e-473a-806e-838a761a2514)


---

<br>

**11.5.3.3 AUTO-Increment 칼럼**

SELECT보다는 INSERT에 최적화된 테이블을 생성하기 위해 다음 두 가지 요소를 갖춘 테이블을 준비합니다.

- 단조 증가 or 단조 감소되는 값으로 PK값 선정
- 세컨더리 인덱스 최소화

InnoDB 스토리지 엔진은 테이블 PK로 자동 클러스터링 합니다. PK로 클러스터링 되지 않게 InnoDB 테이블을 생성할 수 없습니다.

하지만, Auto-Increment 칼럼을 이용하면 클러스터링 되지 않는 테이블의 효과를 얻을 수 있습니다.

MySQL 서버는 자동 증가 값의 채번(고유한 값 생성)을 위한 AUTO-INC 잠금이 필요합니다.

해당 잠금 방식은 `innodb_authoinc_lock_mode`라는 시스템 변수로 변경할 수 있습니다. (InnoDB이외의 스토리지 엔진 사용 테이블에는 영향X)

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cb52f141-c343-40ff-a83f-6001084a9a81)


- 5.7까지는 1, 8.0부터는 기본값이 2로 변경됨
    - 8.0부터 복제의 바이너리 로그 포맷 기본값이 ROW로 변경됐기 때문입니다.
    - 만약 바이너리 로그 포맷을 STATEMENT로 사용중이라면 1로 설정해야 합니다.(명령어 기준이면 정확한 값을 알 수 없으므로)
- 1로 설정해도 시간이 조금씩 지나며 연속된 값에 빈 공간이 생길 가능성이 높습니다.(데이터 삭제 등) 따라서 값이 반드시 연속이어야 한다는 요건에 집착하지 않아도 됩니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f1d33f50-eb31-4235-aa21-2185a419a5de)


---

<br><br><br>

## ✅ 11.6 UPDATE와 DELETE

- 주로 하나의 테이블에 대해 1건 이상의 레코드를 변경, 삭제할때 사용됩니다.
- 또한 여러 테이블을 조인해서 한 개 이상 테이블의 레코드 변경, 삭제도 가능합니다.
(JOIN UPDATE, JOIN DELETE)
- UPDATE, DELETE는 작정법이나 WHERE절의 인덱스 사용법 모두 동일합니다.

<br><br>

### 11.6.1 UPDATE … ORDER BY … LIMIT n

- ORDER BY절과 LIMIT 절을 동시에 사용해 특정 칼럼으로 정렬해 상위 몇 건만 변경, 삭제가 가능합니다.
- 다만 복제 소스 서버에서 사용하면 경고메시지가 발생됩니다.
(바이너리 로그 포맷이 ROW가 아니라 STATEMENT라면 주의가 필요)
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e1436555-702c-4e0c-a6d1-4f997c12f7ea)

    
    - ORDER BY에 의해 정렬되어도 중복된 값의 순서가 복제 소스 서버와 레플리카  서버에서 달라질 수 있기 때문입니다.
    - PK 정렬을 하면 문제가 없지만 여전히 경고 메시지가 기록됩니다.

---

<br><br>

### 11.6.2 JOIN UPDATE

- 두 개 이상의 테이블을 조인해 조인된 결과 레코드를 변경 및 삭제하는 쿼리를 말합니다.
- 조인된 테이블 중에서 특정 테이블의 칼럼값을 다른 테이블의 칼럼에 업데이트 할 때 혹은 양쪽 테이블에 공통으로 존재하는 레코드만 찾아서 업데이트하는 용도로 사용할 수 있습니다.
- 모든 테이블에 대해 읽기 참조만 되는 테이블은 읽기 잠금이, 칼럼이 변경되는 테이블은 쓰기 잠금이 걸립니다.
    - 따라서 웹서비스같은 OLTP 환경에서 데드락을 유발할 가능성이 높으므로 빈번한 사용은 피해야 함
    - 배치 프로그램, 통계용 UPATE 문장에선 유용하게 사용됨

- `JOIN UPDATE 간단 예제`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f17e92c2-f4cd-4c95-ae8d-b07b3904d597)

    
    ```sql
    # 업데이트 이전
    select * from tb_test1;
    +--------+------------+
    | emp_no | first_name |
    +--------+------------+
    |  10001 | NULL       |
    |  10002 | NULL       |
    |  10003 | NULL       |
    |  10004 | NULL       |
    +--------+------------+
    
    # 업데이트 이후
    select * from tb_test1; 
    +--------+------------+
    | emp_no | first_name |
    +--------+------------+
    |  10001 | Georgi     |
    |  10002 | Bezalel    |
    |  10003 | Parto      |
    |  10004 | Chirstian  |
    +--------+------------+
    ```
    
    - 위 JOIN UPDATE 쿼리는 employees 테이블과 사원 번호로 조인하고 first_name 칼럼을 tb_test1 테이블에 복사합니다.
    - JOIN UPDATE 쿼리도 2개 이상의 테이블을 먼저 조인하므로 조인 순서에 따른 UPDATE 문장의 성능이 달라집니다. 따라서 먼저 실행 계획을 확인하는 것이 좋습니다.

- `GROUP BY가  포함된 JOIN UPDATE`
    
    기본적으로 JOIN UPDATE에선 `GROUP BY`, `ORDER BY`절을 사용할 수 없기에 다음과 같이 서브쿼리를 이용해 이를 이용할 수 있습니다.
    
    ```sql
    ALTER TABLE departments ADD emp_count INT;
    
    UPDATE departments d, 
    				(SELECT de.dept_no, COUNT(*) AS emp_count
    				 FROM dept_emp de 
    				 GROUP BY de.dept_no) dc
    SET d.emp_count=dc.emp_count
    WHERE dc.dept_no=d.dept_no;
    ```
    
    - 위 쿼리는 자동으로 임시 테이블이 드라이빙 테이블이 되겠지만, 원하는 조인의 방향이 있다면 JOIN UPDATE 문장에 STRAIGHT_JOIN 키워드를 적용하거나 8.0에 추가된 JOIN_ORDER 옵티마이저 힌트를 적용할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/2ce96c13-229b-4b0b-8179-70d90969998c)

        
        - STRAIGHT_JOIN은 순서를 지정하는 MySQL 힌트이며 조인 키워드로도 사용됩니다. 이를 사용하면 왼쪽에 명시된 테이블이 드라이빙, 우측의 테이블이 드리븐 테이블이 됩니다.
    - 혹은 래터럴 조인을 활용해 JOIN UPDATE를 구현할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/87f7151a-146e-435a-a45a-6bc9092a7dcd)

        

---

<br><br>

### 11.6.3 여러 레코드 UPDATE

- 하나의 문장으로 여러개의 레코드 업데이트는 선택된 레코드를 동일한 값으로 업데이트 합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/092f73ca-7843-4e6d-aa0c-e5fd04fbe075)

    
- MySQL 8.0부터는 `레코드 생성 문법을 이용해 레코드별로 서로 다른 값을 업데이트` 합니다.
    
    ```sql
    CREATE TABLE user_level (
    		user_id BIGINT NOT NULL,
    		user_lv INT NOT NULL,
    		created_at DATETIME NOT NULL,
    		PRIMARY KEY (user_id)
    );
    
    # update 이전
    select * from user_level;
    +---------+---------+---------------------+
    | user_id | user_lv | created_at          |
    +---------+---------+---------------------+
    |       1 |       1 | 2024-02-25 20:49:27 |
    |       2 |       2 | 2024-02-25 20:49:34 |
    |       3 |       3 | 2024-02-25 20:49:39 |
    +---------+---------+---------------------+
    
    # update 이후
    UPDATE user_level ul
    	INNER JOIN (
    		VALUES ROW(1, 1), ROW(2, 4)
    	) new_user_level (user_id, user_lv)
    		ON new_user_level.user_id=ul.user_id
    SET ul.user_lv=ul.user_lv + new_user_level.user_lv;
    
    # user_id = 1, 4인 칼럼에 각각 1, 4라는 user_lv값이 더해짐
    select * from user_level;
    +---------+---------+---------------------+
    | user_id | user_lv | created_at          |
    +---------+---------+---------------------+
    |       1 |       2 | 2024-02-25 20:49:27 |
    |       2 |       6 | 2024-02-25 20:49:34 |
    |       3 |       3 | 2024-02-25 20:49:39 |
    +---------+---------+---------------------+
    ```
    
    - `VALUES ROW(…), ROW(…), …` 문법은 SQL 문장 내에서 임시 테이블을 생성하는 효과를 냅니다.
    - 따라서 2건의 레코드를 갖는 임시테이블(new_user_level)을 생성해 user_level 테이블과 조인하여 업데이트를 수행하는 JOIN UPDATE 문장의 효과를 냅니다.

---

<br><br>

### 11.6.4 JOIN DELETE

- JOIN DELETE는 단일 테이블의 DELETE 문장과는 조금 다른 문법으로 쿼리를 작성합니다.
- `3개의 테이블을 조인해 그 중 하나의 테이블에서만 레코드를 삭제하는 예제`
    
    ```sql
    DELETE e
    FROM employees e, dept_emp de, departments d
    WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
    ```
    
    - 하나의 테이블은 DELETE FROM table과 같은 문법을 사용함
    - JOIN DELETE는 DELETE와 FROM절 사이에 삭제할 테이블을 명시합니다.
- `여러 테이블에서의 레코드 삭제 예제`
    
    ```sql
    DELETE e, de
    FROM employees e, dept_emp de, departments d
    WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
    
    DELETE e, de, d
    FROM employees e, dept_emp de, departments d
    WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
    ```
    
    - 첫번째 쿼리는 e, de 테이블에서 동시에 레코드를 삭제
    - 두 번째 쿼리는 3개의 테이블 모두에서 레코드를 삭제합니다.
    - `JOIN DELETE 또한 JOIN UPDATE와 마찬가지로 SELECT 쿼리로 변환하여 실행 계획을 확인해볼 수 있습니다. -> 어떤 의미인지?`
    - 옵티마이저가 적절한 조인 순서를 결정하지 못하면 STRAIGHT_JOIN, JOIN_ORDER 옵티마이저 힌트를 통해 조인의 순서를 지정할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/792a8a3c-70b8-4455-94de-bc143a224fa3)

        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/87a0d055-8e8d-4004-8a96-e3409f28bbeb)
