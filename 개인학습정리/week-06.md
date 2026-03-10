# week-06

# -1 저번주 질문

## 1. Btree, B+Tree 사용하는 상황

- MySQL은 모든 인덱스가 `B+Tree`를 사용, BTree를 사용하는 경우는 아예 없다
- 책에서 Btree라고 하는 이유는 인덱스 타입 조회시 `BTree`라고 표시되는데, 사실 `B+Tree`임

## 2. 외래키를 가지는 두 테이블에서 자식이 외래키와 관련없는 컬럼을 update하더라도 부모가 대기하는 이유

- 정답
    - 만약 자식 테이블에서 **`parent_id` 와 관련 없는 컬럼**을 수정 중이라도, InnoDB는 **레코드 단위 잠금(row lock)** 을 사용하기 때문에 자식 레코드에 X-lock(배타 잠금)이 걸려있다.
    - 그러나 부모 레코드에는 X-lock이 걸리지 않는다
        - But) parent_id 컬럼을 수정하면 부모레코드에 S-lock이 발생

# 0 질문

## 1. MRR & BKA 방식은 조인 버퍼를 이용해서 드리븐 테이블을 읽는 방식이고, Block Nested Loop Join도 마찬가지이다. 두 방식의 차이점은?

- MRR & BKA 방식은 드리븐 테이블의 인덱싱이 필요하고 랜덤 엑세스가 많아 비효율 적일때 사용
    - 또한 조인 버퍼를 정렬해야하고 드리븐 테이블 접근시 인덱싱 후 순차 접근
    - 풀 테이블 스캔은 발생하지 않음
- Block Nested Loop Join 방식은 드리븐 테이블에 인덱스가 없어서 풀테이블 스캔이 많이 일어나는 경우에 사용
    
    
    | 구분 | Block Nested Loop Join (BNL) | BKA Join (with MRR) |
    | --- | --- | --- |
    | 인덱스 필요 여부 | ❌ 없음 | ✅ 있음 |
    | 조인 버퍼 활용 | 있음 (드라이빙 행 저장) | 있음 (드라이빙 행 저장) |
    | 드리븐 테이블 접근 방식 | 풀스캔 | 인덱스 탐색 (MRR로 랜덤 I/O 최소화) |
    | 효율성 | 상대적으로 비효율적 | 인덱스를 활용하면서 I/O 최적화 |
    | EXPLAIN Extra | `Using join buffer (Block Nested Loop)` | `Using join buffer (Batched Key Access)` |

## 2. MRR & BKA 방식이 Nested Loop 방식보다 느린 경우가 존재하는데, 어떤 오버헤드 때문일까?

- MRR & BKA 방식은 조인 버퍼를 정렬해야하고 드리븐 테이블 접근시 인덱싱 후 순차 접근
- 1대1인 Nested Loop 방식을 개선한것처럼 보이지만 정렬 오버헤드 존재

## 3. num컬럼이 인덱싱 되어 있는 경우 `where num = 20` 은 pk 정렬한 결과가 나오고 `where num between 10 and 20` 은 pk 정렬한 결과가 나오지 않는 이유

- 세컨더리 인덱싱의 데이터 노드에는 (num, pk) 정렬되어있다
- 범위 스캔을 사용하지 않으면 pk 정렬된 결과가 나오지만, 범위 스캔을 사용하는 경우 num=10, 11, 12 … 의 모든 결과가 합쳐져서 나오므로 pk 정렬이 깨진다

## 4.`use_index_extensions` 과 `index_merge_intersection` 은 둘 다 `where col1 and col2` 처럼 두 가지 컬럼이 and 조건일때 최적화하는 방법이다. 어떤 차이가 있을까?

- 힌트
    - 인덱스는 col1, col2 각각 가지고 있다다
    - 인덱스 확장은 더블 룩업을 사용하고, 인덱스 머지는 2개 컬럼 결과 레코드를 교집합 연산을 한다
- 정답
    - 인덱스 확장은 두 컬럼의 인덱싱한 결과 레코드 수가 적을때 최적
        - 또한 한쪽 컬럼의 인덱싱 결과가 훨씬 작다면, 거기를 먼저 인덱싱 해야 최적이다
    - 인덱스 머지는 두 컬럼의 인덱싱한 결과 레코드 수가 많을때 최적

