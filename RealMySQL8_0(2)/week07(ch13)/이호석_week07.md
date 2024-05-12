# 📌 파티션

- `파티션 기능`
    
    논리적으로는 하나의 테이블이지만 물리적으로 여러 개의 테이블로 분리하여 관리할 수 있게 해줌
    
- `목적`
    
    대용량의 테이블을 물리적으로 여러 개의 소규모 테이블로 분산하는 목적
    
- `주의`
    
    어떤 쿼리를 사용하느냐에 따라 오히려 파티션에서의 쿼리가 성능이 나빠지는 경우도 있음
    

## ✅ 13.1 개요

- 파티션이 적용된 테이블에서 insert, select와 같은 쿼리가 어떻게 실행되는지 이해하는게 최적의 사용방식을 이해하는데 도움이 됨

### 13.1.1 파티션을 사용하는 이유

- 테이블의 데이터가 많아진다고 해도, 파티션을 적용하는게 무조건 효율적이지 않음
- 파티션이 필요한 대표적 예
    - 인덱스크기가 커 물리적인 메모리보다 훨씬 큰 경우
    - 데이터 특성상 주기적 삭제 작업이 필요한 경우

---

**13.1.1.1 단일 INSERT와 단일 또는 범위 SELECT의 빠른처리**

- 인덱스의 크기가 커질수록 DDL작업이 전체적으로 느려지는 단점이 있습니다.
    - 특히 한 테이블의 인덱스 크기가 물리적으로 mysql이 사용 가능한 메모리 공간보다 크면 더 심각함
- 인덱스의 Working Set이 실질적인 메모리보다 크면 쿼리 처리가 상당히 느려짐(Disk IO)
- 파티션을 나누어 저장할 경우 인덱스의 Working Set을 좀 더 효율적으로 사용할 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/20a80711-e373-459e-bcde-83eb8e5001ca)

    
- `Working Set`
    - 활발하게 사용되는 데이터를 말하며, 100%의 데이터 중 최신 20 ~ 30%의 데이터를 의미합니다.
    - 워킹 셋의 데이터와 그렇지 않은 부분을 파티션할 수 있다면 효과적인 성능 개선을 기대할 수 있습니다.

---

**13.1.1.2 데이터의 물리적인 저장소를 분리**

- 파티션을 통해 파일의 크기를 조절하거나 파티션별 파일들이 저장될 위치나 디스크를 구분해서 지정해 너무 용량이 큰 데이터 및 인덱스 파일의 관리 문제를 해결할 수 있습니다.
- 하지만, **mysql은 테이블의 파티션 단위로 인덱스를 생성하거나 파티션별로 인덱스를 가지는 형태는 지원**하지 않습니다.

---

**13.1.1.3 이력 데이터의 효율적인 관리**

- 로그 데이터는 단기간 대량 누적, 단기간에 금방 쓸모가 없어집니다. (너무 많이 쌓이게 되므로)
- 따라서 시간이 지나면 별도 아카이빙 혹은 삭제를 하는데 이는 고부하의 작업에 속합니다.
- 이를 파티션 테이블로 관리해 파티션 추가, 삭제로 간단하고 빠르게 해결할 수 있습니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/c144a9eb-466a-4a8c-8033-3dc5355043e2)

    
    - 파티션이 아닌 기간 단위로 로그 테이블을 삭제하면 부하 + 로그 테이블 동시성 문제가 발생할 수 있는데 파티션으로 문제를 대폭 줄일 수 있습니다.

---

### 13.1.2 MySQL 파티션의 내부 처리

```sql
CREATE TABLE tb_article (
		article_id INT NOT NULL,
		reg_date DATETIME NOT NULL,
		...
		PRIMARY KEY(article_id, reg_date)
) PARTITION BY RANGE ( YEAR(reg_date) ) (
		PARTITION p2009 VALUES LESS THAN (2010),
		PARTITION p2010 VALUES LESS THAN (2011),
		PARTITION p2011 VALUES LESS THAN (2012),
		PARTITION p9999 VALUES LESS THAN MAXVALUE
);
```

- 게시글 등록일자(reg_date)의 연도 부분은 `파티션 키`, 어느 파티션에 저장될지 결정합니다.
- 위 테이블에서의 INSERT, UPDATE, SELECT 같은 쿼리가 처리되는 방식을 설명하면

