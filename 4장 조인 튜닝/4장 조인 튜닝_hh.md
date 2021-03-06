# Chapter 4. 조인 튜닝

## NL 조인

- NL조인은 인덱스를 이용한 조인 방식.
- Inner 쪽 테이블은 인덱스를 사용해야 함.
- NL 조인 실행계획
    
    ```sql
    -- 위쪽 사원 테이블 기준으로 아래쪽 고객 테이블과 NL 조인 함.
    
    Excution Plan
    ------------------------------------
    0      SELECT STATEMENT Optimizer=ALL_ROWS
    1  0     NESTED LOOPS
    2  1      TABLE ACCESS (BY INDEX ROWID) OF '사원' (TABLE)
    3  2        INDEX (RANGE SCAN) OF '사원_X1' (INDEX)
    4  3      TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE)
    5  4        INDEX (RANGE SCAN) OF '고객_X1' (INDEX)
    ```
    
- 조인 액세스 횟수가 많을수록 성능이 느려짐.
- 조인 액세스 횟수는 Outer 테이블을 읽고 필터링한 결과 건수에 의해 결정됨.
- 랜덤 액세스 위주의 조인 방식이며, 조인을 한 레코드씩 순차적으로 진행함.
- NL 조인은 소량 데이터를 주로 처리하거나 부분범위 처리가 가능한 온라인 트랜잭션 처리(OLTP) 시스템에 적합한 조인 방식임.

## 소트 머지 조인

- 조인 컬럼에 인덱스가 없을 때, 대량 데이터 조인이어서 인덱스가 효과적이지 않을 떄, 옵티마이저는 NL 조인 대신 소트 머지 조인이나 해시 조인을 선택함.
- 소트 머지 조인(Sort Merge Join)은 아래 두 단계로 진행함.
    - 소트 단계 : 양쪽 집합을 조인 컬럼 기준으로 정렬함.
    - 머지 단계 : 정렬한 양쪽 집합을 서로 머지(Merge)함.
- 소트 머지 조인은 양쪽 테이블로부터 조인 대상 집합(조인 조건 이외 필터 조건을 만족하는 집합)을 일괄적으로 읽어 PGA(또는 Temp 테이블스페이스)에 저장한 후 조인함.

### SGA vs PGA

- SGA (System Global Area / Shared Global Area) : 공유 메모리 영역인 SGA에 캐시된 데이터는 여러 프로세스가 공유할 수 있음.
    - 여러 프로세스가 공유할 수 있지만, 동시에 액세스할 수는 없음.
- PGA (Process/Program/Private Global Area) :오라클 서버 프로세스는 SGA에 공유된 데이터를 읽고 쓰면서, 동시에 자신만의 고유 메모리 영역을 가짐.
    - 같은 양의 데이터를 읽더라도 SGA 버퍼캐시에서 읽을 때보다 훨씬 빠름.

## 해시 조인

- 해시 조인(Hash Join)도 소트 머지 조인처럼 두 단계로 진행됨.
    
    ① Build 단계 : 작은 쪽 테이블(Build Input)을 읽어 해시 테이블(해시 맵)을 생성함.
    
    ② probe 단계 : 큰 쪽 테이블(Probe Input)을 읽어 해시 테이블을 탐색하면서 조인함.
    
- Hash Area에 생성한 해시 테이블(= 해시 맵)을 이용한다는 점만 다를 뿐 해시 조인도 조인 프로세싱 자체는 NL 조인과 같음.
- 해시 조인이 인덱스 기반의 NL 조인보다 빠른 이유는 해시 테이블을 PGA 영역에 할당하기 때문.
- 대량 데이터를 조인할 때 일반적으로 해시 조인이 더 빠름.
- 해시 조인은 NL 조인처럼 조인 과정에서 발생하는 랜덤 액세스 부하가 없고, 소트 머지 조인처럼 양쪽 집합을 미리 정렬하는 부하도 없음.

### 조인 메서드 선택 기준

1. 소량 데이터 조인할 때 → NL 조인
2. 대량 데이터 조인할 때 → 해시 조인
3. 대량 데이터 조인인데 해시 조인으로 처리할 수 없을 때, 즉 조인 조건식이 등치(=) 조건이 아닐 때(조건식이 아예 없는 카테시안 곱 포함) → 소트 머지 조인

### 서브쿼리와 조인

- 메인쿼리와 서브쿼리 간에는 부모와 자식이라는 종속적이고 계층적인 관계가 존재함.
- 서브쿼리는 메인 쿼리에 종속되므로 단독으로 실행할 수 없음.
