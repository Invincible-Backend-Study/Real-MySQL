# 📌 9장: 옵티마이저와 힌트

## ✅ 9.3 고급 최적화

- 옵티마이저의 실행 계획 수립은 통계정보 + 옵티마이저 옵션의 결합입니다.
- 옵티마이저 옵션
    - `조인 관련된 옵티마이저 옵션`
        - MySQL 서버 초기 버전부터 제공되던 옵션
        - 많은 사람들이 그다지 신경쓰지 않았음(조인이 많이 사용되는 서비스라면 알아두는것이 좋음)
    - `옵티마이저 스위치`
        - MySQL 5.5버전부터 지원되기 시작함
        - **MySQL 서버의 고급 최적화 기능을 활성화할지를 제어하는 용도로 사용됨**

<br><br>

### 9.3.1 옵티마이저 스위치 옵션

- optimizer_switch 시스템 변수를이용해 제어함
    - 위 시스템 변수는 여러개의 옵션을 세트로 묶어서 설정하는 방식으로 사용됨
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c9b272d9-38e5-4143-b890-3f70fef1ef7d)

        
    - 위 옵티마이저 스위치 옵션은 `default`, `on`, `off` 중에서 하나를 설정할 수 있음
    - 또한 `글로벌`, `세션별` 모두 설정할 수 있는 시스템 변수이며 `현재 쿼리`만 설정할 수 도 있음
        
        ```sql
        # 글로벌 혹은 세션 설정
        SET GLOBAL optimizer_switch='index_merge=on, index_merge_union=on,...'
        SET SESSION optimizer_switch='index_merge=on, index_merge_union=on,...'
        
        # SET_VAR 옵티마이저 힌트를 이용해 현재 쿼리에만 설정할 수 있음 -> 주석을 통해 힌트를 줌
        SELECT /** SET_VAR(optimizer_switch='condition_fanout_filter=off') */
        ...
        FROM ...
        ```
        
<br>

**MRR과 배치 키 액세스(mrr & batched_key_access)**

- `MRR`: Multi-Range Read (메뉴얼에선 Disk Sweep을 붙인 DS-MRR이라고도 함)
- 기존 네스티드 루프 조인의 경우 드라이빙 테이블에서 한 건 읽고 드리븐 테이블에서 일치하는 레코드를 찾아 조인을 수행했습니다. MySQL 서버 구조상 `조인 처리는 MySQL 엔진`이 `실제 레코드 검색 및 읽기는 스토리지 엔진`이 담당합니다.
- 결국 위와 같이 드라이빙 테이블 레코드별 드리븐 테이블 조회는 스토리지 엔진에서 어떤 최적화도 수행할 수 없게 합니다.
- MRR은 이를 보완하기 위해 드라이빙 테이블의 레코드를 읽어 즉시 조인하지 않고 Join Buffer에 버퍼링을 수행합니다. 이후 버퍼가 차면 해당 레코드들을 한번에 스토리지 엔진에 요청하는 방식을 말합니다.
    - 위 작업을 통해 스토리지 엔진은 데이터 페이지에 정렬된 순서로 접근하고, 디스크의 데이터 페이지 읽기를 최소화 합니다. (InnoDB 버퍼 풀에 데이터 페이지가 있다면 버퍼 풀 접근을 최소화 함)
- 이와 같이 MRR을 응용해서 실행되는 조인 방식을 BKA(Batched_key_access)라고 합니다.

기본적으로 BKA 조인 최적화는 비활성화 되어있는데 BKA 조인을 사용할시 부가적인 정렬 작업이 필요해지면서 오히려 성능이 저하 될 수 있는 단점이 있기 때문입니다.

<br>

**블록 네스티드 루프 조인(block_nested_loop)**

MySQL 서버의 대부분의 조인인 Nested Loop Join은 조인의 연결 조건이 되는 칼럼에 모두 인덱스가 있는 경우 사용되는 조인 방식입니다.

Nested Loop Join은 다음 쿼리에 대해 아래 의사 코드와 같이 `레코드를 다른 버퍼 공간에 저장하지 않고 즉시 드리븐 테이블의 레코드를 찾아서 반환`합니다.

```sql
EXPLAIN SELECT * FROM employees e
INNER JOIN salaries s ON s.emp_no=e.emp_no
		AND s.from_date<=NOW()
		AND s.to_date>=NOW()
WHERE e.first_name='Amor';

+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys        | key          | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no |    9 |    11.11 | Using where |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-------------+
```

```sql
for (row1 IN employees) {
		for (row2 IN salaries) {
				if (condition_matched) return (row1, row2);
		}
}
```

`Block Nested Loop Join`은 일반 nested loop join와는 차이점이 있습니다.

- 조인 버퍼(join_buffer_size 시스템 변수)를 사용함
- 드리븐 테이블을 먼저 읽고 조인 버퍼에 저장되어있는 드라이빙 테이블의 레코드와 일치하는 레코드를 찾는 방식으로 처리됩니다.(nested loop join과는 반대)

일반 Nested Loop Join에선 드라이빙 테이블 일치 레코드 건수 만큼 드리븐 테이블을 검색합니다. 드리븐 테이블 검색시 인덱스를 사용할 수 없는 쿼리라면 풀 테이블 스캔 혹은 풀 인덱스 스캔을 하게 됩니다.

어떤 방식으로도 드리븐 테이블의 풀 테이블, 인덱스 스캔을 피할 수 없으면 옵티마이저는 조인 버퍼를 사용하여 드라이빙 테이블의 레코드를 버퍼링 합니다.

```sql
# 조인 조건이 없으므로 카테시안 조인이 됨
explain 
select * from dept_emp de, employees e 
where de.from_date > '1995-01-01' AND e.emp_no<109004;

+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                                      |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL | 150292 |   100.00 | Using where                                |
|  1 | SIMPLE      | de    | NULL       | ALL   | ix_fromdate   | NULL    | NULL    | NULL | 331143 |    50.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
```

위 쿼리는 조인 버퍼를 사용하여 다음과 같이 실행됩니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/2a7e818a-f94f-44f5-bc8a-b28d6434ebc0)


위의 조회 과정을 보면 드라이빙 테이블의 결과는 우선 조인 버퍼에 담고, 드리븐 테이블을 읽은 결과를 조인 버퍼에서 일치하는 레코드를 찾게 되므로 드리븐 테이블에 의해 정렬이 됩니다.

`일반적인 조인에선 드라이빙 테이블의 순서에 의해 결정되는걸 기대하지만, 조인 버퍼가 사용된 조인에선 결과의 정렬 순서가 흐트러질 수 있습니다.`

→ 8.0.18부터 해시 조인 알고리즘이 도입됐고, 8.0.20부턴 블록 네스티드 루프 조인이 아닌 해시 조인 알고리즘이 사용됩니다. 따라서 `Using join buffer (hash join)` 다음과 같이 표시됩니다.

<br>

**인덱스 컨디션 푸시다운(index_condition_pushdown)**

```sql
mysql> ALTER TABLE employees ADD INDEX ix_lastname_firstname(last_name, first_name);
mysql> SET optimizer_switch='index_condition_pushdown=off';
```

위와 같이 인덱스 컨디션 푸시다운을 비활성화하고 다음 쿼리를 실행하면 `last_name` 조건은 `작업 범위 결정 조건`이 되지만,  `first_name조건`에서 `‘%sal’`은 필터링 조건으로만 사용됩니다.

```sql
EXPLAIN
SELECT * 
FROM employees 
WHERE last_name='Acton' AND first_name LIKE '%sal';
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys         | key                   | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |  189 |    11.11 | Using where |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-------------+
```

또한 쿼리의 실행 계획을 보면 Using where가 표시되어 있는데 이는 InnoDB 스토리지 엔진이 읽어서 반환해준 레코드가 인덱스를 사용할 수 없는 WHERE 조건에 일치하는지 검사하는 과정을 의미합니다. 

여기서는 `first_name=’%sal`’ 검사 과정에서 사용된 조건입니다. 해당 과정은 아래 그림과 같습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/320f6579-9db0-4d0e-8f0a-f00e5c01149d)


만약 위 과정에서 first_name 칼럼이 3개가 아니라 10만건이고 그 중에서 1개만 일치한다면 99_999개의 작업은 불필요해집니다.

