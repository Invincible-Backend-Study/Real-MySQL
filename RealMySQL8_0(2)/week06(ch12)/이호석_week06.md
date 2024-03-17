# 📌 12장 확장 검색

IT기기의 발전으로 데이터의 형태가 다양해졌습니다. 응용 프로그램 또한 단순한 수준을 넘어서 사용자의 위치와 이동경로, 작성한 문서들에 대한 다양한 형태의 검색 기능을 가지고 있습니다.

MySQL은 대표적인 확장 검색 기능으로 `전문 검색`, `공간 검색` 기능을 제공합니다.

<br><br><br>

## ✅ 12.1 전문 검색 (Full-text Search)

- mysql 서버는 용량이 큰 문서를 단어 수준으로 잘게 쪼개어 문서 검색을 하게 해주는 Full-text Search 기능을 제공합니다.
- 8.0 이전에는 일부 스토리지 엔진을 사용하는 테이블만 전문 검색이 가능, 8.0부터는 가장 사용률이 높은 InnoDB 스토리지 엔진에서도 사용할 수 있게 개선됨
- 한국어는 서구권의 언어에 비해 단어 분리를 통해 형태소를 인덱싱 하는 방식이 적합하지 않았음, 이를 보완하기 위해 8.0부터는 형태소, 어원과 관계 없이 특정 길이의 조각(Token)으로 인덱싱하는 n-gram 파서가 도입됐습니다.
    - n-gram은 한글 문서 검색에는 이용 가치가 매우 높은 기능입니다.

<br><br>

### 12.1.1 전문 검색 인덱스의 생성과 검색

mysql 서버는 2가지 알고리즘을 이용해 인덱싱할 토큰을 분리합니다.

- `형태소 분석(서구권 언어의 경우 어근 분석)`
    - 문장의 공백과 같은 띄어쓰기 단위로 단어 분리하고 각 단어의 조사 제거하여 명사 또는 어근을 찾아 인덱싱함. 다만 mysql서버는 단순히 공백과 같은 띄어쓰기 기준을 토큰을 분리해 인덱싱합니다.(형태소, 어근 분석 기능 없음)
- `n-gram 파서`
    - 문장 자체에 대한 이해 없이 공백과 같은 띄어쓰기 단위로 단어를 분리하고 주어진 길이로 쪼개서 인덱싱하는 알고리즘

<br>

**n-gram 알고리즘 사용법**

- n은 숫자를 의미하며 `ngram_token_size` 시스템 변수로 변경할 수 있습니다. (1 ~ 10사이의 값)
    - 각 값이 1, 2, 3이면 `uni-gram`, `bi-gram`, `tri-gram`이라고 합니다.
    - 3 초과의 값도 설정 가능하지만 검색어의 길이 제약이 생기기 때문에 위 3개가 가장 일반적으로 사용됩니다.

- `bi-gram, tri-gram 사용 전문 검색 인덱스 쿼리 결과 비교`
    - ngram_token_size는 읽기 전용이므로 my.cnf 설정 파일에 `사전 설정이 필요함`
    - 전문 검색 인덱스 생성시 WITH PARSER ngram 옵션을 추가해야 n-gram 파서를 사용해 토큰을 생성함(그렇지 않으면 기본 파서를 사용함)
    
    ```sql
    # 테스트 테이블과 데이터를 준비 : bi-gram
    CREATE TABLE tb_bi_gram (
    		id BIGINT NOT NULL AUTO_INCREMENT,
    		title VARCHAR(100),
    		body TEXT,
    		PRIMARY KEY(id),
    		FULLTEXT INDEX fx_msg(title, body) WITH PARSER ngram
    );
    
    INSERT INTO tb_bi_gram VALUES (NULL, 'Real MySQL', '이 책은 지금까지의 매뉴얼 번역이나 단편적인 지식 수준을 벗어나 저자와 다른 많은 MySQL 전문가의...');
    
    # 단어의 선행 2글자 검색(ngram_token_size와 같은 길이)
    SELECT count(*) 
    FROM tb_bi_gram 
    WHERE MATCH(title, body) AGAINST('단편' IN BOOLEAN MODE);
    +----------+
    | count(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어의 후행 2글자 검색(ngram_token_size와 같은 길이)
    SELECT count(*) 
    FROM tb_bi_gram 
    WHERE MATCH(title, body) AGAINST('적인' IN BOOLEAN MODE);
    +----------+
    | count(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어 전체 검색(ngram_token_size보다 큰 길이)
    SELECT count(*) 
    FROM tb_bi_gram 
    WHERE MATCH(title, body) AGAINST('단편적인' IN BOOLEAN MODE);
    +----------+
    | count(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어 전체 검색(ngram_token_size보다 작은 길이)
    SELECT count(*) 
    FROM tb_bi_gram 
    WHERE MATCH(title, body) AGAINST('이' IN BOOLEAN MODE);
    +----------+
    | count(*) |
    +----------+
    |        0 |
    +----------+
    
    # 단어의 선행 1글자 검색(ngram_token_size보다 작은 길이)
    SELECT count(*) 
    FROM tb_bi_gram 
    WHERE MATCH(title, body) AGAINST('책' IN BOOLEAN MODE);
    +----------+
    | count(*) |
    +----------+
    |        0 |
    +----------+
    ```
    
    ```sql
    # 3-gram 예제
    # my.cnf 파일에 ngram_token_size=3으로 설정
    
    # 테이블 및 데이터 준비
    CREATE TABLE tb_tri_gram (
    		id BIGINT AUTO_INCREMENT,
    		title VARCHAR(100),
    		body TEXT,
    		PRIMARY KEY(id),
    		FULLTEXT INDEX fx_msg(title, body) WITH PARSER ngram
    );
    
    INSERT INTO tb_tri_gram VALUES (NULL, 'Real MySQL', '이 책은 지금까지의 매뉴얼 번역이나 단편적인 지식 수준을 벗어나 저자와 다른 많은 MySQL 전문가의 ...');
    
    # 단어의 선행 3글자 검색(ngram_token_size와 같은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('단편적' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어의 후행 3글자 검색(ngram_token_size와 같은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('편적인' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어 전체 검색(ngram_token_size보다 큰 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('단편적인' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        1 |
    +----------+
    
    # 단어 전체 검색(ngram_token_size보다 작은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('이' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        0 |
    +----------+
    
    # 단어의 선행 1글자 검색(ngram_token_size 보다 작은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('책' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        0 |
    +----------+
    
    # 단어의 선행 2글자 검색(ngram_token_size 보다 작은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('단편' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        0 |
    +----------+
    
    # 단어의 후행 2글자 검색(ngram_token_size 보다 작은 길이)
    SELECT COUNT(*) FROM tb_tri_gram
    WHERE MATCH(title, body) AGAINST ('적인' IN BOOLEAN MODE);
    +----------+
    | COUNT(*) |
    +----------+
    |        0 |
    +----------+
    ```
    
    - 검색어의 길이가 ngram_token_size보다 작은 경우: `검색 불가능`
    - 검색어의 길이가 ngram_token_size보다 크거나 같은 경우: `검색 가능`
    - **n-gram 전문 검색 인덱스는 검색어가 단어의 시작 부분이 아니고 단어의 중간, 마지막이어도 검색이 가능합니다.**
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/9442826b-38f9-4787-ac5f-669f16316058)

        
        - 다만 ngram_token_size보다  길이가 작은 단어들은 모두 버립니다.
    - **쿼리의 전문 검색에서도 n-gram 파서가 사용됩니다.**
        
        ```sql
        SELECT COUNT(*) FROM tb_bi_gram
        WHERE MATCH(title, body) AGAINST ('단편적인' IN BOOLEAN MODE);
        ```
        
        - 위와 같은 쿼리에서 4글자를 검색할때도 역시 `ngram_token_size` 변수값에 맞게 토큰을 잘라내어 전문 검색 인덱스와 동등 비교 조건으로 검색합니다.
        - 검색된 결과를 도큐먼트 ID로 그루핑함
            - 전문 검색 인덱스의 경우 PK와 별개로 레코드별로 ID를 가짐 이를 도큐먼트 ID라고 함
            (도큐먼트는 레코드 혹은 로우와 동의어)
        - 그루핑된 결과에서 각 단어의 위치를 이용해 최종 검색어를 포함하는지 식별함
        - 따라서 ngram_token_size 시스템 변수 값은 테이블 생성, 데이터 저장, 쿼리 실행될때 모두 동일한 값으로 유지돼야 합니다.

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/3829a077-009f-4ef5-ba28-c9a39bc8c8b1)