---

**13.1.2.1 파티션 테이블의 레코드 INSERT**

- mysql 서버는 INSERT되는 칼럼의 값 중에서 파티션 키인 reg_date 칼럼의 값을 이용해 파티션 표현식을 평가하고, 결과로 레코드가 저장될 적절한 파티션을 결정합니다.
- INSERT되는 레코드를 위한 파티션이 결정되면 나머지 과정은 일반 테이블과 동일하게 처리됩니다.
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/235941ab-1fdb-4383-ba86-71e5f15ad4b8)

    

---

**13.1.2.2 파티션 테이블의 UPDATE**

- `검색`: WHERE 조건에 파티션 키 칼럼이 조건 존재 여부
    - 존재
        - 저장된 파티션에서 빠르게 대상 레코드를 검색할 수 있음
    - 존재X
        - 모든 파티션을 검색해야 함
- `수정`: UPDATE 쿼리가 어떤 칼럼의 값을 변경하는지 여부
    - 파티션 키 제외 다른컬럼
        - 일반 테이블과 마찬가지로 칼럼값을 변경함
    - 파티션 키
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b9b6743b-67b3-49e9-9bd5-4856964a8323)

        
        - 해당하는 파티션 기존 레코드를 삭제
        - 변경되는 파티션 키 칼럼 표현식 평가
        - 평가 결과를 통해 파티션을 결정해 레코드를 새로 저장

---

**13.1.2.3 파티션 테이블의 검색**

- 파티션 테이블 검색시 성능에 큰 영향을 미치는 조건
    1. WHERE 절 조건으로 검색해야 할 파티션 선택 가능?
    2. WHERE 절 조건이 인덱스를 효율적으로 사용 가능?(인덱스 레인지 스캔)
    (일반 테이블 검색에서도 중요한 사항)
- 파티션 테이블은 1번 결과에 의해 2번 작업 내용이 달라질 수 있음
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/ec278b75-4731-4b3c-a799-0092d6d6a232)

    
    - 3, 4번 방식은 가능하면 피하는게 좋고, 두번째 조합도 테이블에 파티션 개수가 많으면 부하가 높아질 수 있으니 주의해야 합니다.

---

**13.1.2.4 파티션 테이블의 인덱스 스캔과 정렬**

- mysql 파티션 테이블에서 인덱스는 전부 파티션 단위의 로컬 인덱스에 해당됩니다.
(동일한 인덱스가 파티션 단위로 생성, 다른 인덱스가 아님)
- 글로벌 인덱스는 지원X

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/0f9c5779-7d89-4d0f-9159-642cd1b981db)


- 테이블의 reg_userid로 구성된 인덱스가 연도별로 파티션 되어 정렬되고 각 파티션의 순서대로는 reg_userid가 정렬되진 않습니다.
- 인덱스 레인지 스캔시 2개이상의 파티션을 읽어야 할때 reg_userid 기준으로 정렬될것으로 예측되지만 그렇지 않습니다.

```sql
// 2009, 2010 두 개의 파티션에 대해 조회
EXPLAIN
SELECT *
FROM tb_article
WHERE reg_userid BETWEEN 'brew' AND 'toto'
		AND reg_date BETWEEN '2009-01-01' AND '2010-12-31'
ORDER BY reg_userid;

+----+------------+-------------+-------+--------------+--------------------------+
| id | table      | partitions  | type  | key          | Extra                    |
+----+------------+-------------+-------+--------------+--------------------------+
|  1 | tb_article | p2009,p2010 | range | ix_reguserid | Using where; Using index |
+----+------------+-------------+-------+--------------+--------------------------+

```

- Using filesort가 아니라 인덱스 레인지 스캔으로 처리함을 알 수 있습니다.
- 이는 각 파티션으로부터 조건에 일치하는 레코드를 정렬된 순서대로 읽으며 우선순위 큐에 임시 저장합니다.
    - 이후 우선순위큐가 인덱스 순서대로 데이터를 정렬합니다.
- 별도의 정렬은 하지 않지만 큐 처리가 한 번 필요하며 13.6 그림에서 Merge & Sort라고 표시한 부분이 우선순위 큐 작업을 의미합니다.

---

**13.1.2.5 파티션 프루닝**