인덱스 컨디션 푸시다운을 활용하면 인덱스에서 작업 범위 결정 조건으로 활용하지 못하는 `first_name` 칼럼이라도 인덱스에 포함되어 있다면 같이 모아서 스토리지 엔진으로 전달할 수 있게 핸들러 API가 개선됐습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/1914dff3-e9c3-4853-879c-2ced2298743e)


옵티마이저 옵션에서 인덱스 컨디션 푸시다운을 원래대로 돌려두고 실행계획을 보면 위 그림과 같은 결과가 나타납니다.

```sql
mysql> SET optimizer_switch='index_condition_pushdown=on';
EXPLAIN
SELECT *
FROM employees 
WHERE last_name='Acton' AND first_name LIKE '%sal';
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table     | partitions | type | possible_keys         | key                   | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |  189 |    11.11 | Using index condition |
+----+-------------+-----------+------------+------+-----------------------+-----------------------+---------+-------+------+----------+-----------------------+
```

→ `Index Condition Pushdown`은 고도의 기술력을 필요로 하는 기능은 아니지만, 쿼리의 성능이 몇 배에서 몇 십배로 향상될 수 있는 중요한 기능입니다.

<br>

**인덱스 확장(use_index_extensions)**

InnoDB 스토리지 엔진을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 프라이머리 키를 활용할 수 있게 할지를 결정하는 옵션입니다.

```sql
CREATE TABLE dept_emp (
		emp_no INT NOT NULL, 
		dept_no CHAR(4) NOT NULL, 
		from_date DATE NOT NULL, 
		to_date DATE NOT NULL, 
		PRIMARY KEY (dept_no, emp_no),
		KEY ix_fromdate (from_date)
) ENGINE=InnoDB;
```

위와 같은 테이블이 있을때 ix_fromdate의 인덱스는 PK를 포함하기 때문에 인덱스는 마치 `(from_date, dept_no, emp_no)`인 인덱스와 흡사하게 작동할 수 있게 됩니다.

아래 예제에서 key_len 컬럼은 어느 칼럼까지 사용했는지 바이트로 보여줍니다. 또한 사용된 key를 보면 ix_fromdate 인덱스를 활용한 것을 알 수 있습니다.

```sql
EXPLAIN
SELECT COUNT(*) 
FROM dept_emp 
WHERE from_date='1987-07-25' AND dept_no='d001';
+----+-------------+----------+------------+------+---------------------------------------+-------------+---------+-------------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys                         | key         | key_len | ref         | rows | filtered | Extra       |
+----+-------------+----------+------------+------+---------------------------------------+-------------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | PRIMARY,ix_fromdate,ix_empno_fromdate | ix_fromdate | 19      | const,const |    6 |   100.00 | Using index |
+----+-------------+----------+------------+------+---------------------------------------+-------------+---------+-------------+------+----------+-------------+

EXPLAIN
SELECT COUNT(*) 
FROM dept_emp 
WHERE from_date='1987-07-25';
+----+-------------+----------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys                 | key         | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | const |   64 |   100.00 | Using index |
+----+-------------+----------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
```

PK가 세컨더리 인덱스에 포함되어 있으므로 정렬 작업에서도 인덱스를 활용한 정렬을 할 수 있습니다.
Using Filesort가 표시되지 않았으므로 별도의 정렬작업 없이 인덱스 순서대로 읽었음을 만족합니다.

```sql
EXPLAIN 
SELECT * 
FROM dept_emp 
WHERE from_date='1987-07-25' 
ORDER BY dept_no;
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | dept_emp | NULL       | ref  | ix_fromdate   | ix_fromdate | 3       | const |   64 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
```

<br>

**인덱스 머지(index_merge)**

- 대부분의 옵티마이저는 인덱스를 이용한 쿼리에서 테이블별로 하나의 인덱스만 사용하도록 실행 계획을 수립합니다.
- 하지만, 인덱스 머지 실행 계획을 사용하면 하나의 테이블에 2개 이상의 인덱스를 이용해 쿼리를 처리합니다.
- 일반적으로는 WHERE 조건이 여러 개 있더라도 하나의 인덱스에 포함된 칼럼에 대한 조건만으로 인덱스를 검색하고 나머지 조건은 읽어온 레코드에 대해서 체크하는 형태로만 사용됩니다.
- **하지만 인덱스 머지는 쿼리의 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때 MySQL 서버는 인덱스 머지 실행 계획을 선택합니다.**

인덱스 머지 실행 계획은 다음과 같이 3개의 세부 실행 계획으로 나누어 볼 수 있습니다.
여러 개의 인덱스를 통해 결과를 가져오는것은 동일하지만, 각각 **결과를 어떤 방식으로 병합할지에 따른 차이가 있습니다.**

- `index_merge_intersection`
- `index_merge_sort_union`
- `index_merge_union`

→ index_merge 옵티마이저 옵션은 위의 3개의 최적화 옵션을 한 번에 모두 제어할 수 있는 옵션입니다.

<br>

**인덱스 머지 - 교집합(index_merge_intersection)**

다음 쿼리는 2개의 WHERE 조건을 가집니다. 또한 조건 모두 각각의 index가 있습니다.

```sql
EXPLAIN
SELECT * 
FROM employees 
WHERE first_name='Gergi' AND emp_no BETWEEN 10000 AND 20000;

+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys        | key          | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY,ix_firstname | ix_firstname | 62      | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+----------------------+--------------+---------+------+------+----------+-----------------------+
```

현재 제 컴퓨터의 결과는 위와 같지만 `index_merge_intersection` 이 적용됐다면 Extra 칼럼은 다음과 같습니다. `Using intersect(ix_firstname, PRIMARY); Using where`

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/56f1af4c-7cac-46bd-95f2-6adbe55d3f86)


위의 2개의 쿼리의 결과는 다음과 같습니다.

- `first_name=’Georgi’` 일치하는 253건 레코드를 찾고 emp_no 칼럼의 조건에 일치하는 레코드들만 반환하는 형태로 처리돼야 합니다.
- `emp_no BETWEEN 10000 AND 20000` PK를 이용해 10,000건을 읽고 위 조건에 일치하는 레코드만 반환함

다만 3번째 결과를 보면 두 조건 모두 만족하는 레코드는 14건입니다. ix_firstname인덱스를 사용했다면 253건중 239건은 필요가 없고, 클러스터링 인덱스를 사용했다면 9,986건은 의미가 없어집니다.
이런 상황에서 옵티마이저는 좀 더 효율적인 각 인덱스를 검색하고 두 결과의 교집합을 찾아 반환하게 됩니다.

ix_firstname 인덱스가 PK를 포함하고 있기 때문에 좀 더 성능이 좋을것으로 예상된다면 서버 전체, 커넥션, 현재 쿼리 레벨에서 비활성화 할 수 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/93b2f823-6baf-4de9-962c-769ac019db8f)


<br>

**인덱스 머지 - 합집합(index_merge_union)**

인덱스 머지의 `Using union`은 `WHERE`절에 사용된 2개 이상의 조건이 각각의 인덱스를 사용하되 `OR 연산자`로 연결된 경우에 사용되는 최적화 입니다.

```sql
EXPLAIN
SELECT *
FROM employees
WHERE first_name='Matt' OR hire_date='1987-03-31';
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | ix_hiredate,ix_firstname | ix_firstname,ix_hiredate | 58,3    | NULL |  344 |   100.00 | Using union(ix_firstname,ix_hiredate); Using where |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+----------------------------------------------------+
```

위 쿼리의 2개의 조건인 `first_name`과 `hire_date`에는 각각 인덱스가 걸려있습니다.

쿼리의 실행계획을 보면 `Using union` 최적화를 사용합니다.

이는 두 인덱스의 검색결과가 Union 알고리즘으로 병합됐다는 것을 의미합니다. (병합 = `두 집합의 합집합`)

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/8ce0c650-4451-4ea8-8dd4-3f1a7f22214d)


MySQL 서버는 first_name 조건을 검색한 결과와 hire_date 칼럼을 검색한 결과가 `PK로 각각 정렬돼 있다는것을 알 수 있습니다.`

따라서 예제 쿼리는 first_name, hire_date 두 조건의 쿼리로 분리해본다면 다음과 같고 모두 PK키로 정렬돼 있습니다.

