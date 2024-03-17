# 📌 11장 쿼리 작성 및 최적화

## ✅ 11.4 SELECT

### 11.4.8 GROUP BY

group by는 특정 칼럼 값으로 레코드를 그루핑 및 집계된 결과를 하나의 레코드로 조회할때 사용합니다.

<br>

**11.4.8.1 WITH ROLLUP**

- 그루핑된 그룹별로 소계를 가져올 수 있는 롤업 기능
- 출력되는 소계는 단순한 최종 합만 가져오는게 아닌 GROUP BY에 사용된 칼럼의 개수에 따라 소계의 레벨이 달라잡니다.
(엑셀의 피벗 테이블과 거의 동일한 기능)
- `1개 컬럼 그루핑(소계 레코드의 칼럼값은 항상 NULL임)`
    
    ```sql
    SELECT dept_no, COUNT(*)
    FROM dept_emp
    GROUP BY dept_no WITH ROLLUP;
    
    +---------+----------+
    | dept_no | COUNT(*) |
    +---------+----------+
    | d001    |    20211 |
    | d002    |    17346 |
    | d003    |    17786 |
    | d004    |    73485 |
    | d005    |    85707 |
    | d006    |    20117 |
    | d007    |    52245 |
    | d008    |    21126 |
    | d009    |    23580 |
    | NULL    |   331603 |
    +---------+----------+
    
    # dept_no 칼럼 1개만 있으므로 1개의 소계가 조회 됩니다.
    ```
    
