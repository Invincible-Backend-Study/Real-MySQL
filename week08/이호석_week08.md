# 📌 10장 실행 계획

## ✅ 10.2 실행 계획 확인

- MySQL 서버의 실행 계획은 DESC, EXPLAIN 명령으로 확인가능함
- 8.0부터는 EXPLAIN에 사용할 수 있는 새로운 옵션이 등장함
(실행 계획의 출력 포맷 및 실제 쿼리 실행 결과까지 확인할 수 있는 기능)

<br><br>

### 10.2.1 실행 계획 출력 포맷

- 이전 버전은 EXPLAIN EXTENDED or PARTITIONS 명령이 구분됐음
- 8.0부터는 EXPLAIN의 FORMAT 옵션을 사용해 실행 계획의 표시 방법을 JSON, TREE, 단순 테이블 형태를 선택할 수 있음

```sql
# 단순 테이블 형식
EXPLAIN ~

# TREE
EXPLAIN FORMAT=TREE ~

# JSON
EXPLAIN FORMAT=JSON ~
```

포맷 옵션별로 선호도나, 표시 정보의 차이가 있지만 옵티마이저가 수립한 실행 계획의 큰 흐름을 보여주는 것은 동일합니다.

<br><br>

### 10.2.2 쿼리의 실행 시간 확인 (EXPLAIN ANALYZE)

- 8.0.18부터 쿼리의 실행 계획, 단계별 소요 시간 정보가 확인 가능한 EXPLAIN ANALYZE 기능도 추가됐습니다.
    - show profile 명령으로 어떤 부분에서 시간이 많이 소요되는지 확인할 수 있지만, 실행계획읜 단계별로 소요된 시간정보를 보여주진 않음
- **EXPLAIN ANALYZE는 EXPLAIN과 달리 실제 쿼리를 실행하고 사용된 실행 계획과 소요된 시간을 보여줍니다.**
- EXPLAIN ANALYZE는 항상 TREE 포맷으로 보여주기에 FORMAT 설정 불가

```sql
EXPLAIN ANALYZE
SELECT e.hire_date, avg(s.salary)
FROM employees e
		INNER JOIN salaries s ON s.emp_no=e.emp_no
							AND s.salary > 50000
							AND s.from_date <= '1990-01-01'
							AND s.to_date > '1990-01-01'
WHERE e.first_name = 'Matt'
GROUP BY e.hire_ate;

+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-----------------+
| id | select_type | table | partitions | type | possible_keys                    | key          | key_len | ref                | rows | filtered | Extra           |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-----------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_hiredate,ix_firstname | ix_firstname | 58      | const              |  233 |   100.00 | Using temporary |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY,ix_salary                | PRIMARY      | 4       | employees.e.emp_no |    9 |     5.55 | Using where     |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-----------------+

+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|(6) -> Table scan on <temporary>  (actual time=2.89..2.9 rows=48 loops=1)
   (5) -> Aggregate using temporary table  (actual time=2.89..2.89 rows=48 loops=1)
       (4) -> Nested loop inner join  (cost=430 rows=125) (actual time=0.306..2.83 rows=48 loops=1)
           (1) -> Index lookup on e using ix_firstname (first_name='Matt')  (cost=128 rows=233) (actual time=0.231..0.668 rows=233 loops=1)
           (3) -> Filter: ((s.salary > 50000) and (s.from_date <= DATE'1990-01-01') and (s.to_date > DATE'1990-01-01'))  (cost=0.335 rows=0.535) (actual time=0.00798..0.00908 rows=0.206 loops=233)
               (2) -> Index lookup on s using PRIMARY (emp_no=e.emp_no)  (cost=0.335 rows=9.63) (actual time=0.0054..0.00771 rows=9.53 loops=233)
 |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```

위 실행 계획의 실제 실행 순서는 다음 기준으로 읽습니다.

1. `들여쓰기가 같은 레벨에선 상단에 위치한 라인이 먼저 실행`
2. `들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행`

- 위 예시의 실행 계획 설명 (FROM → WHERE → GROUP BY → SELECT?)
    1. employees 테이블의 `ix_firstname` 인덱스를 통해 `first_name='Matt'` 조건에 일치하는 레코드를 찾고
    2. salaries 테이블의 PRIMARY 키를 통해 emp_no가 (1)번 결과의 emp_no와 동일한 레코드를 찾아서
    3. `((s.salary > 50000) and (s.from date <= '1990-01-01') and (s.to_date > '1990-01-01'))`조건에 일치하는 건만 가져와
    4. 1번과 3번의 결과를 조인해서
    5. 임시 테이블에 결과를 저장하면서 GROUP BY 집계를 실행하고
    6. 임시 테이블의 결과를 읽어서 결과를 반환한다.

---

- `EXPLAIN ANALYZE` 명령의 결과에는 `단계별로 실제 소요된 시간(actual time)`과 `처리한 레코드 건수 (rows)`, `반복 횟수(loops)`가 표시됩니다.
- 2번째 실행된 라인에 나열된 필드들의 의미
    - `actual time=0.0054..0.00771`
        - employees 테이블에서 읽은 emp_no 값을 기준으로 salaries 테이블에서 일치하는 레코드를 검색하는 데 걸린 시간(밀리초)을 의미합니다.
        - 첫 번째 숫자는 첫 번 째 레코드를 가져오는 데 걸린 평균 시간(밀리초)을 의미하고, 두 번째 숫자 값은 마지막 레코드를 가져오는 데 걸린 평균 시간(밀리초)을 의미한다. 
        → `HAVETOFIND`: 첫번째 마지막 레코드를 가져오는것이라면 의미가 있을까?!
    - `rows=9.53`
        - employees 테이블에서 읽은 emp_no에 일치하는 salaries 테이블의 평균 레코드 건수를 의미합니다. (salaries 테이블에 중복되는 emp_no의 평균 건수)
    - `loops=233`
        - employees 테이블에서 읽은 emp_no를 이용해 salaries 테이블의 레코드를 찾는 작업이 반복된 횟수를 의미합니다. 결국 여기서는 empolyees 테이블에서 읽은 emp_no의 개수가 233개임을 의미합니다.
- `actual time` 필드와 `rows`의 평균 시간 및 건수의 의미
    - loops=233개의 row를 employees에서 찾은 후 해당 행들을 매번 salaries에서 조회할때 `행을 찾는 시간의 평균과`, `찾은 행의 평균 개수`를 말합니다.

→ EXPLAIN ANALYZE는 실제 실행 후 실행 계획을 가져오므로 아주 오래 걸리는 쿼리의 경우 EXPLAIN으로 실행 계획 확인 및 튜닝 이후 EXPLIAN ANALYZE를 사용하는 방법도 있습니다.

<br><br><br>

## ✅ 10.3 실행 계획 분석

- 실행 계획 출력 포맷보다 중요한 것
    - 실행 계획의 접근 방법의 종류와 사용된 최적화를 아는 것
    - 어떤 인덱스를 사용했는지

- 일반적인 EXPLAIN명령은 테이블 형태의 포맷을 출력합니다.
    - 표의 각 라인은 쿼리 문장에서 사용된 테이블의 개수만큼 출력됨
    (서브쿼리로 임시테이블을 생성했다면 임시테이블도 포함됨)
    - 실행 순서는 위에서 아래 순으로 표시됨
    (UNION, 상관 서브쿼리의 경우 순서대로 표시되지 않을 수 있음)
    - 실행 계획의 위쪽에 출력된 결과일수록(더 작은 id칼럼값) 쿼리의 바깥(outer) 부분이거나 먼저 접근한 테이블
    - 아래쪽에 출력된 결과는 쿼리의 안쪽 부분(inner)또는 나중에 접근한 테이블에 해당됩니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/41cb6ff0-29a8-49ce-85d3-fa9c6270f693)

        
<br><br>

### 10.3.1 id 칼럼

SELECT 문장은 1개 이상의 하위 SELECT 문장을 포함할 수 있습니다.

```sql
SELECT
FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
WHERE tb1.id=tb2.id;
```

