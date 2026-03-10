# week-05

# 0 질문 리스트

## 1. 가상컬럼 인덱스와 복합인덱스의 차이점은?

- 문제
    
    ```sql
    // 복합 인덱스
    CREATE INDEX idx_fullname ON user(first_name, last_name);
    // 가상컬럼 인덱스
    ALTER TABLE user 
      ADD full_name VARCHAR(30) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL,
      ADD INDEX idx_fullname(full_name);
    ```
    
- 정답
    
    
    | 구분 | 복합 인덱스 | 가상 컬럼 + 인덱스 |
    | --- | --- | --- |
    | 인덱스 대상 | 원본 컬럼 조합 `(first_name, last_name)` | 계산된 컬럼 `full_name = CONCAT(first, last)` |
    | 저장 방식 | 인덱스에 두 컬럼 값이 순서대로 저장됨 | 인덱스에 계산된 문자열(full_name) 저장됨 |
    | 검색 활용 | `first_name` 단독 검색 가능 (왼쪽 우선) | `CONCAT(first,last)` 같은 표현식 검색 최적화 → 단독 검색 불가 |
    | 유지 비용 | INSERT/UPDATE 시 원본 두 컬럼만 인덱스 갱신 | INSERT/UPDATE 시 가상 컬럼 계산 + 인덱스 갱신 |
    | 유연성 | 단순한 컬럼 조합에 적합 | 함수/표현식 기반 검색 최적화에 유리 |

## 2. 멀티밸류 인덱스의 B 트리 구성법은?

- 문제
    
    ```sql
    INSERT INTO user VALUES (1, '{"credit_scores" : [1, 2, 3]}');
    INSERT INTO user VALUES (2, '{"credit_scores" : [2, 3, 4]}');
    
    SELECT * FROM user
    	WHERE 2 MEMBER OF(credit_info -> '$.credit_scores');
    ```
    
- 정답
    - 인덱스의 엔트리 키값
        
        ```
        (rowid=1, value=1)
        (rowid=1, value=2)
        (rowid=1, value=3)
        
        (rowid=2, value=2)
        (rowid=2, value=3)
        (rowid=2, value=4)
        ```
        
        - 결국 인덱스는 `(인덱스값, rowid)`의 쌍으로 정렬됩니다.
        - B-Tree 인덱스 특성상, `value`가 먼저 정렬 기준 → 값이 같으면 `rowid`가 두 번째 정렬 기준입니다.
    - 예시 쿼리의 동작방식
        1. **첫 번째 기준**: 배열 요소의 값 (CAST된 타입, 예: UNSIGNED)
            
            → `1 < 2 < 3 < 4` 순으로 인덱스에 저장
            
        2. **두 번째 기준**: 같은 값일 경우 해당 값이 속한 **row의 PK**
            
            → 예: 값 `2`는 row1, row2 모두 존재 → 인덱스에 `(2,row1)`, `(2,row2)`로 저장됨
            
        3. 즉, 옵티마이저는 인덱스에서 **value=2** 엔트리를 바로 찾아가고, 그 엔트리가 가리키는 rowid(row1, row2)를 결과로 반환

## 3. 데드락 발생시 해결 방법은?

- 문제
    - 외래키를 참조하는 상황에서, 서로 순환 대기를 하다가 데드락이 생기는 경우가 발생
- 정답
    - InnoDB가 한쪽 트랜잭션을 강제로 ROLLBACK 해서 데드락을 해결

## 4. 외래키를 가지는 두 테이블에서 자식이 외래키와 관련없는 컬럼을 update하더라도 부모가 대기하는 이유

- 정답
    - 만약 자식 테이블에서 **`parent_id` 와 관련 없는 컬럼**을 수정 중이라도, InnoDB는 **레코드 단위 잠금(row lock)** 을 사용하기 때문에 해당 레코드에 X-lock(배타 잠금)이 걸려있다.

