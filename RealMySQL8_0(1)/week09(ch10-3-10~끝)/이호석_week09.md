# 📌 10장 실행계획

## ✅ 10.3 실행 계획 분석

### 10.3.10 rows 칼럼

- 옵티마이저는 가능한 처리 방식을 나열하고 비용을 비교해 최종적으로 하나의 실행 계획을 수립합니다.
- 대상 테이블에 얼마나 많은 레코드가 포함돼 있는지 혹은 각 인덱스 값을 분포도가 어떤지를 통계정보를 기준으로 조사해 예측합니다.

- **MySQL 실행 계획의 rows 칼럼값은 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여줍니다.**
    - 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해낸 예상값임
    - 예상값이므로 정확하진 않습니다.
- **rows 칼럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미함**
    - 실행 계획의 rows 칼럼에 출력되는 값과 실제 쿼리의 레코드 건수가 일치하지 않는 경우가 많음
- **rows 칼럼은 인덱스를 사용하는 조건에만 일치하는 레코드 건수를 예측할 수 있습니다.**
    
    ```sql
    # 인덱스가 없는 경우에는 범위가 넓든, 좁든 전체 레코드를 조사해야 한다고 표시됩니다.
    mysql> explain select * from dept_emp where to_date <= '9999-01-01';
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | dept_emp | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |    33.33 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    
    mysql> explain select * from dept_emp where to_date <= '2005-01-01';
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | dept_emp | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |    33.33 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    ```
    

- 큰 범위를 조회하는 쿼리
    
    ```sql
    EXPLAIN
    SELECT * FROM dept_emp WHERE from_date >= '1985-01-01';
    
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | dept_emp | NULL       | ALL  | ix_fromdate   | NULL | NULL    | NULL | 331143 |    50.00 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    ```
    
    - MySQL은 위 쿼리를 처리하기 위해 대략 331,143건의 레코드를 읽을것으로 예측함
    - 따라서 인덱스 레인지 스캔이 아니라 풀 테이블 스캔을 선택함

- 좀 더 작은 범위를 조회하는 쿼리
    
    ```sql
    EXPLAIN
    SELECT * FROM dept_emp WHERE from_date >='2002-07-01';
    +----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+-----------------------+
    | id | select_type | table    | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                 |
    +----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+-----------------------+
    |  1 | SIMPLE      | dept_emp | NULL       | range | ix_fromdate   | ix_fromdate | 3       | NULL |  292 |   100.00 | Using index condition |
    +----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+-----------------------+
    ```
    
    - 292건의 레코드만 읽고 체크하면 원하는 결과를 가져올 수 있다고 예측함
    - 따라서 range 인덱스 스캔을 사용했습니다.
    - 292건은 전체 테이블 건수의 8.8%, from_date 칼럼은 DATE타입으로 key_len은 3바이트 표시
    - 옵티마이저가 예측하는 수치는 대략의 값임, 인덱스되지 않은 칼럼, 균등하게 분포되지 않은 경우와 같은 경우를 위해 8.0부터 히스토그램이 도입됨

---

<br><br>

### 10.3.11 filtered 칼럼

- 옵티마이저는 각 테이블에서 일치하는 레코드 개수를 가능한 정확하게 파악해야 효율적인 실행계획을 세웁니다.
- 모든 조건에서 인덱스를 사용하기 어렵고, 조인을 할때 WHERE절에서 인덱스를 사용하지 못하는 조건에 일치하는 레코드 건수 파악도 중요합니다. (드라이빙 테이블 결정을 위해)

```sql
EXPLAIN
SELECT *
FROM employees e, salaries s
WHERE e.first_name='Matt'
		AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
		AND s.emp_no=e.emp_no
		AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01' AND s.salary BETWEEN 50000 AND 60000;

+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys                    | key          | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_hiredate,ix_firstname | ix_firstname | 58      | const              |  233 |    16.72 | Using where |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY,ix_salary                | PRIMARY      | 4       | employees.e.emp_no |    9 |     4.77 | Using where |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
```

employees 테이블과 salaries 테이블을 조인하는데, `e.first_name='Matt'`와 `s.salary BETWEEN 50000 AND 60000` 조건이 인덱스를 사용할 수 있습니다.

이 경우 각 테이블의 나머지 조건을 모두 합쳐서 최종적으로 일치하는 레코드 건수가 적은 테이블이 드라이빙 테이블이 될 가능성이 높습니다.

**filtered 칼럼의 값은 필터링되어 버려지는 레코드의 비율이 아니라 필터링 되고 남은 레코드의 비율을 의미합니다.**

- 실행계획 분석
    - employees 테이블에서 인덱스 조건에 일치하는 레코드 개수 대략 233건 중 16.72%만 인덱스를 사용하지 못하는 `e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'` 조건에 일치함을 알 수 있습니다.
    - employees 테이블에서 salaries 테이블로 조인을 수행한 레코드 건수는 대략 39건(233 * 0.1672)임을 알 수 있습니다.

- 조인 순서를 반대로 할 경우
    
    ```sql
    EXPLAIN
    SELECT /*+ JOIN_ORDER(s, e) */ * FROM employees e, salaries s
    WHERE e.first_name='Matt'
    		AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
    	  AND s.emp_no=e.emp_no
    		AND S.from_date BETWEEN '1990-01-01' AND '1991-01-01'	AND s.salary BETWEEN 50000 AND 60000;
    
    +----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
    | id | select_type | table | partitions | type   | possible_keys                    | key     | key_len | ref                | rows    | filtered | Extra       |
    +----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
    |  1 | SIMPLE      | s     | NULL       | ALL    | PRIMARY,ix_salary                | NULL    | NULL    | NULL               | 2838426 |     4.77 | Using where |
    |  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY,ix_hiredate,ix_firstname | PRIMARY | 4       | employees.s.emp_no |       1 |     5.00 | Using where |
    +----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
    ```
    
    - salaries 테이블을 드라이빙 테이블로 두면 대략 135392(2838426 * 0.0477)건이 조건에 일치했고, 조인을 수행합니다.

레코드 건수 및 다른 요소들도 감안하여 실행 계획을 수립하지만, 조인의 횟수를 줄이고, 읽은 데이터를 저장할 메모리 사용량을 낮추기 위해 레코드 건수가 적은 테이블을 드라이빙 테이블로 선택할 가능성이 높습니다.

결국 filtered 칼럼의 정확도에 따라 조인의 성능이 달라집니다. (8.0부터는 이를 위해 히스토그램 도입됨)

---

<br><br>

### 10.3.12 Extra 칼럼