위 SELECT 문장은 다음과 같이 분리할 수 있는데, SELECT 단위로 구분한 것을 `단위(SELECT) 쿼리`라고 표현합니다.

```sql
SELECT ... FROM tb_test1;
SELECT ... FROM tb1, tb_test2 tb2 WHERE tb1.id=tb2.id;
```

---

실행 계획에서 id 칼럼은 단위 SELECT 쿼리 별로 부여되는 식별자 값입니다. 여러개의 테이블 조인은 조인되는 테이블 개수만큼 실행 계획 레코드 개수가 출력되지만 id값은 같습니다.

```sql
# 여러개의 테이블을 조인해도 id값은 동일함
EXPLAIN
SELECT e.emp_no, e.first_name, s.from_date, s.salary FROM employees e, salaries s
WHERE e.emp_no=s.emp_no LIMIT 10;
+----+-------------+-------+------------+-------+---------------+--------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+--------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | index | PRIMARY       | ix_firstname | 58      | NULL               | 299113 |   100.00 | Using index |
|  1 | SIMPLE      | s     | NULL       | ref   | PRIMARY       | PRIMARY      | 4       | employees.e.emp_no |      9 |   100.00 | NULL        |
+----+-------------+-------+------------+-------+---------------+--------------+---------+--------------------+--------+----------+-------------+

# 쿼리가 단위 SELECT 쿼리로 구성된다면 각 레코드가 다른 id값을 가짐
EXPLAIN
SELECT ((SELECT COUNT(*) FROM employees) + (SELECT COUNT(*) FROM departments)) AS total_count;
+----+-------------+-------------+------------+-------+---------------+-------------+---------+------+--------+----------+----------------+
| id | select_type | table       | partitions | type  | possible_keys | key         | key_len | ref  | rows   | filtered | Extra          |
+----+-------------+-------------+------------+-------+---------------+-------------+---------+------+--------+----------+----------------+
|  1 | PRIMARY     | NULL        | NULL       | NULL  | NULL          | NULL        | NULL    | NULL |   NULL |     NULL | No tables used |
|  3 | SUBQUERY    | departments | NULL       | index | NULL          | ux_deptname | 162     | NULL |      9 |   100.00 | Using index    |
|  2 | SUBQUERY    | employees   | NULL       | index | NULL          | ix_hiredate | 3       | NULL | 299113 |   100.00 | Using index    |
+----+-------------+-------------+------------+-------+---------------+-------------+---------+------+--------+----------+----------------+
```

**id칼럼이 테이블의 접근 순서를 의미하진 않습니다.**

```sql
EXPLAIN
SELECT * FROM dept_emp de
WHERE de.emp_no= (SELECT e.emp_no
										FROM employees e
										WHERE e.first_name='Georgi'
												AND e.last_name='Facello' LIMIT 1);

# 아래 결과는 id값이 de -> e 순서 같아 보이지만
+----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys                      | key                   | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+
|  1 | PRIMARY     | de    | NULL       | ref  | ix_empno_fromdate                  | ix_empno_fromdate     | 4       | const       |    1 |   100.00 | Using where |
|  2 | SUBQUERY    | e     | NULL       | ref  | ix_firstname,ix_lastname_firstname | ix_lastname_firstname | 124     | const,const |    2 |   100.00 | Using index |
+----+-------------+-------+------------+------+------------------------------------+-----------------------+---------+-------------+------+----------+-------------+

# EXPLAIN FORMAT=TREE로 다시 조회하면 실행 순서는 e -> de가 됩니다.
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: (de.emp_no = (select #2))  (cost=1.1 rows=1)
    -> Index lookup on de using ix_empno_fromdate (emp_no=(select #2))  (cost=1.1 rows=1)
    -> Select #2 (subquery in condition; run only once)
        -> Limit: 1 row(s)  (cost=1.21 rows=1)
            -> Covering index lookup on e using ix_lastname_firstname (last_name='Facello', first_name='Georgi')  (cost=1.21 rows=2)
 |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Tree 형식의 실행계획을 보면 Select #2가 id값이 2인 employees테이블을 의미합니다. 

또한 `ix_lastname_firstname` 인덱스를 사용이 가장 깊은 뎁스 이므로 employees 테이블이 가장 먼저 조회되고 그 결과로 dept_emp 테이블이 조회됐습니다.

<br><br>

### 10.3.2 select_type 칼럼

각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼 입니다. 표시될 수 있는 값은 다음과 같습니다.

---

<br>

**SIMPLE**

- UNION, 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우(쿼리에 조인이 포함되어도 마찬가지)
- 쿼리 문장이 아무리 복잡해도 실행 계획에서 select_type이 SIMPLE인 단위 쿼리는 하나만 존재합니다.
(일반적으로 제일 바깥 SELECT 쿼리)

---

<br>

**PRIMARY**

- UNION이나 서브쿼리를 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽(Outer)에 있는 단위 쿼리는 select_type이 PRIMARY로 표시됩니다.
- SIMPLE과 마찬가지로 select_type이 PRIMARY인 단위 SELECT 쿼리는 하나만 존재함
(쿼리의 제일 바깥쪽에 있는 SELECT 단위 쿼리)

---

<br>

**UNION**

- UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시됨
- UNION의 첫 번째 단위 SELECT는 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블(DERIVED)이 select_type으로 표시됨

```sql
EXPLAIN
SELECT * FROM (
	(SELECT emp_no FROM employees e1 LIMIT 10) UNION ALL 
	(SELECT emp_no FROM employees e2 LIMIT 10) UNION ALL 
	(SELECT emp_no FROM employees e3 LIMIT 10)) tb;