---

<br><br>

### 12.1.2 전문 검색 쿼리 모드

- mysql 서버의 전문 검색 쿼리는 `자연어(NATURAL LANGUAGE) 검색 모드`, `불리언(BOOLEAN) 검색 모드`를 지원합니다.
- 특별히 지정하지 않으면 자연어 검색 모드가 사용됨
- mysql 서버는 자연어 검색 모드와 함께 사용할 수 있는 검색어 확장(Query Expansion) 기능도 지원

<br>

**12.1.2.1 자연어 검색(NATURAL LANGUAGE MODE)**

```sql
# 자연어 검색 테스트 위한 tb_bi_gram 테이블 세팅
TRUNCATE TABLE tb_bi_gram;

INSERT INTO tb_bi_gram VALUES
		(NULL, 'Oracle', 'Oracle is database'),
		(NULL, 'MySQL', 'MySQL is database'),
		(NULL, 'MySQL article', 'MySQL is best open source dbms'),
		(NULL, 'Oracle article', 'Oracle is best commercial dbms'),
		(NULL, 'MySQL Manual', 'MySQL manual is true guide for MySQL');
```

- mysql 서버의 자연어 검색은 검색어에 제시된 단어들을 많이 가지고 있는 순서대로 `정렬해서 결과를 변환`함
    
    ```sql
    SELECT id, title, body,
    		MATCH(title, body) AGAINST ('MySQL' IN NATURAL LANGUAGE MODE) AS score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST ('MySQL' IN NATURAL LANGUAGE MODE);
    
    # 정렬된 결과
    +----+---------------+--------------------------------------+--------------------+
    | id | title         | body                                 | score              |
    +----+---------------+--------------------------------------+--------------------+
    |  5 | MySQL Manual  | MySQL manual is true guide for MySQL | 0.5906023979187012 |
    |  2 | MySQL         | MySQL is database                    | 0.3937349319458008 |
    |  3 | MySQL article | MySQL is best open source dbms       | 0.3937349319458008 |
    +----+---------------+--------------------------------------+--------------------+
    ```
    
- 전문 검색 쿼리에서 자연어 문장을 그대로 사용할 수도 있습니다.
(이런 형태를 `Phrase Search` 라고 함)
    
    ```sql
    SELECT id, title, body,
    		MATCH(title, body) AGAINST('MySQL manual is true guide' IN NATURAL LANGUAGE MODE) AS score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST ('MySQL manual is true guide' IN NATURAL LANGUAGE MODE);
    
    +----+---------------+--------------------------------------+--------------------+
    | id | title         | body                                 | score              |
    +----+---------------+--------------------------------------+--------------------+
    |  5 | MySQL Manual  | MySQL manual is true guide for MySQL |    3.5219566822052 |
    |  2 | MySQL         | MySQL is database                    | 0.3937349319458008 |
    |  3 | MySQL article | MySQL is best open source dbms       | 0.3937349319458008 |
    +----+---------------+--------------------------------------+--------------------+
    ```
    
    - 문장이 검색어로 사용되면 검색어를 구분자(공백, \n)로 단어를 분리하고 다시 n-gram 파서로 토큰을 생성한 후 각 토큰에 대해 일치하는 단어의 개수를 확인해 일치율을 계산함
    - 모든 단어가 포함된 레코드 및 일부만 포함하는 결과도 가져옴
    - 검색어가 단일 단어, 문장이면 `“.”`, `“,”` 과 같은 문장기호는 모두 무시

---

<br>

**12.1.2.2 불리언 검색(BOOLEAN MODE)**

- 불리언 검색은 쿼리에 사용되는 검색어의 존재 여부에 대한 논리적 연산이 가능합니다.
    
    ```sql
    # MySQL 존재, manual 제외
    SELECT id, title, body,
    		MATCH(title, body) AGAINST ('+MySQL -manual' IN BOOLEAN MODE) AS score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST ('+MySQL -manual' IN BOOLEAN MODE);
    
    +----+---------------+--------------------------------+--------------------+
    | id | title         | body                           | score              |
    +----+---------------+--------------------------------+--------------------+
    |  2 | MySQL         | MySQL is database              | 0.3937349319458008 |
    |  3 | MySQL article | MySQL is best open source dbms | 0.3937349319458008 |
    +----+---------------+--------------------------------+--------------------+
    
    # MySQL, manual 모두 존재
    SELECT id, title, body,
    		MATCH(title, body) AGAINST ('+MySQL +manual' IN BOOLEAN MODE) AS score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST ('+MySQL +manual' IN BOOLEAN MODE)
    +----+--------------+--------------------------------------+--------------------+
    | id | title        | body                                 | score              |
    +----+--------------+--------------------------------------+--------------------+
    |  5 | MySQL Manual | MySQL manual is true guide for MySQL | 1.5677205324172974 |
    +----+--------------+--------------------------------------+--------------------+
    ```
    
    - `+`: 해당 표시가 붙은 단어는 전문 검색 인덱스 칼럼에 존재해야 함
    - `-`: 해당 표시가 붙은 단어는 전문 검색 인덱스 칼럼에 존재하면 안됨
- 불리언 검색에선 쌍따옴표(”)로 묶인 구는 하나의 단어인 것처럼 취급됩니다.
    
    ```sql
    SELECT id, title, body,
    		MATCH(title, body) AGAINST ('+"MySQL man"' IN BOOLEAN MODE) AS score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST ('+"MySQL man"' IN BOOLEAN MODE);
    +----+--------------+--------------------------------------+--------------------+
    | id | title        | body                                 | score              |
    +----+--------------+--------------------------------------+--------------------+
    |  5 | MySQL Manual | MySQL manual is true guide for MySQL | 0.5906023979187012 |
    +----+--------------+--------------------------------------+--------------------+
    ```
    
    - `+“MySQL man”` 이므로 `MySQL man`이라는 구문을 가진 레코드를 검색합니다.
    - 또한 띄어쓰기까지 정확히 일치하는것을 찾지 않고 MySQL 단어 뒤에 man이라는 단어가 나오면 일치하는 것으로 판단합니다.
- 불리언 검색에서 `“+”`, `“-”`가 없으면 검색어에 포함된 단어 중 아무거나 하나라도 있으면 일치하는 것으로 판단합니다.
    
    ```sql
    SELECT id, title, body,
    		MATCH(title, body) AGAINST ('MySQL doc' IN BOOLEAN MODE) as score
    FROM tb_bi_gram
    WHERE MATCH(title, body) AGAINST('MySQL doc' IN BOOLEAN MODE);
    
    +----+---------------+--------------------------------------+--------------------+
    | id | title         | body                                 | score              |
    +----+---------------+--------------------------------------+--------------------+
    |  5 | MySQL Manual  | MySQL manual is true guide for MySQL | 0.5906023979187012 |
    |  2 | MySQL         | MySQL is database                    | 0.3937349319458008 |
    |  3 | MySQL article | MySQL is best open source dbms       | 0.3937349319458008 |
    +----+---------------+--------------------------------------+--------------------+
    ```
    
    - `MySQL`, `doc` 중 아무거나 하나라도 일치하는 레코드가 있으면 결과로 반환합니다.
    
