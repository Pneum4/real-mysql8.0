# week-07

# 이해를 못한부분

### 1. table pullout 최적화는 서브쿼리 부분이 unique 인덱스나 프라이머리 키 룩업으로 결과가가 1건인 경우에만 사용가능하다?

```sql
SELECT * 
FROM city 
WHERE CountryCode IN (
  SELECT Code 
  FROM country 
  WHERE Continent = 'Asia'
);
// talbe-out 최적화
SELECT *
FROM country a
JOIN city b ON a.Code = b.CountryCode
WHERE a.Continent = 'Asia';

// GPT 설명
🧠 5️⃣ 결론
MySQL 옵티마이저가 Table Pull-out을 자동으로 적용하지 않는 이유는 바로 이거예요.

👉 IN은 “집합(Set)” 개념이므로 중복이 제거된 결과를 반환하지만,
JOIN은 “곱(Product)” 개념이라 중복된 매칭이 생기면 결과가 달라질 수 있다.

따라서 옵티마이저는 다음처럼 판단합니다:

“서브쿼리가 반드시 1건만 반환한다는 확신(UNIQUE KEY / PK lookup)이 없는 한,
JOIN으로 변환(pull-out)하지 않는다.”

-> Duplicated Weed-Out 최적화를 사용가능할듯? 
```

### 2. 비상관관계인 서브쿼리는 FirstMatch 최적화를 사용하면 풀 테이블 스캔을 한다?

```sql
🔍 3️⃣ 그런데 만약 “억지로” JOIN 형태로 바꾸면?

가정을 해보죠.
비상관 서브쿼리를 JOIN으로 재작성하려 하면,
ON 조건이 존재하지 않게 됩니다.

예를 들어:

SELECT *
FROM city c
JOIN (SELECT Code FROM country WHERE Continent='Asia') t;

→ ON 조건이 없습니다.
→ 결과적으로 카테시안 곱(Cartesian Product) 이 되어버립니다.
→ city 전체 × t 전체 = N×M 결과.

따라서 옵티마이저는 이런 JOIN rewrite를 절대 허용하지 않습니다.
(즉, Pull-out 불가능 / FirstMatch도 불가능)
```

### 3. 해시 조인은 중첩 루프 조인(NLP) 보다 반응 속도는 느리고 전체 처리속도는 빠르다. 이유가 뭘까??

- 해시 테이블을 생성해야 조인이 가능하므로 테이블 생성시간때문에 반응 시간이 느리고
- 해시 테이블을 생성했다면, 1대1로 찾는 NLP보다 해싱을 통해 빠른 검색이 가능할 것이다.

### 4. 해시 조인에서 첫번째 해시 테이블의 목적과, 두번째 해시 테이블의 목적은?

- 1차 해시 테이블 : 같은 해시값의 데이터만 묶어 디스크로 분리
- 2차 해시 테이블 : 각 파티션을 메모리로 로드 후 해시 lookup 수행 + 조인된 결과 생성

# .3

- 리마인드
    - 인덱싱에서 중복값은 pk에 정렬되어서 저장되고, 중복값을 모두 조회하면 pk 순으로 가져올 수 있다
    - 인덱싱이 되어있는 컬럼 2개를 OR 연산하게 되면 PQ에 넣어서 합치게 되는데
        - → PK정렬 되어있기 때문에 중복 제거가 가능하다 (UNION 연산이 가능)

## .1 인덱스 머지

### `index_merge_sort_union`

- 예시
    
    ```sql
    where first_name='Matt' or hire_date between '1' and '2'
    ```
    
    - 첫번째 조건은 pk 정렬이 되어서 조회됨
    - 두번째 조건은 1인경우 + 2인경우 각각은 정렬되어도 합치면 정렬이 깨짐
    - → 두번째 조건을 정렬하고 PQ에 Union 연산을 진행하는 것이 `index_merge_sort_union` 이다.

### 세미조인

- 실제 조인없이 다른 테이블에서 레코드 확인만 하는 것
- 만약 최적화를 안한다면?
    - from 테이블이 30만건이라면 풀테이블 스캔을 하고, subquery도 30만번 수행된다

<aside>
IN 서브쿼리 최적화 5대장

</aside>