- 옵티마이저에 의해 일부의 파티션만 읽어도 된다고 판단되면 불필요한 파티션에는 접근하지 않습니다.
- **최적화 단계에서 필요한 파티션만 고르고 불필요한 파티션은 배제하는걸 파티션 프루닝 이라고 합니다.**
    - 이는 실행계획에서 partitions 칼럼에서 확인해볼 수 있습니다.

---

## ✅ 13.2 주의사항

- 파티션은 5.1 ~ 8.0을 지나오며 많은 발전이 있었지만, 아직도 많은 제약을 가지고 지닙니다.
- 살펴보는 제약은 대부분 태생적인 한계이므로 업그레이드가 되어도 여전히 가질 제약사항일 수 있습니다.

### 13.2.1 파티션의 제약 사항

```sql
CREATE TABLE tb_article (
		article_id INT NOT NULL AUTO_INCREMENT,
		reg_date DATETIME NOT NULL,
		...
		PRIMARY KEY(article_id, reg_date)
) PARTITION BY RANGE ( YEAR(reg_date) ) (
		PARTITION p2009 VALUES LESS THAN (2010),
		PARTITION p2010 VALUES LESS THAN (2011),
		PARTITION p2011 VALUES LESS THAN (2012),
		PARTITION p9999 VALUES LESS THAN MAXVALUE
);
```

- PARTIION BY RANGE
    - 레인지 파티션을 사용함을 의미, reg_date에서 연도별로 파티셔닝이 됩니다.

- MySQL 서버의 파티션이 가지는 제약 사항
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f7742655-4b76-4a0a-9ec5-eb2a3cbad792)

    
    - 가장 크게 영향을 주는 제약은 모든 유니크 인덱스에 파티션 키 칼럼이 포함돼야 하는것
    - 파티션 표현식에는 기본적인 산술 연산자인 (+, -, *)같은 연산자 사용이 불가능하고, 다음과 같은 내장함수를 사용할 수 있습니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/423a6abc-53c6-4059-9a21-f1f8f1bfe123)

        
    - 모든 내장함수가 파티션 프루닝을 지원하지 않으므로 버전에 맞는 메뉴얼을 확인해봐야 합니다.
    https://dev.mysql.com/doc/refman/8.0/en/partitioning-limitations-functions.html

---

### 13.2.2 파티션 사용 시 주의사항

- 파티션의 목적이 작업의 범위를 좁히는것인데 유니크 인덱스는 중복 레코드에 대한 체크 작업 때문에 범위가 좁혀지지 않습니다.
    - insert시 중복 체크를 위해 모든 파티션을 검사해야 함
- mysql 파티션은 일반 테이블과 같이 별도의 파일로 관리되는데, mysql 서버가 조작할 수 있는 파일 개수와 연관된 제약도 존재

**13.2.2.1 파티션과 유니크 키(프라이머리 키)**

- 종류와 관계없이 테이블에 유니크 인덱스(PK 포함)가 있으면 파티션 키는 모든 유니크 인덱스의 일부 또는 칼럼을 포함해야 합니다.
- 각 유니크 키에 대해 값이 주어졌을때 해당 레코드가 어느 파티션에 저장돼 있는지 계산할 수 있다면 올바른 테이블 파티션 생성이 된 것 입니다.
- 올바르지 못한 생성
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/570ca95b-b93c-4784-a857-5a5abd015b65)

    
    - 첫 번째 테이블
        - fd1 혹은 fd2 하나만으로 어떠 파티션에 속해야할지 모름
    - 두 번째 테이블
        - fd1, fd2, fd3 각가으로 어떤 파티션인지 알 수 없음
- 올바른 생성

---

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/b12128b9-c585-4406-891a-41586a47f580)


**13.2.2.2 파티션과 open_files_limit 시스템 변수 설정**

- mysql은 일반적으로 테이블을 파일단위로 관리함 따라서 동시에 오픈된 파일의 개수가 많아질 수 있습니다.
- 이를 제한하려면 open_files_limit 시스템 변수에 동시에 오픈할 수 있는 파일수를 설정할 수 있습니다.
- 일반적으로 파티션되지 않은 `일반테이블은 1개당 2~3개의 파일이 오픈`되지만 `파티션 테이블은 파티션의 개수 * 2 ~ 3개`가 됩니다.
    - 따라서 파티션을 많이 사용한다면 시스템 변수 값을 높게 설정 할 필요가 있습니다.

---