+----+-------------+------------+------------+-------+---------------+-------------+---------+------+--------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key         | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+-------------+---------+------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL   | NULL          | NULL        | NULL    | NULL |     30 |   100.00 | NULL        |
|  2 | DERIVED     | e1         | NULL       | index | NULL          | ix_hiredate | 3       | NULL | 299113 |   100.00 | Using index |
|  3 | UNION       | e2         | NULL       | index | NULL          | ix_hiredate | 3       | NULL | 299113 |   100.00 | Using index |
|  4 | UNION       | e3         | NULL       | index | NULL          | ix_hiredate | 3       | NULL | 299113 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+-------------+---------+------+--------+----------+-------------+
```

- e1 테이블을 제외하고 나머지 테이블은 모두 select_type이 UNION입니다.
- 첫 번째 쿼리는 전체 UNION의 결과를 대표하는 select_type으로 설정됨
(3개의 테이블이 UNION ALL될 때 임시테이블을 만들어 사용하므로 DERIVED를 가짐)

---

<br>

**DEPENDENT UNION**

- 이전 UNION select_type과 같이 UNION, UNION ALL로 집합을 결합하는 쿼리에서 표시됩니다.
- DEPENDENT는 UNION, UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미합니다.

```sql
EXPLAIN
SELECT *
FROM employees e1 WHERE e1.emp_no IN (
SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
UNION
SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt');

+----+--------------------+------------+------------+--------+--------------------------------------------+---------+---------+------+--------+----------+-----------------+
| id | select_type        | table      | partitions | type   | possible_keys                              | key     | key_len | ref  | rows   | filtered | Extra           |
+----+--------------------+------------+------------+--------+--------------------------------------------+---------+---------+------+--------+----------+-----------------+
|  1 | PRIMARY            | e1         | NULL       | ALL    | NULL                                       | NULL    | NULL    | NULL | 299113 |   100.00 | Using where     |
|  2 | DEPENDENT SUBQUERY | e2         | NULL       | eq_ref | PRIMARY,ix_firstname,ix_lastname_firstname | PRIMARY | 4       | func |      1 |     5.00 | Using where     |
|  3 | DEPENDENT UNION    | e3         | NULL       | eq_ref | PRIMARY,ix_lastname_firstname              | PRIMARY | 4       | func |      1 |     5.00 | Using where     |
|  4 | UNION RESULT       | <union2,3> | NULL       | ALL    | NULL                                       | NULL    | NULL    | NULL |   NULL |     NULL | Using temporary |
+----+--------------------+------------+------------+--------+--------------------------------------------+---------+---------+------+--------+----------+-----------------+
```

- IN 이하 서브쿼리에서 두 개의 쿼리가 UNION으로 결합되어 있습니다.
- MySQL 옵티마이저는 IN 내부의 서브쿼리를 먼저 처리하지 않고 외부의 employees 테이블을 먼저 읽고 서브쿼리를 실행합니다. (employees 테이블의 칼럼값이 서브쿼리에 영향을 줌
- 내부적으로는 UNION에 사용된 쿼리의 WHERE 조건에 `"e2.emp_no=e1.emp_no"`와 `"e3.emp_no=e1.
emp_no"`라는 조건이 자동으로 추가되어 실행됩니다.
- **정리하면 employees 테이블의 emp_no칼럼이 서브 쿼리에 사용되기 때문에 DEPENDENT UNION이 select_type에 표시됩니다.**

---

<br>

**UNION RESULT**

- UNION 결과를 담아두는 테이블을 의미합니다.
    - 8.0 이전에는 UNION ALL, UNION 쿼리 모두 UNION의 결과를 임시테이블을 생성했지만, 8.0부터는 UNION ALL의 경우 임시 테이블을 사용하지 않도록 기능이 개선됐습니다.
- 하지만 **UNION은 MySQL 8.0버전에서도 여전히 임시 테이블에 결과를 버퍼링** 합니다.
- 실행 계획상에서 위 임시테이블이 사용되면 select_type은 `UNION RESULT`입니다.
(실제 쿼리에서 단위 쿼리가 아니므로 별도의 id값은 없습니다.)

```sql
EXPLAIN
SELECT emp_no FROM salaries WHERE salary > 100000
UNION DISTINCT
SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';
+----+--------------+------------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+
| id | select_type  | table      | partitions | type  | possible_keys                 | key         | key_len | ref  | rows   | filtered | Extra                    |
+----+--------------+------------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+
|  1 | PRIMARY      | salaries   | NULL       | range | ix_salary                     | ix_salary   | 4       | NULL | 188518 |   100.00 | Using where; Using index |
|  2 | UNION        | dept_emp   | NULL       | range | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | NULL |   5325 |   100.00 | Using where; Using index |
|  3 | UNION RESULT | <union1,2> | NULL       | ALL   | NULL                          | NULL        | NULL    | NULL |   NULL |     NULL | Using temporary          |
+----+--------------+------------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+

# UNION ALL을 사용하면 UNION RESULT는 사라짐
EXPLAIN 
SELECT emp_no FROM salaries WHERE salary > 100000
UNION ALL 
SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';
+----+-------------+----------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys                 | key         | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+
|  1 | PRIMARY     | salaries | NULL       | range | ix_salary                     | ix_salary   | 4       | NULL | 188518 |   100.00 | Using where; Using index |
|  2 | UNION       | dept_emp | NULL       | range | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | NULL |   5325 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+-------------------------------+-------------+---------+------+--------+----------+--------------------------+
```

- UNION RESULT 라인의 테이블 컬럼은 `<union1,2>`인데 이것은 아래 그림같이 id값이 일치하는 단위 쿼리의 조회 결과들을 UNION했음을 말합니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/413b6266-91ce-42ed-901e-5c394efc571e)

    

---

<br>

**SUBQUERY**

- select_type의 서브쿼리는 FROM 절 이외에서 사용되는 서브쿼리만을 의미합니다.

```sql
EXPLAIN
SELECT e.first_name,
				(SELECT COUNT(*)
					FROM dept_emp de, dept_manager dm
					WHERE dm.dept_no = de.dept_no) AS cnt
					FROM employees e WHERE e.emp_no=10001;

+----+-------------+-------+------------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref                  | rows  | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
|  1 | PRIMARY     | e     | NULL       | const | PRIMARY       | PRIMARY | 4       | const                |     1 |   100.00 | NULL        |
|  2 | SUBQUERY    | dm    | NULL       | index | PRIMARY       | PRIMARY | 20      | NULL                 |    24 |   100.00 | Using index |
|  2 | SUBQUERY    | de    | NULL       | ref   | PRIMARY       | PRIMARY | 16      | employees.dm.dept_no | 41392 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
```

- FROM 절에 사용된 서브쿼리(인라인 뷰)의 select_type은 DERIVED로 표시되고 그 외의 위치에선 모두 SUBQUERY로 표시됩니다.
    - DERIVED는 파생 테이블과 같은 의미

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cb599e28-74ef-4537-beb7-d1b2fe822bd1)


---

<br>

**DEPENDENT SUBQUERY**

- 서브쿼리가 바깥쪽(Outer) SELECT 쿼리에서 정의된 칼럼을 사용하는 경우, select_type에 DEPENDENT SUBQUERY로 표시됩니다.

```sql
EXPLAIN
SELECT e.first_name,
		(SELECT COUNT(*)
			FROM dept_emp de, dept_manager dm
			WHERE dm.dept_no = de.dept_no AND de.emp_no = e.emp_no) AS cnt
FROM employees e
WHERE e.first_name='Matt';

+----+--------------------+-------+------------+------+------------------------------------+-------------------+---------+----------------------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys                      | key               | key_len | ref                  | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+------------------------------------+-------------------+---------+----------------------+------+----------+-------------+
|  1 | PRIMARY            | e     | NULL       | ref  | ix_firstname,ix_lastname_firstname | ix_firstname      | 58      | const                |  233 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | de    | NULL       | ref  | PRIMARY,ix_empno_fromdate          | ix_empno_fromdate | 4       | employees.e.emp_no   |    1 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | dm    | NULL       | ref  | PRIMARY                            | PRIMARY           | 16      | employees.de.dept_no |    2 |   100.00 | Using index |
+----+--------------------+-------+------------+------+------------------------------------+-------------------+---------+----------------------+------+----------+-------------+
```

- 서브쿼리 결과가 바깥쪽(Outer) SELECT 쿼리의 칼럼에 의존적이기 때문에 DEPENDENT라는 키워드가 붙습니다.
- DEPENDENT UNION과 같이 외부 쿼리가 먼저 수행된 후 내부 쿼리(서브쿼리)가 실행돼야 하므로 DEPENDENT 키워드가 없는 일반 서브쿼리보다는 처리 속도가 느릴 때가 많습니다.

---

<br>

**DERIVED**

- 5.5 버전까지는 서브쿼리가 FROM 절에 사용된 경우 항상 select_type이 DERIVED인 실행 계획을 만들었습니다.
- 하지만 5.6부터는 optimizer_switch(옵티마이저 옵션)에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화를 수행하기도 합니다.
- **DERIVED는 단위 SELECT 쿼리의 실행 결과로 메모리, 디스크에 임시 테이블을 생성**하는걸 의미합니다.
    - select_type이 DERIVED일때 생성된 임시 테이블을 **파생 테이블**이라고 합니다.
- **5.5와달리 5.6부터는 옵티마이저 옵션에 따라 쿼리 특성에 맞게 임시 테이블에도 인덱스를 추가해서 만들 수 있게 최적화 됐습니다.**

```sql
EXPLAIN
SELECT *
FROM (SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no) tb, employees e
WHERE e.emp_no=tb.emp_no;

+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys                         | key               | key_len | ref       | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                                  | NULL              | NULL    | NULL      | 331143 |   100.00 | NULL        |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY                               | PRIMARY           | 4       | tb.emp_no |      1 |   100.00 | NULL        |
|  2 | DERIVED     | de         | NULL       | index  | PRIMARY,ix_fromdate,ix_empno_fromdate | ix_empno_fromdate | 7       | NULL      | 331143 |   100.00 | Using index |
+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
```

- MySQL 버전이 올라가며 조인 쿼리 최적화는 성숙된 상태
- 따라서 파생 테이블에 대한 최적화가 부족한 MySQL 버전을 사용중이라면 가능하면 DERIVED 형태의 실행 계획을 조인으로 해결할 수 있게 쿼리를 바꿔주는 것이 좋습니다.
- 8.0버전 부터는 FROM절 서브쿼리에 대한 최적화도 많이 개선됐기에, 가능하면 서브쿼리는 조인으로 쿼리를 재작성해 처리합니다.

