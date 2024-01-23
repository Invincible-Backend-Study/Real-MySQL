# 📌 9장 옵티마이저와 힌트


## ✅ 9.4 쿼리 힌트

- MySQL 옵티마이저 최적화가 발전했지만, 우리가 서비스하는 비즈니스를 100% 이해하지 못함
- 서비스 개발자나 DBA보다 MySQL 서버가 부족한 실행 계획을 수립할 수 있기에, 옵티마이저에게 실행 계획 수립에 대한 힌트를 줄 수 있습니다.
    - `인덱스 힌트`: 예전 버전의 MySQL에서 사용되어 오던 `USE INDEX`같은 힌트를 의미함
    - `옵티마이저 힌트`: MySQL 5.6버전부터 새롭게 추가되기 시작한 힌트들을 지칭함

→ STRAIGHT_JOIN과 같은 어디에도 포함되지 않는 힌트들도 있음

<br><br>

### 9.4.1 인덱스 힌트

- 인덱스 힌트들은 모두 SQL 문법에 맞게 사용해야 하므로 ANSI-SQL 표준 문법을 준수하지 못하는 단점이 있습니다.
- 5.6부터 추가된 옵티마이저 힌트들은 모두 MySQL 서버를 제외한 다른 RDBMS에선 주석으로 해석하기 때문에 ANSI-SQL 표준을 준수합니다.
- 따라서 인덱스 힌트보단 옵티마이저 힌트를 사용할것을 추천합니다.
- 인덱스 힌트는 SELECT 명령과 UPDATE 명령에서만 사용할 수 있습니다.

<br>

**인덱스 힌트 X:** **STRAIGHT_JOIN**

- 옵티마이저 힌트인 동시에 조인 키워드임
- SELECT, UPDATE, DELETE 쿼리에서 여러 개의 테이블이 조인되는 경우 조인 순서를 고정하는 역할을 함
    
    아래 쿼리는 3개의 테이블을 조인하지만 어떤 테이블이 드라이빙이 되고, 드리븐이 되는지 알 수 없습니다.
    (옵티마이저가 테이블의 통계 정보와 쿼리의 조건을 기반으로 최적이라고 판단되는 순서로 조인함)
    
    ```sql
    EXPLAIN
    SELECT * 
    FROM employees e, dept_emp de, departments d 
    WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;
    +----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------+
    | id | select_type | table | partitions | type   | possible_keys             | key     | key_len | ref                 | rows  | filtered | Extra |
    +----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------+
    |  1 | SIMPLE      | d     | NULL       | ALL    | PRIMARY                   | NULL    | NULL    | NULL                |     9 |   100.00 | NULL  |
    |  1 | SIMPLE      | de    | NULL       | ref    | PRIMARY,ix_empno_fromdate | PRIMARY | 16      | employees.d.dept_no | 41392 |   100.00 | NULL  |
    |  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                   | PRIMARY | 4       | employees.de.emp_no |     1 |   100.00 | NULL  |
    +----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------+
    ```
    
    - 일반적으론 조인 칼럼의 인덱스 여부로 조인의 순서가 결정되고, 인덱스에 문제가 없다면 레코드가 적은 테이블을 드라이빙 테이블로 선택합니다.

`STRAIGHT_JOIN` 인덱스 힌트는 다음과 같이 명시할 수 있습니다.

```sql
SELECT STRAIGHT_JOIN
	e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d 
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

SELECT /*! STRAIGHT_JOIN */
	e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d 
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

EXPLAIN SELECT /*! STRAIGHT_JOIN */ e.first_name, e.last_name, d.dept_name FROM employees e, dept_emp de, departments d  WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;
+----+-------------+-------+------------+--------+---------------------------+-----------------------+---------+----------------------+--------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys             | key                   | key_len | ref                  | rows   | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------------------+-----------------------+---------+----------------------+--------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | index  | PRIMARY                   | ix_lastname_firstname | 124     | NULL                 | 300252 |   100.00 | Using index |
|  1 | SIMPLE      | de    | NULL       | ref    | PRIMARY,ix_empno_fromdate | ix_empno_fromdate     | 4       | employees.e.emp_no   |      1 |   100.00 | Using index |
|  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY                   | PRIMARY               | 16      | employees.de.dept_no |      1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------------------+-----------------------+---------+----------------------+--------+----------+-------------+
```

- 힌트의 표기법이 다르지만 두 쿼리는 모두 동일한 쿼리
- 인덱스 힌트는 SELECT 바로 뒤처럼 사용하는 위치가 결정되어 있음
- **FROM 절에 명시된 테이블의 순서대로 조인을 수행하도록 유도함**

주로 다음 기준에 맞게 **조인 순서가 결정되지 않는 경우 해당 힌트로 조인 순서를 조정하는것이 좋습니다.**

- `임시 테이블(인라인 뷰 또는 파생된 테이블)과 일반 테이블의 조인`: 거의 일반적으로 임시 테이블을 드라이빙 테이블로 선정하는 것이 좋다. 일반 테이블의 조인 칼럼에 인덱스가 없는 경우에는 레코드 건수가 작은 쪽을 먼저 읽도록 드라이빙으로 선택하는 것이 좋은데, 대부분 옵티마이저가 적절한 조인 순서를 선택하기 때문에 쿼리 를 작성할 때부터 힌트를 사용할 필요는 없다. 옵티마이저가 실행 계획을 제대로 수립하지 못해서 심각한 성능 저하가 있는 경우에는 힌트를 사용하면 된다.
- `임시 테이블끼리 조인`: 임시 테이블(서브쿼리로 파생된 테이블)은 항상 인덱스가 없기 때문에 어느 테이블을 먼저 드라이빙으로 읽어도 무관하므로 크기가 작은 테이블을 드라이빙으로 선택해주는 것이 좋다.
- `일반 테이블끼리 조인`: 양쪽 테이블 모두 조인 칼럼에 인덱스가 있거나 양쪽 테이블 모두 조인 칼럼에 인덱스가 없는 경우에는 레코드 건수가 적은 테이블을 드라이빙으로 선택해주는 것이 좋으며, 그 이외의 경우에는 조인 칼럼에 인덱스가 없는 테이블을 드라이빙으로 선택하는 것이 좋다.

→ **언급하는 레코드 건수는 WHERE 조건까지 포함하여 해당 조건을 만족하는 레코드 건수를 의미합니다.**

```sql
# 조건을 나눠서 실행하면 둘 다 종일한 컬럼 개수인데..?
SELECT /*! STRAIGHT_JOIN */
	e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d 
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

# employees 테이블의 건수가 많지만 조건을 만족하는 employees 테이블의 레코드는 건수가 훨씬 적은 경우
# 이럴때는 STRAIGHT_JOIN 힌트를 이용해 employees 테이블을 드라이빙 테이블로 선택하는 것이 좋음
```

- STRAIGHT_JOIN 힌트와 비슷한 옵티마이저 힌트
    - JOIN_FIXED_ORDER: STRAIGHT_JOIN과 동일한 효과
    - JOIN_ORDER
    - JOIN_PREFIX
    - JOIN_SUFFIX
    - 위 3개는 일부 테이블의 조인 순서에 대해서만 제안하는 힌트

<br>

**인덱스 힌트 : USE INDEX / FORCE INDEX / IGNORE INDEX**

- 인덱스 힌트는 사용하려는 인덱스를 가지는 테이블 뒤에 힌트를 명시해야 합니다.
- 옵티마이저는 사용할 인덱스를 무난하게 잘 선택하지만, 3 ~ 4개의 칼럼을 포함하는 비슷한 인덱스가 여러 개 존재하는 경우 실수를 합니다.
- 이런 경우 강제로 특정 인덱스를 사용하도록 힌트를 추가할 수 있습니다.