- `+`, `-`가 사용되지 않은 불리언 검색과 자연어 검색은 거의 흡사하게 작동합니다.
- 다만 서로 다른 방식으로 일치율을 계산하기 때문에 **동일한 검색어를 사용한다고 하더라도 두 모드의 일치율은 다른 값을 보여줍니다.**

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/9726a213-c3de-4be2-975e-ffc0773f5af2)


---

<br>

**12.1.2.3 검색어 확장(QUERY EXPANSION)**

- 사용자가 쿼리에 사용한 검색어로 검색된 결과에서 공통으로 발견되는 단어들을 모아서 다시 한번 더 검색을 수행하는 방식
- **database 키워드가 들어간 게시글에서 MySQL, Oracle과 같은 특정 DBMS의 이름도 검색하고 싶을때** 검색어 확장을 이용할 수 있습니다.

```sql
# 1. 위와 같이 QUERY EXPANSION을 통해 검색하면
SELECT * FROM tb_bi_gram
WHERE MATCH(title, body) AGAINST('database' WITH QUERY EXPANSION);

# 2. 우선 초기 입력 키워드로 전문 검색을 실행합니다.
SELECT * FROM tb_bi_gram
WHERE MATCH(title, body) AGAINST('database');
+----+--------+--------------------+
| id | title  | body               |
+----+--------+--------------------+
|  2 | Oracle | Oracle is database |
|  3 | MySQL  | MySQL is database  |
+----+--------+--------------------+

# 3. 위 검색 결과에서 단어를 다시 수집하여 또다시 전문 검색 쿼리를 실행해 최종 결과를 반환합니다.
+----+----------------+-------------------------------------------+
| id | title          | body                                      |
+----+----------------+-------------------------------------------+
|  3 | MySQL          | MySQL is database                         |
|  4 | MySQL article  | MySQL is best open source dbms            |
|  2 | Oracle         | Oracle is database                        |
|  5 | Oracle article | Oracle is best commercial dbms            |
+----+----------------+-------------------------------------------+
```

- 위 예시는 검색에 알맞은 데이터만 들어있었기에 깔끔한 결과가 보여짐
- MySQL 검색어 확장 기능은 Blind Query expansion 알고리즘을 사용함, 이는 자주 사용된 단어들을 모아 다시 전문 검색 쿼리를 실행하므로 원하지 않은 결과가 많고 추가적인 전문 검색이 불필요하게 많이 실행될 수 있습니다.

---

<br><br>

### 12.1.3 전문 검색 인덱스 디버깅

- InnoDB 스토리지 엔진에서 전문 검색 인덱스를 사용할 수 있게 되며 `불용어`, `토큰 파서` 등의 기능을 제어하는 `시스템 변수`가 좀 더 다양해졌음
- 여러 옵션이 있기에 우리가 몇가지 옵션을 빼먹을 수 있습니다.
    - `ft_stopword_file`: 모든 스토리지 엔진에 대해 적용
    - `innodb_ft_server_stopword_table`: InnoDB 스토리지 엔진에 대해서만 적용됨
    - 인덱스 생성시 `WITH PARSER ngram` 등과 같은 옵션도 잊을 때가 있음
- 전문 검색 인덱스는 원래의 값을 가공하여 인덱싱하므로 오류의 원인을 찾기 어렵지만, mysql 서버는 전문 검색 쿼리 오류의 원인을 쉽게 찾을 수 있게 **전문 검색 인덱스 디버깅 기능을 제공합니다.**

**SET GLOBAL innodb_ft_aux_table = ‘test/tb_bi_gram’;**

- innodb_ft_aux_table에 전문 검색 인덱스를 가진 테이블이 설정되면 mysql 서버는 `information_schema` DB의 다음 테이블을 통해 전문 검색 인덱스가 어떻게 저장, 관리되는지 볼 수 있게 합니다.

```sql
# innodb_ft_config : 전문 검색 인덱스의 설정 내용을 보여줌
SELECT * FROM information_schema.innodb_ft_config;
+---------------------------+-------+
| KEY                       | VALUE |
+---------------------------+-------+
| optimize_checkpoint_limit | 180   |
| synced_doc_id             | 0     |
| stopword_table_name       |       |
| use_stopword              | 1     |
+---------------------------+-------+

# innodb_ft_index_table : 전문 검색 인덱스가 가진 인덱스 엔트리 목록 보여줌
# 각 엔트리는 토큰들이 어떤 레코드에 몇 번 사용됐는지, 레코드별 문자 위치가 어디인지 등의 정보를 관리함
SELECT * FROM information_schema.innodb_ft_index_table;
+------+--------------+-------------+-----------+--------+----------+
| WORD | FIRST_DOC_ID | LAST_DOC_ID | DOC_COUNT | DOC_ID | POSITION |
+------+--------------+-------------+-----------+--------+----------+
| bm   |            4 |           5 |         2 |      4 |       41 |
| bm   |            4 |           5 |         2 |      5 |       42 |
| ce   |            4 |           5 |         2 |      4 |       37 |
| ce   |            4 |           5 |         2 |      5 |       19 |
| cl   |            2 |           5 |         3 |      2 |        3 |
| cl   |            2 |           5 |         3 |      2 |        7 |
...

# innodb_ft_index_cache : 테이블 레코드 insert시 mysql 서버는 전문 검색 인덱스를 위한 토큰을 분리해
# 즉시 디스크로 저장하지 않고 메모리에 임시 저장하는데 이때 임시로 저장되는 공간입니다.
# 해당 캐시 크리를 넘어서면 한꺼번에 모아 디스크 파일로 저장함(전문 검색 인덱스를 사용한다면 캐시 크기도 고려해봐야함)
INSERT INTO tb_bi_gram VALUES(NULL, 'Oracle', 'Oracle is database');

SELECT * FROM information_schema.innodb_ft_index_cache;
+------+--------------+-------------+-----------+--------+----------+
| WORD | FIRST_DOC_ID | LAST_DOC_ID | DOC_COUNT | DOC_ID | POSITION |
+------+--------------+-------------+-----------+--------+----------+
| cl   |            8 |           8 |         1 |      8 |        3 |
| cl   |            8 |           8 |         1 |      8 |        7 |
| le   |            8 |           8 |         1 |      8 |        4 |
| le   |            8 |           8 |         1 |      8 |        7 |
| se   |            8 |           8 |         1 |      8 |       23 |
+------+--------------+-------------+-----------+--------+----------+

# innodb_ft_deleted와 innodb_ft_being_deleted : 테이블의 레코드가 삭제되면 어떤 레코드가 삭제됐는지,
# 그리고 어떤 레코드가 현재 전문 검색 인덱스에서 삭제되고 있는지를 보여줍니다.
DELETE FROM tb_bi_gram where id = 1;

SELECT * FROM information_schema.innodb_ft_deleted;
+--------+
| DOC_ID |
+--------+
|      2 |
+--------+
```

---

<br><br><br>

## ✅ 12.2 공간 검색

<br><br>

### 12.2.1 용어 설명

공간 데이터 관리를 위해 선행되어야 할 용어