## 5. Nested Loop 방식과 JoinBuffer 방식의 조인 처리 차이점

- 문제
    
    <aside>
    
    ## 🔎 1. Nested Loop Join (중첩 루프 조인, 기본 방식)
    
    - MySQL은 대부분의 조인을 **Nested Loop Join** 으로 처리합니다.
    - 동작 방식:
        1. **드라이빙 테이블**에서 한 행(row)을 가져옴
        2. 그 값을 가지고 **드리븐 테이블**을 탐색
            - 인덱스가 있으면 인덱스로 검색 (빠름)
            - 인덱스가 없으면 풀 테이블 스캔 (느림)
        3. 이 과정을 드라이빙 테이블의 모든 행에 대해 반복
    
    📌 장점
    
    - 인덱스가 잘 있으면 빠름 (랜덤 액세스 최소화)
        
        📌 단점
        
    - 드리븐 테이블에 인덱스가 없으면 → **매번 풀 스캔**해야 해서 매우 비효율적
    
    ## 🔎 2. Join Buffer (Block Nested Loop Join)
    
    - 인덱스가 없을 때 단순 Nested Loop로 돌리면 너무 비효율적이므로, MySQL은 **조인 버퍼(join_buffer_size)** 라는 임시 메모리 공간을 활용합니다.
    - 동작 방식:
        1. 드라이빙 테이블에서 여러 행을 **한꺼번에 버퍼에 담음**
        2. 드리븐 테이블을 한 번 스캔하면서, 버퍼에 있는 여러 값들과 비교
        3. 매번 테이블 풀스캔을 반복하지 않고, **드리븐 테이블을 한 번만 읽고 버퍼와 매칭**
    
    📌 장점
    
    - 테이블 풀스캔 횟수를 줄임 → 훨씬 효율적
        
        📌 단점
        
    - 조인 버퍼 크기가 제한적 (`join_buffer_size`) → 너무 크면 메모리 낭비, 너무 작으면 여러 번 반복
    </aside>
    
- 정답
    
    
    | 구분 | Nested Loop Join | Join Buffer (Block Nested Loop Join) |
    | --- | --- | --- |
    | 드리븐 테이블 인덱스 | 있으면 효율적 | 없어도 그나마 효율적 |
    | 방식 | 드라이빙 테이블의 각 행마다 드리븐 테이블 접근 | 드라이빙 테이블의 여러 행을 버퍼에 담아 드리븐 테이블 한 번 스캔 |
    | I/O 비용 | 드리븐 테이블 풀스캔 반복 발생 가능 | 풀스캔 최소화 (버퍼 매칭) |
    | 튜닝 포인트 | 인덱스 설계 | `join_buffer_size` 조정 |

## 6. 드라이빙 테이블 결정 조건 ?

- 정답
    - 조건선택
        - MySQL 옵티마이저는 **비용 기반 최적화(Cost-Based Optimizer, CBO)** 를 사용합니다.
        - 즉, 단순히 "조건 → 인덱스 → 크기" 순으로 기계적으로 따르는 게 아니라, **각 경우의 비용(cost)을 계산해서 가장 낮은 쪽을 선택**합니다.
    - WHERE 필터링
        - 걸러진 결과 record 개수가 적은 테이블 우선 선택
    - 인덱스 사용 가능 여부
        - 드리븐 테이블이 인덱싱이 가능하다면, 드라이빙 테이블을 거기에 맞출 가능성 높다
    - 테이블 크기
        - 일반적으로 작은 테이블을 드라이빙, 큰 테이블을 드리븐으로 두는게 유리
    - STRAIGHT_JOIN
        - 강제로 드라이빙 테이블 지정
        
        ```sql
        SELECT STRAIGHT_JOIN *
        FROM A
        JOIN B ON A.id = B.id;
        // 강제로 A테이블을 드라이빙 테이블로 지정
        ```
        