---

<br>

**DEPENDENT DERIVED**

- 8.0이전 버전에선 FROM 절의 서브쿼리는 외부 칼럼을 사용할 수 없었지만 8.0부터 `LATERAL JOIN`기능이 추가되어 FROM 절의 서브쿼리에서도 외부 칼럼을 참조할 수 있습니다.

```sql
# 이해 잘 안됨
EXPLAIN
SELECT *
FROM employees e
LEFT JOIN LATERAL (SELECT *
										FROM salaries s
										WHERE s.emp_no=e.emp_no
										ORDER BY s.from_date DESC LIMIT 2) AS s2 ON s2.emp_no=e.emp_no;

+----+-------------------+------------+------------+------+---------------+-------------+---------+--------------------+--------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys | key         | key_len | ref                | rows   | filtered | Extra                      |
+----+-------------------+------------+------------+------+---------------+-------------+---------+--------------------+--------+----------+----------------------------+
|  1 | PRIMARY           | e          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               | 299113 |   100.00 | Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | employees.e.emp_no |      2 |   100.00 | NULL                       |
|  2 | DEPENDENT DERIVED | s          | NULL       | ref  | PRIMARY       | PRIMARY     | 4       | employees.e.emp_no |      9 |   100.00 | Using filesort             |
+----+-------------------+------------+------------+------+---------------+-------------+---------+--------------------+--------+----------+----------------------------+

+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Nested loop left join  (cost=239678 rows=598226)
    -> Invalidate materialized tables (row from e)  (cost=30298 rows=299113)
        -> Table scan on e  (cost=30298 rows=299113)
    -> Index lookup on s2 using <auto_key0> (emp_no=e.emp_no)  (cost=1.7..1.95 rows=2)
        -> Materialize (invalidate on row from e)  (cost=1.45..1.45 rows=2)
            -> Limit: 2 row(s)  (cost=1.25 rows=2)
                -> Sort: s.from_date DESC, limit input to 2 row(s) per chunk  (cost=1.25 rows=9.63)
                    -> Index lookup on s using PRIMARY (emp_no=e.emp_no)  (cost=1.25 rows=9.63)
 |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

위 예제는 employees 테이블의 1건의 레코드당 최근 순서대로 최대 2건까지만 가져와서 조인을 실행합니다.

---

<br>

**UNCACHEALBE SUBQUERY**

- 하나의 쿼리 문장에서 조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 내부 캐시 공간에 담습니다.
- **서브쿼리 캐시는 쿼리 캐시 및 파생 테이블(DERIVED)와는 전혀 무관한 기능**입니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cead512b-fddf-43c8-b706-4d1e42b81d5b)


- 위 그림은 select_type이 SUBQUERY인 경우 캐시를 사용하는 방법입니다.
- 캐시는 처음 한 번만 생성됩니다.
- 반면에 **DEPENDENT SUBQUERY**는 Outer 쿼리와 상관관계를 갖으므로 딱 한번만 캐시되는것이 아니라 외부(Outer) 쿼리의 값 단위로 캐시가 만들어지는 방식으로 처리됩니다.

`UNCACHEABLE SUBQUERY`는 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능한 경우가 존재할때 표시됩니다.

- 사용자 변수가 서브쿼리에 사용된 경우
 → 사용자 정의 변수는 세션 단위로 유지 되므로
- NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
→ MySQL은 해당 쿼리의 결과가 시시각각 달라진다고 가정하기에 매번 새로 호출해야 하므로 캐싱이 불가능합니다.
- UUID()나 RAND()와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우
→ 랜덤한 값에 의존하므로 캐싱 불가

```sql
EXPLAIN
SELECT *
FROM employees e WHERE e.emp_no = (
SELECT @status FROM dept_emp de WHERE de.dept_no='d005');

+----+----------------------+-------+------------+------+---------------------------------------+---------+---------+-------+--------+----------+-------------+
| id | select_type          | table | partitions | type | possible_keys                         | key     | key_len | ref   | rows   | filtered | Extra       |
+----+----------------------+-------+------------+------+---------------------------------------+---------+---------+-------+--------+----------+-------------+
|  1 | PRIMARY              | e     | NULL       | ALL  | NULL                                  | NULL    | NULL    | NULL  | 299113 |   100.00 | Using where |
|  2 | UNCACHEABLE SUBQUERY | de    | NULL       | ref  | PRIMARY,ix_fromdate,ix_empno_fromdate | PRIMARY | 16      | const | 165571 |   100.00 | Using index |
+----+----------------------+-------+------------+------+---------------------------------------+---------+---------+-------+--------+----------+-------------+
```

사용자 변수를 사용했기에 WHERE절의 서브쿼리(단위쿼리)의 select_type이 UNCACHEABLE SUBQUERY로 표시됩니다.

<br>

**UNCACHEABLE UNION**

- 위의 `UNCACHEABLE SUBQUERY` 과 동일하게 캐시 자체가 불가능한 UNION일 경우 표시됩니다.

---

<br>

**MATERIALIZED**

- 5.6 버전부터 도입된 select_type으로 주로 FROM 절이나 IN(subquery) 형태의 쿼리에 사용된 서브쿼리 최적화를 위해 사용됩니다.

```sql
EXPLAIN
SELECT *
FROM employees e
WHERE e.emp_no IN (SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000);

+----+--------------+-------------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+--------------------------+
| id | select_type  | table       | partitions | type   | possible_keys     | key       | key_len | ref                | rows | filtered | Extra                    |
+----+--------------+-------------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+--------------------------+
|  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL              | NULL      | NULL    | NULL               | NULL |   100.00 | NULL                     |
|  1 | SIMPLE       | e           | NULL       | eq_ref | PRIMARY           | PRIMARY   | 4       | <subquery2>.emp_no |    1 |   100.00 | NULL                     |
|  2 | MATERIALIZED | salaries    | NULL       | range  | PRIMARY,ix_salary | ix_salary | 4       | NULL               |    1 |   100.00 | Using where; Using index |
+----+--------------+-------------+------------+--------+-------------------+-----------+---------+--------------------+------+----------+--------------------------+
```

- MySQL 5.7부터 도입된 구체화 전략을 통해 위 쿼리에서 서브쿼리가 먼저 임시테이블로 구체화되고, 이후 e와 조인되어 처리됩니다.

---

<br><br>

### 10.3.3 table 칼럼

- MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시됩니다. (같은 id를 가졌어도(조인) 테이블이 다르면 행이 N개로 분기됨) 테이블의 이름에 별칭이 부여된다면 별칭이 표시됩니다.

```sql
# 두 쿼리 모두 테이블을 사용하지 않으므로 table 칼럼이 null을 표시합니다.
EXPLAIN SELECT NOW();
EXPLAIN SELECT NOW() FROM DUAL;
```

- table 칼럼에 <derived N>, <union M,N>과 같이 <>로 둘러싸인 것은 임시 테이블을 의미합니다. 또한 내부 숫자는 단위 SELECT 쿼리의 id값을 지칭합니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f43aa0c0-49ed-4fdb-8778-193b0f480380)


id, select_type, table 칼럼을 통해 실행계획의 각 라인에 명시된 테이블이 어떤 순서로 실행되는지를 판단하는 근거가 됩니다.

```sql
EXPLAIN
SELECT *
FROM (SELECT de.emp_no 
				FROM dept_emp de 
				GROUP BY de.emp_no)tb, employees e
WHERE e.emp_no=tb.emp_no;