- `2개 컬럼 그루핑`
    
    ```sql
    SELECT first_name, last_name, COUNT(*)
    FROM employees
    GROUP BY first_name, last_name WITH ROLLUP;
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6a033fa4-359f-47c6-af03-017c798b21bb)

    
    - first_name 그룹별로 소계를 계산 및 출력하고
    - 맨 마지막에 전체 총계가 출력됩니다.
- `그룹 레코드의 NULL을 사용자가 원하는 이름으로 변경`
    
    ```sql
    SELECT          
    	IF(GROUPING(first_name), 'All first_name', first_name) AS first_name,
    	IF(GROUPING(last_name), 'All last_name', last_name) AS last_name,
    	COUNT(*)
    FROM employees
    GROUP BY first_name, last_name WITH ROLLUP;
    ```
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/efea5f64-b2f0-4795-8eee-cad4e14d7f9c)

    

---

<br>

**11.4.8.2 레코드를 칼럼으로 변환해서 조회**

- 레코드를 그루핑 할 수 있지만, 여러개의 칼럼으로 나누거나 변환하는 SQL 문법은 없습니다.
- `SUM()`, `COUNT()`같은 집합함수 사용시 `CASE WHEN … END` 구문을 이용해 레코드를 칼럼으로 변환하거나 하나의 칼럼을 조건으로 구분해 2개 이상의 칼럼으로 변환하는건 가능합니다.

1. `레코드를 칼럼으로 변환`
    
    ```sql
    SELECT dept_no, COUNT(*) AS emp_count
    FROM dept_emp
    GROUP BY dept_no;
    
    +---------+-----------+
    | dept_no | emp_count |
    +---------+-----------+
    | d001    |     20211 |
    | d002    |     17346 |
    | d003    |     17786 |
    | d004    |     73485 |
    | d005    |     85707 |
    | d006    |     20117 |
    | d007    |     52245 |
    | d008    |     21126 |
    | d009    |     23580 |
    +---------+-----------+
    ```
    
    위 결과를 하나의 레코드에서 각각의 칼럼으로 나눌 수 있습니다.
    
    ```sql
    SELECT
    	SUM(CASE WHEN dept_no='d001' THEN emp_count ELSE 0 END) AS count_d001,
    	SUM(CASE WHEN dept_no='d002' THEN emp_count ELSE 0 END) AS count_d002,
    	SUM(CASE WHEN dept_no='d003' THEN emp_count ELSE 0 END) AS count_d003,
    	SUM(CASE WHEN dept_no='d004' THEN emp_count ELSE 0 END) AS count_d004,
    	SUM(CASE WHEN dept_no='d005' THEN emp_count ELSE 0 END) AS count_d005,
    	SUM(CASE WHEN dept_no='d006' THEN emp_count ELSE 0 END) AS count_d006,
    	SUM(CASE WHEN dept_no='d007' THEN emp_count ELSE 0 END) AS count_d007,
    	SUM(CASE WHEN dept_no='d008' THEN emp_count ELSE 0 END) AS count_d008,
    	SUM(CASE WHEN dept_no='d009' THEN emp_count ELSE 0 END) AS count_d009,
    	SUM(emp_count) AS count_total
    FROM (
      SELECT dept_no, COUNT(*) AS emp_count FROM dept_emp GROUP BY dept_no
    ) tb_derived;
    
    +------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
    | count_d001 | count_d002 | count_d003 | count_d004 | count_d005 | count_d006 | count_d007 | count_d008 | count_d009 | count_total |
    +------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
    |      20211 |      17346 |      17786 |      73485 |      85707 |      20117 |      52245 |      21126 |      23580 |      331603 |
    +------------+------------+------------+------------+------------+------------+------------+------------+------------+-------------+
    ```
    
    - WHEN절에 dept_no에 해당되는 조건만 숫자를 더하고, 나머지 dept_no는 0을 더해 각각의 칼럼으로 분리할 수 있습니다.
    - 다만, 부서번호의 변경이 발생하면 쿼리 자체도 변경해야 한다는 단점도 있습니다.

1. `하나의 칼럼을 여러 칼럼으로 분리`
    
    ```sql
    SELECT de.dept_no,
    	SUM(CASE WHEN e.hire_date BETWEEN '1980-01-01' AND '1989-12-31' THEN 1 ELSE 0 END) AS cnt_1980,
    	SUM(CASE WHEN e.hire_date BETWEEN '1990-01-01' AND '1999-12-31' THEN 1 ELSE 0 END) AS cnt_1990,
    	SUM(CASE WHEN e.hire_date BETWEEN '2000-01-01' AND '2009-12-31' THEN 1 ELSE 0 END) AS cnt_2000,
    	COUNT(*) AS cnt_total
    FROM dept_emp de, employees e
    WHERE e.emp_no=de.emp_no
    GROUP BY de.dept_no;
    
    +---------+----------+----------+----------+-----------+
    | dept_no | cnt_1980 | cnt_1990 | cnt_2000 | cnt_total |
    +---------+----------+----------+----------+-----------+
    | d001    |    11038 |     9171 |        2 |     20211 |
    | d002    |     9580 |     7765 |        1 |     17346 |
    | d003    |     9714 |     8068 |        4 |     17786 |
    | d004    |    40418 |    33065 |        2 |     73485 |
    | d005    |    47007 |    38697 |        3 |     85707 |
    | d006    |    11057 |     9059 |        1 |     20117 |
    | d007    |    28673 |    23571 |        1 |     52245 |
    | d008    |    11602 |     9524 |        0 |     21126 |
    | d009    |    12979 |    10600 |        1 |     23580 |
    +---------+----------+----------+----------+-----------+
    ```
    
    - dept_no로 그루핑하되 조인한 employees 테이블의 hire_date를 기간별로 나눠서 카운팅한 칼럼을 추가합니다.
    - 결과로 80, 90, 00년대 부서별로 입사한 사원의 숫자를 보여줍니다.

---

<br>

### 11.4.9 ORDER BY

- 검색된 레코드를 어떤 순서로 정렬할지 결정합니다.
- ORDER BY를 사용하지 않으면 SELECT의 쿼리 결과 순서는 다음과 같이 정렬됩니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/054de27b-7f71-4aa2-872f-030c8cb6e575)

    
- 결국 ORDER BY절은 필수입니다. `어떤 DBMS도 ORDER BY절이 명시되지 않은 쿼리에 대해서 어떤 정렬도 보장하지 않습니다.`
- ORDER BY 절이 인덱스를 사용하지 못하면 Extra 칼럼에 Using filesort가 표시됩니다.
    - 쿼리를 수행하는 도중 MySQL 서버가 명시적으로 정렬 알고리즘을 수행했다는 의미
    - 정렬대상이 많으면 나누는데 이때 임시 저장을 디스크, 메모리에 함
    - 메모리, 디스크 중 임시저장공간으로 사용된 위치를 알고싶다면 상태값을 확인하면 됩니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/8c3febee-10f2-4a38-8693-bac8c037884f)

        
        - `Sort_merge_passes`: 메모리의 버퍼와 디스크에 저장된 레코드를 몇 번이나 병합했는지 보여줌. 해당 값이 0보다 크다면 정렬해야 할 데이터가 정렬용 버퍼보다 커서 디스크를 이용함을 의미합니다.
        - `Sort_range`: 인덱스 레인지 스캔을 통해 읽은 레코드 정렬 횟수 누적값
        - `Sort_scan`: 풀 테이블 스캔을 통해 읽은 레코드를 정렬한 횟수 누적값
        - `Sort_rows`: 정렬을 수행했던 전체 레코드 건수의 누적된 값

---

<br>

**11.4.9.1 ORDER BY 사용법 및 주의사항**

- 1개 이상의 칼럼으로 정렬 가능(정렬 순서 변경 가능)
- 정렬 대상은 칼럼명, 표현식으로 명시하거나 칼럼의 순번을 명시할 수 있습니다.
    
    `ORDER BY 2`
    
- ORDER BY에 숫자 값이 아닌 문자열 상수를 사용하면 옵티마이저가 ORDER BY절 자체를 무시합니다.
    
    ```sql
    SELECT first_name, last_name
    FROM employees
    ORDER BY "last_name";
    ```
    
    - MySQL에서 쌍따옴표는 문자열 리터럴을 표현하는 데 사용됨

---

<br>

**11.4.9.2 여러 방향으로 동시 정렬**

- `8.0 이전`: 여러 칼럼을 조합해 정렬할 때 각 칼럼의 정렬 순서가 DESC, ASC가 혼용되면 인덱스 사용 불가
- `8.0 포함, 이후`: 인덱스 생성시 DESC, ASC를 혼용하여 생성할 수 있으므로, 혼용 가능
    
    ```sql
    ALTER TABLE salaries ADD INDEX ix_salary_fromdate (salary DESC, from_date ASC);
    ```
    
- `ORDER BY salary DESC LIMIT 10;` 과 같은 쿼리의 경우 오름차순 혹은 내림차순 인덱스 하나만 있어도 최적화가 가능합니다.
    - 만약 내림차순으로만 레코드를 정렬해 사용하면 ix_salary_desc를 생성하는것이 좋습니다.

---

<br>

**11.4.9.3 함수나 표현식을 이용한 정렬**

- 하나 또는 여러 칼럼의 연산 결과를 이용해 정렬할 수 있음
- `8.0 이전`: 연산 결과를 기준으로 정렬하기 위해서는 가상 칼럼을 추가하고 인덱스를 생성하는 방법
- `8.0 포함, 이후`: 함수 기반의 인덱스를 지원함 따라서 연산의 결괏값을 기준으로 정렬하는 작업이 인덱스를 사용하도록 튜닝하는게 가능합니다.

---

<br><br>

### 11.4.10 서브쿼리

- 서브쿼리는 단위 처리별로 쿼리를 독립적으로 작성할 수 있게 합니다.
- 조인처럼 테이블을 섞지 않아 가독성도 높아집니다.
- 과거와 달리 8.0부터는 서브쿼리 처리가 많이 개선됐습니다.

SELECT, FROM, WHERE절에 사용될 수 있습니다. 그러나 사용되는 위치에 따라 쿼리 성능 영향도와 MySQL 서버의 최적화 방법은 완전히 달라집니다.

<br>

**11.4.10.1 SEELCT 절에 사용된 서브쿼리**

- 내부적으로 임시 테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들지 않으므로 적절히 인덱스를 사용할 수 있다면 크게 주의사항은 없습니다.
- SELECT 절에 서브쿼리를 사용하면 항상 칼럼과 레코드가 하나인 결과를 반환해야 합니다. (`오로지 스칼라 서브쿼리만 사용 가능`)
    - 칼럼이 2개 이상 이거나, 레코드가 2개 이상일때는 에러가 발생하지만, 1개의 칼럼에 대해 0개의 레코드가 반환되면 NULL로 처리됨
- 조인으로 처리해도 되는 쿼리를 SELECT 절의 서브쿼리를 사용할 수 있습니다만, **일반적으로 조인이 조금더 빠르기에 가능하면 조인으로 쿼리를 작성하는것이 좋습니다.**
    
    ```sql
    SELECT          
    	COUNT(
    		CONCAT(
    			e1.first_name,
    			(SELECT e2.first_name
    				FROM employees e2
    				WHERE e2.emp_no=e1.emp_no)))
    FROM employees e1;
    
    SELECT COUNT(CONCAT(e1.first_name, e2.first_name))
    FROM employees e1, employees e2
    WHERE e1.emp_no=e2.emp_no;
    ```
    

- SELECT절에 중복되는 서브쿼리가 사용되는 경우 Lateral Join 고려
    
    ```sql
    # emp_no=499999인 사원이 가장 최근에 받은 급여와 해당 날짜를 조회
    SELECT e.emp_no, e.first_name,
     (SELECT s.salary FROM salaries s
         WHERE s.emp_no=e.emp_no
         ORDER BY s.from_date DESC LIMIT 1) AS salary,
     (SELECT s.from_date FROM salaries s
         WHERE s.emp_no=e.emp_no
         ORDER BY s.from_date DESC LIMIT 1) AS salary_from_date,
     (SELECT s.to_date FROM salaries s
         WHERE s.emp_no=e.emp_no
         ORDER BY s.from_date DESC LIMIT 1) AS salary_to_date
    FROM employees e
    WHERE e.emp_no=499999;
    ```
    
    LIMIT 1조건이 있기에 단순 조인은 사용하기 어렵습니다.
    하지만, 래터럴 조인을 사용하여 중복된 서브쿼리를 삭제할 수 있습니다.
    
    ```sql
    SELECT e.emp_no, e.first_name, s2.salary, s2.from_date, s2.to_date
    FROM employees e
     INNER JOIN LATERAL (
       SELECT * FROM salaries s
       WHERE s.emp_no=e.emp_no
       ORDER BY s.from_date DESC
       LIMIT 1) s2 ON s2.emp_no=e.emp_no
    WHERE e.emp_no=499999;
    ```
    
    - 래터럴 조인의 문제점
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e9c63cfc-fe7e-4c8d-a613-f0cc97ba04da)

        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ead036dc-c3d3-445e-8e89-05348a9f6f8b)

        

---

<br>

**11.4.10.2 FROM 절에 사용된 서브쿼리**

- `이전 버전`: FROM절의 서브쿼리는 서브쿼리의 결과를 항상 임시 테이블로 저장하고 필요할때 다시 임시테이블을 읽는 방식으로 처리함
- `5.7부터`: 옵티마이저가 FROM절의 서브쿼리를 외부 쿼리로 병합하는 최적화를 수행하도록 개선됨

```sql
EXPLAIN SELECT * FROM (SELECT * FROM employees) y;

SHOW WARNINGS;

# 옵티마이저가 최적화한 쿼리
/* select#1 */ select
 `employees`.`employees`.`emp_no` AS `emp_no`,