# 8.6 함수 기반 인덱스

- 연산 결과를 인덱스로 만들 수 있다
- 2가지 방식 존재
    - 가상 컬럼
    - 함수 이용
    - 결론 ) 하지만 내부적으로 같은 처리를 한다

### 가상 컬럼 이용 인덱스

- 가상컬럼
    - Virtual Column → 디스크에 저장안하고, 조회할때마다 계산해서 보여줌
    - Stored Generated Column → 디스크에 저장하고, update/insert 시점에 같이 갱신함
- 가상컬럼도 인덱스를 만들 수 있다
- 쿼리
    
    ```sql
    ALTER TABLE user
    	ADD full_name VARCHAR(30) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL,
    	ADD INDEX idx_fullname(full_name);
    ```
    

### 함수를 이용한 인덱스

- 쿼리
    
    ```sql
    CREATE TABLE user {
    	INDEX idx_fullname((CONCAT(first_name, ' ', last_name)))
    	);
    ```
    
- 인덱스를 이용해서 조회할때 조건절 표현식이 인덱스 키와 일치해야 한다
    
    ```sql
    SELECT * FROM user WHERE CONCAT(first_name, ' ', last_name)='Seoyoung Lee';
    ```
    
- 디버그 ) Explain했는데 인덱스를 사용하지 않는다면, 공백 문자 리터럴때문에 동일 콜레이션으로 인식하지 않는것

# 8.7 Multi-Value 인덱스

- 쿼리
    
    ```sql
    // 생성
    CREATE TABLE user {
    	create_info JSON,
    	INDEX midx_creditscores( (CAST(credit_info -> '$.credit_scores' AS UNSIGNED ARRAY)) )
    );
    
    INSERT INTO user VALUES ( ... , '{"credit_scores" : [1, 2, 3]}' );
    
    // 조회 (인덱싱 이용)
    SELECT * FROM user WHERE 2 MEMBER OF(credit_info -> '$.credit_scores');
    ```
    
    - 화살표 함수
        - credit_info JSON의 credic_score 속성을 JSON Array로 반환함
    - CAST 함수
        - JSON Array를 정수 배열로 반환
    - 정수 배열을 가지고 Multi-Value 인덱스를 만듬