```sql
SELECT * FROM employees WHERE first_name='Matt';
SELECT * FROM employees WHERE hire_date='1987-03-31';
```

MySQL 서버는 두 집합에서 하나씩 가져와 서로 비교하며 PK값이 중복된 레코드들을 정렬 없이 걸러낼 수 있습니다.

이런 중복 제거 알고리즘에 사용되는 것 우선순위 큐(`Priority Queue`) 입니다.

<br>

**인덱스 머지 - 정렬 후 합집합(index_merge_sort_union)**

`index_merge_union`은 이미 PK를 통해 정렬되어 있기 때문에 별도의 정렬을 수행하지 않습니다. 하지만 인덱스 머지 작업을 하는 도중에 결과의 정렬이 필요한 경우 MySQL 서버는 인덱스 머지 최적화의 `‘Sort union’` 알고리즘을 사용합니다.

```sql
EXPLAIN SELECT * FROM employees
WHERE first_name='Matt'
OR hire_date BETWEEN '1987-03-01' AND '1987-03-31';

+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+---------------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                                                   |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+---------------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | ix_hiredate,ix_firstname | ix_firstname,ix_hiredate | 58,3    | NULL | 3197 |   100.00 | Using sort_union(ix_firstname,ix_hiredate); Using where |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+---------------------------------------------------------+
# 위 쿼리는 다음과 같이 분해될 수 있습니다.
SELECT * FROM employees WHERE first_name='Matt';
SELECT * FROM employees WHERE hire_date BETWEEN '1987-03-01' AND '1987-03-31';
```

나눠진 첫 번째 쿼리의 결과는 PK로 정렬되지만, 두 번째 결과는 그렇지 않습니다. 따라서 중복 제거를 위한 우선순위 큐를 사용하는 것이 불가능합니다.

MySQL 서버는 두 집합 결과의 중복 제거를 위해 각 집합을 emp_no 칼럼으로 정렬하고 중복제거를 수행합니다.
중복 제거를 위해 강제로 정렬을 수행해야 하는 경우 실행 계획의 `Extra` 칼럼은 `Using sort_union`으로 보여집니다.

<br>

**세미 조인(semijoin)**

다른 테이블과 실제 조인을 수행하지는 않고, 다른 테이블에서 조건에 일치하는 레코드가 있는지 없는지만 체크하는 형태의 쿼리를 말합니다.

```sql
SELECT * 
FROM employees e 
WHERE e.emp_no IN 
		(SELECT de.emp_no 
			FROM dept_emp de 
			WHERE de.from_date='1995-01-01');
```

위 쿼리는 `dept_emp`에서 특정 `from_date`에 해당되는 `emp_no`번호를 가져오고, `employees` 테이블에서 해당 `emp_no`번호에 해당되는 레코드 결과만 가져오는 세미 조인 쿼리입니다.

세미 조인 형태의 쿼리와 안티 세미 조인 형태의 쿼리는 최적화 방법이 조금 차이가 있습니다.

- 8.0 이전의 최적화
    - 세미 조인 형태 ( `=`, `IN`(subquery))
        - 세미 조인 최적화
        - In-to-EXISTS 최적화
        - MATERIALIZATION 최적화
    - 안티 세미 조인 쿼리 (`<>`, `NOT IN` (subquery))
        - In-to-EXISTS 최적화
        - MATERIALIZATION 최적화

8.0 버전부터는 세미 조인 쿼리의 성능을 개선하기 위해 다음과 같은 최적화 전략이 있습니다. MySQL 서버 매뉴얼에서는 아래 최적화 전략들을 모아 `세미 조인 최적화` 라고 부릅니다.

- **Table Pull-out**
- **Duplicate Weed-out**
- **First Match**
- **Loose Scan**
- **Materialization**

쿼리에 사용되는 `테이블`, `조인 조건 특성`에 따라 MySQL 옵티마이저는 사용 가능한 전략들을 선별적으로 사용합니다.

- `Table pull-out` 최적화 전략은 항상 세미 조인 보다 좋은 성능을 내므로 별도로 제어하는 옵티마이저 옵션은 없습니다.
- `First Match`, `Loose Scan` 최적화 전략: `firstmatch`, `loosescan` 옵티마이저 옵션으로 사용여부 결정
- `Duplicate Weed-out`, `Materialization` 최적화 전략: `materialization` 옵티마이저 스위치로 사용여부 결정

→ optimizer_switch 시스템 변수의 `semijoin` 옵티마이저 옵션은 `firstmatch`, `loosescan`, `materialization` 옵티마이저 옵션을 한 번에 활성화하거나 비활성화 할 때 사용합니다.

<br>

**세미조인 - 테이블 풀-아웃(Table Pull-out)**

- Table pullout 최적화는 세미 조인의 서브쿼리에 사용된 테이블을 아우터 쿼리로 끄집어낸 후에 쿼리를 조인 쿼리로 재작성하는 형태의 최적화입니다.
- 서브쿼리 최적화가 도입되기 이전에 수동을 쿼리 튜닝하던 대표적인 방법이지만 현재는 옵티마이저가 지원해줍니다.

```sql
EXPLAIN 
SELECT * 
FROM employees e 
WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.dept_no='d009');
+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys             | key     | key_len | ref                 | rows  | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------+
|  1 | SIMPLE      | de    | NULL       | ref    | PRIMARY,ix_empno_fromdate | PRIMARY | 16      | const               | 46012 |   100.00 | Using index |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                   | PRIMARY | 4       | employees.de.emp_no |     1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------+
```

- 위와 같은 서브쿼리를 포함하는 쿼리의 실행 계획의 `id칼럼 값이 전부 1이라는 것은 서브쿼리가 아닌 조인으로 처리됐음`을 말합니다. https://0soo.tistory.com/235#id
- Table pullout 적용 여부는 별도의 문구가 없기에 위와 같이 테이블의 칼럼값이 동일하여 조인으로 처리됐는지 여부로 판단할 수 있습니다.
- 또한 `EXPLAIN`이후 `SHOW WARNINGS` 명령으로 MySQL 옵티마이저가 재작성한 쿼리를 살펴볼 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d735321f-2d1a-4527-9f3f-038f837b5724)

    
    - 결과를 보면 IN 대신 JOIN으로 쿼리가 재작성됨을 확인할 수 있습니다.

Table pullout 최적화의 제한 사항 및 특성은 다음과 같습니다.

- `제한사항`
    - Table pullout 최적화는 `세미 조인 서브쿼리에서만 사용 가능`하다.
    - Table pullout 최적화는 `서브쿼리 부분이 UNIQUE 인덱스나 프라이머리 키 룩업으로 결과가 1건`인 경우에만 사용 가능하다.
- `특성`
    - Table pullout이 적용된다고 하더라도 기존 쿼리에서 가능했던 최적화 방법이 사용 불가능한 것은 아니므로 MySQL에서는 가능하다면 `Table pullout 최적화를 최대한 적용`한다.
    - Table pullout 최적화는 서브쿼리의 테이블을 아우터 쿼리로 가져와서 조인으로 풀어쓰는 최적화를 수행하는데, 만약 서브쿼리의 모든 테이블이 아우터 쿼리로 끄집어 낼 수 있다면 서브쿼리 자체는 없어진다.
    - MySQL에서는 "최대한 서브쿼리를 조인으로 풀어서 사용해라"라는 튜닝 가이드가 많은데, Table pullout 최적화는 사실 이 가이드를 그대로 실행하는 것이다. 이제부터는 서브쿼리를 조인으로 풀어서 사용할 필요가 없다.

<br>

**세미조인 - 퍼스트 매치(firstmatch)**

IN(subquery) 형태의 세미 조인을 EXISTS(subquery) 형태로 튜닝한 것과 비슷한 방법으로 실행됩니다.

```sql
# 비상관 쿼리
EXPLAIN 
SELECT * 
FROM employees e 
WHERE e.first_name='Matt' AND e.emp_no IN 
		(SELECT t.emp_no 
			FROM titles t 
			WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30');
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
| id | select_type | table | partitions | type | possible_keys        | key          | key_len | ref                | rows | filtered | Extra                                   |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              |  233 |   100.00 | NULL                                    |
|  1 | SIMPLE      | t     | NULL       | ref  | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no |    1 |    11.11 | Using where; Using index; FirstMatch(e) |
+----+-------------+-------+------------+------+----------------------+--------------+---------+--------------------+------+----------+-----------------------------------------+
```