+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys                         | key               | key_len | ref       | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                                  | NULL              | NULL    | NULL      | 331143 |   100.00 | NULL        |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY                               | PRIMARY           | 4       | tb.emp_no |      1 |   100.00 | NULL        |
|  2 | DERIVED     | de         | NULL       | index  | PRIMARY,ix_fromdate,ix_empno_fromdate | ix_empno_fromdate | 7       | NULL      | 331143 |   100.00 | Using index |
+----+-------------+------------+------------+--------+---------------------------------------+-------------------+---------+-----------+--------+----------+-------------+
```

1. 1번 행의 table이 <derived 2>인데 파생테이블을 준비하기 위해 id=2인 라인이 먼저 실행되어 결과가 파생테이블에 저장됨
2. id=2인 select_type 칼럼이 DERIVED이므로 dept_emp 테이블을 읽어 파생 테이블을 생성함을 알 수 있습니다.
3. id=1인 1, 2라인은 조인(<derived2>와 e)되는 쿼리라는 사실을 알 수 있습니다. 이때 <derived2>가 드라이빙 테이블이 됩니다.

8.0부터 서브쿼리 최적화가 이루어지며 `MATERIALIZED`최적화가 도입됐습니다. 구체화를 사용하면 select_type이 MATERIALIZED이고 실행계획에선 `<subquery N>`과 같은 table 칼럼이 나타납니다. (`<derived N>`과 같은 방식으로 해석)

---

<br><br>

### 10.3.4 patitions 칼럼

- 별도의 명령어로 partitions 칼럼을 확인했던 과거와 달리 8.0부터는 EXPLAIN 명령으로 파티션 관련 실행계획을 볼 수 있습니다.

```sql
CREATE TABLE employees_2 (
		emp_no int NOT NULL,
		birth_date DATE NOT NULL, 
		first_name VARCHAR (14) NOT NULL,
		last_name VARCHAR(16) NOT NULL,
		gender ENUM('M', 'F') NOT NULL,
		hire_date DATE NOT NULL,
		PRIMARY KEY (emp_no, hire_date)
) PARTITION BY RANGE COLUMNS (hire_date)
(PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),
 PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),
 PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),
 PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

INSERT INTO employees_2 SELECT * FROM employees;
```

- 별도의 employees_2 테이블을 생성해 hire_date 칼럼값을 5년 단위로 나누어진 파티션을 갖도록 합니다.
- 파티션 생성 시 파티션 키로 사용되는 칼럼은 PK를 포함한 모든 유니크 인덱스의 일부여야 합니다. 따라서 PK 칼럼 및 hire_date 칼럼을 통해 테이블을 생성합니다.

```sql
EXPLAIN
SELECT *
FROM employees_2
WHERE hire_date BETWEEN '1999-11-15' AND '2000-01-15';

+----+-------------+-------------+-----------------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table       | partitions            | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-------------+-----------------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | employees_2 | p1996_2000,p2001_2005 | ALL  | NULL          | NULL | NULL    | NULL | 22297 |    11.11 | Using where |
+----+-------------+-------------+-----------------------+------+---------------+------+---------+------+-------+----------+-------------+
```

- 위 칼럼은 1999년부터 2000년을 조회하므로 `p1996_2000`, `p2001_2005`2개의 파티션을 조회함을 알 수 있습니다.
- 옵티마이저는 쿼리의 `hire_date` 칼럼 조건은 파티션 키 칼럼을 `WHERE`조건에 있음을 파악합니다.
    - 따라서 필요로 하는 데이터가 들어있는 파티션을 파악하고 해당 파티션만 접근하게 됩니다.
    - 불필요한 파티션을 제외하고 쿼리를 수행하기 위한 테이블만 골라내는 과정을 `Partition pruning`이라고 합니다.
- 쿼리의 실행 계획에서도 partitions 칼럼에 참조한 파티션을 표시하게 됩니다.

추가로 type칼럼은 ALL로 표시되는데 이는 풀 테이블 스캔을 의미합니다. MySQL 포함 대부분의 RDBMS는 파티션은 물리적으로 개별 테이블처럼 별도의 저장공간을 가지기 때문입니다. 따라서 조회한 파티션(`p1996_2000`, `p2001_2005`)에 대해서 풀 테이블 스캔을 한다는 의미 입니다.

---

<br><br>

### 10.3.5 type 칼럼

- 실행 계획 type 이후의 칼럼은 MySQL 서버가 각 테이블의 레코드를 읽은 방식을 나타냅니다.
    - 인덱스 사용인지, 풀 테이블 스캔인지 등
- **쿼리 튜닝에서 인덱스의 효율적인 사용 여부가 확인이 중요하므로 실행 계획에서 type 칼럼은 반드시 체크해야 하는 중요 정보입니다.**
- MySQL 메뉴얼은 type 칼럼을 조인 타입으로 소개하지만(하나의 테이블에서의 쿼리도 조인처럼 처리하므로) **조인 보다는 각 테이블의 접근 방법(Access type)으로 해석하는편이 좋다고 합니다.**

- type 칼럼에 표시될 수 있는 값은 현재 대부분 사용하는 버전별 차이가 적고 다음과 같이 표시됩니다.
    - `system`
    - `const`
    - `eq_ref`
    - `ref`
    - `fulltext`
    - `ref_or_null`
    - `unique_subquery`
    - `index_subquery`
    - `range`
    - `index_merge`
    - `index`
    - `ALL` → 풀 테이블 스캔
    
    → ALL을 제외하고 나머지는 모두 인덱스를 사용함
    
    → 하나의 단위 SELECT 쿼리는 접근 방법 중 단 하나만 사용 가능
    
    → index_merge 제외 나머지는 모두 하나의 인덱스만 사용함
    
    → 위에서 아래 순서대로 성능이 빠른 순서임
    
    → 옵티마이저는 접근 방법과 비용을 함께 계산하여 최소 비용 접근 방법을 선택해 쿼리를 처리함
    
    ---
    
<br>

**system**

- 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system이라고 합니다.
- MyISAM, MEMORY 테이블에서만 사용되는 접근 방법

```sql
# MyISAM
CREATE TABLE tb_dual (fd1 int NOT NULL) ENGINE=MyISAM;
INSERT INTO tb_dual VALUES (1);

+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_dual | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+

# InnoDB 사용
CREATE TABLE tb_dual2 (fd1 int NOT NULL) ENGINE=InnoDB;
INSERT INTO tb_dual2 VALUES (1);
EXPLAIN SELECT * FROM tb_dual2;
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_dual2 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
```

- MyISAM을 사용했을때 system이 보입니다.
- 또한 system은 테이블에 레코드가 1건 이하인 경우에만 사용할 수 있는 접근 방법입니다. → 엥?

---

<br>

**const**

- 테이블 레코드 건수와 관계없이 쿼리가 `PK`, `UNIQUE KEY` 칼럼을 이용하는 WEHRE 조건절을 가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식을 말합니다.
- 다른 DBMS는 UNIQUE INDEX SCAN 이라고도 표현합니다.

```sql
EXPLAIN
SELECT * FROM employees WHERE emp_no=10001;