# 9.2

## 4. Group By 처리

- Group By는 스트리밍 처리 불가
    - Having 절은 인덱싱 불가
- 인덱스로 Group By 처리는 가능

### Loose 인덱스 스캔 (Group by)

- Group By로 묶고 MIN/MAX 같은 집계 컬럼을 확인하는 것은
    - 복합인덱스에서 Skip Scan과 유사하게 최적화가 가능하다
    - (group by, where의 컬럼으로 인덱싱 되어있다고 생각)
    - 예 ) col1으로 묶고, Min(col2)를 구하는 경우
        - col1그룹에서 min(col2)를 찾았으면 다음 col1 그룹으로 스킵할 수 있다
- Explain 할때 Extra에 `Using index for group-by` 가 표시됨
    - 임시 테이블도 필요없음
- 유니크 값(도메인) 이 작을수록 성능 향상
- 가능한 경우
    
    ```sql
    // emp_no, from_date 로 인덱싱이 되어있는경우
    SELECT emp_no FROM salaries
    	WHERE from_date='1985-03-01'
    	GROUP BY emp_no;
    // 이렇게 모든 emp_no에 대해서 인덱스로 처리가능
    SELECT emp_no FROM salaries
    	WHERE emp_no=100 AND from_date='1985-03-01';
    
    // Loose 인덱스 처리가 가능한 경우
    SELECT col1, col2 FROM tb GROUP BY col1, col2;
    SELECT col1, MIN(col2) FROM tb GROUP BY col1, col2;
    ```
    
- 불가능한 경우
    
    ```sql
    // sum 집계함수의 경우 모든 record를 확인해야되기때문에 skip 불가
    SELECT col1, SUM(col2) FROM tb GROUP BY col1, col2;
    
    // 인덱스 구성 칼럼이 왼쪽부터 일치하지 않는다
    SELECT col1, col2 FROM tb GROUP BY col2, col3; // col1은 아무거나 선택된다
    
    // col2가 아닌 col3는 스킵할 수 없다 
    SELECT col1, col3 FROM tb GROUP BY col1, col2;
    ```
    
- 인덱싱이 불가능한 Group By를 Join하는 경우
    - Explain의 `Extra = Using Temporary` 임
        - 일단 조인해서 임시테이블을 만듬 (group by 컬럼을 `unique index`로 만든 테이블이다)
        - 그리고 집계함수를 처리함
    - 근데 MySQL 8.0 부터는 Group By한 그룹들을 정렬하지 않는다고함
    - 그래서 `Extra = Using Filesort` 가 아닌것이다
        - 만약 Order By 절이 추가되면 `Using Filesort`가 됨

## 5. Distinct 처리

- Distinct 처리는 Group by 처리와 같다 → 그룹정렬안함
    
    ```sql
    SELECT DISTINCT emp_no FROM salaries;
     = SELECT emp_no FROM salaries GROUP BY emp_no;
    ```
    
- Distinct 함수라는 건 없다
    
    ```sql
    SELECT DISTINCT(first_name), last_name FROM employees;
     = SELECT DISTINCT first_name, last_name FROM employees;
    ```
    
- Join 테이블의 Distinct 처리는 임시 테이블을 이용한다
    - Join 결과는 인덱스가 없기 때문이다
    
    ```sql
    select count(distinct s.salary)
    	from employee e, salaries s
    	where e.emp_no=s.emp_no and e.emp_no betweeen 100 and 200;
    // 조인하고, 조건에 맞는 레코드로만 salary 임시테이블을 만들고, unique index처리를 한다
    
    select count(distinct s.salary), count(distinct e.lastname)
    	from employee e, salaries s
    	where e.emp_no=s.emp_no and e.emp_no betweeen 100 and 200;
    // 마찬가지로 임시테이블을 만드는데 2개가 필요하다
    // salary 임시테이블 생성 + lastname 임시테이블 생성
    
    // 그러나 조인 없는 컬럼은 인덱싱을 이용해서 빠르다
    SELECT count(DISTINCT emp_no) FROM employees;
    ```
    