- `OGC(Open Geospatial Consortium)`
    
    위치 기반 데이터에 대한 표준을 수립하는 단체. 정부와 기업체, 학교 등 모든 기관이 자유롭게 가입할 수 있음
    현재 500여 학교, 기업, 정부 기관이 참여하고 있고 정기적으로 위치 기반 데이터 및 처리에 대한 표준을 제정하고 개선하고 있음
    
- `OpenGIS(Open Geographic Information System)`
    
    OGC에서 제정한 지리 정보 시스템 표준으로 WKT, WKB같은 지리 정보 데이터를 표기하는 방법, 저장하는 방법 그리고 SRID와 같은 표준을 포함한다.
    
    OpenGIS 표준을 준수한 응용 프로그램의 위치 기반 데이터는 상호 변환 없이 교환 가능하도록 설계돼 있음
    
- `SRS(Spatial Reference System)`와 `GCS(Geographic Coordinate System)`, `PCS(Projected Coordinate System)`
    
    공간 참조 시스템 정도로 번역할 수 있음, 흔히 말하는 좌표계
    
    - SRS는 크게 GCS, PCS로 구분됨
    - `GCS`는 지구 구체상의 특정 위치나 공간을 표현하는 좌표계를 의미함 (위도, 경도와 같은 각도 단위의 숫자로 표시됨)
    - `PCS`는 구체 형태의 지구를 종이 지도와 같은 평면으로 투영(프로젝션)시킨 좌표계를 의미함, 주로 미터와 같은 선형적인 단위로 표시됨
    - `GCS`는 지구와 같은 구체 표면에서 특정 위치를 정의함
    (위치 WHERE가 관심사)
    - PCS는 GCS 위치 데이터를 2차원 평면인 종이에 어떻게 표현할지를 정의합니다.
    (어떻게 HOW 표현할지가 주 관심사, PCS좌표는 GCS 좌표도 같이 포함함)
    - 지구상 동일 지점이라도 어떤 공간 좌표계(SRS)를 사용하느냐에 따라 표시 방법이 달라짐
        - SRS는 좌표계, GCS는 지리 좌표계(구면 좌표계), PCS는 투영 좌표계
- `SRID와 SRS-ID`
    
    `SRID: Spatial Reference ID`, SRS를 지칭하는 고유 번호 (SRIS-ID == SRID)
    
    mysql 서버는 SRS 고유 번호를 저장하는 칼럼의 이름으로 SRS_ID를 사용함
    
    함수의 인자나 식별자로 사용될 경우 SRID를 사용합니다.
    
- `WKT와 WKB(Well-Known Text format, Well-Known Binary format)`
    
    OGC에서 제정한 OpenGIS에서 명시한 위치 좌표의 표현 방법, 두 방식 모두 OpenGIS 표준에 명시된 공간 데이터 표기 방법입니다.
    
    - WKT는 사람의 눈으로 쉽게 확인할 수 있는 텍스트 포맷
        - WKT: `POINT(15 20)`, `LINESTRING(0 0, 10 10, 20 25, 50 60)` 등과 점이나 도형의 위치 정보를 정의하는 표준이다.
    - WKB는 컴퓨터에 저장할 수 있는 형태의 이진 포맷의 저장 표준
    - **MySQL 서버가 내부적으로 공간 데이터를 처리하거나 저장할때는 위 두 포맷을 사용하지 않음
    내부적으로는 이진 데이터 포맷은 WKB포맷과 흡사하지만 완전히 동일하지는 않음**
- `MBR과 R-Tree(Minimum Bounding Rectangle)`
    - mysql 매뉴얼을 보면 MBR이라는 표현을 자주 접함
    - 어떤 도형을 감싸는 최소의 사각 상자를 의미합니다.
    - MBR은 mysql 서버의 공간 인덱스(Spatial Index)는 도형의 포함 관계를 이용합니다.
    - 도형들의 포함관계를 이용해 만들어진 인덱스를 R-Tree 라고 합니다.

---

<br><br>

### 12.2.2 SRS(Spatial Reference System)

좌표계(SRS)는 지리 좌표계(GCS)와 PCS(투영 좌표계)로 구분됩니다. mysql 서버에서 지원하는 SRS는 5000여개가 넘습니다.

- mysql 서버에서 지원하는 SRS에 대한 정보는 information_schema DB의 `ST_SPATIAL_REFERENCE_SYSTEMS` 테이블을 통해 확인 가능합니다.

```sql
DESC information_schema.ST_SPATIAL_REFERENCE_SYSTEMS;
+--------------------------+---------------+------+-----+---------+-------+
| Field                    | Type          | Null | Key | Default | Extra |
+--------------------------+---------------+------+-----+---------+-------+
| SRS_NAME                 | varchar(80)   | NO   |     | NULL    |       |
| SRS_ID                   | int unsigned  | NO   |     | NULL    |       |
| ORGANIZATION             | varchar(256)  | YES  |     | NULL    |       |
| ORGANIZATION_COORDSYS_ID | int unsigned  | YES  |     | NULL    |       |
| DEFINITION               | varchar(4096) | NO   |     | NULL    |       |
| DESCRIPTION              | varchar(2048) | YES  |     | NULL    |       |
+--------------------------+---------------+------+-----+---------+-------+
```

- `SRS_ID`, `DEFINITION` 칼럼이 중요함
    - SRS_ID 칼럼은 좌표계를 지칭하는 고유 번호가 저장됨
        
        고유 번호는 칼럼에 데이터를 저장하거나 검색할때 필요하기에 사용자는 저장하는 공간 데이터가 어느 좌표계를 사용하는지 알고있어야 합니다.
        
    - DEFINITION 칼럼은 좌표계가 어떤 좌표계인지에 대한 정의가 저장돼 있음

- SRS_ID=4326인 지리 좌표계의 정의
    
    ```sql
    # GPS가 사용하는 좌표계 정보가 ST_SPATIAL_REFERENCE_SYSTEMS 테이블에 저장하는 방식을 보여줌
    SELECT * FROM information_schema.ST_SPATIAL_REFERENCE_SYSTES WHERE SRS_ID=4326 \G
    *************************** 1. row ***************************
                    SRS_NAME: WGS 84
                      SRS_ID: 4326
                ORGANIZATION: EPSG
    ORGANIZATION_COORDSYS_ID: 4326
                  DEFINITION: GEOGCS["WGS 84",DATUM["World Geodetic System 1984",
    						              SPHEROID["WGS 84",6378137,298.257223563,
    						              AUTHORITY["EPSG","7030"]],AUTHORITY["EPSG","6326"]],
    						              PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],
    						              UNIT["degree",0.017453292519943278,AUTHORITY["EPSG","9122"]],
    						              AXIS["Lat",NORTH],AXIS["Lon",EAST],AUTHORITY["EPSG","4326"]]
                 DESCRIPTION: NULL
    ```
    
    - 좌표계 이름은 `WGS 84`
    (모바일 휴대폰의 GPS장치가 수신하는 대부분의 위경도 좌표가 SRID 4326 좌표계 데이터이므로 다른 좌표계보다 많이 사용될것)
    - 좌표계 `고유 번호는 4326`이고 입력되는 값의 `단위(UNIT)`는 `각도(Degree)`
    - `DEFINITION` 칼럼의 값은 항상 GEOGCS, PROJCS로 시작되는데 전자는 지리 좌표계(GCS), 후자는 투 좌표계(PCS)를 의미합니다.
    - mysql 서버가 지원하는 지리 좌표계는 483개, 투영 좌표계는 4668개입니다. 평면으로 투영해 관리해도 오차가 크지 않으므로 후자가 많음
    - `AXIS` 필드는 두 번 보이는데 각각 위도와 경도 입니다.
    - WGS 84를 사용하려면 POINT(위도 경도)와 같이 표현해야함 (전자 X축, 후자 Y축)
    (ST_X()는 위도값, ST_Y()는 경도 값 반환)