## ✅ MySQL 파티션의 종류

- 4가지의 기본 파티션 기법을 제공함
    - 레인지 파티션
    - 리스트 파티션
    - 해시 파티션
    - 키 파티션
- 해시, 키 파티션에 대해선 Linear 파티션과 같은 추가적인 기법도 제공합니다.

### 13.3.1 레인지 파티션

- 파티션 키의 연속된 범위로 파티션을 정의하는 방법, 가장 일반적으로 사용됨
- 다른 파티션 방법과는 달리 MAXVALUE라는 키워드를 이용해 명시되지 않은 범위의 키 값이 담긴 레코드를 저장하는 파티션을 정의할 수 있습니다.

**13.3.1.1 레인지 파티션의 용도**

- 아래 테이블들은 레인지 파티션을 이용하는게 좋습니다.
    - 날짜를 기반으로 데이터가 누적되고 연도, 월, 일 단위로 분석하고 삭제해야 할 때
    - 범위 기반으로 데이터를 여러 파티션에 균등하게 나눌 수 있을때
    - 파티션 키 위주로 검색이 자주 실행될때
    (모든 파티션에 일반적으로 적용되지만 레인지, 리스트 파티션에 더 필요한 요건)
- 파티션의 장점
    1. 큰 테이블을 작은 크기의 파티션으로 분리
    2. 필요한 파티션만 접근(쓰기, 읽기 모두) → `이 장점의 효과가 매우 큼`

2번이 아니라 1번의 장점에 집중하면 소탐대실의 결과로 귀결되어 파티션 때문에 mysql 서버의 성능을 더 떨어트릴 수 있습니다.

두 가지 이점을 모두 챙기긴 어려워도 레인지 파티션은 어렵지 않게 두 가지 장점을 취할 수 있습니다.

`레인지 파티션은 주로 로그 테이블에 레인지 파티션을 적용한 경우가 대부분이었습니다.`

---

**13.3.1.2 레인지 파티션 테이블 생성**

- 사원의 입사 연도별로 파티션 테이블 만들기
    
    ```sql
    CREATE TABLE employees (
    		id INT NOT NULL,
    		first_name VARCHAR(30),
    		last_name VARCHAR(30),
    		hired DATE NOT NULL DEFAULT '1970-01-01',
    		...
    ) PARTITION BY RANGE( YEAR(hired) ) (
    		PARTITION p0 VALUES LESS THAN (1991),
    		PARTITION p1 VALUES LESS THAN (1996),
    		PARTITION p2 VALUES LESS THAN (2001),
    		PARTITION p3 VALUES LESS THAN MAXVALUE);
    ```
    
    - `PARTITION BY RANGE`: 레인지 파티션 정의
    - `YEAR(hired)`: 칼럼 또는 내장 함수를 이용해 파티션 키 명시, 사원의 입사 연도
    - `VALUES LESS THAN`: 명시된 값 보다 작은 값만 해당 파티션에 저장 LESS THAN에 명시된 값은 그 파티션에 포함되지 않음
    - `VALUES LESS THAN MAXVALUE`: 명시되지 않은 레코드를 저장할 파티션 결정 예제에선 2001 ~ 9999년 사이 입사한 사원 정보가 저장됨(선택사항)
        - MAXVALUE가 없으면 2001 이상의 값을 가진 레코드가 insert되면 `Table has no partition for value 20xx` 라는 메시지 표시됨
    - 테이블과 각 파티션은 같은 스토리지 엔진으로 정의할 수 있음, 기본은 InnoDB

![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/94b38bcb-c32b-4b90-9df7-f69ec3d17181)


---

**13.3.1.3 레인지 파티션의 분리와 병합**

