# 📌 11장 쿼리 작성 및 최적화

## ✅ 11.4 SELECT

INSERT, UPDATE에 비해 SELECT는 여러 개의 테이블로부터 데이터를 조합해 가져오므로 어떻게 읽을지 많은 주의를 기울여야 합니다.

<br><br>

### 11.4.1 SELECT 질의 처리 순서

- `SELECT 문장`: SQL 전체
- `SELECT 절`: 단일 SELECT 키워드 (절일나 SELECT, FROM, JOIN, WHERE 등 작은 단위와 표현식의 함)

```sql
SELECT s.emp_no, COUNT(DISTINCT e.first_name) AS cnt 
FROM salaries s  
		INNER JOIN employees e ON e.emp_no=s.emp_no
WHERE s.emp_no IN (100001, 100002)
GROUP BY s.emp_no HAVING AVG(s.salary) > 1000
ORDER BY AVG(s.salary) LIMIT 10;
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/14fcb0b5-77ff-4ed2-92c5-e34ca7e30388)


- 위와 같이 다양한 절이 사용된 SELECT 문장에서 어떤 절이 먼저 실행될지 모르면 처리 내용이나 결과를 예측하기 어렵습니다.

---

<br>

**일반적인 질의 순서**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/bc9d3bb6-2fdd-49b5-98d3-0b46842fa764)


- 각 절이 비어있는 경우는 가능하지만 대체로 위 순서로 쿼리가 실행됩니다.
(CTE, 윈도우 함수 제외)

---

<br>

**ORDER BY가 조인보다 먼저 실행되는 경우**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/895cb380-11c4-4b81-8984-db57a4e06d16)


- 위 경우는 첫 번째 테이블만 읽어서 정렬 수행 후 나머지 테이블을 읽습니다.
- 주로 GROUP BY 절 없이 ORDER BY만 사용된 쿼리에서 사용될 수 있는 순서입니다.
- 실행 순서를 벗어난 쿼리가 필요하면 주로 서브쿼리로 작성된 인라인 뷰를 사용해야 합니다.
    
    ```sql
    # ORDER BY와 LIMIT의 순서를 변경
    SELECT emp_no, cnt
    FROM (
    		SELECT s.emp_no, COUNT(DISTINCT e.first_name) AS cnt, MAX(s.salary) AS max_salary
    		FROM salaries s
    			inner join employees e ON e.emp_no=s.emp_no
    		WHERE s.emp_no IN (100001, 100002)
    		GROUP BY s.emp_no
    		HAVING MAX(s.salary) > 1000
    		LIMIT 10 # LIMIT와 ORDER BY 순서가 바뀌었기에 이전 쿼리와 다른 결과를 반환할 수 있음
    ) temp_view
    ORDER BY max_salary;
    ```
    
    - 인라인 뷰는 10.3.2.8처럼 임시테이블이 사용되기에 주의
    - LIMIT은 항상 모든 처리의 결과에 대해 레코드 건수를 제한함
    - 8.0버전부턴 서브쿼리를 외부 쿼리와 병합해 최적화 하므로 위 2개의 그림 중 하나의 순서로 실행됨 (조인 사용해)
- 또한 WITH절은 항상 제일 먼저 실행돼 임시테이블로 저장됨 
(그림에서 단독으로 조회되거나 조인되는 테이블로 활용)

<br><br>

### 11.4.2 WHERE  절과 GROUP BY 절, ORDER BY 절의 인덱스 사용

**11.4.2.1 인덱스를 사용하기 위한 기본 규칙**

- WHERE, ORDER BY, GROUP BY가 인덱스를 사용하려면 인덱스된 칼럼의 값 자체를 변환하지 않고 그대로 사용해야 합니다. (`원본값`)
(B-Tree가 칼럼의 값을 변환없이 정렬해 저장하므로)
    
    ```sql
    # 인덱스된 칼럼의 가공때문에 인덱스를 사용할 수 없음
    mysql> explain SELECT * FROM salaries WHERE salary * 10 > 150000;
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    |  1 | SIMPLE      | salaries | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2838426 |   100.00 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)
    
    # 인덱스를 사용할 수 있지만, 풀 테이블 스캔을 하는게 더 효율적이라 판단함
    mysql> explain SELECT * FROM salaries WHERE salary > 150000 / 10;
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    |  1 | SIMPLE      | salaries | NULL       | ALL  | ix_salary     | NULL | NULL    | NULL | 2838426 |    50.00 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)
    ```
    
- 복잡한 연산 수행을 사용해야 한다면 미리 계산된 값을 저장하는 가상 칼럼을 추가하고 해당 칼럼에 인덱스를 생성하거나 함수 기반의 인덱스를 사용하면 됩니다.
- 또한 WHERE절에 사용된 비교조건에서 연산자 양쪽의 비교 대상의 데이터 타입이 일치해야 합니다.
    
    ```sql
    CREATE TABLE tb_test (age VARCHAR(10), INDEX ix_age (age));
    INSERT INTO tb_test VALUES ('1'), ('2'), ('3'), ('4'), ('5'), ('6'), ('7');
    
    # age 칼럼의 데이터 타입과 비교되는 타입이 char, integer로 다르기에 인덱스 풀스캔을 함
    EXPLAIN SELECT * FROM tb_test WHERE age=2;
    +-------+
    | type  |
    +-------+
    | index |
    +-------+
    
    EXPLAIN SELECT * FROM tb_test WHERE age='2';
    +------+
    | type |
    +------+
    | ref  |
    +------+
    ```
    
    → 데이터 타입 불일치에서 오는 최적화를 못하는 현상은 버전이 올라간다고 해결되지 않습니다.
    

---

<br>

**11.4.2.2 WHERE 절의 인덱스 사용**

- 작업 범위 결정 조건, 체크 조건 두 가지 방식으로 인덱스를 사용할 수 있음
- `작업 범위 결정 조건`은 WHERE절에서 동등 비교 조건, IN으로 구성된 조건에 사용된 칼럼들이 인덱스 칼럼 구성의 `좌측부터 얼마나 일치하는가에 따라 달라집니다.`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/fc13b622-08bc-4185-83a4-9d5e8ff43c4f)

    
    - 위와 같이 WHERE절의 순서가 달라도 옵티마이저가 최적화를 수행합니다.
    - `COL_1`, `2`, `3`까지는 작업 범위 결정 조건으로 사용할 수 있지만, `COL_3`의 범위 비교 연산 때문에 `COL_4`의 조건은 체크 조건으로 사용됩니다.
- 8.0부터는 인덱스 구성 칼럼 별로 정렬(오름차순, 내림차순)을 혼합해 사용 가
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/fbe13236-c3fa-4a52-9857-b5fe96cc43cc)

    
- WHERE 절의 순서는 인덱스 적용 여부에 크게 상관이 없습니다. 단지 인덱스를 구성하는 칼럼에 대한 조건이 있는지 여부가 더 중요합니다.

위의 예시는 AND 연산자로 이어졌을 경우이며, OR는 완전히 바뀝니다.

```sql
SELECT *
FROM employees
WHERE first_name='Kebin' OR last_name='Poly';
```

- first_name은 인덱스 사용 가능, last_name은 불가능
- **풀 테이블 스캔 + 인덱스 레인지  스캔보다는 풀 테이블 스캔 한 번이 훨씬 빠름**
- AND는 레코드 건수를 줄이는 역할을 하지만 OR는 비교할 레코드가 더 늘어나므로 주의해야 함

---

<br>

**11.4.2.3 GROUP BY 절의 인덱스 사용**

- GROUP BY의 각 칼럼은 비교 연산자를 가지지 않습니다. (작업 범위 결정, 체크조건 고려 X)
- 다음과 같은 조건이 있음
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/64902a92-f737-46d7-961a-e25cdad70780)

    
    - GROUP BY 절에 명시된 칼럼이 인덱스 칼럼의 `순서와 위치가 같아야 한다.`
    - 인덱스를 구성하는 칼럼 중에서 뒤쪽에 있는 칼럼은 GROUP BY 절에 명시되지 않아도 인덱스를 사용할 수 있지만 `인덱스의 앞쪽에 있는 칼럼이 GROUP BY 절에 명시되지 않으면 인덱스를 사용할 수 없다.`
    - WHERE 조건절과는 달리 `GROUP BY 절에 명시된 칼럼이 하나라도 인덱스에 없으면 GROUP BY 절은 전혀 인덱스를 이용하지 못한다.`
- 다만 WHERE 조건절에서 동등 비교 조건으로 인덱스된 칼럼을 사용하는 경우 앞의 인덱스가 몇개 빠져도 인덱스를 사용가능한 GROUP BY가 될 수 있습니다.
    
    ```sql
    # 1, 2, 3 순서로 인덱스 사용 가능
    ... WHERE COL_1='상수' ... GROUP BY COL_2, COL_3
    # 1, 2, 3, 4
    ... WHERE COL_1='상수' AND COL_2='상수' ... GROUP BY COL_3, COL_4
    # 1, 2, 3, 4
    ... WHERE COL_1='상수' AND COL_2='상수' AND COL_3='상수' ... GROUP BY COL_4
    
    ```
    
    - 인덱스를 이용해 WHERE, GROUP BY가 모두 처리되는지 확인해보려면 WHERE절에 동등 비교 조건을 GROUP BY로 옮겼을때 똑같은 결과가 조회되는지 확인해보면 됩니다.

---

<br>

**11.4.2.4 ORDER BY 절의 인덱스 사용**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ba08b901-8401-4aba-a76b-fe6a471c74a7)


- GROUP BY와 흡사하지만 조건 하나가 더 있음
- 정렬되는 각 칼럼의 ASC, DESC 옵션이 인덱스와 전부 같거나 전부 정반대인 경우에만 사용할 수 있습니다.

---

<br>

**11.4.2.5 WHERE 조건과 ORDER BY(또는 GROUP BY)절의 인덱스 사용**

- WHERE + GROUP BY 혹은 WHERE + ORDER BY일 경우 다음 3가지 중 한 가지 방법으로만 인덱스를 이용합니다.
    - `WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 이용`
        - WHERE절 칼럼과 ORDER BY 절의 칼럼이 하나의 인덱스에 연속해 포함될 경우 이 방식으로 인덱스를 사용
        - 다른 2가지 방식보다 빠른 성능을 보임 → 될 수 있으면 이 방식으로 처리하도록 해야함
    - `WHERE 절만 인덱스를 이용`
        - ORDER BY 절은 인덱스를 이용한 정렬이 불가능하며, 인덱스를 통해 검색된 결과 레코드를 별도의 정렬 처리 과정(Using Filesort)을 거쳐 정렬을 수행합니다.
        - 주로 이 방법은 WHERE 절의 조건에 일치하는 레코드의 건수가 많지 않을 때 효율적인 방식입니다.
    - `ORDER BY 절만 인덱스를 이용`
        - ORDER BY 절은 인덱스를 이용해 처리하지만 WHERE 절은 인덱스를 이용하지 못합니다.
        - 이 방식은 `ORDER BY 절의 순서대로 인덱스를 읽으면서 레코드 한 건씩 WHERE 절의 조건에 일치하는지 비교하고, 일치하지 않을 때는 버리는 형태로 처리`합니다.
        - 주로 아주 많은 레코드를 조회해서 정렬해야 할 때는 이런 형태로 튜닝하기도 함

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/75a8bac5-0df6-4e9a-b2a5-b8395ecf1f42)


- 우측 그림의 COL_3 > ? 조건이 있지만, ORDER BY 절에 COL_3 칼럼을 사용하므로 인덱스를 이용할 수 있습니다.
    
    ```sql
    # 인덱스는 (COL_1, COL_2, COL_3)로 구성됨
    
    # WHERE 범위 비교 조건의 칼럼이 ORDER BY에도 사용됐기에 인덱스 사용 가능
    SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_1, COL_2, COL_3;
    # 범위 비교 조건이 ORDER BY에 없기에 인덱스 사용 불가능
    SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_2, COL_3;
    ```
    

→ GROUP BY도 ORDER BY절과 똑같은 기준이 적용됩니다.

---

<br>

**11.4.2.6 GROUP BY 절과 ORDER BY 절의 인덱스 사용**

- GROUP BY + ORDER BY 절이 사용된 쿼리에서 두 절 모두 하나의 인덱스를 사용하려면 `두 절에 명시된 칼럼의 순서 + 내용`이 모두 같아야 합니다.
- 둘 중 하나라도 인덱스를 사용할 수 없다면 둘 다 인덱스를 사용할 수 없습니다.
- 8.0부터는 GROUP BY 절이 칼럼의 정렬까지 보장하지 않는 형태로 바뀌었습니다. 따라서 그루핑 + 정렬을 하려면 ORDER BY도 필수적으로 명시해야 합니다.

---

<br>

**11.4.2.7 WHERE 조건과 ORDER BY 절, GROUP BY 절의 인덱스 사용**

- 아래 3개의 질문을 기본으로 해서 그림 11.9의 흐름을 적용할 수 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/cec74352-f6d7-4c65-ad01-9197687dd1ca)


<br><br>

### 11.4.3 WHERE 절의 비교 조건 사용 시 주의사항

쿼리가 최적으로 실행되기 위해선 적합한 인덱스와 함께 WHERE 절에 사용되는 비교 조건의 표현식을 적절하게 사용해야 합니다.

<br>

**11.4.3.1 NULL 비교**

- 다른 DBMS와 다르게 MySQL 에서 NULL 값이 포함된 레코드도 인덱스로 관리합니다.
- 이는 인덱스에 서 NULL을 하나의 값으로 인정해서 관리함을 의미함
    - SQL 표준에선 NULL은 비교할 수 없는 값 → 연산 비교에서 한쪽이 NULL이면 결과도 NULL이 반환됨
- 쿼리에서 NULL인지 비교하려면 IS NULL(혹은 `<=>`) 연산자를 사용해야 합니다.

```sql
EXPLAIN SELECT * FROM titles WHERE to_date IS NULL;