- SRS_ID=3857인 투영 좌표계의 정의
    
    ```sql
    SELECT * FROM information_schema.ST_SPATIAL_REFERENCE_SYSTEMS WHERE SRS_ID=3857 \G
    
    *************************** 1. row ***************************
                    SRS_NAME: WGS 84 / Pseudo-Mercator
                      SRS_ID: 3857
                ORGANIZATION: EPSG
    ORGANIZATION_COORDSYS_ID: 3857
                  DEFINITION: PROJCS["WGS 84 / Pseudo-Mercator",
    							              GEOGCS["WGS 84",
    							              DATUM["World Geodetic System 1984",
    							              SPHEROID["WGS 84",6378137,298.257223563,AUTHORITY["EPSG","7030"]],
    							              AUTHORITY["EPSG","6326"]],
    						              PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],
    						              UNIT["degree",0.017453292519943278,AUTHORITY["EPSG","9122"]],
    						              AXIS["Lat",NORTH],AXIS["Lon",EAST],AUTHORITY["EPSG","4326"]],
    						              PROJECTION["Popular Visualisation Pseudo Mercator",AUTHORITY["EPSG","1024"]],
    						              PARAMETER["Latitude of natural origin",0,AUTHORITY["EPSG","8801"]],
    						              PARAMETER["Longitude of natural origin",0,AUTHORITY["EPSG","8802"]],
    						              PARAMETER["False easting",0,AUTHORITY["EPSG","8806"]],
    						              PARAMETER["False northing",0,AUTHORITY["EPSG","8807"]],
    						              UNIT["metre",1,AUTHORITY["EPSG","9001"]],
    						              AXIS["X",EAST],AXIS["Y",NORTH],AUTHORITY["EPSG","3857"]]
                 DESCRIPTION: NULL
    ```
    
    - 값의 단위가 meter
    - DEFINITION의 `(AXIS["X",EAST],AXIS["Y",NORTH])`을 보면 SRID=4326과는 반대로 'POINT(경도 위도)' 순서임
        - 경도는 세로선으로 동, 서를 가르고 위도는 가로선으로 북, 남을 갈라서..?
- mysql 8.0에서 공간데이터를 저장할때 SRID를 별도로 지정하거나 지정하지 않는것 무엇을 택해야할까?
    
    ```sql
    # 평면 좌표계(SRID=0)을 사용하는 공간 데이터
    SELECT ST_Distance(ST_PointFromText('POINT(0 0)', 0),
    		ST_PointFromText('POINT(1 1)', 0)) AS distance;
    +--------------------+
    | distance           |
    +--------------------+
    | 1.4142135623730951 |
    +--------------------+
    		
    # 웹 기반 지도 좌표계(SRID=3857)를 사용하는 공간 데이터
    SELECT ST_Distance(ST_PointFromText('POINT(0 0)', 3857),
         ST_PointFromText('POINT(1 1)', 3857)) AS distance; 
    +--------------------+
    | distance           |
    +--------------------+
    | 1.4142135623730951 |
    +--------------------+
    
    # WGS 84 지리 좌표계(SRID=4326)를 사용하는 공간 데이터
    SELECT ST_Distance(ST_PointFromText('POINT(0 0)', 4326),
    		ST_PointFromText('POINT(1 1)', 4326)) AS distance;
    +--------------------+
    | distance           |
    +--------------------+
    | 156897.79947260793 |
    +--------------------+
    ```
    
    - 1번은 별도의 SRID의 지정없이 계산했으므로 단순히 피타고라스 정리에 의한 수식으로 계산된 거리값입니다.(인간의 실생활과 연관시킬 수 있는 수치 단위가 아님)
    - 2, 3번은 공간 데이터 자체가 SRS(좌표계)에 대한 정보를 가지게 되어 해당 좌표계에 맞게 값이 계산됩니다.
    - SRID가 없거나 0이면 MySQL 서버의 공간 함수들이 실제 데이터의 SRID를 알 수 없기에 사용자가 기대하는 값을 계산하지 못할 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c74e01bc-a5f8-444c-be69-14c1319f3bd4)

    

---

<br><br>

### 12.2.3 투영 좌표계와 평면 좌표계

- mysql 서버의 평면 좌표계는 투영 좌표계나 지리 좌표계에 속하지 않습니다.
- 또한 평면 좌표계와 투영 좌표계는 비슷한 특성을 가집니다.

지구 구체 전체 또는 일부를 평면으로 투영(Projection)해서 표현한 좌표계를 투영 좌표계라고 합니다.
(SRID=0인 좌표계도 평면에 표시되는 좌표계지만 투영 좌표계라고 하진 않고 평면 좌표계라고 함)

SRID=0인 좌표계는 단위를 가지지 않고 X, Y축의 값이 제한을 가지지 않으므로 `무한 평면 좌표계`라고 함
**(mysql 서버에서 평면 좌표계는 SRID=0인 좌표계가 유일함)**

근데 SRID=0, SRID=3857은 값의 단위 및 최대 최솟값을 제외하면 거의 차의가 없음

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/d8765a38-cfe6-420f-8299-f236b02da512)


- 위 그림은 SRID=0인 평면 좌표계를 표현하는데 X, Y축의 단위를 미터로 표현하면 SRID=3857인 투영 좌표계가 됨

mysql서버는 특별히 SRID 지정이 없으면 기본값을 0을 가집니다. 또한 평면, 투영 좌표계에서 거리 계산은 피타고라스 정리 수식에 의해서 좌표 간의 거리가 계산됩니다.

단위가 없는 평면 좌표계에비해 투영 좌표계는 두 점간의 거리를 단위에 따라 결정합니다. (SRID=3857은 meter)

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/11483089-c288-448b-83ed-02b31db37f3f)


- 평면, 투영 좌표계 사용 예시
    
    ```sql
    # 평면 좌표계 사용 예제
    CREATE TABLE plain_coord (
    		id INT NOT NULL AUTO_INCREMENT,
    		location POINT SRID 0,
    		PRIMARY KEY(id)
    );
    
    INSERT INTO plain_coord VALUES (1, ST_PointFromText('POINT(0 0)'));
    INSERT INTO plain_coord VALUES (2, ST_PointFromText('POINT(5 5)', 0));
    
    # 투영 좌표계 사용 예제
    CREATE TABLE projection_coord (
    		id INT NOT NULL AUTO_INCREMENT,
    		location POINT SRID 3857,
    		PRIMARY KEY(id)
    );
    
    INSERT INTO projection_coord
    		VALUES (1, ST_PointFromText('POINT(14133791.066622 4509381.876958)', 3857));
    INSERT INTO projection_coord
    		VALUES (2, ST_PointFromText('POINT(14133793.435554 4494917.464846)', 3857));
    		
    	
    # SRID가 일치하지 않으면 에러 발생
    INSERT INTO plain_coord VALUES (2, ST_PointFromText('POINT(5 5)', 4326));
    ERROR 3643 (HY000): The SRID of the geometry does not match the SRID of the column 'location'.
    The SRID of the geometry is 4326, but the SRID of the column is 0. Consider changing the SRID of the geometry or the SRID property of the column.
    ```
    
    - `ST_PointFromText()` 함수를 이용해 POINT(X Y) 문자열(WKT)을 Geometry 타입 값으로 변환할때 SRID를 설정하지 않아도 자동으로 SRID=0으로 설정됩니다.
    - 반면에 투영 좌표계를 사용하기 위해서는 반드시 SRID=3857을 명시해야 합니다.
    - 테이블 생성시 SRID를 명시적으로 정의하지 않으면 모든 SRID를 저장할 수 있지만 제각각인 SRID에서 mysql 서버는 인덱스를 이용해 빠른 검색을 수행할 수 없습니다. 
    (마치 VARCHAR 타입의 칼럼에 여러 콜레이션을 섞어 저장해둔것과 같은 결과)
    