1. `단순 파티션의 추가`
    - 기본 쿼리
        
        ```sql
        ALTER TABLE employees 
        		ADD PARTITION (PARTITION p4 VALUES LESS THAN (2011));
        ```
        
    - LESS TAHN MAXVALUE가 있을때 파티션 추가 쿼리
        
        ```sql
        ALTER TABLE employees ALGORITHM=INPLACE, LOCK=SHARED,
        REORGANIZE PARTITION p3 INTO (
        		PARTITION p3 VALUES LESS THAN (2011),
        		PARTITION p4 VALUES LESS THAN MAXVALUE
        );
        ```
        
        - MAXVALUE가 있는 상황에서 2011이 추가되면 2011년 레코드는 2개의 파티션에 추가되어야 함 → 하나의 레코드는 하나의 파티션에만 속해야하는 기본 조건을 어김
        - 따라서 위와 같이 수정해주어야 합니다.
        - ALTER TABLE .. REORGANIZE PARTITION 명령은 p3 파티션의 레코드를 모두 2개의 파티션으로 복사해야 하기에 레코드가 많다면 오랜 시간이 걸립니다.
        - **일반적으로 레인지 파티션은 MAXVALUE보다는 미래에 사용될 것으로 예상될 파티션 2~3개를 더 만들어두는 형태로 구성합니다.**
        - 혹은 배치 스크립트를 통해 파티션 테이블의 여유 공간을 확인해 자동으로 파티션을 추가하는 방법도 있습니다.

1. `파티션 삭제`
    - 파티션 테이블에서 특정 파티션을 삭제하는 작업은 아주 빠르게 처리됩니다.
    - 날짜 단위로 파티셔닝된 테이블에서 오래된 데이터를 삭제하는 용도로 많이 사용됩니다.
    
    ```sql
    ALTER TABLE employees DROP PARTITION P0;
    ```
    
    - **레인지 파티션에선 파티션 삭제시 항상 가장 오래된 파티션 순서로만 삭제할 수 있습니다.**
    - **또한 가장 마지막 파티션만 새로 추가할 수 있습니다.**

1. `기존 파티션의 분리`
    - `REORGANIZE PARTITION` 명령을 사용하여 두 개 이상의 파티션으로 분리함
    
    ```sql
    ALTER TABLE employees ALGORITHM=INPLACE, LOCK=SHARED,
    		REORGANIZE PARTITION p3 INTO (
    				PARTITION p3 VALUES LESS THAN (2011),
    				PARTITION p4 VALUES LESS THAN MAXVALUE
    );
    ```
    
    - 기존 p3파티션을
        - 2001년부터 2010년 까지 입사한 사원을 p3파티션에
        - 2011년 이후에 입사한 사원을 p4 파티션에 분리하여 저장합니다.
    - MAXVALUE 파티션 뿐만 아니라 다른 파티션들도 REORGANIZE PARTITION 명령을 이용해 분리 가능
    - 레코드가 많으면 오래걸릴 수 있으므로 온라인 DDL로 실행할 수 있게 ALGORITHM, LOCK 절을 사용하는게 좋습니다.
        - 파티션 재구성은 INPLACE는 가능하지만 최소한 읽기 잠금(Shared Lock)이 필요합니다. (테이블 쓰기 불가능)
        - 따라서 쿼리 처리가 많지 않은 시간대에 진행하는게 좋습니다.

1. `기존 파티션의 병합`
    - 역시 `REORGANIZE PARTITION` 명령으로 처리할 수 있습니다.
    
    ```sql
    ALTER TABLE employees ALGORITHM=INPLACE, LOCK=SHARED,
    		REORGANIZE PARTITION p2, P3 INTO (
    				PARTITION p23 VALUES LESS THAN (2011)
    );
    ```
    
    - 위 쿼리는 p2, p3파티션을 p23 파티션으로 병합합니다.
    - 파티션 병합도 파티션 재구성이 필요하고 역시 INPLACE 및 Shared Lock이 필요합니다.

### 13.3.2 리스트 파티션

- 레인지 파티션은 키 값의 범위로 파티션을 구성하지만 리스트 파티션은 키 값 하나하나를 리스트로 나열해야 합니다. 따라서 MAXVALUE 파티션을 정의할 수 없습니다.

**13.3.2.1 리스트 파티션의 용도**

- 테이블이 다음과 같은 특성을 가지면 리스트 파티션 사용을 권장합니다.
    - 파티션 키 값이 `코드 값`이나 `카테고리`와 같이 고정적일때
    - 키 값이 연속되지 않고 `정렬 순서와 관계없이 파티션`을 해야 할 때
    - (공통) 파티션 키 값을 기준으로 레코드의 건수가 균일하고 검색조건에 파티션 키가 자주 사용될때

---

**13.3.2.2 리스트 파티션 테이블 생성**