- ID값이 모두 1이므로 서브쿼리가 아닌 JOIN이 실행됐다는것을 알 수 있습니다.
- Extra 칼럼의 값이 FirstMatch(e)로 employees 테이블 레코드의 각 emp_no에 대해 titles 테이블에 일치하는 레코드 1건만 찾으면 더이상 titles 테이블을 검색하지 않고 다음 emp_no를 검색하도록 합니다.
- 의미론적으로는 EXISTS(subquery)와 동일하게 처리된 것입니다. 하지만, 조인으로 풀어서 실행합니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a45dd01c-3cb7-46c0-adca-3f2dcd36a92a)


위 그림처럼 일치하는 레코드 하나를 찾는 즉시 레코드를 반환합니다.

- fisrstmatch의 MySQL 5.5의 In-to-EXISTS대비 장점
    - 여러 테이블이 조인되는 경우 원래 쿼리에는 없는 동등 조건을 옵티마이저가 자동으로 추가하는 형태의 최적화가 실행되기도 함(동등 조건 전파), 이전에는 서브쿼리 내에서만 가능했지만 `Firstmatch는 조인 형태로 처리되므로 서브쿼리 및 아우터 쿼리 테이블까지 전파됩니다.` 결국 더 많은 조건이 주어지므로 더 나은 실행 계획을 수립할 수 있습니다.
    - In-to-EXISTS는 아무런 조건 없이 변환이 가능한 경우 무조건 해당 최적화를 수행하는 반면 Firstmatch 최적화는 서브쿼리의 모든 테이블에 대해 Firstmatch를 수행할지 아니면 일부 테이블에만 수행할지를 선택할 수 있는 장점이 있습니다.

- FirstMatch 최적화의 제한사항 및 특징
    - 서브쿼리에서 일치하는 레코드를 검색하면 즉시 반환하고 검색을 멈추는 단축 실행 결로(Short-cout path)이므로 서브쿼리가 참조하는 모든 아우터 테이블이 먼저 조회된 후 실행됩니다.
    - 실행 계획의 Extra 칼럼에는 `FirstMatch(table name)`이 표시됩니다.
    - 비상관 쿼리 뿐만 아니라 상관 서브쿼리에서도 사용할 수 있습니다.
        
        [https://velog.io/@jonghyun3668/서브쿼리](https://velog.io/@jonghyun3668/%EC%84%9C%EB%B8%8C%EC%BF%BC%EB%A6%AC)
        
    - GROUP BY 혹은 집합 함수가 사용된 서브쿼리의 최적화에는 사용될 수 없습니다.
    
<br>

**세미 조인 - 루스 스캔(loosescan)**

세미 조인 서브쿼리 최적화의 LooseScan은 인덱스를 사용하는 GROUP BY 최적화 방법에서 살펴본 `Using index for group-by`의 루스 인덱스 스캔과 비슷한 읽기 방식을 사용합니다.

```sql
EXPLAIN 
SELECT * 
FROM departments d 
WHERE d.dept_no IN (SELECT de.dept_no FROM dept_emp de);
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                                      |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
|  1 | SIMPLE      | de    | NULL       | index | PRIMARY       | PRIMARY | 20      | NULL | 331143 |     0.00 | Using index; LooseScan                     |
|  1 | SIMPLE      | d     | NULL       | ALL   | PRIMARY       | NULL    | NULL    | NULL |      9 |    11.11 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------------------------+
```

위 쿼리에서 dept_emp 테이블은 33만건의 데이터가 있지만, (dept_no + emp_no)칼럼 조합으로 클러스터링 인덱스가 있습니다.

따라서 다음과 같이 그루핑되어 효율적으로 중복 레코드 없이 서브쿼리를 실행함을 알 수 있습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5a2774a9-9053-4ac2-8ccc-97df8b94bb2b)


위 작업이 이뤄지기 위해선 dept_emp 테이블이 드라이빙 테이블이 되어야 합니다. 또한 루스 인덱스 스캔이 적용된 만큼 dept_emp.dept_no를 유니크하게 한 건 씩만 읽습니다. 따라서 실행계획에서도 `LooseScan` 을 확인할 수 있습니다. 

- 실행계획의 ID가 모두 1이므로 조인으로 처리됩니다.
    - 또한 departments의 Extra에 `Using join buffer (hash join)`이 있는데 이는 조인 버퍼를 사용하여 드라이빙 테이블인 `dept_emp` 테이블의 결과는 우선 조인 버퍼에 담고, 드리븐 테이블인 `departments` 테이블을 읽은 결과를 조인 버퍼와 매핑하여 일치하는 레코드를 찾음을 알 수 있습니다.

- LooseScan 최적화의 특징
    - 루스 인덱스 스캔으로 서브쿼리 테이블을 읽고, 다음으로 아우터 테이블(조인을 할때 먼저 읽는 테이블)을 드리븐으로 사용해서 조인을 수행합니다.
    - 따라서 서브쿼리 부분이 루스 인덱스 스캔을 사용할 수 있는 조건이 갖춰져야 사용할 수 있는 최적화 입니다.

<br>

**세미 조인 - 구체화(Materialization)**

Materialization 최적화는 세미 조인에 사용된 서브쿼리를 통째로 구체화해서 쿼리를 최적화한다는 의미입니다.

즉, 내부 임시 테이블을 생성한다는 것을 의미합니다.

```sql
EXPLAIN SELECT *
FROM employees e WHERE e.emp_no IN
		(SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys                 | key         | key_len | ref                | rows | filtered | Extra       |
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL                          | NULL        | NULL    | NULL               | NULL |   100.00 | NULL        |
|  1 | SIMPLE       | e           | NULL       | eq_ref | PRIMARY                       | PRIMARY     | 4       | <subquery2>.emp_no |    1 |   100.00 | NULL        |
|  2 | MATERIALIZED | de          | NULL       | ref    | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | const              |   57 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+-------------------------------+-------------+---------+--------------------+------+----------+-------------+

EXPLAIN SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01';
+----+-------------+-------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys                 | key         | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | de    | NULL       | ref  | ix_fromdate,ix_empno_fromdate | ix_fromdate | 3       | const |   57 |   100.00 | Using index |
+----+-------------+-------+------------+------+-------------------------------+-------------+---------+-------+------+----------+-------------+
```

위 서브쿼리를 분리하여 실행계획을 보면 `ix_fromdate` 인덱스를 사용하여 결과를 가져옵니다. 즉, PK기준으로 정렬되어 있지 않음을 의미합니다. **이를 바로 employees 테이블에서 결과를 가져오려면 풀 테이블 스캔이 발생합니다.**

`Materialization`에선 별도의 임시테이블이 생성하여 클러스터링 키(PK 기준 정렬되어 있음)를 활용해 조인을 하면 인덱스를 사용하여 결과를 가져올 수 있도록 최적화 할 수 있습니다.

이를 서브 쿼리의 구체화 라고 합니다.

Materialization 최적화는 다음과 같은 GROUP BY 절이 있는 서브쿼리에서도 사용할 수 있습니다.

```sql
EXPLAIN
SELECT e.emp_no
FROM employees e 
WHERE e.emp_no 
IN (SELECT de.emp_no # 버그성이 있음
    FROM dept_emp de 
    WHERE de.from_date='1995-01-01' 
    GROUP BY de.dept_no);
```

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a0c168c1-0cc3-43d1-a407-ebdae7f2c3e5)


![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c721c0ad-c6d0-4576-8104-1d3d83ddfb77)


- Materialization 최적화가 사용될 수 있는 형태의 쿼리 제한 및 특성
    - IN(subquery)에서 서브쿼리는 상관 서브쿼리가 아니어야 한다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/22763358-a3e6-481b-99fe-6c1c97113852)

        
    - 서브쿼리는 GROUP BY나 집합 함수들이 사용돼도 Materialization을 사용할 수 있다.
    - Materialization이 사용된 경우에는 내부 임시 테이블이 사용됩니다.
    - 세미 조인이 아닌 서브쿼리의 최적화에도 적용할 수 있습니다.