- 3가지 종류의 인덱스 힌트
    - `USE INDEX`
        - 가장 자주 사용됨
        - 특정 테이블의 인덱스를 사용하도록 권장하는 힌트
        - 따라서 옵티마이저가 무조건 그 인덱스를 사용하는건 아님
    - `FORCE INDEX`
        - 위와 동일하지만 옵티마이저에게 좀 더 강한 주장을 함
        - 근데 USE INDEX에서 안됐으면 FORCE에서도 잘 안된단다.
    - `IGNORE INDEX`
        - 특정 인덱스를 사용하지 못하게 하는 용도로 사용
        - 풀 테이블 스캔 유도시 사용할 수 있음
    
    → 사용자가 부여한 이름이 없는 PK는 `“PRIMARY”`로 명시
    

- 인덱스 용도 명시(선택사항)
    - `USE INDEX FOR JOIN`
        - 테이블 간의 조인 뿐만 아니라 레코드를 검색하기 위한 용도까지 포함하는 용어
        (MySQL은 하나의 테이블에서 검색하는 작업도 JOIN이라고 표현함)
    - `USE INDEX FOR ORDER BY`
        - 명시도니 인덱스를 ORDER BY 용도로만 사용할 수 있게 제한
    - `USE INDEX FOR GROUP BY`
        - 명시도니 인덱스를 GROUP BY 용도로만 사용할 수 있게 제한
    
    → 용도는 보통 옵티마이저가 최적으로 선택하기에 굳이 고려하지 않아도 됨
    

- 예제
    
    ```sql
    # 클러스터링 인덱스 사용
    SELECT * FROM employees WHERE emp_no=10001;
    SELECT * FROM employees FORCE INDEX(primary) WHERE emp_no=10001;
    SELECT * FROM employees USE INDEX(primary) WHERE emp_no=10001;
    
    # 아무 인덱스 사용하지 않음
    SELECT * FROM employees IGNORE INDEX(primary) WHERE emp_no=10001;
    # ix_firstname을 사용할 수 없으므로 아무 인덱스 사용하지 않음
    SELECT * FROM employees FORCE INDEX(ix_firstname) WHERE emp_no=10001;
    ```
    
    - **Full Text Search Index가 있는 경우 가중치를 두고 실행 계획을 수립**하기에 MySQL 옵티마이저는 다른 일반 보조 인덱스를 사용할 수 있는 상황에서 전문 검색 인덱스를 선택하는 경우가 많음
    - 인덱스 사용법, 좋은 실행 계획에 대한 판단이 미흡한 상태라면 그냥 힌트를 사용하지 않는 것을 추천
        - 그떄그때 옵티마이저가 통계 정보를 가지고 선택하게 하는것이 좋을 수 있음
    - 가장 훌륭한 최적화는 쿼리를 서비스에서 없애거나, 튜닝할 필요가 없게 데이터를 최소화 하는 것 혹은 데이터 모델의 단순화를 통해 쿼리를 간결하게 만드는 것입니다.

<br>

**SQL_CALC_FOUND_ROWS**

- MySQL `LIMIT`는 조건을 만족하는 레코드의 수가 LIMIT을 넘겨도 LIMIT에 명시된 수만큼 레코드를 찾으면 즉시 검색 작업을 멈춥니다.
- SQL_CALC_FOUND_ROWS 인덱스 힌트가 포함된 경우엔 LIMIT개수의 레코드를 찾았다 하더라도 끝까지 검색을 수행합니다.
    
    ```sql
    mysql> SELECT SQL_CALC_FOUND_ROWS * FROM employees LIMIT 5;
    +--------+------------+------------+-----------+--------+------------+
    | emp_no | birth_date | first_name | last_name | gender | hire_date  |
    +--------+------------+------------+-----------+--------+------------+
    |  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
    |  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
    |  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
    |  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
    |  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
    +--------+------------+------------+-----------+--------+------------+
    
    mysql> SELECT FOUND_ROWS() total_record_count;
    +--------------------+
    | total_record_count |
    +--------------------+
    |             300024 |
    +--------------------+
    ```
    

- 두 가지 방식의 페이징 처리 비교
    
    ```sql
    # 1. SQL_CALC_FOUND_ROWS 인덱스 힌트 사용
    SELECT SQL_CALC_FOUND_ROWS * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;
    SELECT FOUND_ROWS() total_record_count;
    
    # 2. COUNT(*) 사용
    SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
    SELECT * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;
    ```
    
    1. `first_name=’Georgi’` 처리를 위해 ix_firstname 인덱스를 레인지 스캔으로 실제 값을 읽어옴, 조건 만족의 레코드는 253건이고, **20개만 읽길 원했지만, 인덱스 힌트로 인해 253건 모두를 읽어야 하고 그로 인한 랜덤 I/O가 235번 발생합니다.**
    2. 첫번째 쿼리에서 `first_name=’Georgi’`가 있기에 역시 `ix_firstname` 인덱스를 레인지 스캔합니다. 다만 `COUNT(*)`로 레코드 개수만 읽어오면 되므로 랜덤 I/O가 발생하지 않습니다.
    (**커버링 인덱스 쿼리임**)
        
        두번째 쿼리에선 실제 데이터를 읽어와야 하기에 랜덤 I/O가 발생하지만 20개로 제한되어 있으므로 20번만 발생합니다.
        
    
    → CPU의 연산 작업에 비해 기계적 처리인 디스크 작업은 매우 느리므로 1번 방식보다 2번 방식이 훨씬 효율이 좋습니다.
    
    → SELECT 쿼리 문장이 `UNION(or UNION DISTINCT)`라면 1번 방식에서 `FOUND_ROWS()`함수로 정확한 레코드 건수를 가져올 수 없습니다.
    
    → 또한 8.0.17부터 더이상 사용되지 않고 향후 deprecated 될 예정입니다.
    
    [MySQL :: MySQL 8.0 Reference Manual :: 12.15 Information Functions](https://dev.mysql.com/doc/refman/8.0/en/information-functions.html#function_found-rows)
    
<br><br>

### 9.4.2 옵티마이저 힌트

- 8.0에서 사용가능한 힌트는 많고, 옵티마이저 힌트가 미치는 영향 범위도 매우 다양합니다.

<br>

**옵티마이저 힌트 종류**

- 옵티마이저 힌트는 영향 범위에 따라 4개 그룹으로 나눌 수 있습니다.
    - `인덱스`
        - 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
    - `테이블`
        - 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
    - `쿼리 블록`
        - 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트로서, 특정 쿼리 블록의 이름을 명시하는 것이 아니라 힌트가 명시된 쿼리 블록에 대해서만 영향을 미치는 옵티마이저 힌트
    - `글로벌(쿼리 전체)`
        - 전체 쿼리에 대해서 영향을 미치는 힌트

→ 이 구분으로 힌트의 사용 위치가 달라지진 않음

→ 인덱스 이름이 명시될 수 있으면 `인덱스 수준의 힌트`, 테이블 이름까지만 명시될 수 있는 경우 `테이블 수준의 힌트`로 구분함

→ 특정 힌트는 테이블, 인덱스 이름 모두 명시 가능, 혹은 둘 중하나도 가능, 이를 `인덱스와 테이블 수준의 힌트`라고 함

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e39511c8-9dd6-4a18-b525-c204c040900d)


![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5c6e5f81-bc30-49b8-8ae5-d7a04eb8c494)


- 모든 `인덱스 수준`의 힌트는 반드시 테이블명이 선행돼야 함
- INDEX 힌트를 예시로 들면 다음과 같이 인덱스명 이전에 테이블명을 먼저 명시해야 합니다.
    
    ```sql
    EXPLAIN
    SELECT /*+ INDEX(employees ix_firstname) */ * 
    FROM employees 
    WHERE first_name='Matt';
    
    EXPLAIN 
    SELECT /*+ NO_INDEX(employees ix_firstname) */ * 
    FROM employees 
    WHERE first_name='Matt';
    ```
    