+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
```

- 다만 다중 칼럼 PK나 유니크 키 중에서 인덱스의 일부 칼럼만 조건으로 사용할 때는 const type의 접근 방법을 사용할 수 없습니다.
- 레코드가 1건 밖에 없더라도 **MySQL 엔진이 데이터를 읽어보지 않고서는 확신할 수 없기 때문**

```sql
# dept_emp의 PK는 (dept_no, emp_no)로 구성되어 있음
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no=' d005';
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | PRIMARY       | PRIMARY | 16      | const |    1 |   100.00 | Using where |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
```

**→ PK의 일부만 조건으로 사용할 때는 type 칼럼에 ref가 표시됩니다.**

- 하지만 PK, 유니크 키 인덱스의 모든 칼럼을 동등 조건으로 WHERE 절에 명시하면 const 접근 방식을 사용합니다.

```sql
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no='d005'
AND emp_no=10001;
+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys             | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | dept_emp | NULL       | const | PRIMARY,ix_empno_fromdate | PRIMARY | 20      | const,const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
```

- 참고
    
    **type=const인 실행 계획은 옵티마이저가 쿼리 최적화시 쿼리를 먼저 실행해 통째로 상수화 합니다. 따라서 const라는 표시를 보게 됩니다.**
    
    ```sql
    EXPLAIN
    SELECT COUNT(*)
    FROM employees e1
    WHERE first_name=(SELECT first_name FROM employees e2 WHERE emp_no=100001);
    +----+-------------+-------+------------+-------+------------------------------------+--------------+---------+-------+------+----------+--------------------------+
    | id | select_type | table | partitions | type  | possible_keys                      | key          | key_len | ref   | rows | filtered | Extra                    |
    +----+-------------+-------+------------+-------+------------------------------------+--------------+---------+-------+------+----------+--------------------------+
    |  1 | PRIMARY     | e1    | NULL       | ref   | ix_firstname,ix_lastname_firstname | ix_firstname | 58      | const |  248 |   100.00 | Using where; Using index |
    |  2 | SUBQUERY    | e2    | NULL       | const | PRIMARY                            | PRIMARY      | 4       | const |    1 |   100.00 | NULL                     |
    +----+-------------+-------+------------+-------+------------------------------------+--------------+---------+-------+------+----------+--------------------------+
    ```
    
    서브쿼리로 e2를 읽을때 PK를 검색해 조건에 일치하는 레코드의 이름을 읽으며 1건의 반환 하므로 상수화 됩니다.
    
    결국 옵티마이저에 의해 최적화되는 시점에는 다음과 같은 쿼리로 변환됩니다.
    
    ```sql
    SELECT COUNT (*)
    FROM employees e1
    WHERE first_name='Jasminko'; # Jasminko는 사이 100001인 사원의 first_name 값임
    ```
    

---

<br>

**eq_ref**

- eq_ref 접근 방법은 여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시됩니다.
- **조인시 처음 읽은 테이블의 칼럼값을 다음으로 읽어야 할 테이블의 PK나 유니크 키 칼럼의 검색 조건에 사용할때(조인 동등 조건과 같은) eq_ref라고 합니다.**
    - 이때 두 번째  부터 읽는 테이블의 type 칼럼에 eq_ref가 표시됩니다.
    - **또한 두 번째 이후 읽히는 테이블을 검색하는 유니크 키의 인덱스는 NOT NULL이어야 합니다.**
    - 다중 칼럼으로 만들어진 PK, 유니크 인덱스는 인덱스의 모든 칼럼이 비교 조건에 사용되어야 eq_ref 접근 방식을 사용할 수 있습니다.
- **결국 조인에서 처음 읽은 테이블의 칼럼값으로 두 번째 이후에 테이블을 읽을때 반드시 1건만 존재한다는 보장이 있어야 사용할 수 있는 접근 방법 입니다.**

```sql
EXPLAIN
SELECT * FROM dept_emp de, employees e
WHERE e.emp_no=de.emp_no AND de.dept_no='d005';

+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+--------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys             | key     | key_len | ref                 | rows   | filtered | Extra |
+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+--------+----------+-------+
|  1 | SIMPLE      | de    | NULL       | ref    | PRIMARY,ix_empno_fromdate | PRIMARY | 16      | const               | 165571 |   100.00 | NULL  |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                   | PRIMARY | 4       | employees.de.emp_no |      1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+--------+----------+-------+
```

- 실행 결과에서 1, 2라인이 조인으로 처리됨을 알 수 있습니다.
- dept_emp를 읽고 “e.emp_no=de.emp_no” 조건을 이용해 employees 테이블을 검색하는데 e.emp_no는 PK이므로 반드시 1건만 존재하는게 보장되어 eq_ref로 표시됩니다.

---

<br>

**ref**

- eq_ref와 달리 조인 순서와 관계없이 사용되고 PK, 유니크 키 등의 제약 조건도 없습니다.
- 인덱스 종류에 상관 없이 동등(equal) 조건으로 검색할 때는 ref 접근 방법이 사용됩니다.
- ref 타입은 반환 레코드가 1건이라는 보장이 없기에 const, eq_ref보다 빠르진 않습니다.
(그래도 동등 조건으로 검색하므로 빠른 레코드 조회 방법 중 하나)

```sql
EXPLAIN SELECT * FROM dept_emp WHERE dept_no='d005';

+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key     | key_len | ref   | rows   | filtered | Extra |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | PRIMARY       | PRIMARY | 16      | const | 165571 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
```

- 실행 계획의 ref 칼럼은 ref 접근 방법에서 값 비교에 사용된 입력값이 상수(’d005’) 이므로 const로 표시됩니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d7fe42e4-b0bd-41a9-8ec2-23dbf66a015b)


위 3가지의 공통점은 모두 동등 비교 연산자여야 합니다. ( `=`, `<=>` )

또한 모두 성능 문제를 일으키지 않는 접근 방법 이기에 크게 신경쓰지 않고 넘어가도 괜찮습니다.

<br>

**fulltext**

- **MySQL의 전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법** 입니다.
- 위에서 언급한 type 순서가 일반적인 처리 성능 순서이지만, 데이터 분포나 레코드 건수에 따라 달라 질 수 있습니다. → 비용 기반 옵티마이저에서 통계 정보를 이용해 비용을 계산하는 이유
- 반면에 전문 검색 인덱스는 통계 정보가 관리되지 않고 전혀 다른 SQL 문법을 사용해야 합니다.
- **전문 검색 조건은 우선순위가 상당히 높음**
    - MySQL은 일반 인덱스에서 접근 방법이 `const`, `eq_ref`, `ref`가 아니면 전문 검색 인덱스를 사용하는 조건을 선택해 처리함
- 전문 검색은 `"WATCH (...) AGAINST (...)"` 구문을 사용해서 실행함
    - 반드시 전문 검색용 인덱스가 준비돼 있어야 함

```sql
# 전문 검색 인덱스를 포함한 테이블 정의
CREATE TABLE employee_name (
emp_no int NOT NULL,
first_name varchar (14) NOT NULL,
last_name varchar (16) NOT NULL,
PRIMARY KEY (emp_no),
FULLTEXT KEY fx_name (first_name, last_name) WITH PARSER ngram
) ENGINE=InnoDB;
```

```sql
# 전문 검색 인덱스 전용 구문 사용하여 쿼리 조회
EXPLAIN
SELECT *
FROM employee_name
WHERE emp_no=10001
		AND emp_no BETWEEN 10001 AND 10005
		AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);

+----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table         | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employee_name | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
```

- 위 쿼리는 PK 조회, PK range 조회, 전문 검색 조건와 같이 3개의 조건이 있는데 PK 조회만 만족하면 끝나므로 type=const가 됩니다.

```sql
# emp_no=10001 조건 제외 및 재 질의
EXPLAIN
SELECT *
FROM employee_name
WHERE emp_no BETWEEN 10001 AND 10005
		AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);
+----+-------------+---------------+------------+----------+-----------------+---------+---------+-------+------+----------+-----------------------------------+
| id | select_type | table         | partitions | type     | possible_keys   | key     | key_len | ref   | rows | filtered | Extra                             |
+----+-------------+---------------+------------+----------+-----------------+---------+---------+-------+------+----------+-----------------------------------+
|  1 | SIMPLE      | employee_name | NULL       | fulltext | PRIMARY,fx_name | fx_name | 0       | const |    1 |   100.00 | Using where; Ft_hints: no_ranking |
+----+-------------+---------------+------------+----------+-----------------+---------+---------+-------+------+----------+-----------------------------------+
```

- 이때는 range 타입의 두 번째 조건이 아니라 전문 검색 조건이 선택됐습니다.
- 일반적으로 쿼리에 전문 검색 조건을 사용하면 MySQL 서버는 주저 없이 fulltext 접근 방법을 사용하지만 저자 분은 일반 인덱스를 이용하는 range 접근 방법이 더 빨리 처리되는 경우가 더 많았다고 합니다.
    - 따라서 조건별로 성능을 확인해 보는 편이 좋습니다.

---

<br>

**ref_or_null**

- ref 접근 방법과 같지만 NULL 비교가 추가된 형태 입니다. (ref 조건 + IS NULL 비교 사용시 표시됨)
- 저자분 말로는 많이 활용되진 않지만, 사용된다면 나쁘지 않은 접근 방법 이라고 합니다.

```sql
EXPLAIN
SELECT * FROM titles
WHERE to_date='1985-03-01' OR to_date IS NULL;