- 옵션 끄고 키기
    - optimizer_switch 시스템 변수에서 semijoin, materialization 옵션 모두 ON일 경우에만 사용가능
    - semijoin은 내버려두고 materialization옵션만 OFF로 두면 사용하지 않을 수 있음

<br>

**중복 제거(Duplicated Weed-out)**

Duplicate Weedout은 `세미 조인 서브쿼리를 일반적인 INNER JOIN 쿼리로 바꿔 실행하고 마지막에 중복된 레코드를 제거하는 방법`으로 처리되는 최적화 알고리즘 입니다.

```sql
# 사전 작업
mysql> SET optimizer_switch='materialization=OFF';
mysql> SET optimizer_switch='firstmatch=OFF';
mysql> SET optimizer_switch='loosescan=OFF';
mysql> SET optimizer_switch='duplicateweedout=ON';

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
2 rows in set, 1 warning (0.00 sec)
```

위 쿼리는 다음과 같이 INNER JOIN + GROUP BY 절로 바꿔서 실행하는 것과 동일한 작업으로 쿼리를 처리합니다.

```sql
SELECT e.*
FROM employees e, salaries s
WHERE e.emp_no=s,emp_no AND s.salary > 150000
GROUP BY e.emp_no;
```

일반적으로 쿼리의 실행순서는 다음과 같습니다. 아래 순서를 염두에 두며 그림의 실제 과정을 이해할 수 있습니다.

`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/35250c86-ee01-4f1a-8a5b-a702c8ec5f7c)


- Duplicate Weedout은 실행계획의 Extra 칼럼에 `Start temporary`, `End temporary`라는 문구가 별도로  표시됩니다.
- 또한 실행 결과의 ID값이 동일한데 이는 위 그림에서 조인과 임시테이블로 저장하는 작업이 반복적으로 실행되는 과정이라 하나의 작업으로 볼 수 있습니다.
- 또한 반복 과정이 시작되는 테이블의 실행 계획에는 `Start temporary`가 표시되고, 반복 과정이 끝나는 테이블의 실행 계획 라인에는 `End temporary`가 표시됩니다.
- 따라서 두 문구의 구간이 Duplicate Weedout 최적화의 처리 과정입니다.

- Duplicate Weedout 최적화의 장점 및 제약사항
    - 서브쿼리가 상관 서브쿼리여도 사용할 수 있는 최적화 입니다.
    - 서브쿼리가 GROUP BY나 집합 함수가 사용된 경우에는 사용될 수 없다.
        1. **GROUP BY 사용 시 중요도**: GROUP BY는 특정 열에 대해 그룹화하고 집계 함수를 적용하는데, 이 때 중복된 값을 제거하는 것이 목적입니다. 서브쿼리에서 GROUP BY를 사용할 경우, 그룹화된 결과물이 필요한데, Duplicate Weedout은 중복을 제거하려는데 이미 GROUP BY에 의해 중복이 제거된 상태이므로 사용할 필요가 없습니다.
        2. **집합 함수 사용 시 정확성 문제**: GROUP BY가 없이 집합 함수를 사용할 경우, 결과는 단일 값이 되어야 합니다. Duplicate Weedout은 중복을 제거하기 위한데, 이미 집합 함수를 사용한 경우 중복은 없는 상태입니다. 따라서 중복을 제거하는 최적화가 필요하지 않습니다.
    
<br>

**컨디션 팬아웃(condition_fanout_filter)**

조인시 테이블의 순서는 성능에 큰 영향을 줍니다.

A, B테이블의 레코드가 각각 1만건, 10건일때 A가 드라이빙 테이블 이되면 B테이블의 인덱스를 구성하는 루트 노드를 1만번 읽어야 합니다.

**따라서 MySQL 옵티마이저는 여러 테이블이 조인되는 경우 가능하다면 일치하는 레코드 건수가 적은 순서대로 조인을 실행합니다.**

```sql
# 컨디션 팬아웃 옵션 끄기
SET optimizer_switch='condition_fanout_filter=off';

EXPLAIN 
SELECT * 
FROM employees e 
INNER JOIN salaries s ON s.emp_no=e.emp_no 
WHERE e.first_name='Matt' 
		AND e.hire_date BETWEEN '1985-11-21' AND '1986-11-21';
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys                    | key          | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_hiredate,ix_firstname | ix_firstname | 58      | const              |  233 |   100.00 | Using where |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY                          | PRIMARY      | 4       | employees.e.emp_no |    9 |   100.00 | NULL        |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+

```

위 실행계획의 쿼리는 다음과 같은 절차로 처리 됩니다.

1. employees 테이블서 ix_firstname 인덱스를 이용해 first_name=’Matt’ 조건에 일치하는 233건의 레코드를 검색함
2. `233건의 레코드` 중에서 hire_date가 입력한 기간에 일치하는 레코드만 걸러냄, 이때 실행계획의 filtered `100.00의 의미는 옵티마이저가 233건 모두 hire_date 칼럼의 조건을 만족할 것으로 예측했다는 것을 의미`합니다.
3. employees 테이블을 읽은 결과 233건에 대해 salaries 테이블의 PK를 이용해 레코드를 읽습니다. 이때 MySQL 옵티마이저는 1건의 employees 레코드당 9건의 salaries 테이블 레코드가 일치할 것으로 예상 했습니다.

다시 컨디션 팬아웃을 활성화하고 실행계획을 비교하면 다음과 같습니다.

```sql
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys                    | key          | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_hiredate,ix_firstname | ix_firstname | 58      | const              |  233 |    26.14 | Using where |
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY                          | PRIMARY      | 4       | employees.e.emp_no |    9 |   100.00 | NULL        |
+----+-------------+-------+------------+------+----------------------------------+--------------+---------+--------------------+------+----------+-------------+
```

first_name이 ‘Matt’와 일치하는 레코드의 수는 동일하지만 filtered 칼럼의 값이 100%가 아니라 26.14%로 변경됐습니다.

**컨디션 팬아웃 최적화가 활성화 되면 MySQL 옵티마이저는 인덱스를 사용할 수 있는 first_name 칼럼 조건 이외의 나머지 조건(hire_date칼럼 조건)에 대해서도 얼마나 조건을 충족할지 고려했다는 의미가 됩니다.**

결국 233건에서 실질적으론 `61건 정도가 조건을 모두 충족할 것이라는 예측` 입니다.

- condition_fanout_filter가 활성화 될때 조건을 만족하는 레코드의 비율을 구하기 위한 조건
    - WHERE 조건절에 사용된 칼럼에 대해 인덱스가 있는 경우
    - WHERE 조건절에 사용된 칼럼에 대해 히스토그램이 존재하는 경우

`first_name=’Matt’`에선 `ix_firstname` 인덱스만 사용합니다. 다만, 실행 게획을 수립할때는 `233`건의 레코드건수가 있다는걸 대략적으로 알아내고, `hire_date` 칼럼의 조건을 만족하는 레코드 비율이 `26.14%`일 것으로 예측합니다.

만약 ix_hiredate 인덱스가 없었다면 MySQL 옵티마이저는 first_name 칼럼의 인덱스를 이용해 hire_date 칼럼값의 분포도를 살펴보고 filtered 칼럼 값을 예측 합니다.

- 옵티마이저가 실행계획을 수립할때 순서대로 선택하는 방식
    1. 레인지 옵티마이저를 이용한 예측
        - 실제 인덱스의 데이터를 살펴보고 레코드 건수를 예측하는 방식 (실행 계획 수립 단계에서 빠르게 소량의 데이터 읽음) → 인덱스를 이용해 쿼리가 실행될 수 있을때만 사용됨
        - 다른것들보다 우선순위가 높기에 실행 계획에 표시되는 레코드 건수가 테이블, 인덱스 통계, 히스토그램 정보와 다른값이 표시될 수 있음
    2. 히스토그램을 이용한 예측
    3. 인덱스 통계를 이용한 예측
    4. 추측에 기반한 예측
    

condition_fanout_filter 최적화 기능은 추가적으로 발생하는 오버헤드가 있습니다.(시간, 컴퓨팅 자원) 만약 8.0 이전 버전에서 쿼리 실행 계획이 잘못된 선택을 한 적이 별로 없다면 성능 향상에 크게 도움되지 않을 수 있습니다.

<br>

**파생 테이블 머지(derived_merge)**

과거 MySQL은 FROM 절에 사용된 서브쿼리는 먼저 실행해 결과를 임시 테이블로 만들어 외부 쿼리 부분을 처리합니다.

```sql
SET optimizer_switch='derived_merge=off';