`employees`.`employees`.`birth_date` AS `birth_date`,
`employees`.`employees`.`first_name` AS `first_name`,
`employees`.`employees`.`last_name` AS `last_name`,
`employees`.`employees`.`gender` AS `gender`,
`employees`.`employees`.`hire_date` AS `hire_date`
from `employees`.`employees`
```

- FROM 절의 서브쿼리 뿐만 아니라 FROM절에 사용된 View의 경우에도 옵티마이저는 뷰 쿼리와 외부 쿼리를 병합해서 최적화된 실행 계획을 사용합니다.

- 다음과 같은 기능이 서브쿼리에 사용되면 FROM절의 서브쿼리는 외부 쿼리로 병합되지 못합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cdd45de6-a09c-4e2d-98af-f7cb945a4a38)

    
- FROM 절의 서브쿼리가 ORDER BY 절을 가진 경우 외부 쿼리가 GROUP BY나 DISTINCT와 같은 기능을 사용하지 않았다면 서브쿼리의 정렬조건을 외부쿼리로 같이 병합합니다.
    - 반대의 경우 서브쿼리 정렬 작업이 무의미해지므로 서브쿼리의 ORDER BY절은 무시됩니다.
- FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화는 `optimizer_switch` 시스템 변수로 제어할 수 있습니다. (9.3.1.16절 참고)

---

<br>

**11.4.10.3 WHERE 절에 사용된 서브쿼리**

- 크게 3가지로 구분됩니다.
    1. 동등, 크다, 작다 비교(`= (subquery)`)
    2. IN 비교(`IN (subquery)`)
    3. NOT IN 비교(`NOT IN (subquery)`)

1. `동등 또는 크다 작다 비교`
    
    ```sql
    SELECT * FROM dept_emp de
    WHERE de.emp_no=(SELECT e.emp_no
                     FROM employees e
                     WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
    
    +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
    | id | select_type | table | partitions | type | possible_keys                      | key                   | key_len | ref         | rows | filtered | Extra       |
    +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
    |  1 | PRIMARY     | de    | NULL       | ref  | ix_empno_fromdate                  | ix_empno_fromdate     | 4       | const       |    1 |   100.00 | Using where |
    |  2 | SUBQUERY    | e     | NULL       | ref  | ix_firstname,ix_lastname_firstname | ix_lastname_firstname | 124     | const,const |    2 |   100.00 | Using index |
    +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
    ```
    
    - `5.5 이전`: 서브쿼리 외부의 조건으로 쿼리를 실행하고, 최종적으로 서브쿼리를 체크 조건으로 사용했음 → 풀 테이블 스캔이 필요한 경우가 많아 성능저하가 컸음
    - `5.5 부터`: 정반대로 실행함, 서브쿼리를 먼저 실행한 후 상수로 변환하고 상숫값으로 서브쿼리를 대체해 나머지 쿼리를  처리합니다.
        
        ```sql
        # 위 쿼리의 실행계획
        +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
        | id | select_type | table | partitions | type | possible_keys                      | key                   | key_len | ref         | rows | filtered | Extra       |
        +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
        |  1 | PRIMARY     | de    | NULL       | ref  | ix_empno_fromdate                  | ix_empno_fromdate     | 4       | const       |    1 |   100.00 | Using where |
        |  2 | SUBQUERY    | e     | NULL       | ref  | ix_firstname,ix_lastname_firstname | ix_lastname_firstname | 124     | const,const |    2 |   100.00 | Using index |
        +----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
        
        # dept_emp 테이블을 풀스캔하지 않고 (emp_no, from_date) 조합의 인덱스를 사용함
        # -> 서브쿼리의 우선 실행을 해야 사용가능함을 예측 가능
        
        # FORMAT=TREE 실행계획
        +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
        | EXPLAIN                                                                                                                                                                                                                                                                                                                                                                                            |
        +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
        | -> Filter: (de.emp_no = (select #2))  (cost=0.35 rows=1)
            -> Index lookup on de using ix_empno_fromdate (emp_no=(select #2))  (cost=0.35 rows=1)
            -> Select #2 (subquery in condition; run only once)
                -> Limit: 1 row(s)  (cost=0.454 rows=1)
                    -> Covering index lookup on e using ix_lastname_firstname (last_name='Facello', first_name='Georgi')  (cost=0.454 rows=2)
         |
        +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
        
        # index lookup on e using ix_lastname_firstname를 통해 서브쿼리를 먼저 처리함을 알 수 있습니다.
        ```
        
        - 이런 결과는 동등, 크다, 작다 비교 모두에서 동일한 실행계획을 사용합니다.
    
    - 반면에 튜플 비교의 경우 서브쿼리가 먼저 상수화되긴 하지만 외부 쿼리에선 인덱스를 사용하지 못하고 풀 테이블 스캔을 합니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/17fef515-183f-4b90-b3a8-09522facbb81)

        

1. `IN 비교 (IN (subquery))`
    - 테이블의 레코드가 다른 테이블의 레코드를 이용한 표현식(or 칼럼)과 일치하는지 체크하는 형태를 세미 조인이라고 합니다.
    - 즉, WHERE절에 사용된 IN (subquery) 형태의 조건을 조인의 한 방시긴 세미 조인이라고 봅니다.
    
    ```sql
    SELECT *
    FROM employees e
    WHERE e.emp_no IN
    	(SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
    ```
    
    - `5.5버전까지`: 최적화가 부족했기에 풀 테이블 스캔 사용
    - `5.6 ~ 8.0까지의 최적화 개선`: IN 형태를 2개의 쿼리로 쪼개거나 우회 방법을 찾을 필요가 없습니다.
    - 세미 조인 최적화는 쿼리 특성이나 조인 관계에 맞게 5개의 최적화 전략을 선택적으로 사용합니다.
        - `Table Pull-out`
        - `Firstmatch`
        - `Loosescan`
        - `Materialization`
        - `Duplicated Weed-out`
        
        → 9.3.1.9절을 참조하면 자세히 확인가능
        

1. `NOT IN 비교(NOT IN (subquery))`
    - IN과 비슷하지만 안티 세미 조인이라고 합니다.
    - RDB에서 Not-Equal 비교는 인덱스를 제대로 활용하기 어렵듯, 안티 세미 조인도 최적화할 수 있는 방법이 많지 않습니다.
    - 안티 세미 조인 쿼리는 두 가지 방법으로 최적화를 수행합니다.
        1. `NOT EXISTS`
        2. `Materialization`
        
        → 두 방법 모두 성능 향상에 큰 도움이 되진 않습니다. 만약 WHERE절에 단독 안티 세미 조인 조건만 있다면 풀 테이블 스캔을 피하기 어렵습니다.
        

---

<br><br>

### 11.4.11 CTE(Common Table Expression)

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/51b90eb3-c619-466d-a7d5-edb3db70e07a)


- CTE는 이름을 가지는 임시테이블
- SQL 문장 내에서 한 번 이상 사용될 수 있음
    - SQL문장이 종료되면 자동으로 CTE 임시테이블은 삭제됩니다.
- 재귀적 반복 실행 여부를 기준으로 Non-recursive와 Recursive CTE로 구분됨

---

<br>

**11.4.11.1 비 재귀적 CTE(Non-Recursive CTE)**

- ANSI 표준을 그대로 이용해 WITH 절을 통해 CTE를 정의합니다.
    
    ```sql
    WITH cte1 AS (SELECT * FROM departments)
    SELECT * FROM cte1;
    
    # 혹은 FROM절의 서브쿼리로 사용할 수 있음
    SELECT * FROM (SELECT * FROM departments) cte1;
    
    # 두 쿼리의 실행 계획은 동일합니다.
    # CTE
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    | id | select_type | table       | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    |  1 | SIMPLE      | departments | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |   100.00 | NULL  |
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    # Subquery
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    | id | select_type | table       | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    |  1 | SIMPLE      | departments | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |   100.00 | NULL  |
    +----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+-------+
    ```
    
- CTE 쿼리는 WITH절로 정의함, 이름은 WITH 바로 뒤에 사용자가 원하는 이름으로 정의할 수 있습니다.
- 여러 개의 임시 테이블을 하나의 쿼리에서 사용할 수 있습니다.
    
    ```sql
    WITH cte1 AS (
    	SELECT * FROM departments
    ),
    cte2 AS (
    	SELECT * FROM dept_emp
    )
    SELECT * 
    FROM temp1
    INNER JOIN cte2 ON cte2.dept_no=cte1.dept_no;
    ```
    
- 임시 테이블이 여러번 사용되는 쿼리를 CTE와 FROM절의 서브쿼리로 작성하면 실행계획이 달라짐
    
    ```sql
    # CTE 사용
    EXPLAIN
    WITH cte1 AS (
    	SELECT emp_no, MIN(from_date)
    	FROM salaries
    	GROUP BY emp_no
    )
    SELECT * FROM employees e
    INNER JOIN cte1 t1 ON t1.emp_no=e.emp_no
    INNER JOIN cte1 t2 ON t2.emp_no=e.emp_no;
    
    # cte1이 파생테이블로 메모리 혹은 디스크에 임시테이블을 생성합니다.
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    | id | select_type | table      | partitions | type   | possible_keys     | key         | key_len | ref       | rows   | filtered | Extra                    |
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    |  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL        | NULL    | NULL      | 294784 |   100.00 | NULL                     |
    |  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY     | 4       | t1.emp_no |      1 |   100.00 | NULL                     |
    |  1 | PRIMARY     | <derived2> | NULL       | ref    | <auto_key0>       | <auto_key0> | 4       | t1.emp_no |     10 |   100.00 | NULL                     |
    |  2 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 294784 |   100.00 | Using index for group-by |
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    
    # 서브쿼리 사용
    EXPLAIN
    SELECT * FROM employees e
    	INNER JOIN (SELECT emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t1
    		ON t1.emp_no=e.emp_no
    	INNER JOIN (SELECT emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t2
    		ON t2.emp_no=e.emp_no;
    
    # 동일한 서브쿼리지만, 두 테이블 모두 별도의 파생테이블로 메모리 혹은 디스크에 저장됩니다.
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    | id | select_type | table      | partitions | type   | possible_keys     | key         | key_len | ref       | rows   | filtered | Extra                    |
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    |  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL        | NULL    | NULL      | 294784 |   100.00 | NULL                     |
    |  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY     | 4       | t1.emp_no |      1 |   100.00 | NULL                     |
    |  1 | PRIMARY     | <derived3> | NULL       | ref    | <auto_key0>       | <auto_key0> | 4       | t1.emp_no |     10 |   100.00 | NULL                     |
    |  3 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 294784 |   100.00 | Using index for group-by |
    |  2 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 294784 |   100.00 | Using index for group-by |
    +----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
    ```
    
- CTE로 생성된 임시테이블은 다른 CTE 쿼리에서 참조할 수 있습니다.
    
    ```sql
    	WITH
    	cte1 AS (
    		SELECT emp_no, MIN(from_date) as salary_from_date
    		FROM salaries
    		WHERE salary BETWEEN 50000 AND 51000
    		GROUP BY emp_no
    	), cte2 AS (
    		SELECT de.emp_no, MIN(from_date) as dept_from_date
    		# 직전에 정의한 cte1 테이블과 조인함
    		FROM cte1
    		INNER JOIN dept_emp de ON de.emp_no=cte1.emp_no
    		GROUP BY emp_no)
    	
    	SELECT * 
    	FROM employees e 
    	INNER JOIN cte1 t1 ON t1.emp_no=e.emp_no
    	INNER JOIN cte2 t2 ON t2.emp_no=e.emp_no;
    ```
    
    - 즉, WITH절에 정의된 순서대로 CTE 임시테이블을 재사용할 수 있습니다.
    따라서 cte1에서 cte2를 참조할 수 없습니다.
- Non-Recursive CTE는 기존 FROM절에 사용되던 서브쿼리에 비해 3가지 장점이 존재합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6ffd176f-99bc-47e4-8106-01f9721ea7e5)

    

---

<br>

**11.4.11.2 재귀적 CTE(Recursive CTE)**

```sql
WITH RECURSIVE cte (no) AS (
	SELECT 1 # 비 재귀적 파트
	UNION ALL
	SELECT (no + 1) FROM cte WHERE no < 5 # 재귀적 파트
)
SELECT * FROM cte;
```

- **재귀적 CTE 쿼리는 비 재귀적 쿼리 파트와 재귀적 파트로 구분됩니다.**
- **그리고 반드시 이 둘을 연결하는 UNION 혹은 UNION ALL로 연결하는 형태로 쿼리를 작성해야 합니다.**
- **비재귀적 파트는 딱 한번만 호출하고, 재귀적 파트는 쿼리 결과가 없을때 까지 반복 실행합니다.**
- 실행 순서
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/695d0f0b-8ac2-4b52-9d51-bdfe0e5fd872)

    
    - 1번 과정에서 CTE 임시 테이블의 구조가 결정됨 (테이블의 칼럼명, 칼럼의 데이터 타입)
        - 비 재귀적, 재귀적 파트 결과의 칼럼 개수, 타입, 이름이 다르면 MySQL 서버는 비 재귀적 파트에 정의된 결과를 사용
    - **비 재귀적 파트에서 임시 테이블의 구조를 정의하고, 재귀적 파트에서 데이터를 생성하는 역할**
    - 재귀적 파트에선 직전 단계의 결과만 재귀 쿼리의 입력으로 사용함
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/0e31c10c-d591-4d9a-a00d-b4e714f00b58)

        
    - **재귀 쿼리가 반복을 멈추는 조건은 재귀 파트 쿼리의 결과가 0건일 때까지 입니다.**
- 참고: CTE의 컬럼명을 변경하기 (재귀, 비재귀 모두 포함됨)
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/89dd45fd-74ac-4594-bc2a-b79931c32606)

    

- 재귀 종료조건을 만족하지 못해 무한 반복하게 되면
    - MySQL서버의 cte_max_recursion_depth 시스템 변수를 통해 최대 반복 실행 횟수를 제한할 수 있음
    - 기본값은 1000인데 단순히 번호를 붙이는 쿼리를 제외하곤 큰 값이므로 적절한 낮은 값으로 설정하는걸 권장합니다.
    - 만약 많은 반복이 필요하면 SET_VAR 힌트를 통해 해당 쿼리에만 횟수를 늘리는걸 권장함

---

<br>

**11.4.11.3 재귀적 CTE(Recursive CTE) 활용**

- 아래와 같은 테이블이 있을때
    
    ```sql
    +------------+--------------+------+-----+---------+-------+
    | Field      | Type         | Null | Key | Default | Extra |
    +------------+--------------+------+-----+---------+-------+
    | id         | int          | NO   | PRI | NULL    |       |
    | name       | varchar(100) | NO   |     | NULL    |       |
    | manager_id | int          | YES  | MUL | NULL    |       |
    +------------+--------------+------+-----+---------+-------+
    
    select * from employees;
    +------+---------+------------+
    | id   | name    | manager_id |
    +------+---------+------------+
    |   29 | Pedro   |        198 |
    |   72 | Pierre  |         29 |
    |  123 | Adil    |        692 |
    |  198 | John    |        333 |
    |  333 | Yasmina |       NULL |
    |  692 | Tarek   |        333 |
    | 4610 | Sarah   |         29 |
    +------+---------+------------+
    ```
    
- Adil의 상위 조직장을 찾는 쿼리는 다음과 같습니다.
    
    ```sql
    WITH RECURSIVE managers AS (
    	SELECT *, 1 AS lv FROM employees where id=123 # 123 adil 333 1
    	UNION ALL
    	SELECT e.*, lv + 1 FROM managers m
    			JOIN employees e on e.id=m.manager_id AND m.manager_id IS NOT NULL
    )
    SELECT * FROM managers
    ORDER BY lv DESC;
    ```
    
    - CTE인 managers와 employees 테이블의 조인을 하여 재귀적 쿼리 파트를 통해 최상위의 조직장까지 거치는 사람들 및 레벨을 출력합니다.
- 최상위 조직장들부터 시작하여 상위 조직장들이 순서대로 나열된 칼럼을 만드는 쿼리
    
    ```sql
    WITH RECURSIVE managers AS (
    		SELECT *, CAST(id AS CHAR(100)) AS manager_path, 1 as lv
    		FROM employees WHERE manager_id IS NULL
    	UNION ALL
    		SELECT e.*, CONCAT(e.id, ' -> ', m.manager_path) AS manager_path, lv + 1
    		FROM managers m JOIN employees e on e.manager_id = m.id
    )
    SELECT * FROM managers
    ORDER BY lv ASC;
    
    +------+---------+------------+--------------------------+------+
    | id   | name    | manager_id | manager_path             | lv   |
    +------+---------+------------+--------------------------+------+
    |  333 | Yasmina |       NULL | 333                      |    1 |
    |  198 | John    |        333 | 198 -> 333               |    2 |
    |  692 | Tarek   |        333 | 692 -> 333               |    2 |
    |   29 | Pedro   |        198 | 29 -> 198 -> 333         |    3 |
    |  123 | Adil    |        692 | 123 -> 692 -> 333        |    3 |
    |   72 | Pierre  |         29 | 72 -> 29 -> 198 -> 333   |    4 |
    | 4610 | Sarah   |         29 | 4610 -> 29 -> 198 -> 333 |    4 |
    +------+---------+------------+--------------------------+------+
    ```
    
- 재귀적 쿼리를 활용할 수 있는 업무 요건은 상당히 많으니 잘 알아두면 좋습니다~~

---

<br><br>

### 11.4.12 윈도우 함수(Window Function)

- 조회하는 현재 레코드를 기준으로 연관된 레코드 집합의 연산을 수행하는 함수를 말합니다.
- **집계 함수는 주어진 그룹별로 하나의 레코드로 묶어서 출력하지만, 윈도우 함수는 조건에 일치하는 레코드 건수는 변하지 않고 그대로 유지하는것에 큰 차이가 있습니다.**
- SQL 문장에서 하나의 레코드를 연산할때 다른 레코드의 값을 참조할 수 없으나
- GROUP BY나 집계함수를 사용하면 다른 레코드의 칼럼값을 참조할 수 있음 → 결과 집합의 모양이 바뀜
- 결과 집합을 그대로 유지하며 하나의 레코드 연산에 다른 레코드의 칼럼값을 참조하려면 윈도우 함수를 이용합니다.

---

<br>

**11.4.12.1 쿼리 각 절의 실행 순서**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/47d0609b-a85f-48f5-a65c-64f6b890c326)


- **윈도우 함수를 사용하는 쿼리의 결과에 보여지는 레코드는 FROM, WHERE, GROUP BY and HAVING절에 의해 결정되고 이후 윈도우 함수가 실행됩니다.**
- **그리고 SELECT, ORDER BY, LIMIT 절을 통해 최종 결과가 반환됩니다.**

→ 위 순서를 벗어나고 싶다면 FROM 절의 서브쿼리를 사용해야 합니다.

- FROM절의 서브쿼리에서 LIMIT 5와 서브쿼리 없이 LIMIT 5를 한 것의 차이
    
    ```sql
    SELECT 
    	emp_no, 
    	from_date, 
    	salary,
    	AVG(salary) OVER() AS avg_salary 
    FROM salaries     
    WHERE emp_no=10001     
    LIMIT 5;
    
    # LIMIT 이전에 윈도우 함수가 호출되므로 emp_no=10001에 해당되는
    # 모든 레코드에 대한 평균을 낸 값임 (17건)
    +--------+------------+--------+------------+
    | emp_no | from_date  | salary | avg_salary |
    +--------+------------+--------+------------+
    |  10001 | 1986-06-26 |  60117 | 75388.9412 |
    |  10001 | 1987-06-26 |  62102 | 75388.9412 |
    |  10001 | 1988-06-25 |  66074 | 75388.9412 |
    |  10001 | 1989-06-25 |  66596 | 75388.9412 |
    |  10001 | 1990-06-25 |  66961 | 75388.9412 |
    +--------+------------+--------+------------+
    
    SELECT
    	emp_no,
    	from_date,
    	salary,
    	AVG(salary) OVER() AS avg_salary
    FROM (SELECT * FROM salaries WHERE emp_no=10001 LIMIT 5) s2;
    
    # 10001인 레코드 5개에 대한 평균을 낸 값 (기대했던 결과)
    +--------+------------+--------+------------+
    | emp_no | from_date  | salary | avg_salary |
    +--------+------------+--------+------------+
    |  10001 | 1986-06-26 |  60117 | 64370.0000 |
    |  10001 | 1987-06-26 |  62102 | 64370.0000 |
    |  10001 | 1988-06-25 |  66074 | 64370.0000 |
    |  10001 | 1989-06-25 |  66596 | 64370.0000 |
    |  10001 | 1990-06-25 |  66961 | 64370.0000 |
    +--------+------------+--------+------------+
    ```
    

---

<br>

**11.4.12.2 윈도우 함수 기본 사용법**

```sql
AGGREGATE_FUNC() OVER(<partition> <order>) AS window_func_coloumn
```

- 용도별로 다양한 함수를 사용할 수 있음
- 집계 함수와 달리 함수 뒤에 OVER절을 이용해 연산 대상을 파티션하기 위한 옵션을 명시할 수 있습니다.
- OVER절에 의해 만들어진 그룹을 파티션 또는 윈도우라고 합니다.

- `직원들의 입사 순서를 조회하는 쿼리`
    
    ```sql
    SELECT 
    	e.*, 
    	RANK() OVER(ORDER BY e.hire_date) AS hire_date_rank
    FROM employees e;
    ```
    
    `RANK() OVER(ORDER BY e.hire_date)` 는 소그룹의 구분없이 전체 결과 집합에서 지정된 칼럼으로 정렬후 순위를 매깁니다.
    
- `부서별 입사 순위 매기기`
    
    ```sql
    SELECT 
    	de.dept_no, 
    	e.emp_no, 
    	e.first_name, 
    	e.hire_date,
    	RANK() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
    FROM employees e JOIN dept_emp de ON de.emp_no = e.emp_no
    ORDER BY de.dept_no, e.hire_date;
    ```
    
- `salaries에서 emp_no=10001인 사원의 모든 급여 이력과 평균 급여`(정렬이나 파티셔닝이 필요 없음)
    
    ```sql
    SELECT 
    	emp_no,
    	from_date,
    	salary,
    	AVG(salary) OVER() AS avg_salary
    FROM salaries
    WHERE emp_no=10001;
    
    +--------+------------+--------+------------+
    | emp_no | from_date  | salary | avg_salary |
    +--------+------------+--------+------------+
    |  10001 | 1986-06-26 |  60117 | 75388.9412 |
    |  10001 | 1987-06-26 |  62102 | 75388.9412 |
    |  10001 | 1988-06-25 |  66074 | 75388.9412 |
    |  10001 | 1989-06-25 |  66596 | 75388.9412 |
    |  10001 | 1990-06-25 |  66961 | 75388.9412 |
    |  10001 | 1991-06-25 |  71046 | 75388.9412 |
    |  10001 | 1992-06-24 |  74333 | 75388.9412 |
    |  10001 | 1993-06-24 |  75286 | 75388.9412 |
    |  10001 | 1994-06-24 |  75994 | 75388.9412 |
    |  10001 | 1995-06-24 |  76884 | 75388.9412 |
    |  10001 | 1996-06-23 |  80013 | 75388.9412 |
    |  10001 | 1997-06-23 |  81025 | 75388.9412 |
    |  10001 | 1998-06-23 |  81097 | 75388.9412 |
    |  10001 | 1999-06-23 |  84917 | 75388.9412 |
    |  10001 | 2000-06-22 |  85112 | 75388.9412 |
    |  10001 | 2001-06-22 |  85097 | 75388.9412 |
    |  10001 | 2002-06-22 |  88958 | 75388.9412 |
    +--------+------------+--------+------------+
    ```
    

윈도우 함수의 각 파티션 안에서 연산 대상 레코드별로 연산을 수행할 소그룹이 사용되는데 이를 `프레임`이라고 합니다. 명시적으로 지정하지 않아도 MySQL 서버는 상황에 맞게 프레임을 묵시적으로 선택합니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6380e003-7eaa-442c-8173-ce24c0af9876)


![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/05161f82-3c5b-4a90-846b-28f1aaf91639)


- 프레임을 만드는 기준으로 ROWS, RANGE 중 하나를 선택
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a674108c-a984-49b1-92d3-8dcf3ff9604e)

    
- 프레임의 시작과 끝을 의미하는 키워드들의 의미
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/14ef6871-40ae-4830-a310-1e9886c3f8e5)

    
- 프레임이 ROWS면 expr에는 레코드의 위치, RANGE라면 expr에는 칼럼과 비교할 값이 설정돼야 합니다.  프레임의 시작과 끝이 expr을 가지는 경우 다음 예시와 같이 사용될 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3db8167f-f5b0-4ac9-b599-628179314a9f)

    
- `OVER()내 frame 예제`
    - `ROWS UNBOUNDED PRECEDING`: 파티션의 첫 번째 레코드로부터 현재 레코드까지
    - `ROWS BETWEEN UNBOUND PRECEDING AND CURRENT ROW`: 파티션의 첫 번째 레코드로부터 현재 레코드까지("ROWS UNBOUNDED PRECEDING"와 동일함)
    - `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`: 파티션에서 현재 레코드를 기준으로 앞 레코드부터 뒤 레코드까지
    - `RANGE INTERVAL 5 DAY PRECEDING`: ORDER BY에 명시된 칼럼의 값이 5일 전인 레코드부터 현재 레코드까지
    - `RANGE BETWEEN 1 DAY PRECEDING AND 1 DAY FOLLOWING`: ORDER BY에 명시된 칼럼의 값이 1일 전인 레코드부터 1일 이후인 레코드까지
    
    ```sql
    # 프레임 쉽지 않다..
    SELECT emp_no, from_date, salary,
    
    -- // 현재 레코드의 from_date를 기준으로 1년 전부터 지금까지 급여 중 최소 급여
    MIN(salary) OVER(ORDER BY from_date RANGE INTERVAL 1 YEAR PRECEDING) AS min_1,
    
    -- // 현재 레코드의 from_date를 기준으로 1년 전부터 2년 후까지의 급여 중 최대 급여
    MAX(salary) OVER(ORDER BY from_date
    	RANGE BETWEEN INTERVAL 1 YEAR PRECEDING AND INTERVAL 2 YEAR FOLLOWING) AS max_1,
    
    -- // from_date 칼럼으로 정렬 후, 첫 번째 레코드부터 현재 레코드까지의 평균
    AVG(salary) OVER(ORDER BY from_date ROWS UNBOUNDED PRECEDING) AS avg_1,
    
    -- // from_date 칼럼으로 정렬 후, 현재 레코드를 기준으로 이전 1건부터 이후 1건 레코드까지의 급여 평균
    AVG(salary) OVER(ORDER BY from_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS avg_2
    
    FROM salaries
    WHERE emp_no=10001;
    
    +--------+------------+--------+-------+-------+------------+------------+
    | emp_no | from_date  | salary | min_1 | max_1 | avg_1      | avg_2      |
    +--------+------------+--------+-------+-------+------------+------------+
    |  10001 | 1986-06-26 |  60117 | 60117 | 66074 | 60117.0000 | 61109.5000 |
    |  10001 | 1987-06-26 |  62102 | 60117 | 66596 | 61109.5000 | 62764.3333 |
    |  10001 | 1988-06-25 |  66074 | 62102 | 66961 | 62764.3333 | 64924.0000 |
    |  10001 | 1989-06-25 |  66596 | 66074 | 71046 | 63722.2500 | 66543.6667 |
    |  10001 | 1990-06-25 |  66961 | 66596 | 74333 | 64370.0000 | 68201.0000 |
    |  10001 | 1991-06-25 |  71046 | 66961 | 75286 | 65482.6667 | 70780.0000 |
    |  10001 | 1992-06-24 |  74333 | 71046 | 75994 | 66747.0000 | 73555.0000 |
    |  10001 | 1993-06-24 |  75286 | 74333 | 76884 | 67814.3750 | 75204.3333 |
    |  10001 | 1994-06-24 |  75994 | 75286 | 80013 | 68723.2222 | 76054.6667 |
    |  10001 | 1995-06-24 |  76884 | 75994 | 81025 | 69539.3000 | 77630.3333 |
    |  10001 | 1996-06-23 |  80013 | 76884 | 81097 | 70491.4545 | 79307.3333 |
    |  10001 | 1997-06-23 |  81025 | 80013 | 84917 | 71369.2500 | 80711.6667 |
    |  10001 | 1998-06-23 |  81097 | 81025 | 85112 | 72117.5385 | 82346.3333 |
    |  10001 | 1999-06-23 |  84917 | 81097 | 85112 | 73031.7857 | 83708.6667 |
    |  10001 | 2000-06-22 |  85112 | 84917 | 88958 | 73837.1333 | 85042.0000 |
    |  10001 | 2001-06-22 |  85097 | 85097 | 88958 | 74540.8750 | 86389.0000 |
    |  10001 | 2002-06-22 |  88958 | 85097 | 88958 | 75388.9412 | 87027.5000 |
    +--------+------------+--------+-------+-------+------------+------------+
    ```
    

**주의**

- 프레임이 별도로 명시되지 않으면 무조건 파티션의 모든 레코드가 연산의 대상이 되진 않습니다.
- OVER()의 ORDER BY여부에 따라 묵시적인 프레임 범위가 결정되는데
    - ORDER BY가 있는 경우
        - 파티션의 첫번째 ~ 현재 레코드까지 프레임으로 결정됨
        - `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
    - 없는 경우
        - 파티션의 모든 레코드가 묵시적인 프레임으로 선택됨
        - `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`
- 일부 윈도우 함수들은 프레임이 고정돼 있습니다. (사용자가 작성해도 무시됨) 다음 윈도우 함수들은 프레임이 파티션의 전체 레코드로 설정됩니다.
    
    ```sql
    CUM_DIST()
    DENSE_RANK()
    LAG()
    LEAD()
    NTILE()
    PERCENT_RANK()
    RANK()
    ROW_NUMBER()
    ```
    

---

<br>

**11.4.12.3 윈도우 함수**

- 윈도우 함수는 집계, 비집계 함수 모두 사용가능
    - 집계는 GROUP BY 절과 함께 사용할 수 있는 함수들로, OVER()절 없이도 단독 사용 가능
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/502fbd29-5641-40fa-98f6-4c5cddfbfe1b)

        
    - 비 집계는 반드시 OVER()절을 가지고 있어야 하고, 윈도우 함수로만 사용 가능
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/925f34fc-675f-4cba-8903-c7344e2d89df)

        

- `DENSE_RANK()와 RANK(), ROW_NUMBER()`
    - 모두 ORDER BY 기준으로 매겨전 순위를 반환합니다.
    - `RANK()`
        - 동점인 레코드가 두 건 이상이라면 그 다음 레코드를 동점인 레코드 수만큼 증가시킨 순위 반환
    - `DENSE_RANK()`
        - 동점인 레코드를 1건으로 가정하고 순위를 매기기에 연속된 순위를 가집니다.
    - `ROW_NUMBER()`
        - 똑같이 순위를 매기지만 이름 그대로 각 레코드의 고유한 순번을 반환합니다. 
        (동점에 대한 고려가 없이 정렬된 순서대로 레코드 번호 부여)
    
    ```sql
    # RANK() 사용
    SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date,
    	      RANK() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
    FROM employees e
    	INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    WHERE de.dept_no='d001'
    ORDER BY de.dept_no, e.hire_date
    LIMIT 20;
    +---------+--------+------------+------------+----------------+
    | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
    +---------+--------+------------+------------+----------------+
    | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
    | d001    |  98351 | Florina    | 1985-02-02 |              2 |
    | d001    | 456487 | Jouko      | 1985-02-02 |              2 |
    | d001    | 481016 | Toney      | 1985-02-02 |              2 |
    | d001    | 491200 | Ortrun     | 1985-02-02 |              2 |
    | d001    |  51773 | Eric       | 1985-02-02 |              2 |
    | d001    |  65515 | Phillip    | 1985-02-02 |              2 |
    | d001    |  95867 | Shakhar    | 1985-02-02 |              2 |
    | d001    | 288310 | Mohammed   | 1985-02-02 |              2 |
    | d001    | 288790 | Cristinel  | 1985-02-02 |              2 |
    | d001    | 430759 | Fumiko     | 1985-02-02 |              2 |
    | d001    | 447306 | Pranav     | 1985-02-03 |             12 |
    | d001    | 480861 | Cedric     | 1985-02-03 |             12 |
    | d001    |  43165 | Zhiwei     | 1985-02-03 |             12 |
    | d001    |  64398 | Nectarios  | 1985-02-03 |             12 |
    | d001    |  70562 | Morris     | 1985-02-03 |             12 |
    | d001    |  84982 | Gino       | 1985-02-03 |             12 |
    | d001    |  85622 | Berthier   | 1985-02-04 |             18 |
    | d001    | 277990 | Jianwen    | 1985-02-04 |             18 |
    | d001    | 410831 | Xiaobin    | 1985-02-04 |             18 |
    +---------+--------+------------+------------+----------------+
    
    # DENSE_RANK() 사용
    SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date,
    				DENSE_RANK() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
    FROM employees e
    	INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    WHERE de.dept_no='d001'
    ORDER BY de.dept_no, e.hire_date
    LIMIT 20;
    +---------+--------+------------+------------+----------------+
    | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
    +---------+--------+------------+------------+----------------+
    | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
    | d001    |  98351 | Florina    | 1985-02-02 |              2 |
    | d001    | 456487 | Jouko      | 1985-02-02 |              2 |
    | d001    | 481016 | Toney      | 1985-02-02 |              2 |
    | d001    | 491200 | Ortrun     | 1985-02-02 |              2 |
    | d001    |  51773 | Eric       | 1985-02-02 |              2 |
    | d001    |  65515 | Phillip    | 1985-02-02 |              2 |
    | d001    |  95867 | Shakhar    | 1985-02-02 |              2 |
    | d001    | 288310 | Mohammed   | 1985-02-02 |              2 |
    | d001    | 288790 | Cristinel  | 1985-02-02 |              2 |
    | d001    | 430759 | Fumiko     | 1985-02-02 |              2 |
    | d001    | 447306 | Pranav     | 1985-02-03 |              3 |
    | d001    | 480861 | Cedric     | 1985-02-03 |              3 |
    | d001    |  43165 | Zhiwei     | 1985-02-03 |              3 |
    | d001    |  64398 | Nectarios  | 1985-02-03 |              3 |
    | d001    |  70562 | Morris     | 1985-02-03 |              3 |
    | d001    |  84982 | Gino       | 1985-02-03 |              3 |
    | d001    |  85622 | Berthier   | 1985-02-04 |              4 |
    | d001    | 277990 | Jianwen    | 1985-02-04 |              4 |
    | d001    | 410831 | Xiaobin    | 1985-02-04 |              4 |
    +---------+--------+------------+------------+----------------+
    
    # ROW_NUMBER() 사용
    SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date,
    				ROW_NUMBER() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
    FROM employees e
    	INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    WHERE de.dept_no='d001'
    ORDER BY de.dept_no, e.hire_date
    LIMIT 20;
    +---------+--------+------------+------------+----------------+
    | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
    +---------+--------+------------+------------+----------------+
    | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
    | d001    |  98351 | Florina    | 1985-02-02 |              2 |
    | d001    | 456487 | Jouko      | 1985-02-02 |              3 |
    | d001    | 481016 | Toney      | 1985-02-02 |              4 |
    | d001    | 491200 | Ortrun     | 1985-02-02 |              5 |
    | d001    |  51773 | Eric       | 1985-02-02 |              6 |
    | d001    |  65515 | Phillip    | 1985-02-02 |              7 |
    | d001    |  95867 | Shakhar    | 1985-02-02 |              8 |
    | d001    | 288310 | Mohammed   | 1985-02-02 |              9 |
    | d001    | 288790 | Cristinel  | 1985-02-02 |             10 |
    | d001    | 430759 | Fumiko     | 1985-02-02 |             11 |
    | d001    | 447306 | Pranav     | 1985-02-03 |             12 |
    | d001    | 480861 | Cedric     | 1985-02-03 |             13 |
    | d001    |  43165 | Zhiwei     | 1985-02-03 |             14 |
    | d001    |  64398 | Nectarios  | 1985-02-03 |             15 |
    | d001    |  70562 | Morris     | 1985-02-03 |             16 |
    | d001    |  84982 | Gino       | 1985-02-03 |             17 |
    | d001    |  85622 | Berthier   | 1985-02-04 |             18 |
    | d001    | 277990 | Jianwen    | 1985-02-04 |             19 |
    | d001    | 410831 | Xiaobin    | 1985-02-04 |             20 |
    +---------+--------+------------+------------+----------------+
    ```
    

- `LAG()와 LEAD()`
    - `LAG()`
        - 파티션 내에서 현재 레코드를 기준으로 n번째 이전 레코드를 반환
    - `LEAD()`
        - 파티션 내에서 현재 레코드를 기준으로 n번째 이후 레코드를 반환
    - 모두 3개의 파라미터가 필요하며 두번째 까지는 필수, 세번째는 선택
    
    ```sql
    SELECT from_date, salary,
    		  LAG(salary, 5) OVER(ORDER BY from_date) AS prior_5th_value,
    		  LEAD(salary, 5) OVER(ORDER BY from_date) AS next_5th_value,
    			# 3번째 인자는 default 값
    		  LAG(salary, 5, -1) OVER(ORDER BY from_date) AS prior_5th_with_default,
    		  LEAD(salary, 5, -1) OVER(ORDER BY from_date) AS next_5th_with_default
    FROM salaries
    WHERE emp_no=10001;
    
    +------------+--------+-----------------+----------------+------------------------+-----------------------+
    | from_date  | salary | prior_5th_value | next_5th_value | prior_5th_with_default | next_5th_with_default |
    +------------+--------+-----------------+----------------+------------------------+-----------------------+
    | 1986-06-26 |  60117 |            NULL |          71046 |                     -1 |                 71046 |
    | 1987-06-26 |  62102 |            NULL |          74333 |                     -1 |                 74333 |
    | 1988-06-25 |  66074 |            NULL |          75286 |                     -1 |                 75286 |
    | 1989-06-25 |  66596 |            NULL |          75994 |                     -1 |                 75994 |
    | 1990-06-25 |  66961 |            NULL |          76884 |                     -1 |                 76884 |
    | 1991-06-25 |  71046 |           60117 |          80013 |                  60117 |                 80013 |
    | 1992-06-24 |  74333 |           62102 |          81025 |                  62102 |                 81025 |
    | 1993-06-24 |  75286 |           66074 |          81097 |                  66074 |                 81097 |
    | 1994-06-24 |  75994 |           66596 |          84917 |                  66596 |                 84917 |
    | 1995-06-24 |  76884 |           66961 |          85112 |                  66961 |                 85112 |
    | 1996-06-23 |  80013 |           71046 |          85097 |                  71046 |                 85097 |
    | 1997-06-23 |  81025 |           74333 |          88958 |                  74333 |                 88958 |
    | 1998-06-23 |  81097 |           75286 |           NULL |                  75286 |                    -1 |
    | 1999-06-23 |  84917 |           75994 |           NULL |                  75994 |                    -1 |
    | 2000-06-22 |  85112 |           76884 |           NULL |                  76884 |                    -1 |
    | 2001-06-22 |  85097 |           80013 |           NULL |                  80013 |                    -1 |
    | 2002-06-22 |  88958 |           81025 |           NULL |                  81025 |                    -1 |
    +------------+--------+-----------------+----------------+------------------------+-----------------------+
    ```
    

---

<br>

**11.4.12.4 윈도우 함수와 성능**

- 8.0에 도입된 윈도우 함수는 인덱스를 이용한 최적화가 부족한 부분도 있습니다.

```sql
# 찾은 모든 레코드에 대해 결과를 만들어야 함
EXPLAIN SELECT MAX(from_date) OVER(PARTITION BY emp_no) AS max_from_date 
FROM salaries;
# PK(emp_no, from_date)를 활용하지 못하는 모습
+----+-------------+----------+------------+-------+---------------+-----------+---------+------+---------+----------+-----------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key       | key_len | ref  | rows    | filtered | Extra                       |
+----+-------------+----------+------------+-------+---------------+-----------+---------+------+---------+----------+-----------------------------+
|  1 | SIMPLE      | salaries | NULL       | index | NULL          | ix_salary | 4       | NULL | 2838426 |   100.00 | Using index; Using filesort |
+----+-------------+----------+------------+-------+---------------+-----------+---------+------+---------+----------+-----------------------------+

# 유니크한 emp_no별로 레코드 한건씩만 찾아야 함
EXPLAIN SELECT MAX(from_date)
FROM salaries
GROUP BY emp_no;
+----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys     | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,ix_salary | PRIMARY | 4       | NULL | 294784 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
```

- 윈도우 함수는 인덱스 풀 스캔 및 레코드 정렬까지 진행했습니다.
- GROUP BY는 루스 인덱스 스캔으로 사원별 MAX(from_date)값을 찾음을 알 수 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/12c479ef-a26f-409b-a72e-391833e322bf)


- 위 결과를 보면 윈도우 함수를 이용한 쿼리는 예상보다 많은 레코드를 가공합니다.
- 또한 쿼리 실행 속도에서도 약 5초정도의 시간차이가 발생합니다.

가능하다면 윈도우 함수에 의존하지 말자.

---

<br><br>

### 11.4.13 잠금을 사용하는 SELECT

- InnoDB 테이블 대해서는 레코드를 SELECT할때는 아무 잠금도 걸지 않습니다. 
(Non Locking Consistent Read)
- **SELECT 쿼리를 이용해 읽을 레코드를 업데이트 할때는 다른 트랜잭션이 해당 칼럼의 값을 변경하지 못하게 해야 합니다. 이럴때는 레코드에 강제로 잠금을 걸어 둘 필요가 있습니다.**
    - 이때 사용되는 옵션이 `FOR SHARE`, `FOR UPDATE`가 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f5fd6d52-dfeb-4f77-b03b-9d5e9494fb8f)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/39562cfa-8245-40f4-9ebf-c382bcc00644)

    
- 8.0 이전에는 LOCK IN SHARE MODE와 같은 문법을 사용했지만 지금은 사용하지 않는 편이 좋음
- 위 두가지 잠금 옵션은 모두 `AUTO-COMMIT`이 비활성화된 상태 또는 BEGIN 명령이나  START TRANSACTION 명령으로 `트랜잭션이 시작된 상태에서만 잠금이 유지됩니다.`

- `주의 사항`
    - InnoDB는 MVCC를 통해 잠금 없는 읽기를 지원하기 때문에 FOR UPDATE, FOR SHARE절을 가지지 않는 단순 SELECT 쿼리의 경우 아무런 대기 없이 실행됩니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f1f359c3-9779-41ee-a6c9-36426eea8867)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/4c14042e-2b06-4a3e-bb1d-9e9e95aa6bef)

    

---

<br>

**11.4.13.1 잠금 테이블 선택**

```sql
# 사원의 정보를 조회하는 SELECT 쿼리에 FOR UPDATE를 함께 사용
SELECT *
FROM employees e
	INNER JOIN dept_emp de ON de.emp_no=e.emp_no
	INNER JOIN departments d ON d.dept_no=de.dept_no
FOR UPDATE;
```

- 3개의 테이블 레코드에 모두 쓰기 잠금을 겁니다.
- **만약 쓰기 잠금은 employees에만 걸고 싶다면 8.0버전부터 추가된 잠금을 걸 테이블을 선택하는 기능을 사용할 수 있습니다.**
    
    ```sql
    SELECT *
    FROM employees e
    	INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    	INNER JOIN departments d ON d.dept_no=de.dept_no
    WHERE e.emp_no=10001
    FOR UPDATE OF e;
    ```
    
    - `OF 이름` 과 같이 사용하며 테이블 별명이 사용됐다면 별명을 명시해야 합니다. UPDATE, SHARE절 모두 이용할 수 있습니다.

---

<br>

**11.4.13.2 NOWAIT & SKIP LOCKED**

- 8.0 버전부터 추가된 기능입니다.

```sql
BEGIN;

SELECT * FROM employees WHERE emp_no=10001 FOR UPDATE;

... 응용 프로그램에서 필요한 연산을 수행 ...

UPDATE employees SET ... WHERE emp_no=10001;

COMMIT;
```

- 10001인 사원을 읽고, 그 값을 이용해 연산을 수행하고 employees 테이블에 업데이트 하는 동안, 다른 트랜잭션은 해당 작업이 완료되기를 기다려야 합니다. (innodp_lock_wait_timeout 변수에 설정된 시간만큼)
- `NOWAIT 옵션`
    - **만약 애플리케이션에서 emp_no=10001이 잠김 상태라면 즉시 에러를 반환하도록 하고 싶다면 NOWAIT 옵션을 사용할 수 있습니다.**
        
        ```sql
        SELECT * FROM employees
        WHERE emp_no=10001
        FOR UPDATE NOWAIT;
        ```
        
        - 레코드에 쓰기 잠금을 획득하면 일반 FOR UPDATE처럼 동작하지만, 잠금이 이미 있다면 에러를 반환하며 즉시 쿼리를 종료합니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/9b22e31a-778c-48a3-a4bd-23788235a611)

        
- `SKIP LOCKED 옵션`
    - **SELECT 하려는 레코드가 다른 트랜잭션에 의해 이미 잠긴 상태라면 잠김 레코드는 무시하고 잠금이 걸리지 않은 레코드만 가져옵니다.**
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/1b100bdc-1205-4612-b26d-b165fda3a9c6)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/7e4a5c36-bd02-4eb4-b670-42b8afc289bf)

    
    - SKIP LOCKED절을 가진 SELECT 구문은 확정적이지 않은(NOT-DETERMENISTIC) 쿼리가 됩니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/7b727cba-f166-4b8f-8d29-3addba9c0244)