- 쿼리 실행 계획에서 성능에 관련된 중요한 내용이 Extra 칼럼에 자주 표시됩니다.
- Extra 칼럼에는 고정된 몇 개의 문장이 표시됩니다. (일반적으로 2~3개씩 표시됨)
- 주로 내부적인 처리 알고리즘에 대해 조금 더 깊이 있는 내용을 보여주는 경우가 많음
    - MySQL 버전이 올라가면 더 추가될 것으로 보임

---

<br>

**const row not found**

- 쿼리 실행 계획에서 const 접근 방법으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 존재하지 않으면 표시됨

---

<br>

**Deleting all rows (8.0부터는 실행 계획에서 출력되지 않음)**

- MyISAM 스토리지 엔진과 같이 스토리지 엔진의 핸들러 차원에서 테이블의 모든 레코드를 삭제하는 기능을 제공하는 스토리지 엔진 테이블인 경우 Extra 칼럼에 해당 문구가 표시됨
- WHERE 조건절이 없는 DELETE 문장의 실행 계획에서 자주 표시됨
- 테이블의 모든 레코드를 삭제하는 핸들러 기능(API)을 한 번 호출함으로써 처리됨을 의미합니다.
- 다만 테이블의 모든 데이터 삭제는 DELETE보다 TRUNCATE TABLE 명령을 권장합니다.

---

<br>

**Distinct**

```sql
EXPLAIN
SELECT DISTINCT d.dept_no
FROM departments d, dept_emp de WHERE de.dept_no=d.dept_no;

+----+-------------+-------+------------+-------+---------------------+-------------+---------+---------------------+-------+----------+------------------------------+
| id | select_type | table | partitions | type  | possible_keys       | key         | key_len | ref                 | rows  | filtered | Extra                        |
+----+-------------+-------+------------+-------+---------------------+-------------+---------+---------------------+-------+----------+------------------------------+
|  1 | SIMPLE      | d     | NULL       | index | PRIMARY,ux_deptname | ux_deptname | 162     | NULL                |     9 |   100.00 | Using index; Using temporary |
|  1 | SIMPLE      | de    | NULL       | ref   | PRIMARY             | PRIMARY     | 16      | employees.d.dept_no | 41392 |   100.00 | Using index; Distinct        |
+----+-------------+-------+------------+-------+---------------------+-------------+---------+---------------------+-------+----------+------------------------------+
```

- 실제 조회하려는 값은 dept_no임, departments와 dept_emp 테이블에 모두 존재하는 dept_no만 중복 없이 유니크하게 가져오는 쿼리 입니다.
- ~~즉, 두 테이블을 조인하고 그 결과에 다시 DISTINCT 처리를 한 것 입니다.~~ → 꼭 필요한 쿼리만 하지 않나?

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b80e1001-3a08-41a2-a3f8-f6c70fcb7113)


위 그림을 보면 굳이 조인하지 않아도 되는 항목은 무시합니다.

---

<br>

**FirstMatch**

- FirstMatch 전략이 사용되면 FirsMatch(table_name)이라는 메시지를 칼럼에 출력합니다.

```sql
EXPLAIN SELECT *
FROM employees e
WHERE e.first_name='Matt'
		AND e.emp_no IN (
					SELECT t.emp_no FROM titles t
					WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30'
				);

+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
| id | select_type | table | partitions | type | possible_keys        | key          | key_len | ref                | rows | filtered | Extra                                   |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              |  233 |   100.00 | NULL                                    |
|  1 | SIMPLE      | t     | NULL       | ref  | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no |    1 |    11.11 | Using where; Using index; FirstMatch(e) |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
```

- FirstMatch 메시지에 함께 표시되는 테이블명은 기준 테이블을 의미합니다.
- 위 실행 계획의 경우 employees 테이블을 기준으로 title 테이블에서 첫 번째로 일치하는 한 건만 검색함을 의미합니다.

---

<br>

**Full scan on NULL key**

- `"Col1 IN (SELECT col2 FROM ...)"`과 같은 조건을 가진 쿼리에서 자주 발생함
- col1이 NULL이라면 `NULL IN (…)`과 같이 변경됨
- `SQL 표준에서 NULL은 알 수 없는 값으로 정의`하고 있습니다. 또한 NULL 연산에 대한 규칙이 있는데 해당 규칙은  연산을 수행하기 위해 다음과 같이 비교돼야 합니다.
    - 서브쿼리가 1건이라도 결과 레코드를 가진다면 최종 비교 결과는 NULL
    - 서브쿼리가 1건도 결과 레코드를 가지지 않는다면 최종 비교 결과는 FALSE
- 결국 col1이 NULL이면 서브쿼리에 사용된 테이블에 대해 풀 테이블 스캔을 해야만 결과를 알 수 있습니다.

```sql
EXPLAIN
SELECT d.dept_no, NULL IN (SELECT d2.dept_name FROM departments d2) FROM departments d;

+----+-------------+-------+------------+----------------+---------------+-------------+---------+------+------+----------+-------------------------------------------------+
| id | select_type | table | partitions | type           | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                           |
+----+-------------+-------+------------+----------------+---------------+-------------+---------+------+------+----------+-------------------------------------------------+
|  1 | PRIMARY     | d     | NULL       | index          | NULL          | ux_deptname | 162     | NULL |    9 |   100.00 | Using index                                     |
|  2 | SUBQUERY    | d2    | NULL       | index_subquery | ux_deptname   | ux_deptname | 162     | func |    1 |   100.00 | Using where; Using index; Full scan on NULL key |
+----+-------------+-------+------------+----------------+---------------+-------------+---------+------+------+----------+-------------------------------------------------+
```

- NOT NULL 제약조건이 없어도 col1이 절대 NULL이 될 수 없다는 걸 옵티마이저에게 알려줄 수 있습니다.
    
    ```sql
    SELECT *
    FROM tb_test1
    WHERE col1 IS NOT NULL
    	AND col1 IN (SELECT co12 FROM tb_test2);
    ```
    
    col1이 NULL이면 IS NOT NULL 조건을 통과하지 못하므로 후속 조건을 실행하지 않습니다.
    
- **또한 해당 Full scan on NULL key 코멘트가 Extra 칼럼에 표시됐다 하더라도 IN, NOT IN 연산자의 왼쪽 값이 NULL이 없다면 풀 테이블 스캔은 발생하지 않습니다.**
- **반대로 NULL인 레코드가 있고 서브쿼리에 개별적인 WHERE 조건이 있다면 상당한 성능 문제를 일으킬 수 있음**

---

<br>

**Impossible HAVING**

- 쿼리에 사용된 HAVING 절의 조건을 만족하는 레코드가 없다면 표시됩니다.

```sql
EXPLAIN
SELECT e.emp_no, COUNT(*) AS cnt
FROM employees e
WHERE e.emp_no=10001
GROUP BY e.emp_no
HAVING e.emp_no IS NULL; # PK이므로 NULL 될 수 없음

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra             |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Impossible HAVING |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------+
```