- 테이블에 저장된 평면 좌표, 투영 좌표 데이터 조회
    
    ```sql
    # 평면 좌표계
    SELECT id, location, ST_AsWKB(location) FROM plain_coord \G
    *************************** 1. row ***************************
                    id: 1
              location: 0x00000000010100000000000000000000000000000000000000
    ST_AsWKB(location): 0x010100000000000000000000000000000000000000
    *************************** 2. row ***************************
                    id: 2
              location: 0x00000000010100000000000000000014400000000000001440
    ST_AsWKB(location): 0x010100000000000000000014400000000000001440
    
    SELECT id,
           ST_AsText(location) AS location_wkt,
    	     ST_X(location) AS location_x,
           ST_Y(location) AS location_y
    FROM plain_coord;
    +----+--------------+------------+------------+
    | id | location_wkt | location_x | location_y |
    +----+--------------+------------+------------+
    |  1 | POINT(0 0)   |          0 |          0 |
    |  2 | POINT(5 5)   |          5 |          5 |
    +----+--------------+------------+------------+
    
    # 투영 좌표계
    SELECT id, location, ST_AsWKB(location) FROM projection_coord \G
    *************************** 1. row ***************************
                    id: 1
              location: 0x110F0000010100000076C421E243F56A4172142078B1335141
    ST_AsWKB(location): 0x010100000076C421E243F56A4172142078B1335141
    *************************** 2. row ***************************
                    id: 2
              location: 0x110F00000101000000F10EF02D44F56A417009C05D91255141
    ST_AsWKB(location): 0x0101000000F10EF02D44F56A417009C05D91255141
    
    SELECT id,
           ST_AsText(location) AS location_wkt,
    	     ST_X(location) AS location_x,
           ST_Y(location) AS location_y
    FROM projection_coord;
    +----+---------------------------------------+-----------------+----------------+
    | id | location_wkt                          | location_x      | location_y     |
    +----+---------------------------------------+-----------------+----------------+
    |  1 | POINT(14133791.066622 4509381.876958) | 14133791.066622 | 4509381.876958 |
    |  2 | POINT(14133793.435554 4494917.464846) | 14133793.435554 | 4494917.464846 |
    +----+---------------------------------------+-----------------+----------------+
    ```
    
    - 두 좌표계 첫번째 쿼리는 location 칼럼에 저장된 값을 그대로 조회하여 이진 데이터가 보여집니다.
    (mysql 서버가 내부적으로 사용하는 이진 포맷의 데이터, WKB앞에 SRID를 위한 4바이트 공간이 추가)
        - `ST_AsWKB()`함수의 결괏값이 WKB(Well Known Binary) 포맷의 공간 데이터임
    - 두 번째 쿼리는 ST_X(), ST_Y()를 이용해 공간 좌표 데이터의 X축 값과 Y축 값을 각각 가져옵니다.

- 공간 데이터가 사용하는 SRID는 ST_SRID() 함수를 이용하여 확인할 수 있습니다.
    
    `SELECT ST_SRID(ST_PointFromText('POINT(5 5)', 0));`
    
- 평면, 투영 좌표계에서 두 점 간의 거리를 계산하면 피타고라스 정리에 의해 거리 계산 결과가 보임을 알 수 있습니다. (SRID=3867이어도 계산된 거리 값은 지구 구체상의 실제 거리와는 오차가 있습니다.)
    
    ```sql
    # 평면 좌표계
    SELECT ST_Distance(
    		ST_PointFromText('POINT(0 0)'),
    		ST_PointFromText('POINT(5 5)')) AS distance;
    +--------------------+
    | distance           |
    +--------------------+
    | 7.0710678118654755 |
    +--------------------+
    
    # 투영 좌표계
    SELECT ST_Distance(
    		ST_GeomFromText('POINT(14133790.0 4509380.0)', 3857),
    		ST_GeomFromText('POINT(14133795.0 4509385.0)', 3857)) AS distance;
    +--------------------+
    | distance           |
    +--------------------+
    | 7.0710678118654755 |
    +--------------------+
    ```
    
    - 두 점 간의 거리를 계산하기 위해 `ST_Distance_Sphere()` 함수를 사용하면 되지만, SRID=4326과 같은 지리 좌표에서만 사용할 수 있기에 평면, 투영 좌표계에서 거리 계산에선 사용할 수 없습니다.
    - 투영 좌표계는 지도, 화면에 지도를 표현하는 경우 자주 사용됩니다.
        - 평면, 투영 모두 동일한 함수들이 사용됩니다.
        - 차이점은 단순히 SRID의 다름과 함수에 사용되는 파라미터가 X, Y축 좌푯값인지 위, 경도 좌푯값인지에 대한 차이만 있습니다.

---

<br><br>

### 12.2.4 지리 좌표계

- 학습 목표
    - 지리 좌표계, 투영 좌표계를 사용하는 공간 데이터의 저장 방법
    - mysql 지리 좌표계에서 주의해야 할 사항

<br>

**12.2.4.1 지리 좌표계 데이터 관리**

- 공간 인덱스(Spatial Index)를 생성하는 칼럼은 반드시 NOT NULL이어야 함
- GPS로 부터 받는 위치 정보를 저장하기 위해 WGS 84(SRID=4326)로 칼럼 정의
    
    ```sql
    CREATE TABLE sphere_coord (
    		id INT NOT NULL AUTO_INCREMENT,
    		name VARCHAR(20),
    		location POINT NOT NULL SRID 4326, -- // WGS84 좌표계
    		PRIMARY KEY (id),
    		SPATIAL INDEX sx_location(location)
    );
    
    INSERT INTO sphere_coord VALUES
    		(NULL, '서울숲',     ST_PointFromText('POINT(37.544738 127.039074)', 4326)),
    		(NULL, '한양대학교', ST_PointFromText('POINT(37.555363 127.044521)', 4326)),
    		(NULL, '덕수궁',     ST_PointFromText('POINT(37.565922 126.975164)', 4326)),
    		(NULL, '남산',       ST_PointFromText('POINT(37.548930 126.993945)', 4326));
    ```
    