- 옵티마이저 힌트가 문법에 맞지 않다면 경고 메시지를 표시합니다.
(EXPLAIN은 파싱된 쿼리의 재조립 결과 출력을 위해 1개를 디폴트로 보여줌, 따라서 옵티마이저 힌트가 문법에 맞지 않다면 2개를 출력한다.)
    
    ```sql
    EXPLAIN SELECT /*+ NO_INDEX(ix_firstname) */ *
    FROM employees
    WHERE first_name='Matt';
    
    | Warning | 3128 | Unresolved name `ix_firstname`@`select#1` for NO_INDEX hint
    ```
    
- 특정 쿼리 블록을 외부 쿼리 블록에서 사용하기 (`QB_NAME()`)
    - SELECT는 SQL문장에서 여러번 사용가능
    - SELECT 키워드로 시작하는 서브쿼리 영역을 쿼리 블록이라고 함
    
    ```sql
    EXPLAIN
    SELECT /*+ JOIN_ORDER(e, s@subq1) */ COUNT(*)
    FROM employees e
    WHERE e.first_name='Matt'
    	AND e.emp_no IN (SELECT /** QB_NAME(subq1) */ s.emp_no
    										FROM salaries s
    										WHERE s.salary BETWEEN 50000 AND 50500);
    
    +----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+----------------------------+
    | id | select_type | table | partitions | type | possible_keys        | key          | key_len | ref                | rows | filtered | Extra                      |
    +----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+----------------------------+
    |  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              |  233 |   100.00 | Using index                |
    |  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY,ix_salary    | PRIMARY      | 4       | employees.e.emp_no |    9 |     2.12 | Using where; FirstMatch(e) |
    +----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+----------------------------+
    ```
    
    salaries 테이블이 `세미 조인 최적화(firstmatch)`를 통해 조인으로 처리될 것을 예상하고 `JOIN_ORDER` 힌트를 사용함
    
<br>

**MAX_EXECUTION_TIME**

- 옵티마이저 힌트 중 유일하게 **쿼리의 실행 계획에 영향을 미치지 않는 힌트**
- 단순히 쿼리의 최대 실행 시간을 설정하는 힌트
- 밀리초 단위의 시간을 설정하여, 쿼리가 해당 시간을 초과하면 쿼리는 실패함

```sql
SELECT /*+ MAX_EXECUTION_TIME(100) */ *
FROM employees
ORDER BY last_name LIMIT 1;
```

<br>

**SET_VAR**

- **optimizer_switch 시스템 변수를 일시적으로 제어해** 실행 계획을 일시적으로 변경할 수 있음 또한 조인 버퍼나 정렬용 소트 버퍼의 크기를 일시적으로 증가시켜 대용량 처리 쿼리의 성능을 향상시키는 용도로도 사용할 수 있습니다.
    - MySQL 서버의 경우 시스템 변수가 의해 쿼리의 실행 계획에 상당한 영향을 미칩니다.(join_buffer_size 등)
    
    ```sql
    EXPLAIN
    SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ *
    FROM employees
    WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
    ```
    
    (모든 시스템 변수를 조정할 수는 없음)
    
<br>

**SEMIJOIN & NO_SEMIJOIN**

- 세미 조인 최적화에 대한 옵티마이저 힌트
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b892ba96-6e36-46b9-88da-8bf269d39cd6)

    
    - `Table Pull-out` 전략은 그 전략을 사용할 수 있다면 항상 더 나은 성능을 보장하기 때문에 별도로 힌트가 없음
    - 또한 꼭 세미 조인이 아니라 다른 최적화 전략으로 우회하는 것이 더 나은 성능을 낼 수도 있기 때문에 NO SEMIZOIN 힌트도 제공

- FirstMatch 전략을 사용하는 실행 계획을 다른 최적화 전략을 사용하도록 힌트 주기
    
    ```sql
    EXPLAIN 
    SELECT * 
    FROM departments d 
    WHERE d.dept_no IN
    		(SELECT de.dept_no FROM dept_emp de);
    +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
    | id | select_type | table | partitions | type | possible_keys | key     | key_len | ref                 | rows  | filtered | Extra                      |
    +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
    |  1 | SIMPLE      | d     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL                |     9 |   100.00 | NULL                       |
    |  1 | SIMPLE      | de    | NULL       | ref  | PRIMARY       | PRIMARY | 16      | employees.d.dept_no | 41392 |   100.00 | Using index; FirstMatch(d) |
    +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
    ```
    
    - FirstMatch를 적용하기에는 `departments`에대한 조건이 서브쿼리말고 아무것도 없으므로 서브쿼리 `Materialization` 최적화가 더 적합
        
        ```sql
        # 서브쿼리에 쿼리 블록 이름 지정 및 외부 쿼리 블록에서 세미 조인 힌트 지정
        EXPLAIN
        SELECT /*+ SEMIJOIN(@subq1 MATERIALIZATION) */ *
        FROM departments d
        WHERE d. dept_no IN
        		(SELECT /*+ QB_NAME (subq1) */ de. dept_no
        			FROM dept_emp de);
        ```
        
    - 특정 세미 조인 최적화 전략을 사용하지 않게 함 (`NO_SEMIJOIN`)
        
        ```sql
        EXPLAIN
        SELECT *
        FROM departments d
        WHERE d. dept_no IN
        		(SELECT /*+ NO_SEMIJOIN (DUPSWEEDOUT, FIRSTMATCH) */ de.dept_no
        			FROM dept_emp de);
        
        +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
        | id | select_type | table | partitions | type | possible_keys | key     | key_len | ref                 | rows  | filtered | Extra                      |
        +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
        |  1 | SIMPLE      | d     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL                |     9 |   100.00 | NULL                       |
        |  1 | SIMPLE      | de    | NULL       | ref  | PRIMARY       | PRIMARY | 16      | employees.d.dept_no | 41392 |   100.00 | Using index; FirstMatch(d) |
        +----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
        
        ```
        
<br>

**SUBQUERY**

- 세미 조인 최적화가 사용되지 못할 때 사용하는 최적화 방법
- 다음 2가지 형태로 최적화 가능
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6e23aa8e-ed20-4c3e-9a4f-cc214b127a9f)

    
- 세미 조인을 적용할 수 없는 안티 세미 조인(`NOT IN`, `NOT EXISTS`) 최적화에 위의 2가지 최적화가 사용됨
- 세미 조인 최적화 힌트와 비슷함: `서브쿼리에 힌트 사용`, `서브쿼리에 쿼리 블록 이름` 지정 등

<br>

**BNL & NO_BNL & HASHJOIN & NO_HASHJOIN**

- 8.0.20 버전부터는 Block Nested Loop Join을 HashJoin 알고리즘이 대체하도록 개선됨
- BNL, NO_BNL힌트도 8.0.20포함 이후 버전부턴 힌트를 사용하면 **해시 조인을 사용하도록 유도하는 힌트로 용도가 변경됨**
    - 그리고 HASHJOIN, NO_HASHJOIN은 8.0.18 버전 이후 부터는 효력이 없어짐
- 따라서 다음과 같이 해시 조인을 유도 및 사용하지 않게 할 수 있음
    
    ```sql
    EXPLAIN 
    SELECT /*+ BNL(e, de) */ * 
    FROM employees e 
    INNER JOIN dept_emp de ON de.emp_no=e.emp_no;
    ```
    
<br>

**JOIN FIXED_ORDER & JOIN_ORDER & JOIN_PREFIX & JOIN_SUFFIX**

- MySOL 서버에서는 조인의 순서를 결정하기 위해 전통적으로 `STAIGHT_JOIN` 힌트를 사용해왔지만 다음과 같은 불편함이 있습니다.
    - FROM 절에 사용된 테이블의 순서를 조인 순서에 맞게 변경해야 하는 번거로움 존재
    - 한 번 사용되면 FROM 절에 명시된 모든 테이블에 적용되므로 일부 테이블에만 적용하는것이 불가능

- 따라서 옵티마이저 힌트에서 STRAIGHT_JOIN과 동일한 힌트 포함 4개의 힌트를 제공합니다.
    - `JOIN_FIXED_ORDER`: STRAIGHT_JOIN 힌트와 동일하게 FROM 절의 테이블 순서대로 조인을 실행하게 하는 힌트
    - `JOIN_ORDER`: FROM 절에 사용된 테이블의 순서가 아니라 힌트에 명시된 테이블의 순서대로 조인을 실행하는 힌트
    - `JOIN_PREFIX`: 조인에서 드라이빙 테이블만 강제하는 힌트
    - `JOIN_SUFFIX`: 조인에서 드리 테이블(가장 마지막에 조인돼야 할 테이블들)만 강제하는 힌트
    
    ```sql
    # FROM 절에 나열된 테이블의 순서대로 조인 실행
    SELECT /*+ JOIN_FIXED_ORDER() */ *
    FROM employees e
    		INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    		INNER JOIN departments d ON d.dept_no=de.dept_no;
    
    # 일부 테이블에 대해서만 조인 순서를 나열
    SELECT /*+ JOIN_ORDER(d, de) */*
    FROM employees e
    		INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    		INNER JOIN departments d ON d.dept_no=de.dept_no;
    
    # [ISSUE]아래 두개는 책의 설명이 틀린게 아닌지..?
    # 조인 순서에서 먼저 넣을 테이블을 지정
    SELECT /*+ JOIN_PREFIX(e, de) */ *
    FROM employees e
    		INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    		INNER JOIN departments d ON d.dept_no=de.dept_no;
    
    # 조인 순서에 마지막으로 넣을 테이블을 지정
    SELECT /*+ JOIN_SUFFIX(de, e) */ *
    FROM employees e
    		INNER JOIN dept_emp de ON de.emp_no=e.emp_no
    		INNER JOIN departments d ON d.dept_no=de.dept_no;
    ```
    
<br>

**MERGE & NO_MERGE**

- FROM 절의 서브쿼리는 항상 내부 임시 테이블을 생성(Derived Table), 이는 불필요한 자원 소모를 함
- 그래서 5.7과 8.0 버전에서는 가능하면 임시 테이블을 사용하지 않게 FROM 절의 서브쿼리를 외부 쿼리와 병합하는 최적화를 도입
- 다만 임시 테이블, 병합 중에서 더 나은 선택의 주체가 왔다갔다 할 수 있고, **옵티마이저가 최적의 방법을 선택하지 못한다면 해당 힌트를 사용할 수 있습니다.**

```sql
EXPLAIN
SELECT /*+ MERGE(sub)*/ *
FROM (SELECT *
			FROM employees
			WHERE first_name='Matt') sub LIMIT 10;