+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------------------------+
| id | select_type | table  | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | titles | NULL       | ref  | ix_todate     | ix_todate | 4       | const |    1 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------------------------+
```

- to_date가 NULL인 레코드를 조회하지만 ix_todate 인덱스를 ref(동등비교) 방식으로 적절히 이용하고 있습니다.

ISNULL()이라는 함수도 있지만 주의해야 할 부분이 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c711f4cc-df57-4a23-bf3c-cd9786148493)


3, 4번째 방식처럼 사용하게 되면 인덱스 혹은 테이블을 풀 스캔하는 형태로 처리됩니다. 따라서 `IS NULL 연산자 사용을 권장`합니다.

---

<br>

**11.4.3.2 문자열이나 숫자 비교**

- 문자열 칼럼이나 숫자 칼럼을 비교할 때는 반드시 타입에 맞는 상숫값 사용을 권장합니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/63bdbed0-d463-480b-b71f-47044add3976)


- 숫자와 문자를 비교하면 문자를 숫자로 변환하기 때문에(11.3.1.2) 네 번째 쿼리는 모든 first_name문자열을 숫자로 변환해 비교를 수행합니다.
- 칼럼 타입 변환 때문에 ix_firstname 인덱스를 사용하지 못합니다.

```sql
EXPLAIN SELECT * FROM employees WHERE first_name10001;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | ix_firstname  | NULL | NULL    | NULL | 299113 |    10.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
```

---

<br>

**11.4.3.3 날짜 비교**

- 날짜 타입으로 DATE, DATETIME, TIMESTAMP가 있고 각각 DATE는 날짜를 나머지는 날짜 + 시간을 함께 저장합니다. (시간만 저장하는 TIME도 있음)

- `DATE or DATETIME과 문자열 비교`
    - DATE or DATETIME 과 문자열을 비교하면 문자열 값을 자동으로 DATETIME으로 변환해 비교를 수행합니다.
    - 따라서 명시적으로 STR_TO_DATE와 같은 내장 함수를 사용하지 않아도 알아서 변환해줍니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b9291c50-5e9f-4f7a-8c04-067eb1dfb1ff)

        
    - 또한 DATE, DATETIME 타입의 컬럼을 변경하지 않고 비교하는 상수를 변경하는 형태로 조건을 사용해야 인덱스를 이용할 수 있습니다.

- `DATE와 DATETIME의 비교`
    - DATETIME → DATE 변환을 하려면 DATE() 함수를 사용하여 시간 부분은 버릴 수 있습니다.
    - DATETIME을 그대로 DATE와 비교하면 `MySQL은 DATE → DATETIME으로 변환해 같은 타입을 만들고 비교를 수행`합니다.
        - 위 같은 변환은 인덱스 사용 여부에 영향을 미치지 않기에 쿼리 결과에 주의해서 사용해야 합니다.

- `DATETIME과 TIMESTAMP의 비교`
    - DATE, DATETIME과 TIMESTAMP를 그대로 비교하면 문제없이 작동하는것처럼 보입니다. (인덱스 레인지 스캔을 사용함)
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/289f54ed-38e5-4431-916b-b87b9348bde4)

    
    - UNIX_TIMESTAMP()는 MySQL 내부적으로 단순한 숫자값입니다.
    - 반드시 비교 값으로 사용되는 상수 리터럴을 비교 대상 칼럼의 타입에 맞게 변환해서 사용하는것이 좋습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6ea79453-eaf4-4533-a38f-20bb3d67986d)

        

---

<br>

**11.4.3.4 Short-Circuit Evaluation**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/903fa394-ffc5-4c06-a472-610d5c33f3b7)


- 위와 같은 식에서 in_transaction이 false면 has_modified()는 호출하지도 않고 if문을 벗어납니다.
- 반대로 `||` 이라면 in_transaction이 true면 has_modified()는 호출하지 않고 if문을 실행합니다.
- **Short-Circuit Evaluation**는 ****여러개의 논리 연산에서 선행 표현식의 결과에 따라 후행 표현식을 평가할지 말지를 결정하는 최적화를 말합니다.

```sql
# 1. 타임존을 서울로 변경 했을때 1991-01-01보다 큰 모든 레코드의 수
SELECT COUNT(*) FROM salaries
WHERE CONVERT_TZ(from_date,'+00:00','+09:00') > '1991-01-01';
+----------+
| COUNT(*) |
+----------+
|  2442943 |
+----------+