- where 절에 = (subquery) 이나 IN (subquery) 형태일때는 다음과 같은 세미 조인 최적화가 가능
    - Table Pull-out
        - 비상관 서브쿼리 (unique)
    - Duplicate Weed-out
        - 상관/비상관 서브쿼리 둘 다 → 다른 최적화때문에 잘 안쓰임
    - First Match
        - 상관 서브쿼리 (중복 허용)
    - Loose Scan
        - 비상관 In 서브쿼리
        - 서브쿼리는 많고, 아우터쿼리는 적고
    - Materialization
        - 비상관 서브쿼리 (group by 도 허용)

### Table Pull-out 최적화

- 특징
    - 사용하면 항상 세미조인보다 성능이 최적화
    
    > 최대한 서브쿼리를 조인으로 풀어서 사용해라
    > 
- 조건
    
    
    | 조건 | 설명 |
    | --- | --- |
    | 1️⃣ 비상관 서브쿼리 | 서브쿼리 안에서 외부 쿼리의 컬럼을 참조하지 않아야 함 |
    | 2️⃣ 서브쿼리의 FROM 절이 하나의 테이블로만 구성 | `JOIN`, `GROUP BY`, `DISTINCT`, `LIMIT` 등이 포함되면 pull-out 불가 |
    | 3️⃣ 서브쿼리 결과가 외부 쿼리와 1:1 매핑 가능 | 즉, 단순 필터링 역할일 때만 변환됨 |
    | 4️⃣ 서브쿼리의 중복 인덱스는 불가능 | 프라이머리키 룩업으로 중복값이 발생하면 안됨 |
- GPT가 말하는 조건
    - 비상관 서브쿼리인 경우 가능 (서브쿼리가 바깥 쿼리 컬럼을 참조하지 않음)
        
        ```sql
        SELECT * 
        FROM city c 
        WHERE CountryCode IN (
          SELECT Code 
          FROM country 
          WHERE Population > c.Population
        );
        // 상관 서브쿼리는 불가능
        // 상관 서브쿼리는 서브쿼리를 먼저 실행하는 것이 불가능하기 때문인듯?
        ```
        
- =과 in 은 inner join 과 비슷하기 때문에 가능하다
- 과정
    - 비교하는 테이블과 subquery테이블을 조인하고
    - where 조건으로 필요한 record만 남긴다

```sql
select * from ta a
	where a.id in (select b.id from tb b where b.name='hello')

// 최적화 결과
select a.~ ...
	from ta a
	join tb b
	where a.id = b.id
		and b.name = 'hello'
```

- 최적화 여부 확인법
    - Explain 결과 두 테이블의 id가 같으면 서브쿼리가 아닌 조인으로 처리했음을 의미함

### Firstmatch 최적화

- 서브쿼리의 결과가 `where a.id in (1, 1, 2, 2, 3)` 처럼 중복되어서 나오는 경우에 최적화
    - 마치 exists 처럼 작동하지만 조인으로 작동한다는 차이가 있따
    - (서브쿼리 ← 아우터쿼리) 조인, 찾으면 서브쿼리 정보만 빠르게 반환
- 서브쿼리에 group by 나 집합함수 사용시 못씀
- GPT가 말하는 조건
    - 상관 서브쿼리인 경우 가능
        
        ```sql
        -- FirstMatch 적용 가능 (상관 서브쿼리)
        -- 직원이 월급을 받은 적이 있는지 확인 (1번이라도 받았다면 스킵하는거임)
        SELECT e.id, e.name
        FROM employees e
        WHERE e.id IN (
            SELECT s.emp_id
            FROM salaries s
            WHERE s.emp_id = e.id   -- 상관 조건
        );
        
        -- 마치 exists 처럼 작동한다고 함
        SELECT e.id, e.name
        FROM employees e
        WHERE EXISTS (
            SELECT 1
            FROM salaries s
            WHERE s.emp_id = e.id
        );
        ```
        
- 첫번째 컬럼을 indexing해서 where에 맞는 데이터만 찾고
    - 조인컬럼인 id로 두번째 컬럼을 조인할때 조인 컬럼을 1대1로 비교하면서
    - 서브쿼리의 where절을 1번이라도 만족하면 그대로 스킵하고 다음 조인 컬럼을 확인