```sql
CREATE TABLE product(
		id INT NOT NULL,
		name VARCHAR(30),
		category_id INT NOT NULL,
		...
) PARTITION BY LIST( category_id ) ( # 리스트 파티션이라는걸 명시
		# IN(...) 파티션별 저장할 파티션 키 값의 목록 나열
		PARTITION p_appliance VALUES IN (3), 
		PARTITION p_computer VALUES IN (1,9),
		PARTITION p_sports VALUES IN (2,6,7),
		PARTITION p_etc VALUES IN (4,5,8,NULL) # NULL도 명시 가능
);
# MAXVALUE 파티션은 정의 불가

```

- 정수형 및 문자열 타입일때도 리스트 파티션을 사용할 수 있습니다.

---

**13.3.2.3 리스트 파티션의 분리와 병합**

- VALUES LESS THAN이 아닌 VALUES IN을 사용한다는 것 외에는 레인지 파티션의 추가, 삭제, 병합 작업과 동일합니다.
- 특정 파티션 레코드 건수가 많아져 분리하거나 병합하려면 REORGANIZE PARTITION 명령을 사용하면 됩니다.

---

**13.3.2.4 리스트 파티션 주의사항**

- 명시되지 않은 나머지 값을 저장하는 MAXVALUE 파티션 정의 불가
- 레인지 파티션과 달리 NULL을 저장하는 파티션을 별도로 생성할 수 있음

---

### 13.3.3 해시 파티션

- mysql에서 정의한 해시 함수에 의해 레코드가 저장될 파티션을 결정하는 방법
- 복잡한 알고리즘이 아니라, 파티션 표현식의 결괏값을 파티션의 개수로 나눈 나머지로 저장될 파티션을 결정하는 방식임
    - 파티션 키는 `항상 정수 타입 or 정수를 반환하는 표현식`만 사용 가능
- 파티션의 개수는 레코드를 파티션에 할당하는 알고리즘과 연관되어 있기에 파티션 추가, 삭제 작업에는 `테이블 전체적인 레코드 재분배 작업`이 필요합니다.

**13.3.3.1 해시 파티션의 용도**

- 다음과 같은 특성을 지닌 테이블에 적합함
    - 레인지, 리스트로 데이터를 균등하게 나누는 것이 어려울때
    - 테이블의 모든 레코드가 비슷한 사용 빈도를 보이지만, 테이블이 너무 커서 파티션을 적용해야할 때
- 해시, 키 파티션은 대표적으로 회원 테이블을 예로들 수 있음
    - 회원 정보는 가입 일자가 오래됐거나 최신이라고 더 빈번하게 사용되진 않습니다. 또한 다른 정보들도 사용 빈도에 미치는 영향이 없기에 해시, 키 파티션을 적용하기 적합합니다.
    (전체적으로 비슷한 사용 빈도를 보임)

**13.3.3.2 해시 파티션 테이블 생성**

```sql
# 파티션의 개수만 지정할 때
CREATE TABLE employees (
		id INT NOT NULL,
		first_name VARCHAR(30),
		last_name VARCHAR(30),
		hired DATE NOT NULL DEFAULT '1970-01-01',
		...
		# 해시 파티션 및 키 지정, 표현식은 정수타입, 4개의 파티션 지정
) PARTITION BY HASH(id) PARTITIONS 4; 

# 파티션의 이름을 별도 지정
CREATE TABLE employees (
		id INT NOT NULL,
		first_name VARCHAR(30),
		last_name VARCHAR(30),
		hired DATE NOT NULL DEFAULT '1970-01-01',
		...
) PARTITION BY HASH(id)
PARTITIONS 4 (
		# 파티션의 이름 지정p0 ~ p4
		PARTITION p0 ENGINE=INNODB,
		PARTITION p1 ENGINE=INNODB,
		PARTITION p2 ENGINE=INNODB,
		PARTITION p3 ENGINE=INNODB
);

```

- 해시, 키 파티션에서 특정 파티션의 삭제, 병합 작업은 거의 불필요하므로 이름을 부여하는게 큰 의미가 없습니다.
    - 개수만 지정해도 알아서 p0, p1 …으로 이름을 생성함

---

**13.3.3.3 해시 파티션의 분리와 병합**

- 분리, 병합은 대상 테이블의 모든 파티션에 저장된 레코드를 재분배해야 함 (리스트, 레인지와 다름)
- 분리, 병합으로 레코드 수가 달라짐은 알고리즘의 변경을 의미하므로 전체 파티션이 영향을 받음