+----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const |  233 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+

# 다음과 같이 변경됨
SELECT
  *
FROM
  employees
WHERE
  first_name = 'Matt'
LIMIT 10;

# 파생 테이블을 그대로 생성하게 힌트 사용
EXPLAIN
SELECT /*+ NO_MERGE(sub)*/ *
FROM (SELECT *
			FROM employees
			WHERE first_name='Matt') sub LIMIT 10;

+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL         | NULL    | NULL  |  233 |   100.00 | NULL  |
|  2 | DERIVED     | employees  | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const |  233 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
```

<br>

**INDEX_ MERGE & NO_INDEX_MERGE**

- MySQL 서버는 가능하면 테이블당 하나의 인덱스만을 이용해 쿼리를 처리하려 함
- 하나의 인덱스만으로 검색 대상 범위를 충분히 좁힐 수 없다면 MySQL 옵티마이저는 사용 가능한 다른 인덱스를 이용하기도 함
- 여러 인덱스를 통해 구한 레코드의 교집합, 합집합만을 구해 결과를 반환
- 여러개의 인덱스를 동시에 사용하는(인덱스 머지)것은 성능향상에 도움이 되기도, 안되기도 하기에 이를 제어하기 위한 힌트를 제공합니다.

```sql
# 인덱스 머지 사용
EXPLAIN SELECT *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;

+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys        | key                  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | PRIMARY,ix_firstname | ix_firstname,PRIMARY | 62,4    | NULL |    1 |   100.00 | Using intersect(ix_firstname,PRIMARY); Using where |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+

# 인덱스 머지 사용하지 않도록 힌트 적용
EXPLAIN
SELECT /*+ NO_INDEX_MERGE(employees PRIMARY) */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;

+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys        | key          | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY,ix_firstname | ix_firstname | 62      | NULL |   14 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+

# 인덱스 머지 사용하도록 힌트 적용
EXPLAIN
SELECT /*+ INDEX_MERGE(employees PRIMARY) */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;

+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys        | key                  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | PRIMARY,ix_firstname | ix_firstname,PRIMARY | 62,4    | NULL |    1 |   100.00 | Using intersect(ix_firstname,PRIMARY); Using where |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+----------------------------------------------------+
```

<br>

**NO_ICP  (No Index Condition Pushdown)**

- ICP 최적화는 사용 가능하다면 항상 성능 향상에 도움이 되므로 옵티마이저는 최대한 사용하고자 하는 방향으로 실행 계획을 수립합니다.(따라서 옵티마이저 힌트가 없음)
- 만약 A인덱스에서 컨디션 푸시다운이 가능하고, 일반적인 B인덱스 사이에서 하나를 골라야 하는 상황이 있을때
    - A 인덱스 + 컨디션 푸시다운의 비용이 낮게 예측됐지만 실제로는 B가 더 효율적인 상황이라면
    - 인덱스 컨디션 푸시다운 최적화를 비활성화 시킬 수 있습니다.
    
    ```sql
    # 임시 인덱스 생성
    ALTER TABLE employees ADD INDEX ix_lastname_firstname (last_name, first_name);
    
    EXPLAIN
    SELECT *
    FROM employees
    WHERE last_name='Acton' AND first_name LIKE '%sal';
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
    | id | select_type | table     | partitions | type | possible_keys         | key                   | key_len | ref   | rows | filtered | Extra                 |
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
    |  1 | SIMPLE      | employees | NULL       | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |  189 |    11.11 | Using index condition |
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
    
    # 인덱스 컨디션 푸시다운 최적화 비활성화
    EXPLAIN
    SELECT /*+ NO_ICP (employees ix_lastname_firstname) */ * FROM employees
    WHERE last_name='Acton' AND first_name LIKE '%sal';
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys         | key                   | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |  189 |    11.11 | Using where |
    +----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
    ```
    
<br>

**SKIP_SCAN & NO_SKIP_ SCAN**

- 조건이 누락된 선행 칼럼이 가지는 유니크한 값의 개수가 많아지면 인덱스 스킵 스캔의 성능은 오히려 더 떨어집니다.
- 옵티마이저가 유니크 한 값의 개수 분석 오류, 잘못된 경로로 인해 비효율적인 인덱스 스킵 스캔을 선택할때 힌트를 통해 사용하지 않게 할 수 있습니다.

```sql
ALTER TABLE employees ADD INDEX ix gender_birthdate (gender, birth_date);

# 인덱스 스캡 스캔 사용
EXPLAIN
SELECT gender, birth_date
FROM employees
WHERE birth_date>='1965-02-01';
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys       | key                 | key_len | ref  | rows   | filtered | Extra                                  |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_gender_birthdate | ix_gender_birthdate | 4       | NULL | 100110 |   100.00 | Using where; Using index for skip scan |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------+