+----+-------------+--------+------------+-------------+---------------+-----------+---------+-------+------+----------+--------------------------+
| id | select_type | table  | partitions | type        | possible_keys | key       | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------------+---------------+-----------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | titles | NULL       | ref_or_null | ix_todate     | ix_todate | 4       | const |    2 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------------+---------------+-----------+---------+-------+------+----------+--------------------------+
```

---

<br>

**unique_subquery**

- WHERE 조건절에 사용될 수 있는 `IN(subquery)` 형태의 쿼리를 위한 접근 방법 입니다.
- `unique_subquery` 의미 그대로 서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 사용됩니다.

```sql
# 서브쿼리에서 emp_no=10001이라는 중복되지 않는 유니크한 값만 반환함
EXPLAIN
SELECT * FROM departments
WHERE dept_no IN (SELECT dept_no FROM dept_emp WHERE emp_no=10001);

# 변경된 쿼리
SELECT 
		*
FROM 
		dept_emp de
JOIN 
		departments d
WHERE 
    de.dept_no = d.dept_no
    AND
    de.emp_no = 10001;

# ix_empno_fromdate 인덱스는 복합 인덱스이므로 1건의 반환 결과를 기대할 수 없으므로 ref가
# d.dept_no는 PK이므로 1건의 결과를 예상할 수 있으므로 eq_ref를 표시합니다.
+----+-------------+-------------+------------+--------+---------------------------+-------------------+---------+----------------------------+------+----------+-------------+
| id | select_type | table       | partitions | type   | possible_keys             | key               | key_len | ref                        | rows | filtered | Extra       |
+----+-------------+-------------+------------+--------+---------------------------+-------------------+---------+----------------------------+------+----------+-------------+
|  1 | SIMPLE      | dept_emp    | NULL       | ref    | PRIMARY,ix_empno_fromdate | ix_empno_fromdate | 4       | const                      |    1 |   100.00 | Using index |
|  1 | SIMPLE      | departments | NULL       | eq_ref | PRIMARY                   | PRIMARY           | 16      | employees.dept_emp.dept_no |    1 |   100.00 | NULL        |
+----+-------------+-------------+------------+--------+---------------------------+-------------------+---------+----------------------------+------+----------+-------------+
```

→ 책에서는 dept_emp 테이블에 대해 unique_subquery type이 표시되지만, 나는 그렇지 않음.. ㅎㅎ

---

<br>

**index_subquery : `HAVETOFIND`**

- IN 연산자의 특성상 IN(subquery) 또는 IN(상수 나열) 형태에서 나열된 값들은 중복이 먼저 제거돼야 합니다.
- ~~앞서본 unique_subquery는 IN 조건의 서브쿼리가 중복 된 값을 만들어 내지 않는다는 보장이 있으므로 별도의 중복을 제거할 필요가 없었다.~~
    - `dept_emp는 복합 PK이므로 중복값이 있을 수 있음`
    따라서 서브 쿼리가 다음과 같이 질의 되어야 함 
    `SELECT dept_no FROM dept_emp WHERE dept_no='d005' and emp_no=10001`
- 서브쿼리 결과의 중복된 값을 인덱스를 이용해 제거할 수 있을 때 index_subquery 접근 방법이 사용됩니다.

- `index_subquery`와 `unique_subquery` 접근 방법의 차이
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e1034d9e-47a6-427c-ad3f-462b3b4b92b2)

    

---

<br>

**range**

- 인덱스 레인지 스캔 형태의 접근 방법을 말합니다. (인덱스를 하나의 값이 아니라 범위로 검색하는 경우)
- 주로 <, >, IS NULL, BETWEEN, IN, LIKE 등의 연산자를 이용해 인덱스를 검색할 때 사용됩니다.
- 애플리케이션의 쿼리가 가장 많이 사용하는 접근 방법
- 우선순위가 상당히 낮지만, 상당히 빠르고 range만 사용해도 최적의 성능이 보장됩니다.

```sql
EXPLAIN
SELECT * FROM employees WHERE emp_no BETWEEN 10002 AND 10004;

+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    3 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

현재 책에서 말하는 레인지 스캔은 const, ref, range 3가지 접근 방법을 모두 묶어서 지칭합니다.
(`인덱스를 효율적으로 사용한다`, `작업 범위 결정 조건으로 인덱스를 사용한다` 라는 표현은 위의 3가지 사용을 말함)

**→ 따지고 보면 const, ref, range는 일치하는 레코드를 찾고, 인덱스를 더 스캔하는지 즉시 반환하는지의 차이만 존재하고 본질적으론 인덱스 레인지 스캔임**

---

<br>

**index_merge**

- index_merge 접근 방법은 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들고 결과를 병합한다는 점에서 다른 접근 방법과는 다른점을 보입니다.
- 다만 다음과 같은 특징 때문에 생각보다 효율적으로 동작하진 못합니다.
    - 여러 인덱스를 읽으므로 range 접근 방법보다 효율이 떨어짐
    - 전문 검색 인덱스를 사용하는 쿼리에선 `index_merge` 적용 안됨https://dev.mysql.com/doc/refman/8.0/en/index-merge-optimization.html
    - index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이므로 두 집합의 `합집합`, `교집합`, `중복제거` 같은 부가적 작업 필요함
- 저자분은 메뉴얼의 index_merge 우선순위는 ref_or_null 바로 다음이지만 위의 이유로 range 아래로 옮김
- index_merge 접근 방법은 실행 계획에 좀 더 보완적인 내용이 표시됨(Extra 칼럼 관련)

```sql
EXPLAIN
SELECT * FROM employees
WHERE emp_no BETWEEN 10001 AND 11000 OR first_name='Smith';

+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys        | key                  | key_len | ref  | rows | filtered | Extra                                          |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | PRIMARY,ix_firstname | PRIMARY,ix_firstname | 4,58    | NULL | 1001 |   100.00 | Using union(PRIMARY,ix_firstname); Using where |
+----+-------------+-----------+------------+-------------+----------------------+----------------------+---------+------+------+----------+------------------------------------------------+
```

- 두 개의 조건이 OR 연산자로 연결됐는데 조건이 각각 다른 인덱스를 최적으로 사용할 수 있습니다.
- 따라서 MySQL은 각각 알맞은 인덱스를 선택하여 조회하고 두 결과를 병합하는 형태로 처리하는 실행 계획을 만듭니다.

---

<br>

**index**

- index를 사용하여 효율적으로 접근하는 것이 아니라 인덱스를 처음부터 끝까지 읽는 `인덱스 풀 스캔`을 의미합니다.
- 풀 테이블 스캔과 비교하는 레코드 건수는 같으나, **일반적으로 인덱스의 크기가 데이터 파일 전체 크기보다 작으므로 인덱스 풀 스캔이 풀 테이블 스캔보다 빠르게 처리됩니다.**
- 또한 쿼리 내용따라 정렬된다는 인덱스의 장점을 활용할 수 있기에 훨씬 효율적입니다.
- **아래 조건에서 1, 2번째 혹은 1, 3번째 조건을 충족하는 쿼리에서 사용됩니다.**
    - `range나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우`
    - `인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우(데이터 파일 안읽어도 되는 경우)`
    - `인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우(별도의 정렬 작업을 피하는 경우)`

```sql
EXPLAIN
SELECT * FROM departments ORDER BY dept_name DESC LIMIT 10;

# 본인은 Filesort를 사용함 ㅎㅎ
+----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table       | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | departments | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |   100.00 | Using filesort |
+----+-------------+-------------+------------+------+---------------+------+---------+------+------+----------+----------------+
```

- 책에선 아무런 WHERE 조건이 없어 `1번째 조건을 충`족하고, `3번 조건을 ux_deptname인덱스를 통해 충족`시키므로 type=index임을 기대했지만 `Using filesort`를 사용함
- 만약 index로 사용했을 경우 LIMIT 조건이 없거나 가져와야 할 레코드 건수가 많아지면 상당히 느린 처리를 수행합니다.

---

<br>