- 2호선 뚝섬역 반경 1km 이내에 있는 레코드를 검색하는 쿼리(`ST_Distance_Shpere()` 함수)
    
    ```sql
    # ST_Distance_Sphere() 함수는 두 위치 간의 거리를 meter 단위의 값으로 반환함
    SELECT 
    		id, 
    		name, 
    		ST_AsText(location) AS location,
    		ROUND(ST_Distance_Sphere(location, ST_PointFromText('POINT(37.547027 127.047337)', 4326))) AS distnace_meters
    FROM 
    		sphere_coord
    WHERE
    		ST_Distance_Sphere(location, ST_PointFromText('POINT(37.547027 127.047337)', 4326)) < 1000;
    		
    +----+-----------------+-----------------------------+-----------------+
    | id | name            | location                    | distnace_meters |
    +----+-----------------+-----------------------------+-----------------+
    |  1 | 서울숲            | POINT(37.544738 127.039074) |             772 |
    |  2 | 한양대학교         | POINT(37.555363 127.044521) |             960 |
    +----+-----------------+-----------------------------+-----------------+
    
    +----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
    | id | select_type | table        | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
    +----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
    |  1 | SIMPLE      | sphere_coord | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using where |
    +----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
    ```
    
    - 2건의 레코드 검색을 위해 풀 테이블 스캔을 이용해 2건을 검색함
    - mysql 서버는 아직 인덱스를 이용한 반경 검색 기능(함수)이 없음 (`ST_Distance_Sphere()` 함수)
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5cc7c61b-0ea2-4858-9dd1-ed9e8f96b2a8)

        
- 차선책으로 MBR(Minimum Bounding Rectangle)을 이용한 ST_Within() 함수를 사용할 수 있습니다.
    - 2개의 공간 데이터를 파라미터로 입력, 첫 번째가 두 번째 파라미터의 공간 데이터에 포함되는지를 체크하는 함수 입니다.
    - 공간 좌표계 단위는 각도임, 1km의 거리가 각도(Degree)로 얼마인지 계산해야 하는데 지구는 구면체이므로 위도에 따라 경도 1도에 해당되는 거리가 달라집니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/277c4beb-f44d-43d0-b374-6b1af0645538)

        위도 0도에서 지구의 반지름이 가장 크므로 경도 1도에 해당되는 거리가 가장 길고, 위도가 커질수록 점점 짧아집니다. (반면에 경도가 몇도든 위도1도에 해당되는 거리는 동일합니다)
        
    - 위도에 따라 경도 1도에 해당하는 거리 변화를 고려한 상하좌우 1km에 해당하는 위도 및 경도 좌표 계산방법
        
        ```sql
        # TopRight: 기준점의 북동쪽(우측 상단) 끝 좌표
        Longitude_TopRight = Longitude_Origin + (${DistanceKm}/abs(cos(radians(${Latitude_Origin}))*111.32))
        Latitude_TopRight = Longitude_Origin + (${DistanceKm}/111.32)
        
        # BottomLeft: 기준점의 남서쪽(좌측 하단) 끝 좌표
        Longitude_BottomLeft = Longitude_Origin - (${DistanceKm}/abs(cos(radians(${Latitude_Origin}))*111.32))
        Latitude_BottomLeft = Longitude_Origin - (${DistanceKm}/111.32)
        ```
        
        ```sql
        # 1km 반경의 원을 외접하는 직사각형(MBR) 폴리곤 객체를 반환하는 함수
        DELIMITER ;;
        
        CREATE DEFINER='root'@'localhost'
        		FUNCTION getDistanceMBR(p_origin POINT, p_distanceKm DOUBLE) RETURNS POLYGON
        		DETERMINISTIC
        				SQL SECURITY INVOKER
        		BEGIN
        				DECLARE v_originLat DOUBLE DEFAULT 0.0;
        				DECLARE v_originLon DOUBLE DEFAULT 0.0;
        
        				DECLARE v_deltaLon DOUBLE DEFAULT 0.0;
        				DECLARE v_Lat1 DOUBLE DEFAULT 0.0;
        				DECLARE v_Lon1 DOUBLE DEFAULT 0.0;
        				DECLARE v_Lat2 DOUBLE DEFAULT 0.0;
        				DECLARE v_Lon2 DOUBLE DEFAULT 0.0;
        					         
        				SET v_originLat = ST_X(p_origin); /* = ST_Latitude(p_origin) for SRID=4326*/
        				SET v_originLon = ST_Y(p_origin); /* = ST_Longitude(p_origin) for SRID=4326 */
        
        				SET v_deltaLon = p_distanceKm / ABS(COS(RADIANS(v_originLat))*111.32);
        				SET v_Lon1 = v_originLon - v_deltaLon;
        				SET v_Lon2 = v_originLon + v_deltaLon;
        				SET v_Lat1 = v_originLat - (p_distanceKm / 111.32);
        				SET v_Lat2 = v_originLat + (p_distanceKm / 111.32);
        				
        				SET @mbr = ST_AsText(ST_Envelope(ST_GeomFromText(CONCAT("LINESTRING(", v_Lat1, " ", v_Lon1,", ", v_Lat2, " ", v_Lon2,")"))));
        				RETURN ST_PolygonFromText(@mbr, ST_SRID(p_origin));
        END ;;
        ```
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/528bdb26-40df-476b-9e5c-2e9bd3daf6fa)

        
        - `getDistanceMBR()`
            
             첫 번째 파라미터로 주어진 중심위치(POINT객체) 로부터 두 번째 파라미터에 주어진 km반경의 원을 감싸는 직사각형 POLYGON 객체를 반환함
            
            반환되는 폴리곤 객체의 SRID는 첫 번째 파라미터의 POINT 객체의 SRID를 그대로 계승함
            
    - `뚝섬역을 중심으로 반경 1km 원을 둘러싸는 MBR(직사각형) 조회`
        
        ```sql
        SET @distanceMBR = getDistanceMBR(
                ST_GeomFromText('POINT(37.547027 127.047337)', 4326), 1
        );
        
        SELECT ST_SRID(@distanceMBR), ST_AsText(@distanceMBR) \G
        
        *************************** 1. row ***************************
          ST_SRID(@distanceMBR): 4326
        ST_AsText(@distanceMBR): POLYGON((37.53804388825009 127.03600689589169,
        																	37.55601011174991 127.03600689589169,
        																	37.55601011174991 127.05866710410828,
        																	37.53804388825009 127.05866710410828,
        																	37.53804388825009 127.03600689589169))
        																	
        # ST_Buffer()라는 함수로 손쉽게 특정 위치에서 1km의 점들을 모아 다각형을 만들지만 평면 좌표계에서만 사용할 수 있음
        ```
        
    - `위 과정을 함축하여 실제 반경 검색을 실행하는 쿼리` 간단간단..
        
        ```sql
        SELECT id, name
        FROM sphere_coord
        WHERE ST_Contains(
        		getDistanceMBR(ST_PointFromText('POINT(37.547027 127.047337)', 4326), 1),
        		location
        );
        +----+-----------------+
        | id | name            |
        +----+-----------------+
        |  2 | 한양대학교         |
        |  1 | 서울숲            |
        +----+-----------------+
        
        # ST_Within은 위 함수와 파라미터를 반대로 입력
        SELECT id, name
        FROM sphere_coord
        WHERE ST_Within(location, getDistanceMBR(ST_PointFromText('POINT(37.547027 127.047337)', 4326), 1));
        +----+-----------------+
        | id | name            |
        +----+-----------------+
        |  2 | 한양대학교         |
        |  1 | 서울숲            |
        +----+-----------------+
        
        # 위 두 쿼리의 실행 계획 -> 인덱스 레인지 스캔
        +----+-------------+--------------+------------+-------+---------------+-------------+---------+------+------+----------+-------------+
        | id | select_type | table        | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra       |
        +----+-------------+--------------+------------+-------+---------------+-------------+---------+------+------+----------+-------------+
        |  1 | SIMPLE      | sphere_coord | NULL       | range | sx_location   | sx_location | 34      | NULL |    4 |   100.00 | Using where |
        +----+-------------+--------------+------------+-------+---------------+-------------+---------+------+------+----------+-------------+
        ```
        
        - 다만 위 과정은 1km 반경의 MBR 사각형 내의 점들을 검색하기에 아래 그림과 같이 모서리 부분은 1km를 넘기도 합니다. 따라서 두 도형의 교집합을 제외한 부분만 남기기 위해 8, 16각형으로 생성하면 더 효율적입니다.
            
            ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b00f1215-bd1b-488f-a74c-27328b552703)

            
        - 하지만, 가장 간단한 방법은 한번 더 거리 계산 조건을 적용하는 것입니다.
            
            ```sql
            SELECT id, name 
            FROM sphere_coord
            WHERE ST_Within(location, getDistanceMBR(ST_PointFromText('POINT(37.547027 127.047337)', 4326), 1)) 
            		AND ST_Distance_Sphere(location, ST_PointFromText('POINT(37.547027 127.047337)', 4326)) <= 1000;
            		
            +----+-----------------+
            | id | name            |
            +----+-----------------+
            |  2 | 한양대학교      |
            |  1 | 서울숲          |
            +----+-----------------+
            ```
            
            공간 인덱스를 이용해 ST_Within()을 실행하고, 소량의 결과에 대해 ST_Distance_Sphere() 함수 조건을 비교해 최종 결과를 반환하므로 쿼리의 성능이 크게 떨어지진 않습니다.
            