- 쿼리 실행 계획에서 Extra 칼럼에 표시된다면 쿼리를 점검 하는것이 좋습니다.

---

<br>

**Impossible WHERE**

- WHERE 조건이 항상 FALSE가 될 수 밖에 없는 경우 표시됩니다.

```sql
EXPLAIN
SELECT * FROM employees WHERE emp_no IS NULL;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra            |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Impossible WHERE |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
```

---

<br>

**LooseScan**

- 세미 조인 최적화 중에서 LooseScan 최적화 전략이 사용되면 표시됩니다.

```sql
EXPLAIN
SELECT * FROM departments d WHERE d.dept_no IN (
		SELECT de.dept_no FROM dept_emp de);

# 본인은 FirstMatch가 적용됨 
+----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref                 | rows  | filtered | Extra                      |
+----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
|  1 | SIMPLE      | d     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL                |     9 |   100.00 | NULL                       |
|  1 | SIMPLE      | de    | NULL       | ref  | PRIMARY       | PRIMARY | 16      | employees.d.dept_no | 41392 |   100.00 | Using index; FirstMatch(d) |
+----+-------------+-------+------------+------+---------------+---------+---------+---------------------+-------+----------+----------------------------+
```

---

<br>

**No matching min/max row**

- 쿼리에서 MIN(), MAX()와 같은 집합 함수가 있을때 WHERE 조건절을 만족하는 레코드가 한 건도 없다면 표시됩니다.
- 그리고 MIN(), MAX()의 결과로 NULL이 반환됩니다.

```sql
EXPLAIN
SELECT MIN(dept_no), MAX(dept_no)
FROM dept_emp WHERE dept_no='';

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                   |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No matching min/max row |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
```

`No matching …`, `Impossible …` 등의 메시지는 쿼리 오류가 아니라 쿼리의 실행 계획을 산출하기 위한 기초 자료가 없음을 표현하는 것입니다. (문법적인 오류X)

---

<br>

**no matching row in const table**

- 아래 쿼리와 같이 조인이 사용된 테이블에서 const 방법으로 접근할 때 일치하는 레코드가 없다면 표시됩니다.

```sql
EXPLAIN
SELECT *
FROM dept_emp de,
	(SELECT emp_no FROM employees WHERE emp_no = 0) tb1
WHERE tb1.emp_no=de.emp_no AND de.dept_no='d005';

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | no matching row in const table |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
```

이 역시 실행 계획 수립을 위한 기초자료가 부족함을 말합니다.

---

<br>

**No matching rows after partition prunig**

- 파티션된 테이블에 대한 UPDATE 또는 DELETE 명령시 대상 레코드가 없다면 표시됩니다.

```sql
CREATE TABLE employees_parted (
		emp_no int NOT NULL,
		birth_date DATE NOT NULL, 
		first_name VARCHAR (14) NOT NULL,
		last_name VARCHAR(16) NOT NULL,
		gender ENUM('M', 'F') NOT NULL,
		hire_date DATE NOT NULL,
		PRIMARY KEY (emp_no, hire_date)
) PARTITION BY RANGE COLUMNS (hire_date)
(PARTITION p1986_1990 VALUES LESS THAN ('1991-01-01'),
 PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),
 PARTITION p1996_2000 VALUES LESS THAN ('2001-01-01'),
 PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

INSERT INTO employees_parted SELECT * FROM employees;

SELECT MAX(hire_date) FROM employees_parted;
+----------------+
| MAX(hire_date) |
+----------------+
| 2000-01-28     |
+----------------+

EXPLAIN DELETE FROM employees_parted WHERE hire_date >='2020-01-01';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                    |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------------------+
|  1 | DELETE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No matching rows after partition pruning |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------------------+

# 실제 레코드는 없지만, 파티션에 포함되면 메시지가 표시되지 않음
EXPLAIN DELETE FROM employees_parted WHERE hire_date < '1990-01-01';
+----+-------------+------------------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table            | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+------------------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | DELETE      | employees_parted | p1986_1990 | ALL  | NULL          | NULL | NULL    | NULL | 190322 |   100.00 | Using where |
+----+-------------+------------------+------------+------+---------------+------+---------+------+--------+----------+-------------+
```

- 최대 hire_date의 값이 2000년인데 2020년보다 큰 칼럼에 대해 삭제하게 되면 해당 메시지가 표시됩니다.
- 이는 단순히 삭제할 레코드가 없음이 아니라, **대상 파티션이 없다는 것을 의미합니다.**
- 실제 레코드는 없어도 파티션은 있기 때문에 해당 메시지가 없고 파티션이 선택됩니다.

---

<br>

**No tables used**

- FROM 절이 없는 쿼리 문장이나 “FROM DUAL”(가상의 상수 테이블) 형태의 쿼리 실행 계획은 별도의 테이블을 선택하지 않기에 해당 메시지가 표시됩니다.
(MySQL은 FROM절이 없는 쿼리도 허용)

```sql
EXPLAIN SELECT 1;
EXPLAIN SELECT 1 FROM dual;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```

---

<br>

**Not exists**

- 안티조인은 NOT IN, NOT EXISTS 연산자를 통해 A테이블에는 존재하지만 B 테이블에 없는 값을 조회하는 상황을 말합니다.
- 안티조인 처리를 아우터 조인(LEFT OUTER JOIN)을 이용해서도 구현할 수 있습니다.
    - **일반적으로 안티조인으로 처리해야 할 레코드 건수가 많을때 아우터 조인을 이용하면 빠른 성능을 낼 수 있음**

```sql
EXPLAIN
SELECT *
FROM dept_emp de
	LEFT JOIN departments d ON de.dept_no=d.dept_no
WHERE d.dept_no IS NULL;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                                  |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------------------+
|  1 | SIMPLE      | de    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL                                                   |
|  1 | SIMPLE      | d     | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |      9 |    11.11 | Using where; Not exists; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------------------+
```

- 위 쿼리는 dept_emp에는 있지만, departments에는 없는 dept_no를 조회하는 쿼리를 아우터 조인을 통해 처리했습니다.
- **아우터 조인을 이용해 안티-조인을 수행하는 실행계획의 Extra 칼럼에 Not exists가 표시됩니다.**
- Not exists 메시지는 조인할때 departments 테이블의 레코드가 존재하는지 아닌지만 판단한다는 것을 의미합니다.
- 조인 조건에 일치하는 레코드가 여러 건이 있다고 하더라도 딱 1건만 조회하고 처리를 완료하는 최적화를 의미함

---

<br>

**Plan isn’t ready yet**

- 8.0부터는 다른 커넥션에서 실행 중인 쿼리의 실행 계획을 볼 수 있습니다.