# 2. 1985-01-01 보다 작은 날짜를 가진 레코드 수
SELECT COUNT(*) FROM salaries WHERE to_date < '1985-01-01';
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/e5c18c5b-7ec4-4bb4-94ff-5af3731f173d)


- `1번 조건 && 2번 조건`보다 `2번 조건 && 1번 조건`이 short-circuit evaluation이 적용됐으므로 더 효율적인것을 알 수 있습니다.

하지만, Short-curcuit Evaluation과는 무관하게 WHERE절의 조건 중에서 인덱스를 사용할 수 있는 조건이 있다면 해당 조건을 최우선으로 사용합니다. 
(그래서 조건의 순서가 인덱스 사용 여부를 결정하지 않음)

```sql
# first_name은 인덱스를 효율적으로 사용할 수 있음
SELECT * FROM employees
WHERE last_name='Aamodt'
		AND first_name='Matt';
```

- 이때는 인덱스를 이용해 필요한 레코드만 빠르게 가져오고 나머지 조건을 순서대로 평가합니다.

```sql
# first_name을 먼저 찾음 -> 서브쿼리로 필터링하고 -> last_name 필터링함
SELECT * 
FROM employees e
WHERE e.first_name='Matt'
		AND EXISTS (SELECT 1 FROM salaries s
								WHERE s.emp_no = e.emp_no AND s.to_date > '1995-01-01'
								GROUP BY s.salary HAVING COUNT(*) > 1)
		AND e.last_name = 'Aamodt';

# 상태 초기화
FLUSH STATUS;

# last_name -> 서브쿼리로 진행하도록 순서 변경
SELECT * 
FROM employees e
WHERE e.first_name='Matt'
		AND e.last_name = 'Aamodt'
		AND EXISTS (SELECT 1 FROM salaries s
								WHERE s.emp_no = e.emp_no AND s.to_date > '1995-01-01'
								GROUP BY s.salary HAVING COUNT(*) > 1);

SHOW STATUS LIKE 'Handler%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Handler_commit             | 1     |
| Handler_delete             | 0     |
| Handler_discover           | 0     |
| Handler_external_lock      | 4     |
| Handler_mrr_init           | 0     |
| Handler_prepare            | 0     |
| Handler_read_first         | 0     |
| Handler_read_key           | 9     |
| Handler_read_last          | 0     |
| Handler_read_next          | 15    |
| Handler_read_prev          | 0     |
| Handler_read_rnd           | 0     |
| Handler_read_rnd_next      | 8     |
| Handler_rollback           | 0     |
| Handler_savepoint          | 0     |
| Handler_savepoint_rollback | 0     |
| Handler_update             | 0     |
| Handler_write              | 7     |
+----------------------------+-------+

FLUSH STATUS;

SELECT * 
FROM employees e
WHERE e.first_name='Matt'
		AND EXISTS (SELECT 1 FROM salaries s
								WHERE s.emp_no = e.emp_no AND s.to_date > '1995-01-01'
								GROUP BY s.salary HAVING COUNT(*) > 1)
		AND e.last_name = 'Aamodt';

SHOW STATUS LIKE 'Handler%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Handler_commit             | 1     |
| Handler_delete             | 0     |
| Handler_discover           | 0     |
| Handler_external_lock      | 4     |
| Handler_mrr_init           | 0     |
| Handler_prepare            | 0     |
| Handler_read_first         | 0     |
| Handler_read_key           | 9     |
| Handler_read_last          | 0     |
| Handler_read_next          | 15    |
| Handler_read_prev          | 0     |
| Handler_read_rnd           | 0     |
| Handler_read_rnd_next      | 8     |
| Handler_rollback           | 0     |
| Handler_savepoint          | 0     |
| Handler_savepoint_rollback | 0     |
| Handler_update             | 0     |
| Handler_write              | 7     |
+----------------------------+-------+
```