- 쿼리
    
    ```sql
    select * from ta a where a.col1 = 'one'
    	and a.id in ( select b.id from tb b where b.col2 between 1 and 2 )
    // 서브 쿼리 결과 id(1)col2(1) + id(1)col2(2) 처럼 id가 중복되어서 반환될때
    // 만족하는 id가 1개라도 존재하면 해당 id는 스킵 -> 다음 id확인
    ```
    
- 최적화 여부 확인법
    - Explain 결과 두 테이블의 id가 같으면 서브쿼리가 아닌 조인으로 처리했음을 의미함
    - 거기에 extra에도 FirstMatch라고 보임

### Loosescan 최적화

- outer 테이블은 데이터가 적고, subquery는 데이터가 많을때 사용
- 과정
    - subquery에서 반환하는 col기준으로 loose index 스캔을 먼저 진행
    - outer 테이블의 데이터를 매칭시키고 skip을 진행한다
- gpt
    - 비상관 서브쿼리에서 in인 경우에 사용
        
        ```sql
        SELECT e.id, e.name
        FROM employees e
        WHERE e.dept_id IN (
            SELECT d.id
            FROM departments d
        );
        // departments.id에 인덱스가 있다고 가정하면
        // MySQL은 departments를 풀스캔하지 않고
        // 인덱스를 타고 각 dept_id(d.id)를 한 번씩만 읽어옴.
        // 내부적으로 조인으로 처리 (서브쿼리 <- 아우터쿼리 조인)
        ```
        
- 최적화 여부 확인법
    - Explain 결과 두 테이블의 id가 같으면 서브쿼리가 아닌 조인으로 처리했음을 의미함
    - 거기에 extra에도 LooseScan이라고 보임

### Materialization 최적화

- 비상관 서브쿼리에서 in인 경우에 사용
- 서브쿼리에 group by 나 집합함수 사용해도 가능
- 과정
    - Materialization은 서브쿼리 결과를 **한 번만 계산해서 임시 테이블**에 저장
    - 그 테이블을 인덱스 탐색하듯 사용
- gpt
    
    ```sql
    -- 특정 부서에 속한 직원 찾기
    SELECT e.id, e.name
    FROM employees e
    WHERE e.dept_id IN (
        SELECT d.id
        FROM departments d
        WHERE d.region = 'APAC'
    );
    ```
    
- 최적화 여부 확인법
    - Explain 결과 두 테이블의 id가 다르면 임시테이블을 사용한것임
    - 거기에 select_type에도 MATERIALIZED이라고 보임

### Duplicated Weed-out 최적화

- 서브쿼리의 col 결과가 중복이 나오는 경우 (1, 1, 2, 2, 3) 사용하는데
    - firstmatch 최적화와 적용 조건의 차이를 모르겟음 (둘다 상관 서브쿼리임)
    - 아마 다른 최적화가 가능하면 다른거 쓰는 듯?
- 서브쿼리에 group by 나 집합함수 사용시 못씀
- 과정
    - inner join을 수행하고
    - where in 에서 보는 조건으로 group by를 수행해서 중복 제거
- gpt
    
    ```sql
    -- 각 직원의 부서가 특정 지역에 속해 있는지 확인
    SELECT e.id, e.name
    FROM employees e
    WHERE e.dept_id IN (
        SELECT d.id
        FROM departments d
        WHERE d.region = e.region   -- ❗ 상관 관계
    );
    
    -- inner join으로 바꿈
    SELECT DISTINCT e.id, e.name
    FROM employees e
    JOIN departments d
      ON e.dept_id = d.id
     AND e.region  = d.region;
    ```
    
- 최적화 여부 확인법
    - Explain 결과 두 테이블의 id가 같으면 서브쿼리가 아닌 조인으로 처리했음을 의미함
    - 거기에 Extra에도 Start temporary + End temporary이라고 보임

### `Condition_Fanout_Filter`

- 쿼리
    
    ```sql
    select *
    from employee e
    	inner join salaries s on s.emp_no=e.emp_no
    where e.first_name='Matt' // 인덱스 존재
    	and e.hire_date between '1' and '2' // 인덱스 존재
    ```
    