---

<br>

**12.2.4.2 지리 좌표계 주의 사항**

mysql에서 GIS기능은 도입된지 오래지만, 지리 좌표계나 SRS 관리 기능이 도입된건 8.0이 처음입니다.
따라서 데이터 지리 좌표계 데이터 검색 및 변환 기능의 성능은 미숙한 부분이 보입니다.

기능적으로 성숙하지 않았기에 사용하고자 한다면 정확성, 성능에 대해 주의가 필요합니다.

1. `정확성 주의 사항`
    
    SRID=4326 에서 ST_Contains()함수가 정확하지 않은 결과를 반환함
    
    ```sql
    # 지리 좌표계 SRID=4326으로 ST_Contains() 함수 기능 테스트
    SET @SRID := 4326 /* WGS84 */;
    
    # 경도는 서울 부근, 위도는 북극에 가까운 지점인 POINT객체
    SET @point := ST_GeomFromText('POINT(84.50249779176816 126.96600537699246)', @SRID);
    
    # 경기도 부근의 작은 영역으로 POLYGON 객체 생성
    	SET @polygon := ST_GeomFromText('POLYGON((
    			37.399190586836184 126.965669993654370,
    			37.398563808320795 126.966005172597650,
    			37.398563484045880 126.966005467513980,
    			37.398563277873926 126.966005991634870,
    			37.398563316804754 126.966006603730260,
    			37.398687101343120 126.966426067601090,
    			37.398687548434204 126.966426689671760,
    			37.398687946717580 126.966426805810510,
    			37.398688339525570 126.966426692691090,
    			37.399344708725290 126.966027255005050,
    			37.399344924786035 126.966026657464910,
    			37.399344862437594 126.966026073608450,
    			37.399285401380960 126.965848242476900,
    			37.399284911074280 126.965847737879930,
    			37.399284239833094 126.965847784502270,
    			37.399260339325880 126.965861778579070,
    			37.399191408765940 126.965670657332960,
    			37.399191087858530 126.965670183150050,
    			37.399190586836184 126.965669993654370))', @SRID);
    			
    # 북극의 특정 지점이 경기도 지역 POYLGON에 포함되는지 확인하는 쿼리
    SELECT ST_Contains(@polygon, @point) AS within_polygon;
    +----------------+
    | within_polygon |
    +----------------+
    |              1 |
    +----------------+
    ```
    
    북극에 가까운 지점이 경기도의 특정 지역에 포함되는 것으로 결과를 반환합니다. 이는 간단하게 거리 비교 조건을 추가하면 문제를 회피할 수 있습니다.
    
    ```sql
    # POLYGON의 중심 지점에 대한 POINT 객체 생성
    SET @center:= ST_GeomFromText('POINT(37.398899 126.966064)', @SRID);
    
    # POLYGON의 중심 지점으로부터 100 미터 이내 거리의 위치인지 같이 확인
    # 100미터는 단순 예시이며, 서비스의 요건에 맞게 적절한 거리를 사용할 것
    SELECT (ST_Contains(@polygon, @point)
                      AND ST_Distance_Sphere(@point, @center)<=100) AS within_polygon;
                      
    +----------------+
    | within_polygon |
    +----------------+
    |              0 |
    +----------------+
    ```
    
    ST_Distance_Sphere()는 공간 인덱스를 활용하지 못하므로 단독 사용은 권장하지 않습니다.
    

- 추가적으로 공간 인덱스를 사용할 수 있는지 못하는지의 여부에 따라 다른 결과를 반환할 수 있으므로 꼭 테스트후 사용해야 합니다. (253page 주의 사항 참조)

1. `성능 주의 사항`
    
    일반적으로 지리 좌표계 데이터의 경우 SRID=4326을 많이 사용합니다. 다만, ST_Contains() 함수 등으로 포함 관계를 비교하는 경우 투영 좌표계보다 느린 성능을 보입니다.
    
    (자세한 테스트 과정은 254 ~ 256page 참조)
    
    - SRID=4326(지면 좌표계)
        - 140개 정도의 점으로 구성된 POLYGON 및 POINT 객체 준비
        - BENCHMARK 함수를 통해 SRID=4326인 POINT 객체 및 POLYGON 객체의 포함 관계를 비교하는 ST_Contains() 함수를 100만번 실행
        - 약 41.44초 소요
    - 동일 데이터 SRID=3857(투영 좌표계)
        - 총 소요시간 13.09초 (3배 빠름)
    
    위 테스트에서 개당 소요시간은 각각 41, 13밀리초 이지만 매우 많은 레코드에 대해서는 큰 성능차이가 발생합니다.
    
2. `좌표계 변환`
    
    일반적인 휴대폰 GPS 좌표는 WGS 84이고 SRID=4326인 지리 좌표계로 표현됨
    SRID=3857은 WGS 84좌표를 평면으로 투영한 좌표 시스템이므로 4326 ↔ 3857간 상호 변환이 가능합니다.
    
    mysql 서버도 ST_Transform() 함수를 제공하여 좌푯값을 변환할 수 있지만, SRID 3857에 대한 변환은 지원하지 않습니다. 
    **(3857은 투영 좌표계이므로 거리 계산시 실제 구면체에서의 거리와는 큰 오차를 보임)**
    
    ```sql
    # 지리 좌표계(SRID=4326)에서의 거리 계산
    SET @p_seoul := ST_GeomFromText('POINT(37.566253 126.977999)', 4326);
    SET @p_busan := ST_GeomFromText('POINT(35.179766 129.075139)', 4326);
    SELECT ST_Distance_Sphere(@p_seoul, @p_busan) AS distance_as_meters;
    +--------------------+
    | distance_as_meters |
    +--------------------+
    | 325050.50026583584 |
    +~~--------------------+~~
    
    # 투영 좌표계(SRID=3857)에서의 거리 계산
    SET @p_seoul := ST_GeomFromText('POINT(14135126.188661173 4518331.820651973)', 3857);
    SET @p_busan := ST_GeomFromText('POINT(14368578.745550888 4188337.539032124)', 3857);
    SELECT ST_Distance(@p_seoul, @p_busan) AS distance_as_meters;
    +--------------------+
    | distance_as_meters |
    +--------------------+
    | 404223.10945831076 |
    +--------------------+
    ```
    
    - 실제 거리는 4326의 결과와 가깝습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/5c58a276-ce64-4084-a256-663d67b5edb0)

        
    
    - 3857은 ST_Conatins() 처리가 빠르고 정확한 결과를 보여주긴 하지만 투영 좌표계이므로 원하는 값이 아닐 수 있습니다.
    - 따라서 3857을 사용하는 경우 4326으로 변환하거나 구면체 기반의 거리 계산이 필요할때는 여러 함수들을 사용하면 됩니다.
    - 자세한 예시는 258 ~ 261page참조