```sql
show processlist;
+-----+-----------------+-----------+-----------+---------+--------+------------------------+----------------------------------------+
| Id  | User            | Host      | db        | Command | Time   | State                  | Info                                   |
+-----+-----------------+-----------+-----------+---------+--------+------------------------+----------------------------------------+
|   5 | event_scheduler | localhost | NULL      | Daemon  | 202300 | Waiting on empty queue | NULL                                   |
| 281 | root            | localhost | employees | Query   |     13 | User sleep             | select * from employees where sleep(1) |
| 282 | root            | localhost | NULL      | Query   |      0 | init                   | show processlist                       |
+-----+-----------------+-----------+-----------+---------+--------+------------------------+----------------------------------------+

explain for connection 281;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299113 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
```

- 281번 프로세스의 실행 계획을 살펴보면 풀 테이블 스캔을 하고 있습니다.
- EXPLAIN FOR CONNECTION 명령은 옵티마이저가 의도된 인덱스를 잘 사용하는지 여부를 확인할때 유용합니다.
- 이때 Extra에 `Plan is not ready yet` 이라는 메시지가 표시될 때가 있는데 이는 해당 커넥션에서 아직 실행 계획을 수립하지 못한 상태에서 조회했을때 표시됩니다.

---

<br>

**Range checked for each record(index map:N)**

- 레코드마다 인덱스 레인지 스캔을 체크한다라는 의미입니다.

```sql
EXPLAIN
SELECT *
FROM employees e1, employees e2
WHERE e2.emp_no > e1.emp_no;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                          |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+------------------------------------------------+
|  1 | SIMPLE      | e1    | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL | 299113 |   100.00 | NULL                                           |
|  1 | SIMPLE      | e2    | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL | 299113 |    33.33 | Range checked for each record (index map: 0x1) |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+------------------------------------------------+
```

- employees 테이블에 1억건의 데이터가 있을때 (1번부터 순차적인 id)
- e1.emp_no가 1이라면 e2 테이블에선 1억건 전부를 읽어야 하고, e1.emp_no가 1억이라면 e2테이블은 한 건만 읽으면 됩니다.
    - 전자의 경우 풀 테이블 스캔을 활용할 것이고, 후자의 경우 인덱스 레인지 스캔으로 접근할 것입니다.
- 이렇게 레코드마다 인덱스 레인지 스캔을 체크합니다.

- 실행 계획 처리 시나리오
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f8359aad-4833-49da-873d-dfca0f0cd855)

    
- `index map: 0x1 의미`
    - 0x1은 16진수로 표현된 숫자 입니다.
    - 해당 표시는 사용할지 말지를 판단하는 후보 인덱스의 순번을 나타냅니다.
    - 순번은 SHOW CREATE TABLE employees 명령으로 테이블 구조를 조회 했을때 일치하는 순번의 인덱스를 의미합니다.
    (여기서는 첫번째 인덱스를 의미함)
    - Range checked for each record(index map:N) 메시지가 있다면 type=ALL로 표시됩니다.
    - 이는 index map 후보 인덱스를 사용할지 여부 검토를 했을때 사용하지 않는다면 풀 테이블 스캔을 하기 때문에 ALL로 표시됩니다.
- 조금 더 복잡한 index map 예제
    
    ```sql
    CREATE TABLE to_member (
    		mem_id INTEGER NOT NULL,
    		mem _name VARCHAR(100) NOT NULL,
    		mem_nickname VARCHAR (100) NOT NULL,
    		mem_region TINYINT,
    		mem_gender TINYINT,
    		mem_phone VARCHAR (25),
    		PRIMARY KEY (mem_id),
    		INDEX ix_nick_name (mem_nickname, mem_name),
    		INDEX ix nick_region (mem_nickname, mem_region),
    		INDEX ix nick gender (mem_nickname, mem gender),
    		INDEX ix nick_ phone (mem_nickname, mem_phone)
    );
    ```
    
    위와 같은 테이블을 대상으로 실행한 쿼리의 실행계획에서 index map: 0x19라고 나왔을때
    
    - 0x19 = 11001(2)  입니다. 비트 배열의 해석은 다음과 같습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e5551521-02dd-4c41-979a-e2946220257a)

        
    - 위 표를 기반으로 0x19는 다음 3개의 인덱스가 후보로 선정됐음을 의미합니다
        - PRIMARY KEY
        - ix_nick_gender
        - ix_nick_phone
    - 다만 실제로 어떤 인덱스가 사용됐는지는 알 수 없고, 단순히 1로 set된 자리의 인덱스가 후보군이라는것만 알 수 있습니다.
    
- 참고
    
    Range checked for each record가 표시되는 쿼리가 많이 실행되는 MySQL서버에선 SHOW GLOBAL STATUS의 상태값 중 Select_range_check의 값이 크게 나타납니다.
    

---

<br>

**Recursive**

- 8.0버전 부터는 Common Table Expression을 이용해 재귀 쿼리를 작성할 수 있습니다.

```sql
WITH RECURSIVE cte (n) AS
(
	SELECT 1
	UNION ALL
	SELECT n + 1 FROM cte WHERE n < 5
)
SELECT * FROM cte;

+------+
| n    |
+------+
|    1 |
|    2 |
|    3 |
|    4 |
|    5 |
+------+
```

1. “n”이라는 칼럼 하나를 가진 cte라는 이름의 내부 임시테이블 생성
2. “n” 칼럼의 값이 1부터 5까지 1씩 증가하여 5개의 레코드를 만들고 cte 임시테이블에 저장
3. WITH절 이후 임시테이블 cte를 풀 스캔해 결과를 반환합니다. 이런 재귀쿼리의 실행 계획은 Recursive 구문이 표시됩니다.
    
    ```sql
    +----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------+
    | id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                  |
    +----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------+
    |  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL                   |
    |  2 | DERIVED     | NULL       | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used         |
    |  3 | UNION       | cte        | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Recursive; Using where |
    +----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------+
    ```
    

- 주의
    
    WITH 구문을 이용한 CTE사용이 무조건 Recursive를 표시하진 않음, WITH 구문이 `재귀 CTE`로 사용될 경우에만 Recursive 메시지가 표시됩니다.
    
    아래 예제는 비재귀 CTE 이므로 Recursive가 표시되지 않습니다.
    
    ```sql
    EXPLAIN WITH RECURSIVE cte (n) AS
    (
    	SELECT 1
    )
    SELECT * FROM cte;
    
    +----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
    | id | select_type | table      | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
    +----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
    |  1 | PRIMARY     | <derived2> | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL           |
    |  2 | DERIVED     | NULL       | NULL       | NULL   | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
    +----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
    ```
    

---

<br>

**Rematerialize**