- 옵티마이저가 where 조건으로 데이터를 걸러낼때, 일부분만 미리 해보고 전체 데이터에서 조건으로 걸러지는 데이터 비율을 예측한다
    - 단순 통계로는 예측이 틀릴거라서 일부분 실제 통계로 필터링진행
- filter가 없다면 e.first_name만 보고 예측을 하는데
- filter가 있다면 e.hire_date까지 고려해서 예측을 한다 (인덱스 필수)
- 그리고 예측한 내용을 바탕으로 조인할때 드라이빙-드리븐 테이블을 최적화 할 수도 있다
    
    ```sql
    옵티마이저가 employee 테이블의 통계를 봄
    first_name='Matt' 으로 필터하면 예를 들어 10% 남는다.
    hire_date BETWEEN ... 으로 필터하면 예를 들어 20% 남는다.
    
    두 조건이 서로 독립적이라고 가정하면
    전체 selectivity = 10% × 20% = 2% (0.1 × 0.2)
    하지만 실제 데이터는 상관관계가 있을 수 있음
    (예: Matt라는 이름은 특정 연도에 몰려 있을 수도 있음)
    
    이때 옵티마이저는 condition_fanout_filter 단계를 수행해
    first_name='Matt' 인 인덱스 범위를 일부 실제로 스캔
    그 결과에서 hire_date 조건이 얼마나 더 걸러지는지를 “샘플링 기반으로 추정”
    이렇게 보정된 필터 비율을 사용해 조인 순서 및 접근 경로(index join order) 를 다시 결정
    ```
    

### 파생 테이블 머지( Derived_Merge)

- Derived Table
    - From절에 사용된 파생 테이블 (인라인뷰를 말하는듯?)
    - 예전에는 최적화 없이 파생 테이블은 임시로 메모리나 디스크에 저장했음
- 그러나 아우터 쿼리와 서브쿼리를 merge할 수 있다면 임시 테이블을 만들 필요가 없음
- 쿼리
    
    ```sql
    select * from (
    	select * from employees where first_name='Matt'
    	) derived_table
    	where derived_table.hire_date='1'
    // 머지 최적화
    select *
    	from derived_table
    	where first_name='Matt' and erived_table.hire_date='1'
    ```
    
    - 예시는 같은 테이블에서 where조건만 나눠져 있는 경우임
- 머지가 가능하면 하는데, 안되는 경우가 있다
    - 서브쿼리에서 집계함수, 윈도우 함수 사용
    - 서브쿼리 Distinct, Group By, limit, union 등
    - 등등 기존 테이블을 이용한 새 테이블이 반
    - 
    - 
    - 
        - 환된 경우인듯?

### 인비저블 인덱스(Use_Invisible_Index)

- 옵티마이저가 인덱스를 사용하지 못하게 숨기는것
- 쿼리
    
    ```sql
    alter table employees alter index idx invisible // 옵티마이저가 사용 못함
    alter table employees alter index idx visible // 사용가능
    
    // 이렇게 설정하면 invisible인덱스도 사용함
    set optimizer_switch='use_invisible_indexes=on' 
    ```
    

### 스킵 스캔 (Skip_Scan)

- 인덱스 스킵 스캔을 말하는 것이다
- Where 절의 선행 컬럼의 소수의 유니크 값을 가질때에만 효율적
- 다수의 유니크 값을 선행컬럼이 가진다면, 비효율
- 쿼리
    
    ```sql
    select * from employee
    	Where gender='M' and birth_date >= '1'
    //gender의 경우 M,F 밖에 없기때문에 인덱스 스킵 스캔 가능
    ```
    

### 해시조인 (Hash_Join)

- 드리븐 테이블이 인덱스가 없을 때 사용하는 Blocked Nested Loop Join 과 비슷한데, 해싱을 통해서 빠른 검색이 가능하다
    - 그래서 메모리 할당 공간으로 join_buffer_size 변수를 공유
    - 해당 크기 이상의 해시 테이블을 가지려면 디스크 공간을 사용해야한다.