EXPLAIN 
SELECT * 
FROM (SELECT * FROM employees WHERE first_name='Matt') derived_table 
WHERE derived_table.hire_date='1986-04-03';
+----+-------------+------------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
| id | select_type | table      | partitions | type        | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                                                  |
+----+-------------+------------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL         | NULL                     | NULL                     | NULL    | NULL |    2 |   100.00 | NULL                                                   |
|  2 | DERIVED     | employees  | NULL       | index_merge | ix_hiredate,ix_firstname | ix_hiredate,ix_firstname | 3,58    | NULL |    1 |   100.00 | Using intersect(ix_hiredate,ix_firstname); Using where |
+----+-------------+------------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
```

- derived_merge를 사용하지 않으면 FROM 절의 서브쿼리의 결과로 임시테이블을 생성하고 다시 바깥 WHERE를 적용합니다. (FROM 절에 사용된 서브쿼리를 파생 테이블이라고 함)
- 위 방식은 몇가지 문제점이 있습니다.
    - 서브쿼리의 결과를 읽고 → 다시 임시테이블에 insert하는 오버헤드 발생
    - 초기에는 메모리에 생성하지만 크기가 커지면 디스크에 생성 → Disk I/O발생

하지만 derived_merge 최적화 옵션을 적용하면 다음과 같은 실행계획 및 옵티마이저가 쿼리를 최적화 시켜줍니다.

```sql
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
| id | select_type | table     | partitions | type        | possible_keys            | key                      | key_len | ref  | rows | filtered | Extra                                                  |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
|  1 | SIMPLE      | employees | NULL       | index_merge | ix_hiredate,ix_firstname | ix_hiredate,ix_firstname | 3,58    | NULL |    1 |   100.00 | Using intersect(ix_hiredate,ix_firstname); Using where |
+----+-------------+-----------+------------+-------------+--------------------------+--------------------------+---------+------+------+----------+--------------------------------------------------------+
```

```sql
select `employees`.`employees`.`emp_no` AS `emp_no`,
`employees`.`employees`.`birth_date` AS `birth_date`,
`employees`.`employees`.`first_name` AS `first_name`,
`employees`.`employees`.`last_name` AS `last_name`,
`employees`.`employees`.`gender` AS `gender`,
`employees`.`employees`.`hire_date` AS `hire_date` 
from `employees`.`employees` 
where ((`employees`.`employees`.`hire_date` = DATE'1986-04-03') 
			and (`employees`.`employees`.`first_name` = 'Matt'))
```

→ 서브쿼리를 삭제하고 외부 쿼리 WHERE절에 병합

다만 다음 조건에선 옵티마이저가 자동으로 서브쿼리를 외부 쿼리로 병합할 수 없으니 수동으로 병합하여 작성하는 것을 권장합니다.

- 왜 안될까? 찾아보면 좋을듯 → `HAVETOFIND`
- SUM() 또는 MIN(), MAX() 같은 집계 함수와 윈도우 함수(Window Function)가 사용된 서브쿼리
- DISTINCT가 사용된 서브쿼리
- GROUP BY HAVING이 사용된 서브쿼리
- LIMIT이 사용된 서브쿼리
- UNION 또는 UNION ALL을 포함하는 서브쿼리
- SELECT 절에 사용된 서브쿼리
- 값이 변경되는 사용자 변수가 사용된 서브쿼리

<br>

**인비저블 인덱스(use_invisible_indexes)**

8.0부터 추가된 인덱스의 가용 상태를 제어할 수 있는 기능입니다.

8.0이전에는 인덱스가 존재하면 옵티마이저가 실행 계획 수립시 인덱스를 검토하고 사용했습니다.

8.0부터는 인덱스의 삭제 없이 해당 인덱스의 사용가능 여부를 제어할 수 있는 기능을 제공합니다.

```sql
# 옵티마이저가 ix_hiredate 인덱스를 사용하지 못하게 변경
ALTER TABLE employees ALTER INDEX ix_hiredate INVISIBLE;

# 옵티마이저가 ix_hiredate 인덱스를 사용할 수 있게 변경
ALTER TABLE employees ALTER INDEX ix_hiredate VISIBLE;
```

다만 use_invisible_indexes 옵션을 on으로 두면 INVISIBLE로 설정된 인덱스라 하더라도 옵티마이저가 사용하게 제어할 수 있습니다. (기본값은 off)

<br>

**스킵 스캔(skip_scan)**

인덱스는 칼럼의 순서에 따라 정렬되어 있습니다. 따라서 (A, B, C) 칼럼 인덱스가 있다면

WHERE절에 (A, B), (A), (A, B, C)에 대한 조건 모두 (A, B, C)인덱스 해당 칼럼까지 인덱스를 탈 수 있습니다.

**다만 (B, C)와 같은 조건의 경우 인덱스를 활용할 수 없는데 스킵 스캔은 제한적이지만 이런 제약 사항을 넘을 수 있는 최적화 기법입니다.**

```sql
# employees 테이블에 다음과 같은 인덱스가 있다고 가정해보자.
ALTER TABLE employees ADD INDEX ix_gender birthdate (gender, birth_date);

# ix_gender_birthdate 인덱스를 사용하지 못하는 쿼리 -> gender 조건 누락
SELECT * FROM employees WHERE birth_date>='1965-02-01';

# ix gender_birthdate 인덱스를 사용할 수 있는 쿼리
SELECT * FROM employees WHERE gender='M' AND birth_date>='1965-02-01';
```

- 8.0 부터는 선행 칼럼 없이 후행 칼럼의 조건만으로도 인덱스를 이용한 쿼리 성능 개선이 가능합니다.
- gender 칼럼의 값을 가져와 세번째 쿼리와 같이 gender 칼럼의 조건이 있는 것처럼 쿼리를 최적화 합니다.
    
    → 다만 선행 칼럼의 선택도(유니크한 칼럼 수)가 높다면 인덱스 스킵 스캔이 비효율적일 수 있습니다.
    
    → 따라서 8.0 옵티마이저는 선택도가 낮은 값을 가질때만 최적화를 사용 합니다.
    

다음과 같이 활성화 여부 및 힌트를 줄 수 있습니다.

```sql
# 현재 세션에서 인덱스 스킵 스캔 최적화를 활성화
SET optimizer_switch='skip_scan=on';

# 현재 세션에서 인덱스 스킵 스캔 최적화를 비활성화
SET optimizer_switch='skip_scan=off';

# 특정 테이블에 대해 인덱스 스킵 스캔을 사용하도록 힌트를 사용
SELECT /*+ SKIP_SCAN (employees)*/ COUNT (*)
FROM employees
WHERE birth_date>= 1965-02-01 ;

# 특정 테이블과 인덱스에 대해 인덱스 스킵 스캔을 사용하도록 힌트를 사용
SELECT /*+ SKIP_SCAN (employees ix_gender_birthdate)*/ COUNT (*)
FROM employees
WHERE birth_date>=' 1965-02-01';