- 8.0부터 사용된 LATERAL JOIN 기능은 선행 테이블의 레코드별로 서브쿼리를 실행해 임시테이블에 결과를 저장합니다. 이 과정을 Rematerializing이라고 합니다.

```sql
EXPLAIN
SELECT * FROM employees e
		LEFT JOIN LATERAL (
				SELECT *
				FROM salaries s
				WHERE s.emp_no=e.emp_no
				ORDER BY s.from_date DESC LIMIT 2) s2 ON s2.emp_no=e.emp_no
	WHERE e.first_name='Matt';

+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys | key          | key_len | ref                | rows | filtered | Extra                      |
+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
|  1 | PRIMARY           | e          | NULL       | ref  | ix_firstname  | ix_firstname | 58      | const              |  233 |   100.00 | Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0>  | 4       | employees.e.emp_no |    2 |   100.00 | NULL                       |
|  2 | DEPENDENT DERIVED | s          | NULL       | ref  | PRIMARY       | PRIMARY      | 4       | employees.e.emp_no |    9 |   100.00 | Using filesort             |
+----+-------------------+------------+------------+------+---------------+--------------+---------+--------------------+------+----------+----------------------------+
```

- employees 테이블의 레코드마다 salaries 테이블에서 emp_no가 동일한 레코드들을 from_date 역순으로 2개만 가져와 derived2라는 임시테이블에 저장합니다.
- 이후 employees 테이블과 derived2 테이블이 조인합니다.
    - 이때 derived2 테이블은 employees 테이블의 레코드마다 새로운 내부 임시 테이블을 생성합니다.
- **이렇게 매번 새로운 임시테이블이 생성되면 Rematerialize 문구가 표시됩니다.**

---

<br>

**Select tables optimized away**

- `MIN(), MAX()만 SELECT절에 사용`되거나, `GROUP BY로 MIN(), MAX()를 조회하는 쿼리`에 **인덱스 오름차순 or 내림차순으로 1건만 읽는 형태의 최적화가 적용**된다면 표시되는 문구입니다.
- 혹은 MyISAM 테이블에 대해서 GROUP BY없이 COUNT(*)만 SELECT해도 이런 형태의 최적화가 적용됩니다.
    - MyISAM은 테이블 전체 레코드 건수를 별도로 관리하기 때문에 인덱스 및 데이터를 읽지 않아도 빠르게 조회 가능(WHERE 있다면 불가)

```sql
# emp_no는 PK이므로 1건만 읽음
EXPLAIN
SELECT MAX(emp_no), MIN(emp_no) FROM employees;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+

# (emp_no, from_date)가 PK이므로 이들을 조합해 1건만 읽을 수 있음
EXPLAIN
SELECT MAX(from_date), MIN(from_date) FROM salaries WHERE emp_no=10002;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
```

- 1번 쿼리
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/06ff138d-0f49-4bc9-bc6d-3f5431b77ee9)

    
- 2번 쿼리
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e9c5c9df-b525-4655-9728-48f185f55466)

    

---

<br>

**Start temporary, End temporary**

- 세미 조인 최적화 중 Duplicate Weed-out 최적화 전략이 사용되면 표시됩니다.

```sql
EXPLAIN 
SELECT * 
FROM employees e 
WHERE e.emp_no IN 
		(SELECT s.emp_no 
			FROM salaries s 
			WHERE s.salary>150000);
+----+-------------+-------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+-------------------------------------------+
| id | select_type | table | partitions | type   | possible_keys     | key       | key_len | ref                | rows | filtered | Extra                                     |
+----+-------------+-------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+-------------------------------------------+
|  1 | SIMPLE      | s     | NULL       | range  | PRIMARY,ix_salary | ix_salary | 4       | NULL               |   36 |   100.00 | Using where; Using index; Start temporary |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY           | PRIMARY   | 4       | employees.s.emp_no |    1 |   100.00 | End temporary                             |
+----+-------------+-------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+-------------------------------------------+
```

- Duplicate Weed_out은 중복건을 제거하기 위해 내부 임시 테이블을 사용합니다.
- 따라서 임시 테이블에 저장되는 테이블 식별을 위해 조인 첫번째 테이블과 마지막 테이블에 해당 문구를 표시합니다.
- `salaries ~ employees` 테이블의 내용을 임시 테이블에 저장한다는 의미입니다.

---

<br>

**unique row not found**

- 두 개의 테이블이 각각 유니크 칼럼으로 아우터 조인을 수행하는 쿼리에서 아우터  테이블에 일치하는 레코드가 존재하지 않을때 표시됩니다.

```sql
# 테스트 케이스를 위한 테스트용 테이블 생성
CREATE TABLE tb_test1 (fdpk INT, PRIMARY KEY(fdpk));
CREATE TABLE tb_test2 (fdpk INT, PRIMARY KEY(fdpk));

# 생성된 테이블에 레코드 삽입
INSERT INTO tb_test1 VALUES (1), (2);
INSERT INTO tb_test2 VALUES (1);

# tb_test1을 tb_test2에 left outer join
EXPLAIN
SELECT t1.fdpk
FROM tb_test1 t1
	LEFT JOIN tb_test2 t2 ON t2.fdpk=t1.fdpk WHERE t1.fdpk=2;

+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+----------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra                |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+----------------------+
|  1 | SIMPLE      | t1    | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index          |
|  1 | SIMPLE      | t2    | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    0 |     0.00 | unique row not found |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+----------------------+
```

- tb_test2는 fdpk=2인 레코드가 없으므로 해당 문구가 출력됩니다.

---

<br>

**Using filesort**

- ORDER BY시 적절한 인덱스를 찾지 못하면 MySQL 서버가 조회된 레코드를 다시 정렬해야 합니다.
- 이때 정렬용 메모리 버퍼에 복사해 퀵 혹은 힙 소트 알고리즘을 이용해 정렬을 수행한다는 의미로 Using filesort가 표시됩니다.

```sql
EXPLAIN
SELECT * FROM employees
ORDER BY last_name DESC;

+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra          |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299113 |   100.00 | Using filesort |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
```

- last_name 칼럼에 인덱스가 없으므로 정렬 작업시 소트 버퍼를 사용합니다.
- Extra에 Using filesort를 출력하는 쿼리는 많은 부하를 일으키므로 가능하면 쿼리 튜닝 혹은 인덱스를 생성하는것이 좋습니다.

---

<br>

**Using index(커버링 인덱스)**

- 데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을때 표시됩니다.
- 쿼리에서 인덱스 처리시 가장 큰 부하를 차지하는 부분은 인덱스 검색에서 일치하는 키 값들의 레코드를 읽기 위해 데이터 파일을 검색하는 작업임
- 최악의 경우 인덱스를 통해 검색한 결과 전부를 디스크에서 읽어와야할 수 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d8d71afe-540a-46d4-8a39-7df94802d4b6)