- 쿼리 최적화
    
    ```sql
    select count(distinct first_name), count(distinct last_name)
    	from employees
    	where emp_no between 100 and 200;
    // 이쿼리의 경우 빠른 인덱싱을 위해서 인덱스 생성이 필요
    CREATE INDEX idx_empno_fname ON employees(emp_no, first_name);
    CREATE INDEX idx_empno_lname ON employees(emp_no, last_name);
    // 이 인덱싱은 가능한데 최적은 아님
    CREATE INDEX idx_empno ON employees(emp_no);
    ```
    

## 6. 임시테이블

- 임시테이블은 메모리에 생성
    - 너무 커지면 디스크로
    - 예외 적으로 디스크에 만들 수도 있음
        - `internal_tmp_disk_storage_engine` 설정
- 쿼리 처리가 끝나면 자동 삭제

### 메모리 임시테이블

- 기존에 쓰던 MEMORY 스토리지 엔진
    - 가변 길이 타입은 최대 공간을 할당하기 때문에 비효율적
- 8.0 버전의 TempTable (기본값)
    - 가변 길이 타입 지원
    - 1GB 이상은 메모리 데이터를 MMAP파일로 디스크에 저장
        - MMAP파일이 디스크 InnoDB 테이블 파일보다 오버헤드가 적다
        - `temptable_use_mmap = OFF` 인 경우에는 InnoDB테이블로 저장
- 임시테이블이 필요한 쿼리
    - order by 와 group by와 관련
        - 인덱싱이 불가능한 경우
    - Union (Distinct) 연산
        - → Explain에 Using Temporary가 표시되지 않더라도 임시테이블을 이용한다
    - from 절에 서브쿼리(select_type = DERIVED)
        - → Explain에 Using Temporary가 표시되지 않더라도 임시테이블을 이용한다
- 디스크 생성이 필요한 경우
    - 칼럼 길이가 512byte 이상
    - 임시 테이블 크기가 `tmp_table_size` / `temptable_max_ram` 보다 큰 경우
- 디스크 생성 확인 방법
    
    ```sql
    show session status like 'created_tmp%';
    ```
    
    - `created_tmp_tables = 1`
        - 만들어진 모든 임시테이블(디스크, 메모리)의 개수
    - `created_tmp_disk_tables = 1`
        - 디스크 임시테이블 개수

# 9.3

## 1. 옵티마이저 스위치

- 옵티마이저에서 제공하는 최적화 기능을 켜고 끌 수 있다
- 쿼리
    
    ```sql
    set global optimizer_switch='index_merge=on, index_maerge_union=on';
    set session optimizer_switch='index_merge=on, index_maerge_union=on';
    // 현재 쿼리에만 적용
    select /*+ SET_VAR(optimizer_switch='index_merge=off')
    	from ...
    ```
    

### MRR & BKA

- `Nested Loop Join`
    - 조인할때 드라이빙 테이블의 1개 레코드씩 드리븐 테이블과 조인하는것
- `MRR (Multi-Range Read)` = `BKA (Batched Key Access)` 조인
    - 조인할때 조인처리는 MySQL 엔진이, 레코드 접근은 Storage 엔진이 한다
    - 드라이빙 테이블의 레코드 여러개를 Join Buffer에 저장하고 버퍼를 Storage 엔진에 넘긴다
    - Storage 엔진은 레코드를 정렬된 순서로 접근하고 여러 레코드를 동시에 반환
    - 단점 ) 정렬 오버헤드가 필요

### 블록 Nested Loop Join

- 1레코드씩 조인시키는 건 `Nested Loop Join`
- 조인 버퍼를 이용해서 N레코드씩 드리븐 레코드를 검색하는 건 `Block Nested Loop Join`
- 사용조건
    - 드리븐 테이블에 적절한 인덱스가 없는 경우
    - 풀 테이블 스캔을 피할 수 없는 경우
- MRR & BKA 방식과의 차이
    - `MRR & BKA` 방식 → 드리븐 테이블에 인덱스 존재 + 랜덤 엑세스가 많은 경우
        - Explain `Using join buffer = Batched Key Access`
        - 또한 조인 버퍼를 정렬해야하고 드리븐 테이블 접근시 인덱싱 후 순차 접근
        - 풀 테이블 스캔은 발생하지 않음
        - 1대1인 Nested Loop 방식보다 좋아보이지만 정렬 오버헤드 존재
    - `Block Nested Loop` → 드리븐 테이블에 인덱스가 없는 경우 + 풀 테이블 스캔
        - Explain `Using join buffer = Block Nested Loop`