# 인덱스 스킵 스캔 비활성화
EXPLAIN
SELECT /*+ NO_SKIP_SCAN(employees ix_gender_birthdate) */ gender, birth_date
FROM employees
WHERE birth_date>='1965-02-01';
+----+-------------+-----------+------------+-------+---------------+---------------------+---------+------+--------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key                 | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+---------------------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | index | NULL          | ix_gender_birthdate | 4       | NULL | 300363 |    33.33 | Using where; Using index |
+----+-------------+-----------+------------+-------+---------------+---------------------+---------+------+--------+----------+--------------------------+
```

<br>

**INDEX & NO_INDEX**

- INDEX, NO_INDEX는 과거 MySQL 서버에서 사용되던 인덱스 힌트를 대체하는 용도로 제공되며 옵션은 아래와 같습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/53600db7-098e-4e92-b867-1135061f8716)

    
- 인덱스 힌트는 특정 테이블 뒤에 사용했으므로 별도로 힌트 내에 테이블명 없이 인덱스 이름만 나열했지만, **옵티마이저 힌트에선 테이블명, 인덱스 이름을 함께 명시해야 합니다.**
    
    ```sql
    # 인덱스 힌트 사용
    EXPLAIN
    SELECT *
    FROM employees USE INDEX(ix_firstname)
    WHERE first_name='Matt';
    
    # 옵티마이저 힌트 사용
    EXPLAIN
    SELECT /*+ INDEX employees ix_firstname) */ *
    FROM employees
    WHERE first_name='Matt';
    ```
    
<br><br><br><br>

# 📌 10장 실행 계획

- DBMS는 많은 데이터를 안전하게 저장 및 관리하고 사용자가 원하는 데이터를 빠르게 조회 할 수 있게 해주는 것이 주목적
- 옵티마이저가 사용자의 쿼리를 최적으로 처리될 수 있게 하는 쿼리 실행 계획을 수립해야 함
- 하지만 관리자, 사용자 개입 없이 항상 좋은 실행 계획을 만들기 어려움
    - 따라서 옵티마이저가 수립한 실행계획을 확인할 수 있게 EXPLAIN 명령 제공
- 더 깊은 이해를 위해선 MySQL 서버가 보여주는 실행 계획 + 서버가 데이터를 처리하는 로직을 이해해야 함
    - 실행 계획에 가장 큰 영향을 미치는 건 `통계 정보`
    - MySQL 서버가 보여주는 `실행 계획을 읽는 순서`, `실행 계획에 출력되는 키워드`, `알고리즘`

<br><br><br>

## ✅ 10.1 통계 정보

- 5.7까지는 테이블, 인덱스에 대한 개괄적인 정보만 가지고 실행 계획 수립함
    - 칼럼 값 분포를 모르므로 실행 계획의 정확도가 떨어지는 경우가 다수 존재
- 8.0부터는 인덱스되지 않은 칼럼들에 대해서도 데이터 분포도를 수집해 저장하는 히스토그램 정보가 도입됨
(물론 인덱스의 통계 정보도 같이 필요함)

<br><br>

### 10.1.1 테이블 및 인덱스 통계 정보

- **비용 기반 최적화에서 가장 중요한 것은 통계 정보
(통계 정보가 비 정확 하다면 그만큼의 잘못된 실행계획이 도출될 수 있기 때문)**
- 기존 MySQL은 다른 DBMS보다 통계정보의 정확도가 낮았고 휘발성(메모리에만 관리됨)이 강했기에, 쿼리의 실제 실행 계획 수립시 실제 테이블 데이터 일부를 분석해 통계 정보를 보완함

<br>

**MySQL 서버의 통계 정보**

- 5.6 버전부터 InnoDB 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적(Persistent)으로 관리할 수 있게 개선함
- 5.5버전 까지는 각 테이블의 통계 정보가 메모리에만 관리했기에 서버가 재시작 되면 모든 통계 정보가 사라졌습니다.
(오로지 SHOW INDEX 명령으로만 테이블 인덱스 칼럼 분포도 확인 가능)
- 5.6 버전부터는 각 테이블의 통계 정보를 `mysql` 데이터베이스의 `innodb_index_stats` 테이블 과 `innodb_table_stats` 테이블로 관리할 수 있게 개선하여 재시작되어도 기존의 통계 정보를 유지함
    
    ```sql
    USE mysql;
    SHOW TABLES LIKE '%_stats' ;
    +---------------------------+
    | Tables_in_mysql (%_stats) |
    +---------------------------+
    | innodb_index_stats        |
    | innodb_table_stats        |
    +---------------------------
    ```
    
- 5.6에선 테이블 생성시 STATS_PERSISTENT 옵션을 통해 테이블 단위로 영구적인 통계 정보 보관 여부를 결정할 수 있습니다.
    
    ```sql
    CREATE TABLE tab_test (fd1 INT, fd2 VARCHAR(20), PRIMARY KEY(fd1))
    ENIGNE=InnoDB
    STATS_PERSISTENT={ DEFAULT | 0 | 1 } # 3개중에 고름
    ```
    
    - `STATS_PERSISTENT=0`: 테이블의 통계 정보를 MYSQL 5.5 이전의 방식대로 관리하며, mysql 데이터베이스의 Innodb_index_stats와 innodb_table_stats 테이블에 저장하지 않음
    - `STATS_PERSISTENT=1`: 테이블의 통계 정보를 mysql 데이터베이스의 innodb_index_stats와 innodb_table_stats 테이블에 저장함
    - `STATS_PERSISTENT=DEFAULT`: 테이블을 생성할 때 별도로 STATS_PERSISTENT 옵션을 설정하지 않은 것과 동일하며, 테이블의 통계를 영구적으로 관리할지 말지를 `innodb_stats_persistent` 시스템 변수의 값으로 결정한다.
    
    디폴트 설정은 `ON(1)`입니다.
    
    ```sql
    # 테이블 통계정보 영구 저장
    CREATE TABLE tab_persistent (fd1 INT PRIMARY KEY, fd2 INT)
    ENGINE=InnODB STATS_PERSISTENT=1;
    
    # 테이블 통계 정보 저장 X
    CREATE TABLE tab_transient (fd1 INT PRIMARY KEY, fd2 INT)
    ENGINE=InnoDB STATS_PERSISTENT=0;
    
    SELECT * FROM mysql. innodb_table_stats
    WHERE table_name IN ('tab_persistent', 'tab_transient') \G
    ******************* 1.roW *******************
    				   	database_name: test
    				  	 	 table_name: tab_persistent
    				      last_update: 2013-12-28 17:11:30
    		               n_rows: 0
        clustered _index size: 1
     sum_of_other index sizes: 0
    
    # ALTER TABLE 명령으로 설정을 변경할 수 있음
    ALTER TABLE employees.employees STATS_PERSISTENT=1;
    
    SELECT *
    FROM mysql.innodb_index_stats
    WHERE database_name='employees'
    AND TABLE_NAME='employees';
    
    +-----------------------+--------------+------------+-------------+-----------------------------------+
    | index_name            | stat_name    | stat_value | sample_size | stat_description                  |
    +-----------------------+--------------+------------+-------------+-----------------------------------+
    | PRIMARY               | n_diff_pfx01 |     299113 |          20 | emp_no                            |
    | PRIMARY               | n_leaf_pages |        886 |        NULL | Number of leaf pages in the index |
    | PRIMARY               | size         |        929 |        NULL | Number of pages in the index      |
    | ix_firstname          | n_diff_pfx01 |       1352 |          20 | first_name                        |
    | ix_firstname          | n_diff_pfx02 |     325698 |          20 | first_name,emp_no                 |
    | ix_firstname          | n_leaf_pages |        496 |        NULL | Number of leaf pages in the index |
    | ix_firstname          | size         |        609 |        NULL | Number of pages in the index      |
    | ix_gender_birthdate   | n_diff_pfx01 |          1 |           3 | gender                            |
    | ix_gender_birthdate   | n_diff_pfx02 |       8844 |          20 | gender,birth_date                 |
    | ix_gender_birthdate   | n_diff_pfx03 |     284666 |          20 | gender,birth_date,emp_no          |
    | ix_gender_birthdate   | n_leaf_pages |        361 |        NULL | Number of leaf pages in the index |
    | ix_gender_birthdate   | size         |        417 |        NULL | Number of pages in the index      |
    | ix_hiredate           | n_diff_pfx01 |       4851 |          20 | hire_date                         |
    | ix_hiredate           | n_diff_pfx02 |     292515 |          20 | hire_date,emp_no                  |
    | ix_hiredate           | n_leaf_pages |        294 |        NULL | Number of leaf pages in the index |
    | ix_hiredate           | size         |        353 |        NULL | Number of pages in the index      |
    | ix_lastname_firstname | n_diff_pfx01 |       1679 |          20 | last_name                         |
    | ix_lastname_firstname | n_diff_pfx02 |     275954 |          20 | last_name,first_name              |
    | ix_lastname_firstname | n_diff_pfx03 |     300955 |          20 | last_name,first_name,emp_no       |
    | ix_lastname_firstname | n_leaf_pages |        460 |        NULL | Number of leaf pages in the index |
    | ix_lastname_firstname | size         |        545 |        NULL | Number of pages in the index      |
    +-----------------------+--------------+------------+-------------+-----------------------------------+
    
    SELECT *
    FROM mysql.innodb_table_stats
    WHERE database_name='employees'
    AND TABLE_NAME='employees';
    
    +---------------+------------+---------------------+--------+----------------------+--------------------------+
    | database_name | table_name | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
    +---------------+------------+---------------------+--------+----------------------+--------------------------+
    | employees     | employees  | 2024-01-09 16:32:51 | 299113 |                  929 |                     1924 |
    +---------------+------------+---------------------+--------+----------------------+--------------------------+
    ```
    
    - 통계 정보의 각 칼럼은 다음과 같은 값을 저장하고 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/86a50b45-ef90-40ab-8d52-36d3f36ed4ae)

        
    - innodb_table_stats.`sum_of_other_index_sizes` 칼럼의 값은 테이블의 `STATS_AUTO_RECALC` 옵션에 따라 0으로 보일 수도 있는데, 그 경우 테이블에 대해 `ANALYZE TABLE` 명령을 실행하면 통곗값이 저장됩니다.
        
        `ANALYZE TABLE employees.employees;`
        
    
- MySQL 5.5까지는 테이블의 통계 정보가 메모리에만 저장됐고, 재시작시 통계 정보가 초기화 됐기에, 다음과 같은 이벤트가 발생하면 자동으로 통계 정보가 갱신됐습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/eb8d8ef8-e7d7-4982-ae47-0457de78a587)

    
    - 잦은 통계 정보의 갱신은 인덱스 레인지 스캔으로 잘 처리하던 쿼리가, 갑자기 풀 테이블 스캔으로 실행되는 상황이 발생할 수 있습니다. → **내용이 시시각각 변하므로**
    - `영구적인 통계정보 도입`과, `innodb_stats_auto_recalc`를 off로 두어 통계 정보의 자동 갱신을 막을 수 있습니다.
    - 통계정보 자동 수집 여부도 STATS_AUTO_RECALC 옵션을 이용해 테이블 생성시 조정할 수 있습니다.
        - `STATS_AUTO_RECALC=1`: 테이블의 통계 정보를 MySQL 5.5 이전의 방식대로 자동 수집한다.
        - `STATS_AUTO_RECALC=0`: 테이블의 통계 정보는 ANALYZE TABLE 명령을 실행할 때만 수집된다.
        - `STATS AUTO_RECALC=DEFAULT`: 테이블을 생성할 때 별도로 STATS_AUTO_RECALC 옵션을 설정하지 않은 것과 동일 하며, 테이블의 통계 정보 수집을 `innodb_stats_auto_recalc` 시스템 설정 변수의 값으로 결정한다.
    
- MySQL 5.5 버전에선 테이블 통계 정보 수집시 몇 개의 InnoDB 테이블 블록을 샘플링 할지 결정하는 옵션이 있었지만, 5.6부터는 해당 옵션은 사라지고, 2개의 시스템 변수로 분리됐습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ea4c2e95-049c-4275-9268-7a99898d7a5f)

    
    - 영구적인 통계 정보 사용은 더 정확한 통계 정보 수집을 위해 많은 시간이 소요되지만, 이후 **통계 정보의 정확성을 통해 얻을 수 있는쿼리의 성능과 직결되기에 시간을 투자할 가치가 있습니다.**
    - `innodb_stats_persistent_sample_pages` 시스템 변수에 높은 값을 설정하면 더 정확한 통계 정보를 수집할 수 있습니다. (너무 올리면 시간이 길어짐)

<br><br>

### 10.1.2 히스토그램

- 5.7버전까진 통계 정보는 단순히 인덱스 칼럼의 유니크한 값의 개수 정도만 가졌습니다. → 계획 수립하기에 정보 부족 → 매꾸기 위해 랜덤한 인덱스 페이지를 가져와 참조
- 8.0부터는 칼럼의 데이터 분포도를 참조할 수 있는 히스토그램 정보를 활용할 수 있습니다.

<br>

**히스토그램 정보 수집 및 삭제**

- MySQL 8.0 버전에서 히스토그램 정보는 칼럼 단위로 관리되는데, 이는 자동으로 수집되지 않고
`ANALYZE TABLE ••• UPDATE HISTOGRAM` 명령을 실행해 `수동`으로 수집 및 관리됩니다.
- 수집된 정보는 `시스템(데이터) 딕셔너리`에 함께 저장됨
- MySQL 서버가 시작되면 딕셔너리 히스토그램 정보를 `information_schema` 데이터베이스의 `column-statistics` 테이블로 로드
    
    쿼리는 다음과 같고 실제 결과는 `책 400~401page 참조`
    
    ```sql
    SELECT * 
    FROM information_schema.COLUMN_STATISTICS 
    WHERE SCHEMA_NAME='employees' AND TABLE_NAME='employees';
    ```
    
- 8.0 버전은 2종류의 히스토그램 타입이 지원됨
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/fae18f42-29bf-4469-8ede-b5165c643dae)

    
- 히스토그램은 버킷(Bucket) 단위로 구분되어 레코드 건수나 칼럼값의 범위가 관리
    - `싱글톤 히스토그램`
        - 칼럼이 가지는 값 별로 버킷 할당
        - 각 버킷은 `칼럼의 값`, `발생 빈도의 비율` 2개의 값을 가짐
    - 높이 균형 히스토그램
        - 개수가 균등한 칼럼값의 범위별로 하나의 버킷이 할당됨
        - `범위 시작 값`, `마지막 값`, `발생 빈도율`, `각 버킷에 포함된 유니크한 값의 개수` 4개의 값을 가짐

- 그림 10.1과 그림 10.2는 employees 테이블의 gender 칼럼과 hire_date 칼럼의 히스토그램 데이터를 그 래프로 표현한 것입니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/407df181-c101-4279-9ec4-13ca41f81cf7)

    
    - 싱글톤 히스토그램
        - `ENUM(’M’, ‘F’)` 타입인 gender 칼럼이 가질 수 있는 값에 대한 누적 레코드 건수 비율을 보여줍니다.
        - 싱글톤 히스토그램은 유니크한 값의 개수가 상대적으로 적은 칼럼에 대해 사용됩니다.
        - 값은 `누적`되므로 F의 비율은 `1 - 0.59985298`로 구합니다.
    - 높이 균형 히스토그램
        - 칼럼값의 각 범위에 대해 레코드 건수 비율이 `누적`으로 표시됩니다.
        - 그래프의 기울기가 일정하므로 각 칼럼의 범위가 비슷한 레코드의 건수를 가짐을 알 수 있습니다.

- `information_schema.colum_statistics` 테이블의 `HISTOGRAM` 칼럼이 가진 나머지 필드 설명
    - `sampling-rate`
        - 히스토그램 정보를 위해 스캔한 페이지의 비율, 0.35의 경우 전체 데이터 페이지의 35%를 스캔했다는 의미가 됩니다.
        - 샘플링 비율은 `histogram_generation_max_mem_size` 시스템 변수로 설정할 수 있습니다.(default 20MB) 비율이 너무 크다면 시스템 자원의 소모와 부하가 높아집니다.
    - `histogram-type`
        - 히스토그램의 종류를 저장합니다.
    - `number-of-buckets-specified`
        - 히스토그램을 생성할 때 설정했던 버킷의 개수를 저장합니다.
        - 별도로 지정하지 않았다면 100개의 버킷이 사용됩니다.
        - 최대 1024개 설정 가능 (다만 100개 정도가 충분함)

→ 8.0.19 미만까지는 `histogram_generation_max_mem_size` 값에 상관없이 테이블 풀 스캔을 샘플링하여 히스토그램을 생성했지만 8.0.19부턴 InnoDB 스토리지 엔진 자체의 샘플링 알고리즘을 사용합니다.

- 히스토그램 삭제 (딕셔너리 내용만 삭제하므로 다른 쿼리 처리에 영향을 주지 않고 즉시 완료됨)
    - 다만 쿼리의 실행 계획이 달라 질 수 있기에 주의 해야 합니다.
    
    ```sql
    ANALYZE TABLE employees.employees
    DROP HISTOGRAM ON gender, hire_date;
    ```
    
- 히스토그램 삭제 없이 optimizer_switch 시스템 변수 설정으로 히스토그램 사용 중지
    
    ```sql
    SET GLOBAL optimizer_switch='condition_fanout_filter-off';
    ```
    
    다만 위 설정에 의해 다른 최적화 기능들이 사용되지 않을 수 있으므로 주의해야 합니다.
    
- 특정 커넥션, 특정 쿼리에서 히스토그램 사용 X
    
    ```sql
    # 현재 커넥션에서 실행되는 쿼리만 히스토그램을 사용하지 않게 설정
    SET SESSION optimizer_switch='condition_fanout_filter=off';
    
    # 현재 쿼리만 히스토그램을 사용하지 않게 설정
    SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */*
    FROM ...
    ```
    
<br>

**히스토그램의 용도**

- 기존의 통계 정보는 `테이블의 전체 레코드 건수`, `인덱스된 칼럼이 가지는 유니크한 값의 개수` 였습니다.
- 하지만, 단순히 유니크한 값의 개수를 전체 레코드에 나눠서 대략 몇개의 레코드가 일치할 것 이라는 계산은 균등한 분포에서만 잘 작동합니다. 따라서 이를 해결하기 위해 히스토그램이 도입 됐습니다.
- 히스토그램은 특정 칼럼의 모든 값에 대한 분포도 정보를 가지지는 않지만, `각 범위(버킷)별 로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정확한 예측`을 할 수 있음

- 히스토그램 유무에 따른 예측치 차이
    
    ```sql
    # 히스토그램 정보 수집 사용 전
    EXPLAIN
    SELECT *
    FROM employees
    WHERE first_name='Zita'
    AND birth_date BETWEEN '1950-01-01' AND '1960-01-01';
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const |  224 |    11.11 | Using where |
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    
    # 히스토그램 정보 수집
    ANALYZE TABLE employees
    UPDATE histogram ON first_name, birth_date;
    
    # 히스토그램 정보 수집 사용 후
    EXPLAIN
    SELECT *
    FROM employees
    WHERE first_name='Zita'
    AND birth_date BETWEEN '1950-01-01' AND '1960-01-01';
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const |  224 |    60.85 | Using where |
    +----+-------------+-----------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
    
    # 실제 데이터 조회
    SELECT
    SUM(CASE WHEN birth_date between '1950-01-01' and '1960-01-01' THEN 1 ELSE 0 END)
    		/ COUNT(*) as ratio
    FROM employees WHERE first_name='Zita';
    
    +--------+
    | ratio  |
    +--------+
    | 0.6384 |
    +--------+
    ```
    
    - 사용전
        - `first_name=’Zita’` 조건에 일치하는 레코드가 224건이 있고, 그중에서 대략 `11.11%`인 24.8명 정도의 `bitrth_date`가 1950년대 출생일 것으로 예측
    - 사용후
        - 60.85%인 136.2명이 1950년대 출생일 것으로 예측했습니다.
        - 실제 데이터를 조회해보면 대략 63.84%인 143명이 1950년대 출생인 것을 알 수 있다.
    
    → **따라서 히스토그램 사용 유무의 차이가 매우 큼을 알 수 있습니다.**
    

- 히스토그램 정보에서 특정 범위의 데이터의 많고 적음을 식별하는 것의 차이
    
    ```sql
    # 조인에 사용되는 테이블의 레코드 개수
    +----------+
    | salaries |
    +----------+
    |  2844047 |
    +----------+
    +-----------+
    | employees |
    +-----------+
    |   300024  |
    +-----------+
    
    SELECT /*+ JOIN_ORDER(e, s) */ *
    FROM salaries s
    		INNER JOIN employees e ON e.emp_no=s.emp_no
    				AND e.birth_date BETWEEN '1950-01-01' AND '1950-02-01'
    WHERE s.salary BETWEEN 40000 AND 70000;
    
    # Empty set (0.15 sec)
    
    SELECT /*+ JOIN_ORDER(s, e) */ *
    FROM salaries s
    		INNER JOIN employees e ON e.emp_no=s.emp_no
    				AND e.birth_date BETWEEN '1950-01-01' AND ' 1950-02-01'
    WHERE s.salary BETWEEN 40000 AND 70000;
    # Empty set, 65535 warnings (1.32 sec)
    ```
    
    - E → S 조인은 조인을 해야하는 건수가 반대의 경우보다 훨씬 적습니다.
    - birth_date, salary칼럼은 인덱스가 되지 않았으므로 칼럼들의 히스토그램을 참조해 실행계획을 세웁니다.
    - 만약 히스토그램이 없고, 옵티마이저 힌트도 제거했다면 전체 레코드 건수, 크기와 같은 단순 정보로 드라이빙 테이블을 결정하므로 어떤 테이블이라도 조인의 드라이빙 테이블이 될 수 있고, 성능이 복불복이 될 수 있습니다.
    - 결국 최대 10배 정도의 성능 차이가 나게 됩니다.
    - 그만큼 히스토그램 정보가 있어야 옵티마이저가 더 정확한 판단을 할 수 있습니다.
    
<br>

**히스토그램과 인덱스**

- 히스토그램과 인덱스는 완전히 다른 객체이므로 비교 대상은 아님
- 다만 인덱스는 부족한 통계 정보를 수집하기 위해 사용됨 (히스토그램과 약간의 공통점)
    - 가능한 인덱스들에서 조건절에 일치하는 레코드 건수를 대략 파악후 최종적으로 가장 나은 실행 계획 선택
    - 조건절에 일치하는 레코드 건수 예측을 위해 실제 인덱스 B-Tree를 샘플링해 살펴보는데 이를 `인덱스 다이브`라고 표현함

```sql
SELECT *
FROM employees
WHERE first_name='Tonny'
		AND birth_date BETWEEN '1954-01-01' AND '1955-01-01';