```sql
EXPLAIN
SELECT first_name, birth_date
FROM employees
WHERE first_name BETWEEN 'Babette' AND 'Gad';

+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | ix_firstname  | NULL | NULL    | NULL | 299113 |    31.36 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
```

- `ix_firstname` 인덱스를 사용했을때 위 쿼리에서 발생하는 비효율은 다음과 같습니다.
    - ix_firstname을 통해 where 조건절에 일치하는 레코드 5만개를 찾아옴
    - birth_date를 읽기 위해 데이터 페이지를 5만번 읽음
    - 위 같은 비효율 때문에 옵티마이저는 인덱스가 아니라 풀 테이블 스캔이 효율적이라고 판단함

- birth_date를 제외한 쿼리
    
    ```sql
    EXPLAIN
    SELECT first_name
    FROM employees
    WHERE first_name BETWEEN 'Babette' AND 'Gad';
    
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    | id | select_type | table     | partitions | type  | possible_keys | key          | key_len | ref  | rows  | filtered | Extra                    |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    |  1 | SIMPLE      | employees | NULL       | range | ix_firstname  | ix_firstname | 58      | NULL | 93802 |   100.00 | Using where; Using index |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    ```
    
    - first_name 칼럼만 있으면 쿼리를 완료할 수 있고, 인덱스만으로도 해결 가능하기에 매우 빠른 속도로 처리합니다.
    - `이렇게 인덱스만으로 쿼리를 처리할 수 있는 경우를 커버링 인덱스라고 합니다.`
    
- InnoDB의 모든 테이블은 클러스터링 인덱스로 구성돼 있습니다. 따라서 세컨더리 인덱스는 데이터 레코드의 주솟값으로 PK값을 가집니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cd3ca7ce-8ba6-45a5-ab36-2c9131c969e0)

    
- 따라서 first_name 칼럼만으로 인덱스를 만들어도 PK인 emp_no칼럼이 같이 저장되는 효과를 냅니다.
- 이런 클러스터링 인덱스 특징 덕분에 커버링 인덱스로 처리될 가능성이 높습니다.
    
    ```sql
    # PK를 같이 조회해도 레인지 스캔으로 처리합니다.
    EXPLAIN
    SELECT emp_no, first_name
    FROM employees
    WHERE first_name BETWEEN 'Babette' AND 'Gad';
    
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    | id | select_type | table     | partitions | type  | possible_keys | key          | key_len | ref  | rows  | filtered | Extra                    |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    |  1 | SIMPLE      | employees | NULL       | range | ix_firstname  | ix_firstname | 58      | NULL | 93802 |   100.00 | Using where; Using index |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+-------+----------+--------------------------+
    ```
    
- 레코드 건수에 따라 차이는 있지만 커버링 인덱스 사용 유무에서 성능 차이는 수십배에서 수백배까지 날 수 있습니다.
- 다만 무조건 커버링 인덱스로 처리하기 위해 인덱스에 많은 칼럼을 추가하는것도 좋지 않습니다. (너무 과도한 인덱스의 크기 및 저장, 변경 작업 매우 느려짐)
- 접근 방법(type칼럼)이 `eq_ref`, `ref`, `range`, `index_merge`, `index`등 인덱스를 사용하는 실행계획에서 Using index가 표시될 수 있습니다.
- 커버링 인덱스는 실행 계획의 type에 관계없이 사용될 수 있습니다.

---

<br>

**Using index condition**

- 옵티마이저가 인덱스 컨디션 푸시다운 최적화를 사용하면 표시됩니다.

```sql
EXPLAIN SELECT * FROM employees WHERE last_name='Acton' AND first_name LIKE '%sal';

+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table     | partitions | type | possible_keys         | key                   | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |  189 |    11.11 | Using index condition |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
```

---

<br>

**Using index for group-by**

- MySQL 서버의 GROUP BY처리는 기준 칼럼을 이용해 정렬 작업을 수행하고, 정렬된 결과를 그루핑하는 고부하 작업을 필요로 합니다.
- GROUP BY처리에 인덱스를 이용하면 인덱스 칼럼을 순서대로 읽으며 그루핑하면 되므로 효율적이고 빠릅니다.
- GROUP BY 처리에 인덱스가 사용되면 해당 메시지가 표시됩니다.
- GROUP BY 처리를 위해 인덱스를 순서대로 쭉 읽는 것과 인덱스의 필요한 부분만 듬성듬성 읽는 루스 인덱스 스캔은 다릅니다.

- 타이트 인덱스 스캔(인덱스 스캔)을 통한 GROUP BY 처리
    - 인덱스를 이용해 GROUP BY 처리를 해도 AVG(), SUM(), COUNT()처럼 모든 인덱스를 다 읽어야 할 때는 필요 레코드만 듬성듬성 읽을 수 없습니다.
    - 이런 쿼리는 GROUP BY를 위해 인덱스를 사용하지만, 루스 인덱스 스캔이라 불리진 않습니다.
    - 이런 상황에선 Using index for group-by가 출력되지 않습니다.
    
    ```sql
    EXPLAIN
    SELECT first_name, COUNT(*) AS counter
    FROM employees GROUP BY first_name;
    
    +----+-------------+-----------+------------+-------+------------------------------------+--------------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type  | possible_keys                      | key          | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+-------+------------------------------------+--------------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | index | ix_firstname,ix_lastname_firstname | ix_firstname | 58      | NULL | 299113 |   100.00 | Using index |
    +----+-------------+-----------+------------+-------+------------------------------------+--------------+---------+------+--------+----------+-------------+
    ```
    

- 루스 인덱스 스캔을 통한 GROUP BY 처리
    - 단일 칼럼으로 구성된 인덱스에선 그루핑 칼럼 제외 아무것도 조회하지 않는 쿼리에서 루스 인덱스 스캔을 사용할 수 있습니다.
    - 또한 `다중 칼럼 인덱스`, `MIN()`, `MAX()`와 같이 인덱스의 첫번째 혹은 마지막 레코드만 읽는 쿼리의 경우 루스 인덱스 스캔이 사용될 수 있습니다.
    (인덱스를 필요한 부분만 읽을 수 있음)
    
    ```sql
    EXPLAIN
    SELECT emp_no, MIN(from_date) AS first_changed_date, MAX(from_date) AS last_changed_date
    FROM salaries
    GROUP BY emp_no;
    
    +----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
    | id | select_type | table    | partitions | type  | possible_keys     | key     | key_len | ref  | rows   | filtered | Extra                    |
    +----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
    |  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,ix_salary | PRIMARY | 4       | NULL | 294784 |   100.00 | Using index for group-by |
    +----+-------------+----------+------------+-------+-------------------+---------+---------+------+--------+----------+--------------------------+
    
    ```
    
    - 위 쿼리는 (emp_no, from_date)인 PK에서 emp_no 그룹별 from_date의 첫번째와 마지막 값을 읽을 수 있기에 루스 인덱스 스캔 방식으로 처리할 수 있습니다.