- having절 때문에 세미 조인 최적화를 사용하지 못함
- e.first_name의 조건에만 인덱스를 사용할 수 있고 나머지는 불가능합니다.
- 인덱스를 사용하지 못하는 두 조건의 순서를 바꿔서 실행할 경우 Short-curcuit Evaluation 때문에 차이가 발생할 수 있습니다.
    - 책에선 last_name을 우선시 했을때 260건 정도로 처리를 완료함을 보여주지만 실제로 그렇진 않았습니다.
    - 반면에 서브쿼리를 먼저 수행하면 많은 레코드를 읽고 쓰는 작업을 합니다.

→ MySQL 서버에서 쿼리 작성시 가능하면 복잡한 연산 혹은 다른 테이블의 레코드를 읽어야 하는 서브쿼리 조건은 WHERE 절의 맨 뒤쪽으로 배치하는 것이 좋습니다.
(인덱스가 있는 칼럼들의 위치는 상관 없음)

---

<br><br>

### 11.4.4 DISTINCT(9.2.5 참조)

- 특정 칼럼의 유니크한 값을 조회하기 위해 사용됩니다.
- 다만 성능 적인 문제 + 쿼리의 의도와 결과가 달라지는 경우가 있습니다.
- 테이블 간 조인시 중복을 줄이기 위해 남용하는 경우가 있는데 조인 쿼리 작성시 각 테이블간 1:1, 1:M 조인인지 파악하는 업무적인 특성일 잘 이해하는것이 중요합니다.