- 조회가 가능한 함수
    - MEMBER OF ( )
        - 단일 서치 가능
        - `WHERE 2 MEMBER OF(credit_info -> '$.credit_scores');`
    - JSON_CONTAINS ( )
        - 배열로 서치 가능 + 부분집합으로 확인
        - `JSON_CONTAINS(credit_info->'$.credit_scores', '[2,3]');`
    - JSON_OVERLAPS ( )
        - 교집합 있는지 서치 → 빠르다
        - `WHERE JSON_OVERLAPS(credit_info->'$.credit_scores', '[2,4,6]');`
    
    ## 8.8 클러스터링 인덱스
    
    - PK를 키값으로 하는 B+트리 유지
    - 조회는 빠른데, 삽입 삭제가 좀 느림
    - 우선순위
        1. PK
        2. Unique + Not Null 중에서 첫번째 인덱스
        3. 둘 다 없으면, 자동으로 증가하는 Unique 컬럼을 생성고 키로 선택
            1. 사용자는 접근이 불가능
    
    ### MyISAM의 인덱싱과 InnoDB 인덱싱의 차이점
    
    | 구분 | MyISAM (세컨더리 인덱스 = 물리주소) | InnoDB (세컨더리 인덱스 = PK) |
    | --- | --- | --- |
    | **데이터 접근 속도** | 인덱스 → 주소 → 바로 레코드 (빠름, 1번 접근) | 인덱스 → PK 조회 → 클러스터링 인덱스 → 데이터 (2번 접근, 다소 느림) |
    | **데이터 이동/삭제** | 레코드 위치가 바뀌면 모든 인덱스의 주소도 갱신 필요 (비효율적) | PK 값은 변하지 않으므로 인덱스 안정적 (데이터 위치 변경돼도 문제 없음) |
    | **저장 구조** | 인덱스와 데이터 파일 분리 (`.MYI`, `.MYD`) | 인덱스+데이터 통합 (테이블스페이스) |
    | **범위 검색** | 가능하지만 주소 기반이므로 디스크 접근 비효율 가능 | B+Tree + 리프 노드 연속 링크 → 범위 검색 최적화 |
    | **외래키 지원** | ❌ 지원 안 함 | ✅ 지원 |
    | **트랜잭션** | ❌ 지원 안 함 | ✅ 지원 (ACID) |
    - 단점
        - 클러스터링 키 값의 크기가 크면 인덱스 크기가 커짐
        - Double Lookup
        - PK변경 시 레코드를 Delete 하고 Insert 하기 때문에 느리다
    - 결론 ) 빠른 읽기 + 느린 쓰기
    
    ### PK 최적화
    
    - PK크기에 비례해서 Secondary Index의 크기가 커지기 때문에 최적화 필요
    - 방법
        1. Auto-Increment보다는 컬럼을 지정하는게 좋다
            1. 검색에서 PK가 빈번하게 사용되기 때문
        2. PK는 반드시 지정하는게 좋음
            1. 자동생성 PK컬럼도 Auto_Increment이기 때문에 접근 할 수 있게 지정하는게 좋다
        3. 복합 PK라서 키가 큰 경우
            1. 왠만하면 세컨더리 인덱스 안쓰고 컬럼 다 묶어서 복합 PK를 쓰는게 좋다
            2. 세컨더리 인덱스가 필요하다면, Auto_Increment로 PK크기를 줄이는게 좋다
                1. 특히 INSERT가 많을때 사용하면 좋음
    
    ### Unique 인덱스
    
    - Unique 인덱스를 읽을때, 트리 탐색하고 중복값을 읽는데 디스크에 접근해서 읽는게 아니라 CPU에서 단순히 컬럼을 비교해서 찾는다
        - 레코드 1건당 읽는 속도는 Unique나 중복 col이나 비슷하다
        - 결론 ) 중복 컬럼을 위해 Unique 인덱스를 새로 생성할 필요가 없다
    - Unique 인덱스 처리
        - 중복 체크할때 읽기잠금 + 쓰기잠금 → 데드락 발생 가능성
        - 시간 단축을 위해 인덱스
    
    ### FK
    
    - 부모 ← 자식 관계
        - 자식이 부모의 pk에 해당하는 col을 가지고 있음
        - 자식은 부모를 참조하고, 부모는 자식에게 참조되는 관계
    - 잠금 경합 첫번째 상황 → 부모 테이블 변경중에 자식 fk를 변경하는 경우
        - 자식이 fk 컬럼을 (update, insert)할때 fk 컬럼이 가리키는 부모의 record가 lock인지 확인함
        - 만약 참조되는 부모 record가 update 중이면 대기
        - 부모 record 트랜잭션이 완료되면, 자식이 update 처리 가능
    - 잠금 경합 2번째 상황 → 자식 컬럼 변경 중에 부모 테이블 변경하는 경우
        - fk가 아닌 컬럼을 update하더라도 x-lock은 record 단위이기 때문에, 대기 발생
    - 결론
        - 부모가 update할때 fk를 가지고 있는 모든 자식의 무결성을 확인
        - 자식이 update할때, 부모가 record를 가지고 있는지 존재를 확인
        - 두 상황 모두 락 충돌 + 대기 발생

# 9.1 옵티마이저

- 규칙 기반 최적화 → 미리 정의해놓은 우선순위로 어떤 상황에서도 동일한 실행 방법 결정
- 비용 기반 최적화 → 예측 통계 정보를 이용해 최소 비용 계획을 선택

# 9.2 데이터처리