GROUP BY에서 인덱스를 사용하려면 우선 GROUP BY 조건에서 인덱스를 사용할 수 있는 요건이 되는지를 확인해야 합니다.

또한 WHERE 절에 사용되는 인덱스에 의해서도 GROUP BY 절의 인덱스 사용 여부가 영향을 받습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/93c43719-faff-4f37-bbc2-d4bba1c9b750)


- 참고
    
    루스 인덱스 스캔을 사용할 수 있더라도 WHERE 조건에 검색된 레코드 건수가 적으면 풀 인덱스 스캔을 해도 빠르기 때문에 사용하지 않을 수 있습니다.
    (옵티마이저의 손익 분기점 판단)
    

---

<br>

**Using index for skip scan**

- MySQL 옵티마이저가 인덱스 스킵 스캔 최적화를 사용하면 해당 메시지를 표시합니다.

```sql
ALTER TABLE employees
	ADD INDEX ix_gender_birthdate (gender, birth_date);

EXPLAIN
SELECT gender, birth_date
FROM employees
WHERE birth_date >='1965-02-01';

+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+------+----------+----------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys       | key                 | key_len | ref  | rows | filtered | Extra                                  |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+------+----------+----------------------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_gender_birthdate | ix_gender_birthdate | 4       | NULL |  299 |   100.00 | Using where; Using index for skip scan |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+------+----------+----------------------------------------+
```

---

<br>

**Using join buffer(Block Nested Loop), Using join buffer(Batched Key Access), Using join buffer(hash join)**

- 조인 칼럼에 대한 드리븐 테이블의 인덱스 유무는 중요합니다.
- 두 테이블에서 하나만 인덱스가 있다면 일반적으로 인덱스가 없는 테이블을 드라이빙 테이블로 선정합니다.
- 만약 드리븐 테이블에 적절한 인덱스가 없다면 `블록 네스티드 루프 조인` or `해시 조인을 사용하는데`, 이때 조인 버퍼를 사용합니다.
    - 따라서 조인 버퍼를 사용하는 실행 계획은 Using join buffer라는 메시지가 표시됩니다.
- 조인 버퍼 사이즈는 부족하거나 과다해지면 안됩니다. (일반적은 1MB정도면 충분)
    - 대용량의 쿼리라면 조인 버퍼를 더 크게 설정
- 8.0부터 추가된 해시조인도 조인 버퍼를 이용합니다.

```sql
EXPLAIN
SELECT *
FROM dept_emp de, employees e
WHERE de.from_date > '2005-01-01' AND e.emp_no < 10904;

+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | de    | NULL       | range | ix_fromdate   | ix_fromdate | 3       | NULL |    1 |   100.00 | Using index condition                      |
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY       | PRIMARY     | 4       | NULL |  903 |   100.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
```

- 위와 같은 카테시안 조인에선 항상 조인 버퍼를 사용합니다.
- 5.6 버전부터는 `Batched key Access`, `Hash join`이 도입되어 Using join buffer + 추가 문구에 알고리즘이 표시됩니다.

---

<br>

**Using MRR**

- MySQL 엔진이 실행 계획을 수립하고, 실행 계획에 맞게 핸들러 API를 호출해 쿼리를 처리합니다.
- 하지만, 옵티마이저와 달라 InnoDB를 포함한 스토리지 엔진 레벨에선 쿼리 실행의 전체적인 부분을 알지 못하기 때문에 최적화에 한계가 있습니다.
- 많은 레코드를 읽는 과정에서 스토리지 엔진은 MySQL 엔진이 넘겨주는 키 값을 기준으로 레코드를 한 건씩 읽어 반환하도록 동작합니다.
    - 매번 읽어서 반환하는 레코드가 동일 페이지에 있더라도 레코드 단위로 API 호출이 필요

MySQL에서 `Multi Range Read` 라는 최적화를 도입해 위 같은 단점을 보완했습니다. 

- MySQL 엔진은 여러 개의 키 값을 한 번에 스토리지 엔진으로 전달
- 스토리지 엔진은 넘겨 받은 키 값들을 정렬해 최소한의 페이지 접근만으로 필요 레코드를 읽게 최적화
- MRR 덕분에 각 스토리지 엔진은 디스크 접근을 최소화 합니다.