# 특정 테이블에 대해 인덱스 스킵 스캔을 사용하지 않도록 힌트를 사용
SELECT /*+ NO_SKIP_SCAN (employees)*/ COUNT (*)
FROM employees
WHERE birth_date>='1965-02-01';
```

<br>

**해시 조인(hash_join)**

MySQL 8.0.18 버전부터 추가로 지원되기 시작한 최적화 입니다.

해시 조인이 네스티드 루프 조인보다 무조건적으로 빠르진 않습니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/733251c6-4656-42c6-baed-24249f3be99e)


두 조인은 모두 똑같은 시점에 시작됐지만 첫 레코드를 찾아낸 시점은 중첩 루프 조인이 빠릅니다.

반면에 마지막 레코드를 찾는 시점은 해시 조인이 더 빠른걸 알 수 있습니다.

**따라서 해시조인은 Best Throughput 전략에 중첩 루프 조인은 온라인 트랜잭션(OLTP)와 같은 응답 속도가 더 중요한 전략에 어울립니다.**

MySQL은 범용 RDBMS이고 온라인 트랜잭션 처리를 위한 데이터베이스 서버입니다. 따라서 주로 조인 조건의 칼럼이 인덱스가 없거나 조인 대상 테이블 중 일부의 레코드 건수가 매우 적은 경우 등에 대해서만 해시 조인 알고리즘을 사용하도록 설계돼있습니다.

**→ 해시 조인 최적화는 중첩 루프 조인이 사용하기 적합하지 않은 부분의 차선책입니다. 블록 중첩 루프 조인 또한 해시 조인 이전의 차선책이었지만 현재는 거의 사용되지 않습니다.**

```sql
EXPLAIN 
SELECT * 
FROM employees e IGNORE INDEX (PRIMARY, ix_hiredate) 
INNER JOIN dept_emp de IGNORE INDEX(ix_empno_fromdate, ix_fromdate) 
		ON de.emp_no=e.emp_no AND de.from_date=e. hire_date;
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
|  1 | SIMPLE      | de    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL                                       |
|  1 | SIMPLE      | e     | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299587 |     0.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
```

실행계획을 보면 IGNORE INDEX로 쿼리 조건에 적용될 수 있는 인덱스를 사용하지 못하도록 하면 **중첩 루프 조인을 사용할 수 없으므로 차선책인 해시 조인을 사용함**을 알 수 있습니다. 

해시조인은 빌드 단계(Build-phase), 프로브 단계(Probe-phase)로 나뉘어 처리됩니다.

- `빌드 단계`
    - 조인 대상 테이블에서 레코드 건수가 적어 해시 테이블로 만들기 용이한 테이블을 골라 해시 테이블을 메모리에 생성(빌드)하는 작업 수행
    - 이때 사용되는 원본 테이블을 빌드 테이블이라 함
- `프로브 단계`
    - 빌드 테이블을 제외한 나머지 테이블의 레코드를 읽어 해시 테이블의 일치 레코드를 찾는 과정을 의미합니다.
    - 이때 읽히는 나머지 테이블을 프로프 테이블이라 함

빌드, 프로브 테이블은 다음과 같이 `EXPLAIN FORMAT=TREE` or `EXPLAIN ANALYZE` 명령을 사용하여 쉽게 구분할 수 있습니다.

```sql
EXPLAIN FORMAT=TREE 
SELECT * 
FROM employees e IGNORE INDEX (PRIMARY, ix_hiredate) 
INNER JOIN dept_emp de IGNORE INDEX(ix_empno_fromdate, ix_fromdate) 
		ON de.emp_no=e.emp_no AND de.from_date=e. hire_date;
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                  |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Inner hash join (e.hire_date = de.from_date), (e.emp_no = de.emp_no)  (cost=9.92e+9 rows=331143)
    -> Table scan on e  (cost=0.0595 rows=299587)
    -> Hash
        -> Table scan on de  (cost=33851 rows=331143)
 |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

→ 최하단 제일 안쪽의 de(dept_emp) 테이블이 빌드 테이블

- `메모리에서 모두 처리 가능한 해시 조인`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3494d8aa-188b-4504-9085-10c634b0a4a6)

    
    - 빌드 테이블인 dept_emp를 읽어 메모리에 해시 테이블 생성
    - 프로프 테이블인 employees 테이블을 스캔하며 해시 테이블에서 레코드를 찾아 결과를 사용자에게 반환
    
- 해시 테이블이 조인 버퍼 메모리 보다 큰 경우의 `해시 조인 1차 처리`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/27356b57-6ff8-4f8e-84d8-287abc042d30)

    
    **해시 테이블을 메모리 저장시 join_buffer_size로 크기 제어가 가능한 조인 버퍼를 사용합니다.**
    
    - dept_emp 테이블을 읽으며 메모리의 해시 테이블을 준비합니다.
    - 지정된 메모리 크기(join_buffer_size)를 넘어서면 나머지 레코드들은 디스크에 청크로 구분하여 저장합니다.
    - 메모리에 올라가 있는 테이블에 대해 employees 테이블의 emp_no 값을 이용해 1차 조인 결과를 생성합니다.

- 해시 테이블이 조인 버퍼 메모리보다 큰 경우의 `해시 조인 2차 처리`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/fc7d8cd7-70e1-4c9e-b23d-4c41b5002469)

    
    2차에선 N개의 청크에 대한 해시 조인 처리를 진행합니다.
    
    - 1차 조인 결과를 생성함과 동시에 employees 테이블에서 읽은 레코드를 디스크에 청크로 구분하여 저장합니다.
    - 1차 조인이 완료됐다면, 각 빌드 테이블의 첫번째 청크들을 가져와서 다시 메모리 해시 테이블을 구축하여 1차 조인과정을 진행합니다.
    - 이는 디스크의 저장된 청크 개수만큼 반복 처리해 최종적으로 완성된 조인 결과를 만듭니다.
        - 이를 위해 2차 해시 함수(해시 키 생성 알고리즘과 달리 다른 해시 함수를 사용한 청크 분리)를 이용해 **빌드 테이블과 프로브 테이블을 동일 개수의 청크로 쪼개어 디스크로 저장**합니다.

MySQL 옵티마이저는 메모리에서 모두 처리 가능한 해시 조인의 경우 `클래식 해시 조인 알고리즘`을 해시 조인 1차 처리가 필요한 경우 `그레이스 해시 조인 알고리즘`을 하이브리드(섞어서) 활용하도록 구현되어 있습니다.

MySQL 서버에서 해시키를 만들기 위한 알고리즘을 xxHash64 해시 함수를 사용합니다.

해시조인 pseudo code

```cpp
result = []
join_buffer = []
partitions = 0
on_disk = false;

## h_tab :: Hash table
## p_tab :: Probe table

// 빌드 테이블(해시테이블)의 레코드를 돎
for (h_tab_row in h_tab){
		// 각 행의 키에 해시 적용
		hash = xxHash64(h_tab_row.join_column)
		// 디스크에 붙여야 하는 상황이 아니라면 버퍼에 붙임 -> 버퍼에 공간이 있다면
		if (not on_disk){
				join_buffer.append(hash)

				// 버퍼 가득 차면
				if (is_full(join_buffer) ){
						## Spill out partition to disk
						// 디스크 플래그 true set
						on_disk = true
						// 현재까지의 버퍼를 디스크에 쓰고 partitions에 청크 크기를 반환
						partitions = write_buffer_to_disk(join_buffer)
						// 조인버퍼 초기화
						join_buffer = []
		
				} else {
						// 버퍼 가득 차지 않았다면 해시를 디스크에 씀 -> 메모리에 써야하는거 아닌가..?
						write_hash_to_disk(hash)
				}
		}
}

// 디스크 쓰기가 false라면
if (not on_disk) {
		// 프로브 테이블의 각 레코드는
		for (p_tab_row in p_tab) {
				// 해시를 적용하고
				hash = xxHash64(p_tab_row.join_column)
				// 조인 버퍼에 레코드가 있다면
				if (hash in join buffer) {
						// 빌드, 프로브 테이블의 행을 가져와서
						h_tab_row = get_row(hash)
						p_tab_row = get_row(hash)
						// 결과에 조인하여 붙여줌
						result.append(join_rows(h_tab_row, p_tab_row))
				}
		}
// 디스크 쓰기가 true라면
} else {
		// 프로브 테이블의 각 레코드는
		for (p_tab_row in p_tab) {
				// 해시 적용 후
				hash = xxHash64(p_tab_row.join_column)
				// 해시를 디스크(메모리)에 씀
				write_hash_to_disk(hash)
		}

		// 이후 청크의 갯수만큼 돌며
		for (part in partitions) {
				// 각 청크들을 조인 버퍼로 불러와서
				Join_buffer = load_build_from_disk(part)
				// N번째 청크의 해시들을 가져와서
				for (hash in load hash_from_disk(part)) {
						// 프로브 해시가 조인 버퍼에 포함된다면 -> 키가 일치한다면
						if (hash in join buffer) {
								// 빌드, 프로브 테이블 행을 가져와서
								h_tab_row = get_row(hash)
								p_tab_row = get_row(hash)
								// 결과에 조인하여 붙여줌
								result.append(join_rows(h_tab_row, p_tab_row))
						}
				}
		// 조인버퍼 초기화
		join_buffer = []
		}
}
```