### 풀 테이블 스캔 vs 풀 인덱스

- 풀 테이블 스캔 선택하는 상황
    - 레코드가 너무 작다 (테이블이 1페이지 인 경우)
    - Where 조건절에 적당한 인덱스가 없다
    - 반환하는 레코드 결과가 너무 많다
- 풀 테이블 스캔의 멀티 쓰레딩
    - 풀 테이블이 포함된 페이지가 여러개인 경우
    - Read Ahead 기법으로 필요한 페이지를 예측한다
    - Foreground Thread와 Background Thread를 이용
        - 포어그라운드 스레드는 읽어온 버퍼 풀의 데이터를 사용
        - 백그라운드 스레드는 디스크에서 버퍼 풀로 페이지 읽음
    - `innodb_read_ahead_threshold`
        - 백그라운드 스레드가 디스크에 접근하기 전에 처음 포어그라운드 스레드가 읽어오는 페이지 개수
    - 쿼리
        
        ```sql
        // 레코드 주소를 알 필요 없다 -> 풀 인덱스 스캔(빠름)
        // Primary 인덱스 없이 Secondary 인덱스 트리만 접근하면 해결
        SELECT count(*) FROM employees;
        
        // 레코드에 직접 접근해야한다 -> 풀 테이블 스캔(느림)
        SELECT * FROM employees;
        ```
        

### 병렬 스레드

- 하나의 쿼리를 여러 쓰레드가 처리하는 경우
    - 오직 조건 없이 테이블 전체 건수를 가져오는 쿼리만 가능
    
    ```sql
    SET SESSION innodb_parallel_read_threads=1;
    // 2 3 4 로 늘리면 count쿼리가 점점 빨라짐
    ```
    
    - SET SESSION → 현재 내 세션에만 영향
        - SHOW SESSION VARIABLES LIKE ‘innodb_parallel_read_threads’;
    - SET GLOBAL → 전체 서버 인스턴스에 영향
        - SHOW GLOBAL VARIABLES LIKE ‘innodb_parallel_read_threads’;

### Order BY 처리

- 커버링 인덱스인 경우 → 인덱스 트리 이용
    - 읽기는 빠르다
    - update가 느리다
- 인덱스가 없거나 조회하는 레코드 개수가 적은 경우 → File Sort 이용
    - 인덱스가 없어도 된다 (공간 절약)
    - 레코드가 적을때 충분히 빠르다
    - 레코드가 많으면 느리다
- File Sort를 사용하는 상황
    - 정렬 기준이 너무 많다
    - Group By나 Distinct 처리 후 정렬하는 경우
    - 임시 테이블의 결과를 정렬하는 경우 (Union)
    - 렌덤하게 레코드를 가져오는 경우
- File Sort 과정
    - `sort_buffer_size` 시스템 변수만큼 메모리 할당
        - 50~1000KB가 적절. 이 범위를 벗어나면 성능 저하
    - 메모리보다 정렬 데이터가 큰 경우, 디스크에서 읽고 메모리에서 정렬하고 디스크로 다시 기록하는 과정 반복
    - 디스크로 다시 기록할때는 정렬된 버퍼끼리 Multi-Merge 과정이 필요
    - 외부정렬
- 옵티마이저 트레이스
    
    ```sql
    SET OPTIMIZER_TRACE="enabled=on:, END_MARKERS_IN_JSON=on; //JSON 결과에 END_MAERKER 추가해서 가독성
    SET OPTIMIZER_TRACE_MAX_MEM_SIZE = 1000000; // 옵티마이저 로그의 최대 메모리 크기 지정
    
    SELECT * FROM employees ORDER BY last_name LIMIT 1000000, 1; (OFFSET, LIMIT)
    
    SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE \G
    ```
    
    - sort_mode 필드 분석
        - <sort_key, rowid> : 정렬 키와 레코드 Row ID만 읽어서 최적화 정렬 = 투 패스 정렬
        - <sort_key, additional_fileds> : 정렬 키와 레코드 전체를 읽어서 정렬 = 싱글 패스 정렬