**ALL**

- 풀 테이블 스캔을 의미합니다.
- 지금까지 설명한 접근 방법으로는 처리할 수 없을 때 가장 마지막에 선택하는 `가장 비효율적인 방법`입니다.
- InnoDB는 풀 테이블 스캔이나 인덱스 풀 스캔과 같은 대량의 디스크 I/O작업과 같이 한꺼번에 많은 페이지를 읽어 들이는 기능을 제공합니다.
    - 이를 `Read Ahead`라고 합니다.
    - 데이터 웨어하우스, 배치 프로그램처럼 대용량의 레코드를 처리하는 쿼리에선 잘못 튜닝된 쿼리(억지로 인덱스를 사용하게 한 쿼리)보다 더 나은 방법이기도 합니다.
    - **쿼리 튜닝이 무조건 인덱스 풀 스캔, 풀 테이블 스캔을 사용하지 못하게 하는 것은 아닙니다.**

`index`, `ALL` 접근 방법은 작업 범위를 제한하는 조건이 아니므로 사용자에게 빠른 응답을 줘야하는 웹 서비스, 온라인 트랜잭션 처리 환경에선 적합하지 않습니다.

실제로 사용하려면 테이블에 데이터를 어느 정도 저장한 상태에서 쿼리의 성능을 확인해보고 적용하는것이 좋습니다.

- MySQL Read Ahead 이용시 언제 실행할지, 몇 개의 스레드를 사용할지 설정 할 수 있음
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/1b826f0d-9bb2-4152-8fca-817be484404a)

    

---

<br><br>

### 10.3.6 possible_keys 칼럼

- 옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록
(사용될 법했던 인덱스의 목록)
- possible_keys 칼럼에 인덱스 이름이 나열됐다고 해서 그 인덱스를 사용한다고 판단하지 않도록 주의해야 합니다.
`(쿼리를 튜닝할때 인덱스의 교체 만으로 요행을 바라지 말라는 뜻?)`

---

<br><br>

### 10.3.7 key 칼럼

- 최종 선택된 실행 계획에서 사용하는 인덱스를 의미합니다.
- 쿼리를 튜닝할때는 key 칼럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요합니다.
- index_merge만 key칼럼에 2개 이상의 인덱스가 표시됩니다.
- type이 ALL이어서 풀 테이블 스캔을 한다면 key 칼럼은 null

---

<br><br>

### 10.3.8 key_len 칼럼

- 사용자가 쉽게 무시하는 정보지만 매우 중요한 정보 중 하나입니다.
- 실제 업무에선 단일 보단 다중 칼럼 인덱스가 더 많습니다.
- **이때 key_len 칼럼의 값은 쿼리를 처리하기 위해 다중 칼럼으로 구성된 인덱스에서 몇 개의 칼럼까지 사용했는지 알 수 있게 도와줍니다.**
    - **더 정확하게는 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려줌**
- 다중 칼럼 및 단일 칼럼 인덱스에서도 같은 지표 제공

```sql
EXPLAIN SELECT * FROM dept_emp WHERE dept_no='d005';
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key     | key_len | ref   | rows   | filtered | Extra |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | PRIMARY       | PRIMARY | 16      | const | 165571 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
```

- dept_no는 칼럼의 타입이 CHAR(4)이므로 PK 앞쪽 4글자 16바이트만 유효하게 사용했다는 의미가 됩니다.
- dept_no 칼럼은 utf8mb4 문자 집합을 사용하는데 문자 하나가 차지하는 공간이 1바이트에서 4바이트까지 가변적입니다.
    - 하지만, **MySQL 서버가 utf8mb4문자를 위해 메모리 공간을 할당해야 할 때는 문자와 관계없이 고정적으로 4바이트로 계산합니다.**

```sql
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no='d005' AND emp_no=10001;

+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys             | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | dept_emp | NULL       | const | PRIMARY,ix_empno_fromdate | PRIMARY | 20      | const,const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+---------------------------+---------+---------+-------------+------+----------+-------+
```

- emp_no는 INTEGER(4바이트 차지)입니다. 따라서 PK를 사용했으므로 key_len = 16 + 4 = 20바이트 사용

```sql
CREATE TABLE titles (
		emp_no int NOT NULL,
	  title varchar (50) NOT NULL,
		from_date date NOT NULL,
		to_date date DEFAULT NULL,
		PRIMARY KEY (emp_no, from_date, title),
		KEY ix_todate (to_date)
);

EXPLAIN SELECT * FROM titles WHERE to_date <=' 1985-10-10';

+----+-------------+--------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | titles | NULL       | range | ix_todate     | ix_todate | 4       | NULL |   51 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
```

- 때로는 key_len 필드의 값이 데이터 타입의 길이보다 조금 더 길게 표시될 수 있으니 테이블 생성 이후 쿼리의 실행 계획을 살펴봐야 합니다.
- DATE 타입은 3바이트를 사용하는 key_len은 4바이트인데, to_date 칼럼이 DATE 타입을 사용하면서 NULLABLE 칼럼으로 정의됐습니다.
- MySQL은 NOT NULL이 아닌 칼럼은 칼럼의 NULL 구분을 위해 1바이트를 추가로 더 사용합니다.

---

<br><br>

### 10.3.9 ref 칼럼

- 접근 방법이 ref면 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여줍니다.
- 상수값이라면 const, 다른 테이블의 칼럼값이라면 그 테이블명과 칼럼명이 표시됩니다.
- ref 칼럼 출력은 몇개 케이스를 제외하고 크게 신경쓰지 않아도 괜찮습니다.

1. ref 칼럼이 func라면 참조용으로 사용되는 값을 콜레이션 변환 혹은 값 자체의 연산을 거쳐서 참조됨을 의미합니다.
    
    ```sql
    EXPLAIN SELECT * FROM employees e, dept_emp de WHERE e.emp_no=de.emp_no;
    
    # de는 드라이빙 테이블이므로 따로 제공하진 않음
    +----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+--------+----------+-------+
    | id | select_type | table | partitions | type   | possible_keys     | key     | key_len | ref                 | rows   | filtered | Extra |
    +----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+--------+----------+-------+
    |  1 | SIMPLE      | de    | NULL       | ALL    | ix_empno_fromdate | NULL    | NULL    | NULL                | 331143 |   100.00 | NULL  |
    |  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY           | PRIMARY | 4       | employees.de.emp_no |      1 |   100.00 | NULL  |
    +----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+--------+----------+-------+
    ```
    
    위 쿼리는 emp_no 칼럼에 어떤 변환, 가공도 수행하지 않았기에 조인 대상 칼럼의 이름이 그대로 표기됩니다.
    
    ```sql
    # 조인 조건에 간단한 산술 표현식을 넣은 쿼리
    EXPLAIN SELECT * FROM employees e, dept_emp de WHERE e.emp_no=(de.emp_no-1);
    
    +----+-------------+-------+------------+--------+---------------+---------+---------+------+--------+----------+-------------+
    | id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-------+------------+--------+---------------+---------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | de    | NULL       | ALL    | NULL          | NULL    | NULL    | NULL | 331143 |   100.00 | NULL        |
    |  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | func |      1 |   100.00 | Using where |
    +----+-------------+-------+------------+--------+---------------+---------+---------+------+--------+----------+-------------+
    ```
    
    dept_emp 테이블의 emp_no값에서 1을 뺀 값으로 employees 테이블과 조인합니다. 따라서 ref 칼럼의 값이 func로 표시됩니다.
    

- 위 예제는 사용자가 명시적으로 변환시켰지만, MySQL 서버가 내부적으로 값을 변환 해야 할 때도 ref 칼럼에는 func가 출력됩니다.
- 만약 문자집합이 일치하지 않는 두 문자열 칼럼을 조인하거나 숫자 타입의 칼럼과 문자열 타입의 칼럼으로 조인하면 MySQL 서버가 내부적으로 값을 변화하여 func가 표시됩니다.
- 따라서 **MySQL 서버가 변환을 하지 않도록 조인 칼럼의 타입은 일치시키는것이 좋습니다.**