---

<br><br>

### 11.4.5 LIMIT n

- LIMIT은 쿼리 결과에서 지정된 순서에 위치한 레코드만 가져오고자 할 때 사용합니다.

```sql
SELECT * FROM employees
WHERE emp_no BETWEEN 10001 AND 10010
ORDER BY first_name
LIMIT 0, 5;
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a5984fdc-d3cf-407b-88be-16ba83231b13)


LIMIT은 쿼리의 가장 마지막에 실행되고, 찾고자 하는 레코드를 모두 찾았다면 즉시 쿼리를 종료합니다.(정렬이어도 상위 5건이 정렬되면 쿼리 종료)

- **GROUP BY, DISTINCT를 LIMIT와 같이 사용했을때 작동 방법**
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6ec150e2-eba7-47c5-846e-ffb9c2e29315)

    
    - 1번은 LIMIT가 없다면 풀 테이블 스캔을 하겠지만, LIMIT 덕분에 범위를 확 줄일 수 있기에 쿼리가 상당히 빨리 끝납니다.
    - GROUP BY를 먼저하고 LIMIT를 하기에 GROUP BY 작업이 반드시 완료되어야 합니다. 따라서 크게 서버의 작업 내용을 줄여주진 않습니다.
    - DISTINCT 처리를 위해 풀테이블 스캔을 통해 employees 테이블의 레코드를 읽으며 중복 제거를 합니다.(임시테이블 사용) 위 작업을 하다가 LIMIT의 개수만큼 채워지면 쿼리를 멈춥니다. 정렬 조건도 없기에 LIMIT절이 작업량을 많이 줄여줍니다.
    - WHERE 조건에 일치하는 레코드를 정렬하는데 10건이 완성되면 즉시 반환합니다. 정렬을 하려면 모든 레코드와 비교하여 정렬해야 하기에 크게 작업량을 줄여주진 못합니다.

→ 위 예제는 인덱스를 적절히 이용하지 못하는 경우인데 만약 ORDER BY, DISTINCT, GROUP BY가 인덱스를 이용해 처리될 수 있다면 LIMIT절은 꼭 필요한 만큼의 레코드만 읽게 만들기에 작업량을 줄여줍니다.

- LIMIT는 인자 2개 혹은 1개를 사용해 지정합니다. (첫번째 인자는 offset)
    
    ```sql
    SELECT * FROM employees LIMIT 10; # 0 ~ 10 가져옴
    SELECT * FROM employees LIMIT 10, 10; # 11 ~ 20 가져옴 
    ```
    
- LIMIT 인자로 표현식이나 별도의 서브쿼리를 사용할 수 없습니다.
- LIMIT는 결과 레코드의 건수가 아니라 해당 결과를 만들기 위해 어떤 작업들을 했는지 파악하는것이 중요합니다.
    
    ```sql
    # 2000000건을 버리고 마지막 10건만 반환 -> 느림
    # JPA는 어떻게 하더라..?
    SELECT * FROM salaries ORDER BY salary LIMIT 2000000,10;
    
    # WHERE 조건으로 읽을 위치를 찾고 해당 위치에서 10개만 읽는 쿼리
    SELECT * FROM salaries ORDER BY salary LIMIT 0, 5;
    
    # 두번째 페이지 읽기
    SELECT * FROM salaries
    WHERE salary >= 38836 AND NOT (salary = 38836 AND emp_no <= 64198)
    ORDER BY salary LIMIT 0, 5;
    
    ```
    
    - `NOT (salary = 38836 AND emp_no <= 64198)`
        - 이전 페이지에서 이미 조회됐던 레코드를 제외하기 위한 조건
        - 다만 salary 칼럼은 중복을 허용하므로 단일 조건으로하면 중복이 발생할 수 있으니 emp_no 조건과 함께 조건을 걸어줍니다.

---

<br><br>

### 11.4.6 COUNT()

- 결과 레코드의 건수를 반환하는 함수 입니다.
- `“*”` 를 사용하면 레코드 자체를 의미하기에, 모든 레코드의 건수를 계산할 수 있습니다. (COUNT(1), COUNT(*)모두 동일한 처리 성능)
- InnoDB 스토리지 엔진을 사용하는 테이블에서 COUNT(*)는 **WHERE 조건이 없더라도 직접 데이터, 인덱스를 읽어야 레코 드 건수를 가져올 수 있기 때문에 주의**해야 합니다.
(대략적인 정보만 필요하면 SHOW TABLE STATUS 명령도 활용해보면 좋음)
- 레코드 건수를 세는 `COUNT(*)`쿼리와 무관한 ORDER BY, LEFT JOIN같은 작업을 같이 포함하면 안됩니다.
(8.0부터는 옵티마이저가 `COUNT(*)` ORDER BY 절은 무시하도록 개선됨)
- 인덱스를 제대로 사용하도록 튜닝되지 못한 COUNT(*) 쿼리도 많은 부하를 일으키기에 주의깊게 작성해야 함
- COUNT(colName)은 해당 칼럼에서 NULL은 제외하고 카운팅하게 됩니다.
- 주의
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f36a2743-6752-4bda-a8ea-aeadf3ef0ffa)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f563e363-380c-42c1-8511-8342314ce9d9)

    
    `HAVETOFIND` 직접 레코드를 삭제하는 경우가 아니라면 문제가될 수 있을수도 있겠다?
 ---   

<br><br>

### 11.4.7 JOIN


**11.4.7.1 JOIN의 순서와 인덱스**

- 8.3.4.1절 인덱스 레인지 스캔은 인덱스 탐색 단계와 스캔 하는 과정으로 구분할 수 있습니다.
- 탐색을 통해 스캔하는 레코드 건수는 일반적으로 소량이기에 탐색 단계의 부하가 상대적으로 높은 편입니다.
- 조인 작업에서 드라이빙 테이블을 읽을땐 한 번의 탐색 작업 이후 스캔만 실행하면 됩니다.
- 반면에 드리븐 테이블에선 탐색 작업과 스캔 작업을 드라이빙 테이블에서 읽은 레코드 건수만큼 반복됩니다.
- 따라서 옵티마이저는 드리븐 테이블을 최적으로 읽을 수 있게 실행 계획을 수립합니다.

```sql
SELECT * 
FROM employees e, dept_emp de 
WHERE e.emp_no=de.emp_no;
```

- employees 테이블의 emp_no 칼럼과 dept_emp 테이블의 emp_no 칼럼에 인덱스가 있거나 없을때의 조인순서 차이
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/aad41ced-9ee6-4e3c-9180-beb8455cd73e)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/01bd30f8-68c8-48b4-8266-990e484de0c6)

    

---

<br>

**11.4.7.2 JOIN 칼럼의 데이터 타입**

- 조인 칼럼간 비교에서 각 칼럼의 데이터 타입이 일치하지 않으면 인덱스를 효율적으로 이용할 수 없습니다.

```sql
CREATE TABLE tb_test1 (user_id INT, user_type INT, PRIMARY KEY(user_id));
CREATE TABLE tb_test2 (user_type CHAR(1), type_desc VARCHAR(10), PRIMARY KEY(user_type));