1. `해시 파티션 추가`
    - 파티션의 개수가 변경되면 MOD 연산의 값의 변경이 발생하므로 데이터 재분배가 필요합니다.
    
    ```sql
    # 파티션 1개 추가, 파티션 이름 부여
    ALTER TABLE employees ALGORITHM=INPLACE, LOCK=SHARED,
        ADD PARTITION(PARTITION p5 ENGINE=INNODB);
        
    # 동시에 6개의 파티션을 별도의 이름 없이 추가하는 경우
    ALTER TABLE employees ALGORITHM=INPLACE, LOCK=SHARED,
        ADD PARTITION PARTITIONS 6;
    ```
    
    - 파티션 추가를 위해 별도의 영역, 범위는 명시하지 않아도 되며, 추가할 개수만 명시하면 됨
    - 파티션에서 발생하는 재분배 작업(INPLACE, 공유잠금)
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/f4f97863-483a-42c7-89bc-c435cb78ae2a)

        

1. `해시 파티션 삭제`
    - 해시, 키 파티션은 파티션 단위로 레코드 삭제 방법이 없음 (에러 발생함)
    - 파티션 키 값을 가공해 데이터를 파티션 별로 분산했으므로 삭제하는 작업은 의미도 없고 해선 안됩니다.

1. `해시 파티션 분할`
    - 분할 안됨, 개수를 늘리는것만 가능함

1. `해시 파티션 병합`
    - 해시, 키 파티션은 2개 이상의 파티션을 하나의 파티션으로 통합하는 기능을 제공하지 않음, 단지 파티션의 개수만 줄일 수 있음
    - 개수를 줄이려면 COALESCE PARTITION 명령을 사용
    - 아래 명령어는 4개의 파티션이 존재한다면 3개의 파티션을 가진 테이블로 재구성함
        
        ```sql
        ALTER TABLE emplyees ALGORITHM=INPLACE, LOCK=SHARD
        		COALESCE PARTITION 1; # 줄이고자 하는 파티션의 개수
        ```
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/a2f0d282-6d08-494c-9e1f-a176765640b4)

        

1. `해시 파티션 주의사항`
    
    ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/151e3601-2bb4-4629-a84c-039ee5e2e3d4)

    

---

### 13.3.4 키 파티션

- 해시 파티션과 사용법, 특성이 거의 같음
- 키 파티션은 해시 값의 계산도 mysql 서버에서 MD5() 함수를 이용해 계산하여 해당 값으로 MOD연산을해 각 파티션으로 분배합니다. (`해시 파티션과의 유일한 차이점`)
- 따라서 정숫값 뿐만 아니라 대부분의 데이터 타입에 대해 파티션 키를 적용할 수 있습니다.

**13.3.4.1 키 파티션의 생성**

```sql
# 프라이머리 키가 있는 경우 자동으로 프라이머리 키가 파티션 키로 사용됨
CREATE TABLE k1 (
		id INT NOT NULL,
		name VARCHAR(20),
		PRIMARY KEY (id)
)
# 괄호의 내용을 비워 두면 자동으로 프라이머리 키의 모든 칼럼이 파티션 키가 됨
# 그렇지 않고 프라이머리 키의 일부만 명시할 수도 있음
PARTITION BY KEY()
		PARTITIONS 2;

# 프라이머리 키가 없는 경우 유니크 키(존재한다면)가 파티션 키로 사용됨
CREATE TABLE k1 (
id INT NOT NULL,
name VARCHAR(20),
UNIQUE KEY (id)
)
# 괄호의 내용을 비워 두면 자동으로 프라이머리 키의 모든 칼럼이 파티션 키가 됨
# 그렇지 않고 프라이머리 키의 일부만 명시할 수도 있음
PARTITION BY KEY()
PARTITIONS 2;

# 프라이머리 키나 유니크 키의 칼럼 일부를 파티션 키로 명시적으로 설정
CREATE TABLE dept_emp (
emp_no INT NOT NULL,
dept_no CHAR(4) NOT NULL,
...
PRIMARY KEY (dept_no, emp_no)
)
# 괄호의 내용에 프라이머리 키나 유니크 키를 구성하는 칼럼 중에서
# 일부만 선택해 파티션 키로 설정하는 것도 가능하다.
PARTITION BY KEY(dept_no)
PARTITIONS 2; # 생성할 파티션의 개수 지정

```