```sql
EXPLAIN
SELECT /*+ JOIN_ORDER(s, e) */ * 
FROM employees e, 
		salaries s
WHERE e.first_name='Matt'
		AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
		AND s.emp_no=e.emp_no
		AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01' AND s.salary BETWEEN 50000 AND 60000;

# Using MRR이 아니라 풀 테이블 스캔을 함
+----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys                    | key     | key_len | ref                | rows    | filtered | Extra       |
+----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
|  1 | SIMPLE      | s     | NULL       | ALL    | PRIMARY,ix_salary                | NULL    | NULL    | NULL               | 2838426 |     4.77 | Using where |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY,ix_hiredate,ix_firstname | PRIMARY | 4       | employees.s.emp_no |       1 |     5.00 | Using where |
+----+-------------+-------+------------+--------+----------------------------------+---------+---------+--------------------+---------+----------+-------------+
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/fec399aa-2836-4b1a-99e1-af5622350c1c)


- 책의 내용대로라면 salaries 테이블에서 읽은 레코드를 이용해 employees 테이블 검색을 위한 조인 키 값을 모아 MRR 엔진을 전달하고
- MRR 엔진은 키 값을 정렬해 employees 테이블을 최적화된 방법으로 접근했음을 의미합니다.

---

<br>

**Using sort _union(...), Using union(...), Using intersect(...)**

- 실행 계획의 `type=index_merge`로 실행되는 경우 2개 이상의 인덱스가 동시에 사용됩니다.
- Extra 칼럼에는 두 인덱스로부터 읽은 결과의 병합 방식을 설명하기 위해 3가지 메시지 중 하나를 선택합니다.
    - `Using intersect(...)`
        - 인덱스를 사용할 수 있는 조건들이 AND로 연결된 경우 각 처리 결과에서 교집합을 추출해 내는 작업을 수행함
    - `Using union(...)`
        - 인덱스를 사용할 수 있는 조건들이 OR로 연결된 경우 각 처리 결과에서 합집합을 추출해내는 작업을 수행함
        - 동등비교(=)처럼 일치 레코드 건수가 많지 않은 경우 사용됨
    - `Using sort_union(...)`
        - Using union과 같은 작업을 수행하지만 Using union으로 처리될 수 없는 경우(OR로 연결 된 상대적으로 대량의 range 조건들) 이 방식으로 처리됩니다.
        - Using sort_union과 Using union의 차이점은 Using sort_union은 프라이머리 키만 먼저 읽어서 정렬하고 병합한 이후 레코드를 읽어서 반환합니다.
        - 크다, 작다와 같이 상대적으로 많은 레코드에 일치하는 조건이 사용될 경우 사용됨
    

Using union, Using sort_union은 실제로는 비교 조건이 동등이면 전자가, 그렇지 않으면 후자가 사용되는 경향이 있습니다.

---

<br>

**Using temporary**

- MySQL 서버에서 쿼리 처리시 중간 결과를 담아두기 위한 임시 테이블을 사용했다면 표시되는 메시지 입니다.
- 다만 실행 계획만으로 임시테이블이 메모리, 디스크 중 어디에 생성됐는지 판단할 수 없습니다.

```sql
EXPLAIN 
SELECT gender, MIN(emp_no) 
FROM employees 
GROUP BY gender 
ORDER BY MIN(emp_no);
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys       | key                 | key_len | ref  | rows   | filtered | Extra                                        |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index | ix_gender_birthdate | ix_gender_birthdate | 4       | NULL | 299113 |   100.00 | Using index; Using temporary; Using filesort |
+----+-------------+-----------+------------+-------+---------------------+---------------------+---------+------+--------+----------+----------------------------------------------+
```

- GROUP BY, ORDER BY 칼럼이 다르기에 임시 테이블이 필요합니다.
- 인덱스를 사용하지 못하는 GROUP BY 형태 쿼리는 Using temporary가 표시되는 가장 대표적인 형태의 쿼리입니다.

- 주의
    
    **Using temporary가 표시되지 않아도 내부적으로 임시테이블을 사용할 때도 많음**
    
    - FROM 절에 사용된 쿼리는 무조건 임시 테이블 생성 → 파생 테이블이라 부르지만 임시테이블임
    - `COUNT(DISTINCT column1)`을 포함하는 쿼리도 인덱스를 쓸 수 없다면 임시 테이블이 만들어짐
    - UNION, UNION DISTINCT도 항상 임시 테이블을 사용해 결과 병합함
    (8.0부터 UNION ALL은 임시테이블을 사용하지 않도로 개선됨)
    - 인덱스를 사용하지 못한 Using filesort도 임시 버퍼 공간을 쓰기에 실체는 임시 테이블과 같습니다.
    
    또한 임시 테이블이나 버퍼가 메모리, 디스크 중 어디에 저장됐는지는 MySQL 서버 상태 변숫값을 통해 알 수 있습니다.
    
    ```sql
    EXPLAIN
    SELECT COUNT(DISTINCT last_name)
    FROM employees;
    
    # Using temporary가 표시되진 않았지만, 임시 테이블이 사용됨
    +----+-------------+-----------+------------+-------+-----------------------+-----------------------+---------+------+------+----------+--------------------------+
    | id | select_type | table     | partitions | type  | possible_keys         | key                   | key_len | ref  | rows | filtered | Extra                    |
    +----+-------------+-----------+------------+-------+-----------------------+-----------------------+---------+------+------+----------+--------------------------+
    |  1 | SIMPLE      | employees | NULL       | range | ix_lastname_firstname | ix_lastname_firstname | 66      | NULL | 1634 |   100.00 | Using index for group-by |
    +----+-------------+-----------+------------+-------+-----------------------+-----------------------+---------+------+------+----------+--------------------------+
    
    FLUSH STATUS;
    
    # 컴퓨터가 좋은지 생성안됨 ㅋ
    SHOW STATUS LIKE 'Created_tmp%';
    +-------------------------+-------+
    | Variable_name           | Value |
    +-------------------------+-------+
    | Created_tmp_disk_tables | 0     |
    | Created_tmp_files       | 0     |
    | Created_tmp_tables      | 0     | // 기댓값 1
    +-------------------------+-------+
    3 rows in set (0.00 sec)
    ```
    

---

<br>

**Using where**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/4b6355ff-ee49-4d86-927b-f9428341a7ee)


- `MySQL 아키텍쳐 = MySQL 엔진 + 스토리지 엔진`
- 스토리지 엔진은 디스크, 메모리 상에서 필요한 레코드를 읽거나 저장하는 역할
- MySQL 엔진은 스토리지 엔진으로부터 받은 레코드를 가공, 연산하는 작업 수행
- **MySQL 엔진 레이어에서 별도의 가공을 해서 필터링 작업을 처리한 경우에만 Using where 코멘트가 표시됩니다.**

- 8.3.7.1에서 언급한 작업 범위 결정 조건의 경우 스토리지 엔진 레벨에서 처리되지만 체크 조건은 MySQL 엔진 레이어에서 처리됩니다. (Using where)
    
    ```sql
    EXPLAIN
    SELECT *
    FROM employees
    WHERE emp_no BETWEEN 10001 AND 10100
    		AND gender='F';
    
    +----+-------------+-----------+------------+-------+-----------------------------+---------+---------+------+------+----------+-------------+
    | id | select_type | table     | partitions | type  | possible_keys               | key     | key_len | ref  | rows | filtered | Extra       |
    +----+-------------+-----------+------------+-------+-----------------------------+---------+---------+------+------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | range | PRIMARY,ix_gender_birthdate | PRIMARY | 4       | NULL |  100 |    50.00 | Using where |
    +----+-------------+-----------+------------+-------+-----------------------------+---------+---------+------+------+----------+-------------+
    ```
    
    - 작업 범위 결정 조건: `emp_no BETWEEN 10001 AND 10100`
    - 체크 조건: `gender='F'`
    
    **스토리지 엔진은 100건을 읽고, MySQL 엔진에게 넘겨주고 MySQL 엔진은 63건의 레코드를 필터링해서 버렸다는 의미가 됩니다.**
    

8.0부터는 실행 계획에 filtered 칼럼이 같이 표시되므로 Using where가 성능상 문제를 일으킬지 말지를 좀 더 쉽게 알아낼 수 있습니다.

만약 위 쿼리에서 (emp_no, gender) 인덱스가 있었다면 두 조건 모두 작업 범위 제한 조건으로 사용되므로 필요한 37건의 레코드만 정확하게 읽을 수 있습니다.

---

<br>

**Zero limit**

- MySQL 서버에서 데이터 값이 아닌 쿼리 결괏값의 메타데이터만 필요한 경우 `LIMIT 0`을 사용하면 됩니다.
- 옵티마이저는 이런 의도를 알아채고 실제 테이블의 레코드는 전혀 읽지 않고 메타정보만 반환합니다.
- 이때 Extra 칼럼에는 Zero limit이 출력됩니다.

```sql
EXPLAIN SELECT * FROM employees LIMIT 0;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Zero limit |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
```