EXPLAIN
SELECT *
FROM tb_test1 tb1, tb_test2 tb2
WHERE tb1.user_type=tb2.user_type;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL                                       |
|  1 | SIMPLE      | tb2   | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    1 |   100.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
```

- 조인 칼럼의 타입이 다르므로 두 테이블 모두 풀 테이블 스캔으로 접근합니다. 또한 해시 조인이 실행됨을 알 수 있습니다.
- 타입이 서로 다르기 때문에 char 타입을 숫자로 변환해서 비교하는데 이때 인덱스의 변형이 필요하므로 tb_test2 인덱스를 활용할 수 없게 됩니다.
- 옵티마이저는 풀 테이블 스캔이 일어나는걸 알고 그나마 빨리 실행되는 조인버퍼를 활용한 해시조인을 합니다.
- 데이터 타입 불일치는 `CHAR-VARCHAR`, `INT-BIGINT`, `DATE-DATETIME` 사이에선 발생하지 않습니다.
- 다만 다음과 같은 경우는 문제가 될 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/381ffa17-cdf2-4589-a80d-80c9b6a9c9ea)

    

---

<br>

**11.4.7.3 OUTER JOIN의 성능과 주의사항**

- 아우터를 사용하지 않아도 될 상황에 사용하기
    
    ```sql
    EXPLAIN
    SELECT *
    FROM employees e
    		LEFT JOIN dept_emp de ON de.emp_no=e.emp_no
    		LEFT JOIN departments d ON d.dept_no=de.dept_no AND d.dept_name='Development';
    
    +----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
    | id | select_type | table | partitions | type   | possible_keys       | key               | key_len | ref                  | rows   | filtered | Extra       |
    +----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
    |  1 | SIMPLE      | e     | NULL       | ALL    | NULL                | NULL              | NULL    | NULL                 | 299113 |   100.00 | NULL        |
    |  1 | SIMPLE      | de    | NULL       | ref    | ix_empno_fromdate   | ix_empno_fromdate | 4       | employees.e.emp_no   |      1 |   100.00 | NULL        |
    |  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY,ux_deptname | PRIMARY           | 16      | employees.de.dept_no |      1 |   100.00 | Using where |
    +----+-------------+-------+------------+--------+---------------------+-------------------+---------+----------------------+--------+----------+-------------+
    ```
    
    - employees 테이블은 드라이빙 테이블로 풀스캔, `dept_emp`와 `departments` 테이블을 드리븐 테이블로 사용합니다.
    - 테이블의 데이터가 일관되지 않은 경우 아우터 조인이 필요한데, 현재`employees.emp_no`가 `dept_emp`에 없는 경우가 없으므로 굳이 사용할 필요가 없습니다.
    - 옵티마이저는 절대 아우터로 조인되는 테이블을 드라이빙 테이블로 선택하지 못하기에 풀 스캔이 필요한 employees 테이블을 드라이빙 테이블로 선택합니다.
    (레코드 건수가 많아서 성능이 떨어짐)
    - 이런 경우는 이너조인을 사용하는 편이 좋습니다.

- 아우터로 조인되는 테이블에 대한 조건을 WHERE절에 함께 명시하는 것
    
    ```sql
    SELECT *
    FROM employees e
    		LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no
    WHERE mgr.dept_no='d001';
    ```
    
    - 위와 같은 쿼리는 옵티마이저가 WHERE 조건을 보고 LEFT JOIN을 INNER JOIN으로 변경해서 처리합니다. —> ???
    - 따라서 LEFT JOIN의 ON절로 옮겨야 합니다.
        
        ```sql
        SELECT * 
        FROM employees e
        LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no
        		AND mgr.dept_no='d001';
        ```
        

- 예외적으로 OUTER JOIN으로 연결되는 테이블의 칼럼에 대한 조건을 WHERE절에 사용하는 경우가 있는데 이는 안티 조인 효과를 기대할 때 사용됩니다.
    
    ```sql
    SELECT *
    FROM employees e
    		LEFT JOIN dept_manager dm ON dm.emp_no=e.emp_no
    WHERE dm.emp_no IS NULL
    LIMIT 10;
    ```
    
    - dm.emp_no가 NULL인 칼럼만 찾으므로 dept_manager에는 없는 employees 칼럼만 조회합니다.
- `그 외의 경우 아우터 조인으로 연결되는 테이블의 칼럼이 WHERE MySQL 서버는 LEFT JOIN을 INNER JOIN으로 자동 변환 합니다.`

---

<br>

**11.4.7.4 JOIN과 외래키(FOREIGN KEY)**

- 데이터베이스에 외래키가 없어도 조인과는 아무런 연관이 없습니다.
- 외래키의 주 목적은 데이터의 무결성 보장입니다. (참조 무결성)
    - 부서 테이블을 사원 테이블이 참조하는데 버그로 인해 부서 테이블에 없는 부서 코드가 사원 테이블이 갖게 되는 경우 참조 무결성이 깨지므로 DBMS차원에서 막기 위해 외래키를 생성합니다.
- 따라서 조인을 사용하기 위해 외래키가 필요한 것은 아닙니다.

---

<br>

**11.4.7.5 지연된 조인(Delayed Join)**

- 조인을 사용해 데이터를 조회하는 쿼리에 GROUP BY, ORDER BY를 사용할 때 각 처리 방법에서 인덱스를 사용한다면 이미 최적으로 처리될 가능성이 높습니다.
- 그렇지 않다면 MySQL은 모든 조인 실행 이후 GROUP BY, ORDER BY를 처리할 것입니다.
    - 조인 이후엔 레코드 건수가 많아지는데 조인을 실행하기 전에 적용하는 것보다 더 많은 레코드를 처리해야 함
- **지연된 조인은 조인이 실행되기 전에 GROUP BY, ORDER BY를 처리하는 방식을 말합니다. (주로 LIMIT이 함께 사용된 쿼리에서 더 큰 효과를 얻습니다.)**

- 지연된 조인 사용 전
    
    ```sql
    # 인덱스를 사용하지 못하는 쿼리
    EXPLAIN
    SELECT e.*
    FROM salaries s, employees e
    WHERE e.emp_no=s.emp_no
    		AND s.emp_no BETWEEN 10001 AND 13000 
    GROUP BY s.emp_no
    ORDER BY SUM(s.salary) DESC
    LIMIT 10;
    
    +----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
    | id | select_type | table | partitions | type  | possible_keys     | key     | key_len | ref                | rows | filtered | Extra                                        |
    +----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
    |  1 | SIMPLE      | e     | NULL       | range | PRIMARY           | PRIMARY | 4       | NULL               | 3000 |   100.00 | Using where; Using temporary; Using filesort |
    |  1 | SIMPLE      | s     | NULL       | ref   | PRIMARY,ix_salary | PRIMARY | 4       | employees.e.emp_no |    9 |   100.00 | NULL                                         |
    +----+-------------+-------+------------+-------+-------------------+---------+---------+--------------------+------+----------+----------------------------------------------+
    ```
    
    - employees 테이블을 드라이빙 테이블로 선택, `s.emp_no BETWEEN 10001 AND 13000` 조건을 만족하는 3000건의 레코드 읽음
    - salaries 테이블을 조인하는데 수행 횟수는 대략 3000 * 4 정도라는 걸 알 수 있습니다.
    - 조인의 결과에서 그룹처리를 하고, 정렬 및 상위 10건만 최종 반환합니다.

- 지연된 조인 사용
    
    ```sql
    EXPLAIN
    SELECT e.*
    FROM
    		(SELECT s.emp_no
    		FROM salaries s
    		WHERE s.emp_no BETWEEN 10001 AND 13000
    		GROUP BY s.emp_no 
    		ORDER BY SUM(s.salary) DESC
    		LIMIT 10) x,
    employees e
    WHERE e.emp_no=x.emp_no;
    
    +----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
    | id | select_type | table      | partitions | type   | possible_keys     | key     | key_len | ref      | rows  | filtered | Extra                                        |
    +----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
    |  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL    | NULL    | NULL     |    10 |   100.00 | NULL                                         |
    |  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY | 4       | x.emp_no |     1 |   100.00 | NULL                                         |
    |  2 | DERIVED     | s          | NULL       | range  | PRIMARY,ix_salary | PRIMARY | 4       | NULL     | 56844 |   100.00 | Using where; Using temporary; Using filesort |
    +----+-------------+------------+------------+--------+-------------------+---------+---------+----------+-------+----------+----------------------------------------------+
    ```
    
    - FROM 절에 서브쿼리가 사용됐기에 파생테이블로 처리 됐습니다.
    - 서브쿼리를 위해 id=2인 값을 보면 56844건(실제는 28606건)의 레코드를 읽어 임시테이블에 저장합니다.( `<derived2>` )
    - 이후 GROUP BY를 통해 3000건으로 줄이고, ORDER BY를 통해 상위 10건의 데이터만 employees 테이블과 조인합니다.

- 어찌보면 느리다고 볼 수 있지만, 조인 이전에 임시테이블에 저장할 레코드가 10건밖에 남지 않으므로 메모리를 이용해 빠르게 처리됩니다.
- 실제로는 지연된 조인으로 개선된 쿼리가 3~4배정도 빠르게 실행된다고 합니다.
- 모든 쿼리를 지연된 조인 형태로 개선하기 어렵습니다. 다음과 같은 조건이 갖춰져야 지연된 쿼리로 변경해 사용할 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ede5e0c8-1acb-4548-8a0d-8e304af75085)

    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f5178b31-0284-488d-bc23-b90028e23ec9)

    

---

<br>

**11.4.7.6 래터럴 조인(Lateral Join)**

- 8.0 부터는 래터럴 조인을 이용해 특정 그룹별로 서브쿼리를 실행해 그 결과와 조인하는 것이 가능해졌습니다.

```sql
SELECT *
FROM employees e
		LEFT JOIN LATERAL (
						SELECT *
						FROM salaries s
						WHERE s.emp_no=e.emp_no
						ORDER BY s.from_date DESC LIMIT 2) s2
		ON s2.emp_no=e.emp_no		