```

- 옵티마이저는 위 쿼리에서 ix_firstname을 사용할지, 풀 스캔을 할지 고민합니다.
- 8.0 서버에선 인덱스된 칼럼을 조건으로 사용하는 경우 **칼럼의 히스토그램이 아니라 실제 인덱스 다이브를 통해 직접 수집한 정보를 활용합니다.**
    - 실제 검색 조건의 대상 값에 대한 샘플링을 실행하는 것
    - 항상 히스토그램보다 더 정확한 값을 기대할 수 있음
- 그래서 **8.0에서 히스토그램은 인덱스되지 않은 칼럼에 대한 데이터 분포도를 참조하는 용도로 사용됨**
- 다만 인덱스 다이브도 비용이 들어가며 때로는 (IN 절에 값이 많이 명시된 경우) 실행 계획 수립만으로 상당한 인덱스 다이브를 실행해 비용이 커집니다.

<br><br>

### 10.1.3 코스트 모델(Cost Model) → 약간 작업들에 가중치를 하드웨어 별로 조절해 옵티마이저가 올바른 실행 계획을 수립할 수 있게 해주는 건가?

- MySQL 서버가 쿼리를 처리하려면 다음과 같은 다양한 작업을 필요로 합니다.
    - 디스크로부터 데이터 페이지 읽기
    - 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
    - 인덱스 키 비교
    - 레코드 평가
    - 메모리 임시 테이블 작업
    - 디스크 임시 테이블 작
- MySQL 서버는 사용자 쿼리에 대해 위 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과를 바탕으로 최적의 실행 계획을 찾습니다.
    - 전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용을 **코스트 모델(Cost Model)**이라고 합니다.
    - 5.7이전에는 작업들의 비용을 서버 소스코드에 상수화 시켰지만 하드웨어 스펙에 따라 작업 비용이 상이해 집니다.
- **5.7버전부턴 각 단위 작업 비용을 DBMS 관리자가 조정할 수 있게 개선됨**
- 다만 8.0이전까지는 인덱스 되지 않은 칼럼의 데이터 분포, 메모리에 상주중인 페이지의 비율와 같은 정보가 부족하여 비용 계산에 어려움이 있었습니다.
- **8.0부터 히스토그램을 통해 칼럼의 데이터 분포나 인덱스별 메모리에 적재된 페이지의 비율이 관리되며 옵티마이저의 실행 계획 수립에 사용되기 시작함**

- MySQL 8.0 서버의 코스트 모델은 다음 2개 테이블에 저장된 설정값을 사용합니다.
    - `server_cost`
        - 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
    - `engine_cost`
        - 레코드를 가진 데이터 페이지를 가져오는 데 필요한 비용 관리

- 2개의 테이블이 가진 공통 칼럼
    - `cost_name`
        - 코스트 모델의 각 단위 작업
    - `default_value`
        - 각 단위 작업의 비용(기본값이며, 이 값은 MySQL 서버 소스 코드에 설정된 값)
    - `cost_value`
        - DBMS 관리자가 설정한 값이 값이 NULL이면 MySQL 서버는 default_value 칼럼의 용 사용)
    - `last_updated`
        - 단위 작업의 비용이 변경된 시점
        - 옵티마이저에 영향 X 단순 정보성
    - `comment`
        - 비용에 대한 추가 설명
        - 옵티마이저에 영향 X 단순 정보성

- engine_cost 테이블이 가진 추가 칼럼
    - `engine_name`
        - 비용이 적용된 스토리지 엔진
        - 스토리지 엔진별로 각 단위 작업의 비용을 설정할 수 있음 (기본값 `default`)
        - default는 특정 스토리지 엔진의 비용이 설정되지 않았다면 해당 스토리지 엔진의 비용으로 이 값을 적용한다는 의미
    - `device_type`
        - 디스크 타입
        - 8.0에선 아직 칼럼의 값 활용하지 않음 (0만 설정 가능)

- MySQL 8.0 버전의 코스트 모델에서 지원하는 8개의 단위 작업
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/246d87a5-48dd-41e7-97df-ef4ae601bc08)

    
    - `row-evaluate_cost`
        - 스토리지 엔진이 반환한 레코드가 쿼리의 조건에 일치하는지를 평가하는 단위 작업을 의미
        - 값이 증가하면 많은 레코드를 처리하는 쿼리(풀 테이블 스캔)의 비용이 높아지고, 적은 수의 레코드를 처리하는 쿼리(레인지 스캔)의 비용이 낮아짐
    - `key_compare_cost`
        - 키 값의 비교 작업에 필요한 비용
        - 값 증가 → 레코드 정렬과 같이 키 값 비교 처리가 많은 경우 쿼리의 비용이 높아진다.

- MySQL 서버에서 각 실행 계획의 계산된 비용(Cost) 확인
    
    ```sql
    EXPLAIN FORMAT=TREE
    SELECT *
    FROM employees WHERE first_name='Matt';
    +--------------------------------------------------------------------------------------------+
    | EXPLAIN                                                                                    |
    +--------------------------------------------------------------------------------------------+
    | -> Index lookup on employees using ix_firstname (first_name='Matt')  (cost=81.5 rows=233)
    |
    +--------------------------------------------------------------------------------------------+
    
    EXPLAIN FORMAT=JSON
    SELECT *
    FROM employees WHERE first_name='Matt';
    +---------------------------------------------------------+
    | EXPLAIN                                                 |
    +---------------------------------------------------------+
    | {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "81.55"
        },
        "table": {
          "table_name": "employees",
          "access_type": "ref",
          "possible_keys": [
            "ix_firstname"
          ],
          "key": "ix_firstname",
          "used_key_parts": [
            "first_name"
          ],
          "key_length": "58",
          "ref": [
            "const"
          ],
          "rows_examined_per_scan": 233,
          "rows_produced_per_join": 233,
          "filtered": "100.00",
          "cost_info": {
            "read_cost": "58.25",
            "eval_cost": "23.30",
            "prefix_cost": "81.55",
            "data_read_per_join": "30K"
          },
          "used_columns": [
            "emp_no",
            "birth_date",
            "first_name",
            "last_name",
            "gender",
            "hire_date"
          ]
        }
      }
    } |
    +---------------------------------------------------------+
    ```
    
    - 각 단위 작업 비용을 이용해 실행 계획에 표시되는 비용을 직접 하는것은 어려움
    - 옵티마이저는 다양한 추가정보들을 이용해 쿼리의 비용을 계산하는데 이런 정보는 사용자에게 모두 표시되지 않으므로 직접 계산하기란 어렵습니다.
        - 인덱스의 B-Tree 깊이
        - 인덱스 키 검색을 위해 읽어야 하는 페이지의 개수
        - 디스크와 메모리(InnoDB 버퍼 풀)에서  읽어야 하는 데이터 페이지 개수
        - 레코드 정렬 작업에 사용되는 알고리즘 별 키 값 비교 작업 횟수
    

**결국 코스트 모델에서 가장 중요한 것은 각 단위 작업에 설정된 비용 값이 커지면 어떤 실행 계획들이 고비용으로 바뀌고 반대로 저비용으로 바뀌는 실행 계획들을 파악하는 것입니다.**

- 각 단위 작업의 비용이 변경됐을때 예상 할 수 있는 결과 (각 단위 작업의 비용 조절을 연습해볼 수 있음)
    - `key_compare_cost`
        - 비용을 높이면 MySQL 서버 옵티마이저가 가능하면 정렬을 수행하지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.
    - `row_evaluate_cost`
        - 비용을 높이면 풀 스캔을 실행하는 쿼리들의 비용이 높아지고, MySQL 서버 옵티마이저는 가능하면 인덱스 레인지 스캔을 사용하는 실행 계획을 선택할 가능성이 높아진다.
    - `disk_temptable_create_cost`와 `disk_temptable_row_cost`
        - 비용을 높이면 MySQL 옵티마이저는 디스크에 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.
    - `memory_temptable_create_cost`와 `memory_temptable_row_cost`
        - 비용을 높이면 MySQL 서버 옵티마이저는 메모리 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.
    - `io_block_read_cost`
        - 비용이 높아지면 MySQL 서버 옵티마이저는 가능하면 InnoDB 버퍼 풀에 데이터 페이지가 많이 적재돼 있는 인덱스를 사용하는 실행 계획을 선택할 가능성이 높아진다.
    - `memory_block_read_cost`
        - 비용이 높아지면 MySQL 서버는 InnoDB 버퍼 풀에 적재된 데이터 페이지가 상대적으로 적다고 하더라도 그 인덱스를 사용할 가능성이 높아진다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/582f1f99-1ac4-4cf7-8776-01f9fe442bc2)