<br>

**인덱스 정렬 선호(prefer_ordering_index)**

MySQL 옵티마이저는 ORDER BY 또는 GROUP BY 특정 인덱스를 사용해 처리 가능한 경우 쿼리의 실행 계획에서 해당 인덱스의 가중치를 높이 설정해 실행하도록 합니다.

```sql
EXPLAIN 
SELECT * 
FROM employees 
WHERE hire_date BETWEEN '1985-01-01' AND '1985-02-01' 
ORDER BY emp_no;

+----+-------------+-----------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                 |
+----+-------------+-----------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_hiredate   | ix_hiredate | 3       | NULL |   25 |   100.00 | Using index condition; Using filesort |
+----+-------------+-----------+------------+-------+---------------+-------------+---------+------+------+----------+---------------------------------------+
```

위 쿼리는 2가지 실행 계획을 선택할 수 있습니다.

1. ix_hiredate 인덱스를 이용해 BETWEEN 조건에 일치하는 레코드를 찾고 emp_no로 정렬하여 결과 반환
2. employees 테이블의 PK인 emp_no를 정순으로 읽으며 hire_date 칼럼의 조건에 일치하는지 비교 후 결과 반환

상황에 따라 둘 다 모두 효율적일 수 있지만, 일반적으로 1번 조건의 레코드가 많지 않다면 1번이 효율적입니다. 
다만 가끔 실수로 잘못된 실행계획을 선택할 수 있는데, 아주 가끔 발생한다고 합니다.

- 8.0.20까지는 다른 실행 계획 사용을 위해 IGNORE INDEX 힌트를 사용하곤 했습니다.
- **8.0.21부터는 MySQL 서버 옵티마이저가 ORDER BY를 위한 인덱스에 너무 큰 가중치를 부여하지 않도록 prefer_ordering_index 옵티마이저 옵션을 on, off할 수 있습니다.**

<br><br>

### 9.3.2 조인 최적화 알고리즘

MySQL은 조인 쿼리의 실행 계획 최적화를 위한 알고리즘이 2개 있습니다.

조인 최적화의 개선이 많이 이루어졌지만, 테이블의 개수가 많아지면 최적화된 실행 계획을 찾기란 어렵고, 하나의 쿼리에서 조인되는 테이블도 많아지면 실행 계획 수립만 몇 분이 걸릴 수 있습니다.

현재 파트에선 왜 그런 현상이 생기는지, 어떻게 하면 피하는지에 대해 설명합니다.

다음과 같은 4개의 테이블 조인 예시를 이용해 2가지 알고리즘을 설명합니다.

```sql
SELECT * FROM t1, t2, t3, t4 WHERE ...
```

<br>

**Exhaustive 검색 알고리즘**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/795e901c-1dcd-405c-b2c1-7b32daf75b9d)


- 5.0 및 이전 버전에서 사용되던 조인 최적화 기법
- FROM 절에 명시된 모든 테이블의 조합에 대해 실행 계획의 비용을 계산해서 최적의 조합 1개를 찾는 방법
- 그림은 4개지만 20개의 테이블이라면 가능한 조인 조합은 20!으로 많은 시간이 소모됩니다.
(테이블 10개에서 1개만 늘어나도 11배의 시간이 걸림)

<br>

**Greedy 검색 알고리즘**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/8815a5cb-c0c2-4fe0-a98a-318300965208)


- Greedy 검색 알고리즘은 Exhaustive 검색 알고리즘의 시간 소모적인 문제점을 해결하기 위해 5.0부터 도입된 조인 최적화 기법
- 위 예시는 4개의 테이블을 그리디 검색 알고리즘을 및 `optimizer_search_depth=2`로 두고 최적의 조인 순서를 검색하는 방법을 보여줍니다.

순서는 다음과 같습니다.

1. N개의 테이블 중 `optimizer_search_depth` 값에 정의된 테이블로 가능한 조인 조합 생성
→ 4개중에서 2개니까 $4P2 = 12$
2. 조인 조합 중에서 최소 비용의 실행 계획 선정
3. `optimizer_search_depth`개중에서 첫번째 테이블을 `부분 실행 계획`의 첫 번째 테이블로 선정
    1. 선정한 1개 테이블 제외 N - 1개의 테이블에서 `optimizer_search_depth` 시스템 설정 변수에 정의된 개수의 테이블로 가능한 조인 조합 생성 $3P2 = 6$
    2. 생성된 조인 조합들을 3번의 `부분 실행 계획`에 대입해 실행 비용 계산 하고, 최적 실행 계획에서 두 번째 테이블을 부분 실행 계획의 두 번째 테이블로 선정
    3. a ~ b과정을 남은 테이블이 모두 없어질 때까지 반복 실행하여 `부분 실행 계획`에 테이블의 조인 순서를 기록함
4. 최종적으로 `부분 실행 계획`이 테이블의 조인 순서로 결정됨

그리디 검색 알고리즘은 optimizer_search_depth 변수의 값에 따라 조인 최적화 비용이 상당히 줄어들 수 있습니다. (기본값은 62)

MySQL은 조인 최적화를 위한 시스템 변수로 `optimizer_prune_level`과 `optimizer_search_depth`가 제공됩니다.

- **optimizer_search_depth**
    - Greedy, Exhaustive 검색 알고리즘 중에서 어떤 알고리즘을 사용할지 결정하는 시스템 변수
    - `1~62 설정 시`: Greedy 검색 대상을 지정된 개수로 한정해서 최적의 실행 계획을 산출함
    - `0 설정 시`: Greedy 검색을 위한 최적의 조인 검색 테이블의 개수를 MySQL 옵티마이저가 자동으로 결정함
    - 조인에 사용된 테이블의 개수 기준
        - `optimizer_search_depth보다 클 때`: 설정 값의 개수만큼의 테이블은 Exhaustive 검색 사용 나머지 테이블에 Greedy 검색이 사용됨
        - `optimizer_search_depth보다 작을때`: Exhaustive 검색만 사용됨
    - 많은 테이블이 조인되는 쿼리에선 해당 값이 크다면 무리가 됨, optimizer_prune_level 변수가 0이라면 4~5 정도의 설정 권장
- **optimizer_prune_level**
    - 5.0부터 추가된 Heuristic 검색이 작동하는 방식을 제어합니다.
    - Exhaustive, Greedy 검색 알고리즘은 모두 테이블 조인 순서를 결정하기 위해 많은 조인 경로를 비교합니다.
    - Heuristic의 핵심은 이미 계산했던 조인 순서의 비용보다 커진다면 언제든지 계산을 중간에 포기하도록 합니다.
    - 또한 아우터 조인으로 연결되는 테이블은 우선순위에서 제거하는 등 경험 기반의 최적화도 Heuristic 검색 기반 최적화에는 포함돼 있습니다.
    - `1 설정`: 옵티마이저가 조인 순서 최적화에 경험 기반의 Heuristic 알고리즘을 사용합니다.
    - `0 설정`: 사용하지 않음(테이블이 몇개 없더라도 성능차이가 크므로 0으로 설정은 지양)

- optimizer_prune_level 사용에 따른 성능 측정
    
    실제 성능 비교를 위해 책에선 optimizer_prune_level을 1과 0으로 설정하여 각 200개의 테스트 데이터를 가진 30개의 테이블을 조인합니다.
    
    이후 optimizer_search_depth의 값을 5부터 증가시키며 62까지 변화시켜가며 쿼리 실행 계획 수립에 걸리는 시간을 측정합니다.
    
    1. `optimizer_prune_level=1`일 경우
        
        거의 시간 차이 없이 0.1초 이내로 완료됨 
        (과거 버전은 시간차이가 났으니 현재는 optimizer_search_depth값 변화와 관계없이 빠른 시간내에 해결됨)
        
    2. `optimizer_prune_level=0`일 경우
        
        10개의 테이블 부터 급격한 성능저하를 보이고, 그 이후부터는 너무 많은 시간이 소요되어 측정 불가
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/4142896b-3403-4ac8-ae0f-bd39fcbac729)

        
    
    결과적으로 과거의 휴리스틱과 달리 현재 휴리스틱은 많이 개선되었으며 성능차이도 엄청나기에 optimizer_prune_level은 1로 고정해야 함이 필수입니다.