WHERE e.first_name='Matt';
```

- employees 테이블에서 이름이 `Matt`인 사원에 대해 사원별로 가장 최근 급여 변경 내역 최대 2건을 반환합니다.
- 래터럴 조인의 핵심은 FROM 절에 사용된 서브쿼리(Derived Table)에서 외부 쿼리의 FROM 절에 정의된 테이블의 칼럼을 참조할 수 있다는 사실 입니다.
    - 예제에선 salaries 테이블을 읽는 서브쿼리에서 employees 테이블의 emp_no를 참조
- FROM 절에 사용된 서브쿼리가 외부 쿼리의 칼럼을 참조하려면 LATERAL 키워드가 명시돼야 합니다.
- **LATERAL 키워드를 가진 서브쿼리는 조인 순서 후순위로 밀리고 외부 쿼리의 결과 레코드 단위로 임시 테이블이 생성되기에 꼭 필요한 경우에만 사용해야 합니다.**

---

<br>

**11.4.7.7 실행 계획으로 인한 정렬 흐트러짐**

- 8.0 부터는 해시 조인 방식이 도입됐습니다.
- 해시 조인 방식은 쿼리 결과의 레코드 정렬 순서가 달라 집니다. (8.0 이전의 블록 네스티드 루프 조인도 정렬 순서 달라짐)

```sql
EXPLAIN
SELECT e.emp_no, e.first_name, e.last_name, de.from_date
FROM dept_emp de, employees e
WHERE de.from_date > '2001-10-01' AND e.emp_no < 10005;

+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                                   |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY       | PRIMARY     | 4       | NULL |    4 |   100.00 | Using where                                             |
|  1 | SIMPLE      | de    | NULL       | range | ix_fromdate   | ix_fromdate | 3       | NULL | 2763 |   100.00 | Using where; Using index; Using join buffer (hash join) |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------------------------+
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5cd61256-0956-4b13-9499-bc5b007c18b6)


- 네스티드 루프 방식으로 조인이 처리되면 드라이빙 테이블을 읽은 순서대로 조회되어야 하지만 그렇지 않습니다.
- 실행 계획은 MySQL 옵티마이저에 의해 그때그때 상황에 따라 달라질 수 있기때문에 정렬된 결과가 필요한 경우라면 드라이빙 테이블의 순서에 의존하지 않고 명시적인 ORDER BY 절을 사용하는것이 좋습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/81dc6768-c69d-4c65-88d0-66e6c0c2dd55)