---

**13.3.4.2 키 파티션의 주의사항 및 특이사항**

- mysql 서버가 내부적으로 MD5() 함수를 이용해 파티션하기 때문에 키가 꼭 정수 타입이 아니어도 됩니다. (해시 파티션이 어렵다면 키 파티션 적용 고려)
- PK나 유니크 키 구성 칼럼중 일부만으로 파티션할 수 있음
- 유니크 키를 파티션 키로 사용하려면 유니크 키가 반드시 NOT NULL이어야 함
- 해시 파티션에 비해 더 균등하게 분할할 수 있기에 더 효율적임

---

### 13.3.5 리니어 해시 파티션/리니어 키 파티션

- 해시, 키 파티션에서 파티션 추가, 줄일때 전체 파티션의 데이터 재분배 작업을 최소화 하기 위해 고안된 알고리즘
- 리니어 해시 파티션, 리니어 키 파티션은 각 레코드 분배를 위해 2의 승수 알고리즘을 이용합니다. (다른 파티션의 추가, 통합 시 타 파티션에 미치는 영향을 최소화함)

**13.3.5.1 리니어 해시 파티션/리니어 키 파티션의 추가 및 통합**

- 나머지 연산이 아니라 2의 승수 분배 방식을 사용기 때문에 파티션 추가, 통합시 특정 파티션의 데이터에 대해서만 이동 작업을 하면 됨
- 위와 같은 이유로 파티션을 추가하거나 통합하는 작업에서 나머지 파티션의 데이터는 재분배 대상이 되지 않음

1. `파티션의 추가`
    - 파티션 추가 명령은 해시, 키 파티션과 동일
    - `Power-of-two` 알고리즘으로 레코드가 분배되어있기에 새로운 파티션 추가시에도 아래 그림과 같이 특정 파티션의 레코드만 재분배 하면 됩니다.
    (해시, 키 파티션에서의 추가보다 빠르게 처리됨)
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/67b3f2b6-817c-463c-b0a0-81a44a58ec65)

        
    
2. `파티션의 통합`
    - 통합 작업 또한 일부 레코드에 대해서만 통합 작업이 필요합니다.
        
        ![image](https://github.com/Invincible-Backend-Study/Real-MySQL/assets/66772624/6591a1af-32a0-452b-900a-76923dc95cbd)

        
3. `주의사항`
    - 해시, 키 파티션과 달리 Power-of-two 알고리즘을 사용하기에 추가, 통합시 재분배 파티션의 범위를 최소화 할 수 있으나, 각 파티션이 가지는 레코드 건수는 해시, 키 파티션 보다 덜 균등해질 수 있음
    - 파티션 추가, 삭제 빈도가 높다면 해시, 키 파티션을 사용하는게 좋음

---

### 13.3.6 파티션 테이블의 쿼리 성능

- 파티션 전체, 일부를 읽느냐에 따라 성능에 아주 큰 영향을 줍니다.
- 따라서 얼마나 많은 파티션을 프루닝 할 수 있는지가 관건입니다.
    - 이는 쿼리의 실행 계획으로 확인할 수 있습니다.
- 레인지, 리스트 파티션은 직접 명시해주어 생성하므로 파티션의 개수가 주로 10~20개 내외로 적습니다.
- 반면에 키 파티션은 개수만 명시하면 되므로 많은 파티션을 가진 테이블도 쉽게 생성합니다.
    
    ```sql
    CREATE TABLE user (
    		user_id BIGINT NOT NULL,
    		name VARCHAR(20),
    		...,
    		PRIMARY KEY (id),
    		INDEX ix_name (name)
    ) PARTITION BY KEY() PARTITIONS 1024;
    
    SELECT * FROM user WHERE name='toto';
    ```
    
    - 일반적인 테이블이라면 위 쿼리는 B-Tree를 한 번만 룩업하면 되니 효율적이지만, 1024개의 파티션에선 모든 파티션을 찾아봐야 하므로 비효율적입니다.
- 파티션으로 쪼개고 일부만 사용하지 못하고 균등하게 사용하면 오버헤드만 심해집니다. (반면에 샤딩은 효과적일것)
- 파티셔닝 이전에 파티션 프루닝이 얼마나 도움될지 예측해보고 적용하는걸 추천합니다.