- 진행 순서
    - 드라이빙 테이블에서 조건 레코드만 버퍼에 저장
    - 드리븐 테이블에서 버퍼로 검색
    - 드리븐 테이블의 검색결과에 드라이빙 테이블을 조인시킴
    - 결론 ) 조인버퍼를 이용하면 드라이빙 테이블 순서로 결과가 나오지 않음

### 인덱스 컨디션 pushdown

- 복합 인덱스에서 첫번째 컬럼으로 인덱싱을하는데, 두번째 컬림이 인덱싱이 불가능한 Where 조건을 달아준 경우
    - `index_condition_pushdown=off` 일때
        - Secondary 인덱스에서 첫번째 컬럼을 만족하는 모든 pk에 대해서 Primary 인덱스의 랜덤 액세스함
        - 테이블에 직접 접근해서 두번째 조건을 확인
    - `index_condition_pushdown=on` 일때
        - Secondary 인덱스에서 두번째 컬럼을 순차 비교한다 → 시간 최적화 가능
- explain으로 확인 → Extra의 `Using index condition` 을 확인

### 인덱스 extension

- `use_index_extensions`
- Secondary 인덱스 컬럼 + Primary 인덱스 컬럼으로 탐색하는 경우
- 마치 1개의 복합 인덱스가 형성된 것처럼 인덱싱이 가능
    - 실제로 복합 인덱스처럼 트리를 1번만 타는건 아니고, 2번 탐색한다 (더블룩업)
    - 하지만 explain `key_len` 값 (인덱싱에 이용된 모든 컬럼의 byte합)은 복합인덱스랑 똑같다

### 인덱스 Merge

- 3가지 방법이 있다
    - `index_merge_intersection`
    - `index_merge_sort_union`
    - `index_merge_union`
- `index_merge_intersection`
    - 상황
        - 여러 컬럼이 where and 조건으로 연결되어 있는 경우
        - 각 컬럼에 세컨더리 인덱스가 있는 경우
    - 방법
        - col1의 인덱스로 결과 레코드를 찾고
        - col2의 인덱스로 결과 레코드를 찾은뒤
        - 두 레코드 사이의 교집합만 반환하는 방법이다
    - 특징
        - 더블 LookUp은 조건에 맞는 레코드 개수가 두 컬럼 모두 많을때 최적이다
        - 만약 조건에 맞는 레코드가 적은 컬럼이 1개라도 있다면, 거기만 먼저 인덱스 탐색 후 두번째 조건 검사를 하는게 빠르다
- `index_merge_union`
    - 상황
        - 여러 컬럼이 where or 조건으로 연결되어 있는 경우
        - 각 컬럼에 세컨더리 인덱스가 있는 경우
    - 방법
        - col1의 인덱스로 결과 레코드를 찾고
        - col2의 인덱스로 결과 레코드를 찾은뒤
        - 두 결과는 모두 pk에 정렬되어서 나오므로 하나씩 비교하면서 큐에 넣으면 중복제거가 가능하다
        - 이걸 Prioity Queue라고 하던데 heap 자료구조를 사용하는게 아니라 그냥 정렬되어있어서 그렇게 부르는것 같다
- `index_merge_sort_union`
    - 상황
        - 여러 컬럼이 where or 조건으로 연결되어 있는 경우
        - 각 컬럼에 세컨더리 인덱스가 있는 경우
        - 컬럼 중 범위 스캔이 필요한 것이 있는 경우
            - 범위 스캔의 결과는 Pk에 정렬되어 나오지 않는다
            - 예를들어 a=1, a=2 각각은 pk 정렬되어있는데 범위 스캔으로 합치면 아니게 된다
    - 방법
        - col1의 인덱스로 결과 레코드를 찾고
        - col2의 인덱스로 결과 레코드를 찾은뒤
        - 범위 스캔을 사용한 컬럼은 따로 정렬을 진행한다
        - 이후 똑같이 하나씩 비교하면서 큐에 넣는다