- `STATEMENT`의 경우 명령문 기반의 로깅으로 비확정적인 쿼리가 사용된 경우 정확한 결과값을 추론할 수 없기 때문에 데이터가 불일치할 수 있습니다.
- `ROW`의 경우 데이터 자체(레코드)를 로깅하는 형태기 때문에 소스, 레플리카 서버의 데이터를 일관적으로 유지할 수 있습니다. 다만 사용자 계정 생성 및 구한 또는 CREATE , ALTER, DROP 등 DDL 문의 경우 STATEMENT 형태로 로깅합니다.
- `MIXED`의 경우 STATEMENT, ROW 방식을 혼합합니다. 기본적으로는 STATEMENT를 사용하다가 쿼리, 스토리지 엔진의 종류에 따라 자동으로 ROW 방식을 사용합니다. 비 확정적인 쿼리의 경우 ROW 포맷으로 전환되어 기록 됩니다.

- `예제: NOWAT, SKIP LOCKED 기능으로 큐와 같은 기능을 MySQL 서버에서 구현하기
(간단 쿠폰 발급 기능)`
    - 하나의 쿠폰은 한 사용자만 사용 가능
    - 쿠폰의 개수는 1000개 제한, 선착순으로 요청한 사용자에게 발급
    
    ```sql
    CREATE TABLE coupon (
      coupon_id     BIGINT NOT NULL,
      owned_user_id BIGINT NULL DEFAULT 0, /* 쿠폰이 발급되면 소유한 사용자의 id를 저장 */
      coupon_code   VARCHAR(15) NOT NULL,
      ...
      PRIMARY KEY (coupon_id),
      INDEX ix_owneduserid (owned_user_id)
    );
    
    # 응용 프로그램에서의 절차
    BEGIN;
    
    # 다른 트랜잭션에서 해당 쿠폰을 가지지 못하게 FOR UPDATE 사용
    SELECT * FROM coupon
            WHERE owned_user_id=0 ORDER BY coupon_id ASC LIMIT 1 FOR UPDATE;
    
    ... 응용 프로그램 연산 수행 ...
    
    UPDATE coupon SET owned_user_id=? WHERE coupon_id=?;
    
    COMMIT;
    ```
    
    - 만약 1000명의 사용자가 동시 요청하면 1명이 수행하고 999명은 대기해야 하는 상황이므로 거의 에러를 반환한다고 할 수 있습니다.
    - 8.0부터는 `FOR UPDATE SKIP LOCKED`를 사용해 다른 트랜잭션에서 이미 사용중인 레코드를 스킵하고 각자의 트랜잭션을 실행할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cff99a68-f6e2-4ed8-861f-6b124d830497)

        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d7701a49-dabb-4a6f-9635-c89997dde975)

        
        - A: 하나의 트랜잭션이 처리되는데 걸리는 시간
        - B: 잠겨진 레코드 1건을 스킵하는 시간
    
- 저자분의 뼈있는 한마디..
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e821be85-14fb-4fb1-b4f5-e9acec6343a5)

    

- `NOWAIT`, `SKIP LOCKED`는 `SELECT … FOR UPDATE` 구문에서만 사용할 수 있습니다.
- NOWAIT, SKIP LOCKED절은 쿼리 자체를 비확정적으로 만들기에 UPDATE, DELETE 문장에서 사용하면 실행때마다 DB상태를 다른 결과로 만들기에 사용할 수 없습니다.