- 싱글 패스 정렬
    - 레코드 전체를 메모리로 가져오기 때문에 정렬 결과 그대로 리턴하면됨
- 투 패스 정렬 (두번 디스크 접근)
    - 정렬에 필요한 col만 읽어서 정렬하고, 디스크에 한 번 더 접근해서 나머지 컬럼도 붙여서 리턴
    - 메모리는 절약되지만, 느리다 → 보통 싱글 패스 이용
    - 투 패스 정렬이 필수인 상황
        - 레코드 크기가 `max_length_for_sort_data` 보다 큼
        - 컬럼 데이터가 크고 가변적 (BLOB이나 TEXT타입)

### 조인 Nested-loop 방식 vs 조인 버퍼 이용

- Nested-loop 방식
    - 먼저 드라이빙 테이블을 선택
    - 드라이빙 테이블의 조건에 맞는 record만 탐색
    - 탐색 결과를 1 by 1으로 드리븐 테이블에서 record를 탐색
        - 드리븐 테이블에서 탐색하는 컬럼에 인덱싱이 되어 있으면 속도 상승
- 조인 버퍼 이용
    - 먼저 드라이빙 테이블을 선택
    - 드라이빙 테이블의 조건에 맞는 record만 탐색
    - 탐색 결과의 일부를 조인 버퍼에 저장하고 드리븐 테이블에서 탐색하는데, 조인 버퍼에 여러 개 원소 중에서 일치하는 결과가 있는지 확인한다
        - 테이블 풀스캔 횟수가 줄어드는 장점
- 드라이빙 테이블 선택 조건
    - 조건절에서 인덱스로 시간을 줄일 수 있는 테이블 선택

### 정렬 처리 방법

- 속도 ) 인덱스 이용 > 조인에서 드라이빙 테이블만 정렬 > 조인 임시 테이블을 다시 정렬
- 인덱스 이용 정렬
    - 조건1 ) ORDER BY 절이 드라이빙 테이블의 인덱싱 컬럼이어야됨
    - 조건2 ) WHERE 절이 ORDER BY와 같은 인덱스로 처리 가능해야됨
    - 만약 조인되는 컬럼도 인덱싱이 되어 있다면, Nested-Loop 방식으로 빠른 처리 가능
- 드라이빙 테이블의 결과를 정렬
    - 조건 ) ORDER BY 절이 드라이빙 테이블의 인덱싱이 안된 경우
    - 인덱싱이 되어있는 Where 절의 컬럼으로 트리를 탐색하고
    - 탐색 결과를 ORDER BY절의 컬럼으로 FILE SORT를 진행
    - 정렬 후 드리븐 테이블과 조인
- 임시 테이블을 이용한 정렬
    - 조건 ) ORDER BY 절이 드리븐 테이블의 컬럼인 경우
    - 인덱싱이 되어있는 Where 절의 컬럼으로 트리를 탐색하고
    - 탐색 결과를 먼저 조인
    - 조인한 결과를 임시 테이블로 만들고, 임시 테이블을 드리븐 테이블의 컬럼으로 정렬

### 쿼리처리 방식

- 스트리밍 방식
    - 첫번째 레코드 탐색 결과를 얻자마자 클라이언트한테 전달
    - 풀 테이블 스캔의 첫번째 레코드 결과는 비교적 빨리 가능
    - LIMIT으로 반환 데이터를 줄이면 쿼리 실행 속도 감소
- 버퍼링 방식
    - ORDER BY나 GROUP BY절은 첫번째 레코드만 빨리 얻을 수 없다
    - 모든 결과를 조회한 후 첫 번째 결과를 얻을 수 있기 때문에 LIMIT으로 반환 데이터를 줄이더라도 속도 성능의 차이가 없다