- NLP보다 반응 속도는 느린데, 처리속도는 빠르다
- 드라이브 테이블 = 빌드 테이블 / 드리븐 테이블 = 프로브 테이블
- 1차 캐싱만 하는경우 (In-memory Hash Join)
    - 만약 해시테이블이 메모리에 전부 올릴 수 있는 경우
    - 1차 캐싱으로 조인 완료
- 2차 캐싱을 하는 경우 (Grace Hash Join)
    - Grace Hash Join 방식을 사용해야 한다
    - 메모리에 캐시 테이블을 2번 만든다
        - 첫번째 캐시 테이블 - 빌드 테이블과 프로브 테이블의 동일한 해시값을 가지는 데이터를 동일 한 청크로 넣고, 같은 청크 크기로 파티셔닝을 진행
        - 두번째 캐시 테이블 - 빌드 테이블과 프로브 테이블의 청크를 1개씩 메모리에 로드하면서 해쉬  테이블로 조인 결과를 생성함

### 인덱스 정렬 선호 (Prefer_Ordering_Index)

- 쿼리
    
    ```sql
    select *
    	from employees
    	where hire_date between '1' and '2'
    	order by emp_no; #pk
    ```
    
- 보통 이 쿼리는 인덱스로 빠르게 where조건을 적용하고, 적은 데이터를 정렬하는 것이 빠르다
- 그런데 옵티마이저가 order by 절에 컬럼인 emp_no의 primary 인덱스에 가중치를 주고 정렬을 먼저하는 경우가 있다 (풀스캔해야되어서 비효율적)
- 그래서 `prefer_ordering_index` 옵션을 비활성화해서 ordering_index에 큰 가중치를 주지 않도록 설정할 수 있다

## 옵티마이저 힌트

### 🧩 2️⃣ 문법 형식

MySQL 5.7.7 이후부터 표준 형식은 다음과 같습니다:

```sql
SELECT /*+ HINT_NAME(param, ...) */ ...
FROM ...
```

📌 즉, `/*+` 와 `*/` 사이에 옵티마이저 힌트를 넣습니다.

일반 주석처럼 보이지만 실제로는 옵티마이저가 인식합니다.

---

### STRAIGHT_JOIN과 차이점

- 힌트는 옵티마이저가 선택적으로 무시할 수 있음
- STRAIGHT_JOIN은 반드시 선택됨

### 🧠 3️⃣ 힌트 종류별 예시

① **조인 순서 제어**

```sql
SELECT /*+ JOIN_ORDER(e, s) */ *
FROM employee e
JOIN salaries s ON s.emp_no = e.emp_no;
```

👉 옵티마이저에게 “employee → salaries 순서로 조인하라”고 강제

---

② **드라이빙 테이블 고정**

```sql
SELECT /*+ LEADING(e) */ *
FROM employee e
JOIN salaries s ON s.emp_no = e.emp_no;
```

👉 `employee`를 드라이빙 테이블로 강제

---

③ **특정 인덱스 사용**

```sql
SELECT /*+ INDEX(e idx_firstname) */ *
FROM employee e
WHERE e.first_name='Matt';

```

👉 옵티마이저가 `idx_firstname` 인덱스를 반드시 사용하도록 지시

---

④ **특정 최적화 비활성화**

```sql
SELECT /*+ NO_ICP(e) */ *
FROM employee e
WHERE e.hire_date BETWEEN '1990-01-01' AND '1999-12-31';

```

👉 ICP (Index Condition Pushdown) 최적화를 사용하지 않게 함

---

⑤ **서브쿼리 최적화 제어**

```sql
SELECT /*+ SEMIJOIN(DUPSWEEDOUT) */ *
FROM employee e
WHERE e.emp_no IN (SELECT s.emp_no FROM salaries s);
```

👉 `IN (subquery)` 최적화 전략 중 `Duplicate Weed-out` 방식을 지정

(`SEMIJOIN`, `MATERIALIZATION`, `LOOSESAN`, `FIRSTMATCH` 등의 방식 선택 가능)

---

⑥ **조인 알고리즘 선택**

```sql
SELECT /*+ BNL(s) */ *
FROM employee e
JOIN salaries s ON s.emp_no = e.emp_no;
```

👉 조인 시 **Block Nested Loop Join** 사용 강제

(`BNL`, `BKA`, `HASH_JOIN` 등 가능)