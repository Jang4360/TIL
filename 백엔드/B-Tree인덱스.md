# B-Tree 인덱스는 왜 조회를 빠르게 만들지만 쓰기를 비싸게 만드는가

주문 목록 API가 느려져서 `created_at`에 인덱스를 추가했다.

배포 직후 조회는 빨라졌다. 그런데 몇 시간 뒤부터 주문 생성 TPS가 떨어지고, 특정 시간대에는 결제 완료 트랜잭션이 간헐적으로 대기한다. 더 이상한 점은 단순 `INSERT`인데도 DB lock wait이 늘어난다는 것이다.

처음에는 이렇게 생각하기 쉽다.

> 인덱스는 조회 최적화 도구니까 쓰기와는 별개 아닌가?

하지만 운영 DB에서 인덱스는 읽기 경로만 바꾸지 않는다.

데이터가 추가되고 수정되고 삭제될 때마다 인덱스도 함께 유지되어야 한다. 그 과정에서 page 접근, page split, tree height, buffer pool, redo log, range lock, selectivity, covering 여부가 쓰기 비용과 잠금 범위를 바꾼다.

핵심 질문은 이것이다.

> 인덱스는 조회를 빠르게 만들지만 왜 쓰기 비용과 잠금 범위를 함께 바꾸는가?

B-Tree 인덱스는 읽을 때는 탐색 범위를 줄인다. 하지만 쓸 때는 정렬된 자료구조를 계속 유지해야 한다. 그래서 page 변경, tree 재균형, 보조 인덱스 갱신, range lock이 추가 비용으로 따라온다.

---

## 먼저 구조를 잡자

B-Tree 인덱스는 값을 정렬된 형태로 저장해두고, 루트에서 리프까지 내려가며 원하는 key 범위를 찾는 구조다.

DB에서 말하는 B-Tree 계열 인덱스는 단순한 메모리 객체가 아니다. 기본적으로 디스크에 저장되는 자료구조이고, 실행 중 자주 쓰이는 일부 page가 메모리의 buffer pool 또는 shared buffer에 캐시된다.

짧게 보면 이렇게 나뉜다.

```text
disk
  - B-Tree 인덱스 전체
  - table data page
  - undo/redo 관련 파일

memory
  - 자주 접근한 index page
  - 자주 접근한 data page
  - dirty page
  - 실행 계획과 통계 일부
```

여기서 page는 DB가 데이터를 읽고 쓰는 기본 단위에 가깝다.

MySQL InnoDB 기준으로 인덱스 page의 기본 크기는 보통 16KB다. 하나의 page 안에는 key 하나만 들어가는 것이 아니라, 수십 개에서 수백 개의 index entry가 들어간다. key가 작으면 더 많이 들어가고, 긴 문자열이나 여러 컬럼이 섞이면 덜 들어간다.

조회 요청은 대략 이렇게 움직인다.

```text
1. 조건절의 컬럼과 인덱스 key 순서를 비교한다.
2. root page에서 시작해 branch page를 따라간다.
3. leaf page에서 조건에 맞는 key를 찾는다.
4. 필요한 row를 table 또는 clustered index에서 다시 읽는다.
5. 조건이 range라면 leaf page를 옆으로 따라가며 scan한다.
6. 필요한 컬럼이 인덱스에 모두 있으면 table 접근을 생략한다.
```

예를 들어 다음 쿼리를 보자.

```sql
SELECT order_id, status
FROM orders
WHERE user_id = 10
  AND created_at >= '2026-06-01'
ORDER BY created_at DESC
LIMIT 20;
```

이 쿼리에 `(user_id, created_at)` 인덱스가 있으면 DB는 `user_id = 10` 구간으로 바로 들어간 뒤 `created_at` 범위를 따라간다.

반대로 `(created_at, user_id)` 인덱스만 있으면 특정 날짜 이후의 넓은 범위를 먼저 훑고 그 안에서 `user_id`를 필터링할 수 있다.

같은 컬럼을 포함해도 key 순서가 다르면 range scan의 폭이 달라진다.

쓰기 요청은 더 많은 일을 한다.

```text
1. table 또는 clustered index에 row를 쓴다.
2. 모든 secondary index에 새 key를 추가한다.
3. key가 들어갈 leaf page를 찾는다.
4. leaf page에 빈 공간이 없으면 page split을 수행한다.
5. split 결과가 상위 page에 전파될 수 있다.
6. 변경된 page를 redo log, undo, dirty page 관리 대상에 올린다.
7. 격리 수준과 조건에 따라 record lock 또는 gap/range lock을 잡는다.
```

그래서 인덱스는 조회용 부속품이 아니다.

쓰기 경로에 함께 참여하는 정렬된 복제 구조에 가깝다. 인덱스가 하나 늘 때마다 조회 후보 경로는 늘지만, 쓰기 때 갱신해야 하는 구조도 하나 늘어난다.

---

## 내부에서는 무슨 일이 일어나는가

### B-Tree page는 탐색 횟수를 줄이지만 변경 단위를 만든다

인덱스를 추가한 뒤 조회는 빨라졌는데 쓰기 지연이 늘어나는 상황이 있다.

원인은 B-Tree가 row 단위로만 움직이지 않고 page 단위로 탐색되고 변경되기 때문이다.

B-Tree의 각 노드는 보통 고정 크기 page로 저장된다. root page에는 큰 범위의 경계 key가 있고, branch page는 더 작은 범위로 나누며, leaf page에는 실제 index key와 row 위치 정보가 있다.

조회는 root에서 leaf까지 내려가면 된다.

전체 row 수가 많아져도 fan-out이 크면 tree height는 크게 증가하지 않는다. 수천만 건의 row가 있어도 tree height는 몇 단계에 머무를 수 있다. 그래서 equality lookup은 전체 테이블을 훑는 대신 몇 개 page 접근으로 후보 위치를 찾는다.

이게 인덱스가 조회를 빠르게 만드는 핵심이다.

하지만 쓰기에서는 그 leaf page가 변경 대상이 된다.

새 key를 정렬 순서에 맞춰 넣어야 하고, page에 공간이 없으면 page split이 발생한다.

page split은 단순히 한 page를 둘로 나누는 일이 아니다.

```text
Before
  Page A: 10, 20, 30, 40, 50, 60

Insert 35

After
  Page A: 10, 20, 30
  Page B: 35, 40, 50, 60
```

여기서 끝나지 않는다.

부모 page에는 "35 이상은 Page B로 가라"는 경계 key가 추가되어야 한다. 부모 page도 꽉 차 있으면 split이 상위로 전파될 수 있다.

page split은 다음 비용을 만든다.

- 새 page 할당
- 기존 key 일부 이동
- 부모 page 수정
- 필요하면 부모 page split
- redo log 기록
- 여러 page dirty 처리
- 동시성 제어를 위한 latch 경합 가능성

랜덤한 값으로 인덱스를 만들면 page split이 더 자주 발생할 수 있다.

UUID를 primary key로 쓸 때 insert 성능이 나빠질 수 있는 이유도 여기에 있다. 반대로 `AUTO_INCREMENT`처럼 증가하는 key는 보통 오른쪽 끝 page에 append되므로 page 이동이 상대적으로 적다.

정리하면 이렇다.

> B-Tree는 읽을 때는 몇 단계만 내려가면 되게 해주지만, 쓸 때는 정렬 상태를 유지하기 위해 page를 찾아가고 필요하면 나눠야 한다.

### B-Tree는 디스크에 있고, 일부 page만 메모리에 올라온다

B-Tree 인덱스 전체가 항상 메모리에 있는 것은 아니다.

전체 인덱스는 디스크의 tablespace 또는 데이터 파일에 저장된다. 쿼리가 실행되면 필요한 index page와 data page가 메모리의 buffer pool에 올라온다. 이미 buffer pool에 있으면 디스크 I/O 없이 메모리에서 읽는다.

```text
disk
  orders.ibd
    - clustered index page
    - secondary index page

memory
  InnoDB buffer pool
    - 최근 읽은 index page
    - 최근 읽은 data page
    - 수정됐지만 아직 flush되지 않은 dirty page
```

buffer pool은 단순 캐시가 아니다.

`UPDATE`가 실행되면 DB는 page를 buffer pool로 읽어오고, 메모리 안의 page를 수정한다. 이 page는 dirty page가 된다. 변경 내용은 redo log에 먼저 기록되고, 실제 data page는 checkpoint나 background flush를 통해 나중에 디스크로 반영된다.

그래서 쓰기는 보통 이렇게 움직인다.

```text
1. 필요한 page를 buffer pool에서 찾는다.
2. 없으면 disk에서 읽어온다.
3. buffer pool 안의 page를 수정한다.
4. redo log에 변경 기록을 남긴다.
5. page는 dirty page가 된다.
6. 나중에 background thread가 dirty page를 disk로 flush한다.
```

이 구조 덕분에 매번 data file을 즉시 쓰지 않아도 된다.

하지만 dirty page가 너무 많이 쌓이면 flush가 몰린다. 이때 쓰기 지연이 순간적으로 튈 수 있다. 인덱스가 많으면 수정해야 하는 page도 늘어나고, dirty page와 redo log pressure도 함께 커질 수 있다.

### clustered index와 secondary index는 다르게 동작한다

InnoDB에서는 primary key가 clustered index다.

clustered index의 leaf page에는 row 데이터 자체가 저장된다. 즉 primary key로 row를 찾으면 leaf page에 도달하는 순간 row 데이터를 바로 읽을 수 있다.

반면 secondary index의 leaf page에는 secondary key와 primary key 값이 들어 있다.

예를 들어 다음 테이블을 보자.

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  age INT NOT NULL,
  INDEX idx_age(age)
);
```

`idx_age`의 leaf entry는 개념적으로 이렇게 볼 수 있다.

```text
age = 20 -> id = 3
age = 20 -> id = 8
age = 21 -> id = 5
age = 30 -> id = 1
```

`WHERE age = 20`이면 DB는 `idx_age`에서 `age = 20`인 entry를 찾고, 거기서 얻은 `id = 3`, `id = 8`을 이용해 clustered index를 다시 탐색한다.

이걸 back to table, bookmark lookup, clustered index lookup처럼 부를 수 있다.

범위 조회도 마찬가지다.

`WHERE age BETWEEN 20 AND 30`이라고 해서 value 하나에 범위 안의 모든 위치 정보가 들어 있는 것이 아니다.

DB는 `age = 20` 근처의 leaf entry로 이동한 뒤, leaf page를 순서대로 따라가며 `age <= 30`인 entry를 스캔한다. 각 entry에서 primary key를 얻고, 필요한 row를 다시 clustered index에서 찾는다.

이 구조 때문에 secondary index 조회는 후보가 적을 때 효과적이다.

후보가 많으면 secondary index를 타고 많은 primary key lookup을 반복해야 한다. 이 경우 sequential scan이 더 나을 수 있다.

또 하나 중요한 점이 있다.

InnoDB secondary index는 primary key를 포함하므로 primary key가 길면 모든 secondary index도 커진다. 그래서 InnoDB에서는 primary key를 짧고 안정적으로 잡는 것이 중요하다.

### tree height는 낮아도 range scan은 넓어질 수 있다

실행 계획에는 index range scan이 찍혀 있는데도 느린 쿼리가 있다.

원인은 B-Tree 탐색 비용과 range scan 비용을 구분하지 않았기 때문이다.

B-Tree에서 특정 시작점까지 가는 비용은 tree height에 가깝다. 하지만 시작점 이후 조건을 만족하는 leaf entry를 얼마나 많이 따라가느냐는 별개의 문제다.

`WHERE created_at >= yesterday` 같은 조건이 전체 row의 40%를 포함한다면, 시작점은 빨리 찾더라도 이후 leaf page를 아주 많이 읽어야 한다.

여기서 중요한 개념이 selectivity다.

selectivity는 조건이 전체 데이터 중 얼마나 좁은 후보를 남기는지를 말한다.

```text
좋은 selectivity
  user_id = 10
  -> 전체의 0.01%

나쁜 selectivity
  status = 'PAID'
  -> 전체의 80%
```

낮은 selectivity 컬럼에 단독 인덱스를 만들면 DB는 인덱스를 타더라도 많은 row를 다시 읽어야 한다.

예를 들어 다음 쿼리를 보자.

```sql
SELECT *
FROM orders
WHERE status = 'PAID';
```

`status` 값이 대부분 `PAID`라면 `status` 인덱스는 별 도움이 되지 않는다.

인덱스 leaf에서 많은 key를 읽고, 각 key마다 table row를 다시 읽는 random access가 생긴다. 이 경우 DB는 차라리 full scan이 낫다고 판단할 수 있다.

그래서 봐야 할 것은 "인덱스를 탔는가"가 아니다.

> 인덱스를 타고 몇 건을 읽어서 몇 건을 버렸는가.

실행 계획에서 access method만 보지 말고 rows estimate, filtered, actual rows, heap/table fetch 수를 같이 봐야 한다.

### 복합 인덱스는 key 순서가 성능을 결정한다

복합 인덱스는 여러 컬럼을 하나의 정렬 기준으로 묶는다.

```sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at);
```

이 인덱스는 먼저 `user_id`로 정렬되고, 같은 `user_id` 안에서 `created_at`으로 정렬된다.

그래서 다음 쿼리에 잘 맞는다.

```sql
SELECT *
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

하지만 다음 쿼리에는 제한적이다.

```sql
SELECT *
FROM orders
WHERE created_at >= ?
ORDER BY created_at DESC
LIMIT 20;
```

왼쪽 컬럼인 `user_id` 조건이 없기 때문이다.

MySQL 문서도 복합 인덱스에서는 leftmost prefix가 중요하다고 설명한다. `(col1, col2, col3)` 인덱스가 있으면 `(col1)`, `(col1, col2)`, `(col1, col2, col3)` 형태의 탐색에 유리하다.

복합 인덱스 순서를 잡을 때는 다음을 같이 본다.

- 동등 조건 컬럼이 있는가
- 범위 조건 컬럼이 있는가
- `ORDER BY`와 같은 순서로 읽을 수 있는가
- `LIMIT`으로 빨리 멈출 수 있는가
- 해당 컬럼의 selectivity가 충분한가
- 쓰기 시 자주 변경되는 컬럼인가

예를 들어 주문 목록 API라면 다음 두 인덱스는 다르게 동작한다.

```sql
INDEX idx_user_created (user_id, created_at)
INDEX idx_created_user (created_at, user_id)
```

`WHERE user_id = ? ORDER BY created_at DESC LIMIT 20`이면 `idx_user_created`가 자연스럽다.

DB는 특정 사용자의 구간으로 바로 들어가서 최신순으로 20개만 읽으면 된다.

반대로 `idx_created_user`를 쓰면 최신 주문 전체를 훑으며 특정 user만 걸러야 할 수 있다.

같은 컬럼을 넣었다고 같은 인덱스가 아니다.

정렬 순서가 곧 탐색 순서다.

### covering index는 table 접근을 줄이지만 쓰기 비용을 늘린다

range scan 자체는 줄였는데 여전히 느린 조회가 남는 경우가 있다.

원인은 인덱스에서 후보를 찾은 뒤 table row를 다시 읽는 비용이 크기 때문이다.

일반적인 secondary index leaf에는 인덱스 key와 row를 찾기 위한 primary key가 들어 있다. 쿼리가 필요한 컬럼을 인덱스가 모두 가지고 있지 않으면 DB는 인덱스에서 후보를 찾은 뒤 원본 row를 다시 읽어야 한다.

후보가 수십 건이면 괜찮다.

하지만 후보가 수만 건이면 table 접근 비용이 커진다.

covering index는 이 비용을 줄인다.

쿼리가 필요한 컬럼을 인덱스 안에 모두 포함시키면 table row 접근을 생략할 수 있다.

```sql
-- 조회 패턴
SELECT order_id, status, created_at
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;

-- 후보 인덱스
CREATE INDEX idx_orders_user_created_cover
ON orders (user_id, created_at, order_id, status);
```

이 경우 DB는 `user_id`로 범위를 좁히고 `created_at` 순서대로 leaf를 읽으면서 `order_id`, `status`까지 인덱스에서 바로 반환할 수 있다.

PostgreSQL은 `INCLUDE`를 사용해 검색 key가 아닌 payload 컬럼을 인덱스에 포함할 수 있다.

```sql
CREATE INDEX idx_orders_user_created_cover
ON orders (user_id, created_at)
INCLUDE (order_id, status);
```

하지만 covering index도 공짜가 아니다.

인덱스 entry가 커지면 한 page에 들어가는 key 수가 줄어든다. page 수가 늘고 cache 효율이 떨어질 수 있다. insert/update/delete 때 더 큰 인덱스를 유지해야 한다.

특히 자주 변경되는 컬럼을 covering 목적으로 넣으면 update 때마다 index maintenance가 발생한다.

covering index는 읽기 hot path에 제한적으로 쓰는 편이 좋다.

"언젠가 필요할 수 있는 컬럼"을 계속 붙이면 조회 하나는 빨라질 수 있지만 전체 쓰기 비용과 저장 공간, buffer pool 경쟁을 키운다.

### index maintenance는 쓰기 하나를 여러 쓰기로 만든다

애플리케이션에서는 row 하나만 insert했다.

하지만 DB 내부 비용은 row 하나처럼 보이지 않는다.

원인은 모든 인덱스가 해당 row의 다른 정렬 사본이기 때문이다.

`orders` 테이블에 다음 인덱스가 있다고 하자.

```sql
PRIMARY KEY (id)
INDEX idx_user_created (user_id, created_at)
INDEX idx_status_created (status, created_at)
INDEX idx_payment_id (payment_id)
INDEX idx_created_at (created_at)
```

주문 하나를 넣으면 DB는 primary 구조뿐 아니라 네 개 secondary index에도 entry를 추가해야 한다.

`status`가 바뀌면 `idx_status_created`도 갱신 대상이다.

`payment_id`가 나중에 채워지는 구조라면 해당 update는 secondary index insert를 동반한다.

인덱스가 많을수록 쓰기 지연은 늘 수 있다.

물론 DB 엔진은 buffer pool, change buffer, WAL/redo log, group commit 같은 장치로 비용을 완화한다. 하지만 durable하게 유지해야 하는 정렬 구조가 늘어난다는 사실은 사라지지 않는다.

인덱스는 많을수록 안전한 것이 아니다.

운영 테이블의 인덱스는 조회 성능과 쓰기 비용 사이의 예산 배분이다.

사용되지 않는 인덱스는 조회에 도움을 주지 않으면서 insert/update/delete 경로에 계속 비용을 부과한다.

### 인덱스는 lock range도 바꾼다

인덱스를 추가하거나 제거한 뒤 lock wait 양상이 바뀌는 경우가 있다.

같은 SQL인데 어떤 때는 작은 범위만 잠그고, 어떤 때는 넓은 범위가 막힌다.

원인은 DB가 조건을 찾는 경로에 따라 잠금 대상도 달라질 수 있기 때문이다.

특히 InnoDB 같은 엔진에서는 격리 수준과 쿼리 형태에 따라 record lock, gap lock, next-key lock이 사용될 수 있다.

`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`가 range 조건을 사용하면 DB는 조건에 해당하는 record뿐 아니라 삽입을 막기 위한 gap까지 잠글 수 있다.

이때 적절한 인덱스가 있으면 잠금 범위가 조건에 맞는 좁은 key range로 제한된다.

반대로 조건을 효율적으로 찾을 인덱스가 없으면 더 넓은 범위를 스캔하며 잠금을 잡을 수 있다.

예를 들어 다음 쿼리를 보자.

```sql
UPDATE coupons
SET used = TRUE
WHERE user_id = ?
  AND used = FALSE
  AND expires_at > NOW()
LIMIT 1;
```

`(user_id, used, expires_at)` 인덱스가 있으면 특정 사용자의 미사용 쿠폰 중 만료 전 범위로 좁힐 수 있다.

하지만 `expires_at` 단독 인덱스만 있으면 만료 전 전체 쿠폰 범위를 훑으며 많은 row를 검사할 수 있다.

이 차이는 단순 조회 성능뿐 아니라 lock wait과 deadlock 가능성에도 영향을 준다.

인덱스는 lock 문제를 줄이는 장점이 맞다.

다만 "아무 인덱스나 있으면 lock이 줄어든다"가 아니다.

> 쓰기 쿼리의 `WHERE`, `ORDER BY`, `LIMIT`, cardinality에 맞는 인덱스가 lock 범위를 줄인다.

---

## 언제 문제가 되는가

대표 시나리오는 주문 테이블에 조회 요구가 늘면서 인덱스를 계속 추가한 경우다.

처음에는 사용자별 주문 목록이 느려서 `(user_id, created_at)`을 추가한다.

이후 운영자 페이지의 상태별 조회가 느려져 `(status, created_at)`을 추가한다.

정산 배치에서 결제 ID 조회가 필요해 `payment_id` 인덱스를 추가한다.

최근 주문 대시보드 때문에 `created_at` 단독 인덱스도 추가한다.

읽기 쿼리는 하나씩 좋아진다.

하지만 주문 생성은 primary key insert에 더해 여러 secondary index insert를 수행한다. 주문 상태 변경도 일부 인덱스를 갱신한다.

특정 시간대에 주문이 몰리면 같은 B-Tree page 근처에 쓰기가 집중되고, page split과 dirty page flush가 겹치며 응답 시간이 흔들린다.

더 위험한 지점은 lock이다.

결제 완료 처리에서 다음 쿼리를 쓴다고 하자.

```sql
UPDATE orders
SET status = 'PAID'
WHERE user_id = ?
  AND status = 'PENDING'
  AND created_at >= ?
ORDER BY created_at
LIMIT 1;
```

이 쿼리가 의도한 것은 "특정 사용자의 최근 대기 주문 하나를 결제 완료로 바꾸는 것"이다.

하지만 인덱스가 `(created_at)` 중심이면 DB는 최근 주문 범위를 넓게 훑고 `user_id`, `status`를 필터링할 수 있다. 이 과정에서 많은 후보를 읽고, 격리 수준과 실행 계획에 따라 불필요하게 넓은 lock range가 생길 수 있다.

반대로 `(user_id, status, created_at)` 인덱스가 있으면 특정 사용자의 대기 주문 범위로 바로 들어간다.

조회 건수도 줄고, 잠금 후보도 줄어든다.

같은 `UPDATE 1 row`라도 내부적으로 찾는 범위가 다르면 쓰기 지연과 충돌 확률이 달라진다.

인덱스를 추가하면 항상 lock이 줄어드는 것은 아니다.

잘못된 key 순서의 인덱스는 optimizer가 선택하지 않을 수 있고, selectivity가 낮은 인덱스는 넓은 range scan을 유도할 수 있다.

인덱스의 효과는 존재 여부가 아니라 실제 실행 계획과 접근 row 수로 확인해야 한다.

운영에서 문제가 되는 패턴은 보통 세 가지다.

```text
1. 조회 장애를 급히 막기 위해 인덱스를 계속 추가한다.
2. 쓰기 TPS가 늘어난 뒤 index maintenance 비용이 드러난다.
3. UPDATE와 DELETE가 넓은 range를 훑으며 lock wait을 만든다.
```

이때 단순히 DB 스펙을 올리면 당장은 버틸 수 있다.

하지만 사용되지 않는 인덱스, 낮은 selectivity 인덱스, key 순서가 틀어진 복합 인덱스가 남아 있으면 데이터가 늘수록 같은 문제가 반복된다.

---

## 트레이드오프

인덱스 설계는 조회를 빠르게 만드는 일이면서 동시에 쓰기 비용을 어디까지 감당할지 정하는 일이다.

### 인덱스를 추가하는 선택

인덱스를 추가하면 full scan을 줄이고, range scan 시작점을 빠르게 찾고, 특정 쿼리의 p99를 낮출 수 있다.

`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`에서는 조건에 맞는 row를 빨리 찾아 lock range를 줄이는 효과도 기대할 수 있다.

하지만 잃는 것도 명확하다.

- 쓰기마다 index maintenance가 추가된다.
- 저장 공간이 늘어난다.
- buffer pool 사용량이 늘어난다.
- page split 가능성이 늘어난다.
- dirty page flush와 redo log 양이 늘 수 있다.
- optimizer가 선택할 후보가 늘어 실행 계획이 흔들릴 수 있다.

조회 빈도가 낮고 쓰기 빈도가 높은 컬럼에 인덱스를 붙이면, 읽기에서 얻는 이익보다 쓰기 비용이 더 커질 수 있다.

특히 insert가 몰리는 테이블, 이벤트 로그 테이블, 주문/결제처럼 쓰기 경로가 사용자 응답 시간에 직접 연결된 테이블에서는 인덱스 하나가 곧 쓰기 예산을 소비한다.

### 복합 인덱스를 만드는 선택

복합 인덱스는 조건 탐색과 정렬, range scan을 한 번에 줄일 수 있다.

`(user_id, created_at)`은 사용자별 최신 목록에 강하고, `(status, created_at)`은 상태별 시간 범위 조회에 강하다.

하지만 범용성은 떨어진다.

복합 인덱스는 왼쪽 prefix에 강하게 의존한다. `(user_id, created_at)`은 `user_id` 없이 `created_at`만 조건으로 쓰는 쿼리에 제한적이다.

여러 쿼리를 하나의 복합 인덱스로 모두 해결하려고 하면 key 순서가 애매해지고, optimizer가 선택하지 않는 인덱스가 될 수 있다.

PostgreSQL 문서도 multicolumn index는 sparingly 사용하라고 설명한다. 컬럼이 너무 많은 인덱스는 매우 정형화된 조회 패턴이 아니라면 도움이 제한적이고, 공간과 갱신 비용을 늘린다.

### covering index를 만드는 선택

covering index는 table lookup을 줄인다.

목록 API처럼 특정 조건으로 작은 범위를 읽고 정해진 컬럼만 반환하는 쿼리에서는 효과가 크다. buffer pool miss와 random I/O를 줄일 수 있다.

하지만 인덱스 크기가 커진다.

포함 컬럼이 늘수록 leaf entry가 커지고 page fan-out이 줄어든다. 자주 변경되는 컬럼을 포함하면 update 비용도 커진다.

조회 하나를 빠르게 만들기 위해 전체 쓰기 경로와 cache 효율을 희생할 수 있다.

`SELECT *`에 가깝게 많은 컬럼을 반환하거나, 반환 컬럼이 자주 바뀌거나, 포함하려는 컬럼이 긴 문자열 또는 큰 값인 경우에는 신중해야 한다.

### 인덱스를 제거하는 선택

사용되지 않는 인덱스를 제거하면 쓰기 비용, 저장 공간, buffer pool 경쟁, DDL 관리 복잡도를 줄일 수 있다.

하지만 예외적인 조회 경로의 안전망을 잃을 수 있다.

평소에는 안 쓰이던 배치, 운영자 검색, 장애 대응 쿼리가 느려질 수 있다. 대용량 테이블에서는 인덱스 재생성이 긴 DDL 작업이 될 수도 있다.

인덱스 제거는 사용량 통계, slow query, 배치 일정, 운영자 기능을 함께 보고 결정해야 한다.

짧은 관측 기간만 보고 제거하면 월말 정산, 주간 리포트, 이벤트성 배치에서 문제가 터질 수 있다.

### 유니크 인덱스를 추가하는 선택

unique index는 조회 최적화뿐 아니라 중복 방지 불변조건을 제공한다.

예를 들어 주문 번호가 유일해야 한다면 다음 제약은 성능보다 정합성에 더 가깝다.

```sql
ALTER TABLE orders
ADD CONSTRAINT uk_orders_order_no UNIQUE (order_no);
```

얻는 것은 명확하다.

애플리케이션 버그나 재시도 문제가 있어도 같은 주문 번호가 두 번 저장되지 않는다.

하지만 쓰기 경로에서는 중복 검사를 위한 인덱스 접근이 필요하다. 또 어떤 컬럼 조합이 정말 유일해야 하는지 잘못 잡으면 정상 요청까지 막는다.

unique index는 성능 튜닝보다 도메인 규칙에 가깝게 봐야 한다.

### 부분 인덱스와 함수 인덱스를 쓰는 선택

PostgreSQL은 partial index를 지원한다.

예를 들어 대부분 주문은 완료 상태이고, 자주 조회하는 것은 미처리 주문뿐이라면 전체 `status` 인덱스보다 부분 인덱스가 더 나을 수 있다.

```sql
CREATE INDEX idx_orders_pending_created
ON orders (created_at)
WHERE status = 'PENDING';
```

얻는 것은 작은 인덱스다.

쓰기 비용과 저장 공간을 줄이면서 hot query에 맞출 수 있다.

하지만 조건이 정확히 맞아야 한다. 쿼리 조건이 partial index predicate와 맞지 않으면 optimizer가 사용하지 못할 수 있다.

MySQL에서는 PostgreSQL과 같은 형태의 partial index는 제한적이므로 generated column이나 별도 설계가 필요할 수 있다.

함수 인덱스도 비슷하다.

```sql
CREATE INDEX idx_users_lower_email
ON users (lower(email));
```

이 인덱스는 `WHERE lower(email) = ?`에 도움이 될 수 있다. 하지만 함수 결과를 유지해야 하므로 쓰기 비용이 추가된다.

---

## 어떻게 분석할까

먼저 증상을 분리해야 한다.

조회가 느린지, 쓰기가 느린지, lock wait인지, flush나 log 병목인지 구분하지 않으면 인덱스를 잘못 추가하거나 제거하기 쉽다.

```text
1. 느린 SQL을 먼저 특정한다.
2. 실행 계획에서 access method와 rows estimate를 확인한다.
3. 실제 실행 통계에서 read rows와 returned rows 차이를 본다.
4. lock wait이 있으면 어떤 SQL이 blocker인지 확인한다.
5. 인덱스 사용량과 쓰기 빈도를 함께 본다.
6. timeout, retry rate, queue depth가 같이 튀는지 본다.
```

### 조회 쿼리는 읽은 row와 반환 row를 비교한다

조회 쿼리에서는 `EXPLAIN` 또는 `EXPLAIN ANALYZE`를 본다.

핵심은 인덱스를 사용했는가가 아니다.

얼마나 읽고 얼마나 반환했는가다.

```sql
EXPLAIN ANALYZE
SELECT order_id, status
FROM orders
WHERE user_id = 10
  AND created_at >= '2026-06-01'
ORDER BY created_at DESC
LIMIT 20;
```

좋은 상태는 다음에 가깝다.

```text
read rows: 25
returned rows: 20
```

나쁜 상태는 다음에 가깝다.

```text
read rows: 500,000
returned rows: 20
```

후자는 DB가 많은 row를 읽고 대부분 버렸다는 뜻이다.

이런 쿼리는 CPU, buffer pool, 디스크 I/O를 많이 쓰고, `UPDATE`나 `DELETE`라면 불필요한 lock 범위도 커질 수 있다.

확인할 항목은 다음이다.

- 사용한 인덱스가 기대한 인덱스인가
- 조건이 복합 인덱스의 leftmost prefix와 맞는가
- range scan 범위가 넓지 않은가
- 별도 sort가 발생하는가
- table lookup이 과도한가
- estimated rows와 actual rows 차이가 큰가

estimated rows와 actual rows 차이가 크면 통계가 부정확하거나 데이터 분포가 skew되어 있을 수 있다.

PostgreSQL 문서도 인덱스 사용을 판단할 때 `ANALYZE`로 통계를 최신화하고, 실제 데이터로 실험해야 한다고 설명한다.

### 쓰기 지연은 인덱스 개수와 변경 컬럼을 본다

insert가 느리다면 secondary index 수를 본다.

update가 느리다면 변경 컬럼이 인덱스에 포함되어 있는지 본다.

```sql
UPDATE orders
SET status = 'PAID'
WHERE id = ?;
```

`status`가 어떤 인덱스의 key에 들어 있다면 이 update는 row만 바꾸는 것이 아니다.

기존 index entry를 제거하거나 새 entry를 추가해야 할 수 있다.

확인할 항목은 다음이다.

- 테이블에 인덱스가 몇 개인가
- 쓰기 hot path에서 변경하는 컬럼이 어떤 인덱스에 포함되어 있는가
- primary key가 너무 길어 secondary index를 비대하게 만들고 있지는 않은가
- 랜덤 key insert로 page split이 자주 발생할 가능성이 있는가
- dirty page와 redo log pressure가 늘고 있는가

### lock wait은 실행 계획과 함께 본다

lock wait이 늘면 blocker와 waiter를 먼저 본다.

MySQL/InnoDB라면 다음이 도움이 된다.

```sql
SHOW ENGINE INNODB STATUS\G
```

또는 MySQL 8의 performance schema를 볼 수 있다.

```sql
SELECT *
FROM performance_schema.data_lock_waits;

SELECT *
FROM performance_schema.data_locks;
```

PostgreSQL이라면 `pg_locks`, `pg_stat_activity`, `EXPLAIN`을 함께 본다.

중요한 것은 lock wait을 애플리케이션 로그와 연결하는 것이다.

특정 API의 trace에서 DB span이 길어지고, 동시에 retry rate가 올라가고, queue depth가 쌓인다면 단순 slow query가 아니라 대기 전파일 수 있다.

DB connection pool이 고갈되면 원래 빠른 쿼리까지 대기한다.

### 인덱스 사용량을 본다

사용되지 않는 인덱스는 쓰기 비용만 남긴다.

PostgreSQL에서는 `pg_stat_user_indexes` 같은 통계로 index scan 횟수를 볼 수 있다.

```sql
SELECT
  schemaname,
  relname,
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

MySQL에서는 `performance_schema`와 `sys` schema를 통해 사용되지 않는 인덱스를 확인할 수 있다.

```sql
SELECT *
FROM sys.schema_unused_indexes;
```

다만 "사용량 0"만 보고 바로 제거하면 안 된다.

관측 기간에 월말 배치나 운영자 기능이 포함되어 있는지 확인해야 한다.

### 지표는 하나만 보지 않는다

인덱스 문제는 여러 지표가 같이 움직인다.

봐야 할 지표는 다음이다.

- API p95, p99
- DB query latency
- rows examined/read
- rows returned
- buffer pool hit rate
- dirty page count
- pages read/created/written
- redo log write/fsync
- lock wait time
- deadlock count
- DB connection pool active count
- retry rate
- queue depth
- disk IOPS

고정 임계치보다 변화량이 중요하다.

같은 트래픽 조건에서 read rows, lock wait time, dirty page, redo log fsync, queue depth가 함께 튄다면 인덱스 설계를 다시 봐야 한다.

---

## 해결 전략

인덱스 문제를 해결할 때는 "느리니 인덱스 추가"로 바로 가지 않는 편이 좋다.

순서는 다음처럼 잡는다.

```text
1. 느린 API와 SQL을 특정한다.
2. 실제 실행 계획과 rows read/returned를 확인한다.
3. 쿼리의 WHERE, ORDER BY, LIMIT을 기준으로 후보 인덱스를 설계한다.
4. 쓰기 hot path에서 변경되는 컬럼인지 확인한다.
5. lock wait이 있으면 잠금 범위와 blocker를 확인한다.
6. 기존 인덱스와 중복되는지 확인한다.
7. staging 또는 replica에서 실행 계획과 쓰기 비용을 비교한다.
8. 배포 후 p99, rows examined, write latency, lock wait을 같이 본다.
```

### 1. 쿼리 모양을 먼저 고정한다

인덱스는 쿼리 모양에 맞춰 만든다.

다음 정보를 먼저 적는다.

```text
WHERE
  user_id = ?
  status = ?
  created_at >= ?

ORDER BY
  created_at DESC

LIMIT
  20

SELECT
  order_id, status, created_at
```

이렇게 보면 어떤 컬럼이 탐색에 필요하고, 어떤 컬럼이 정렬에 필요하고, 어떤 컬럼이 반환에만 필요한지 나뉜다.

그 다음 후보 인덱스를 만든다.

```sql
CREATE INDEX idx_orders_user_status_created
ON orders (user_id, status, created_at);
```

여기서 `user_id`, `status`는 동등 조건이고 `created_at`은 range 또는 order 조건이다.

이 순서가 항상 정답은 아니다.

상태값 분포, 사용자별 row 수, 정렬 방향, limit, 실제 호출 빈도에 따라 달라진다. 그래서 실행 계획과 실제 row 수로 검증해야 한다.

### 2. 낮은 selectivity 단독 인덱스를 의심한다

`status`, `type`, `is_deleted`, `used` 같은 컬럼은 단독 인덱스로 효과가 작을 수 있다.

값 종류가 적고 특정 값에 데이터가 몰리기 때문이다.

```sql
CREATE INDEX idx_orders_status
ON orders (status);
```

`status = 'PAID'`가 전체의 80%라면 이 인덱스는 많은 row를 읽게 만든다.

하지만 복합 인덱스 안에서는 의미가 생길 수 있다.

```sql
CREATE INDEX idx_orders_user_status_created
ON orders (user_id, status, created_at);
```

여기서는 `user_id`로 먼저 좁힌 뒤, 그 안에서 `status`를 보는 구조가 된다.

낮은 selectivity 컬럼은 단독으로 볼 것이 아니라 다른 조건과 조합해서 봐야 한다.

### 3. 정렬과 LIMIT을 인덱스로 흡수한다

목록 API는 보통 `ORDER BY`와 `LIMIT`이 있다.

```sql
SELECT order_id, status, created_at
FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

이 쿼리에 `(user_id, created_at)` 인덱스가 있으면 특정 사용자의 주문을 시간순으로 읽다가 20개에서 멈출 수 있다.

별도 sort가 필요 없고, 많이 읽을 필요도 줄어든다.

반대로 `ORDER BY`를 인덱스가 처리하지 못하면 DB는 조건에 맞는 row를 모은 뒤 sort하고 limit을 적용해야 한다.

데이터가 많아지면 이 비용이 커진다.

### 4. covering index는 반환 컬럼이 안정적인 곳에만 쓴다

목록 API가 자주 호출되고 반환 컬럼이 작고 안정적이라면 covering index를 고려한다.

MySQL이라면 필요한 컬럼을 뒤쪽 key로 포함시킨다.

```sql
CREATE INDEX idx_orders_user_created_cover
ON orders (user_id, created_at, order_id, status);
```

PostgreSQL이라면 `INCLUDE`를 고려한다.

```sql
CREATE INDEX idx_orders_user_created_cover
ON orders (user_id, created_at)
INCLUDE (order_id, status);
```

하지만 긴 문자열, JSON, 자주 바뀌는 컬럼은 넣지 않는 편이 좋다.

covering index는 인덱스 크기를 키운다. 인덱스 크기가 커지면 buffer pool이나 shared buffer에 더 많은 압박을 준다.

### 5. 쓰기 경로에서 바뀌는 컬럼을 확인한다

인덱스를 추가하기 전에 이 컬럼이 얼마나 자주 바뀌는지 봐야 한다.

```text
created_at
  - insert 후 거의 변경 없음

status
  - CREATED -> PAID -> CANCELLED 등 자주 변경 가능

updated_at
  - 거의 모든 update에서 변경
```

`updated_at`을 무심코 여러 인덱스에 넣으면 대부분 update가 index maintenance를 동반한다.

상태값도 마찬가지다.

`status`는 조회에 자주 쓰이지만, 변경도 자주 된다. 상태별 조회가 정말 hot path인지, 다른 조건과 묶어 좁힐 수 있는지 봐야 한다.

### 6. UPDATE/DELETE는 잠금 범위 기준으로 설계한다

쓰기 쿼리의 인덱스는 lock range를 줄이는 도구다.

다음 쿼리가 있다.

```sql
UPDATE orders
SET status = 'EXPIRED'
WHERE status = 'PENDING'
  AND expires_at < NOW()
ORDER BY expires_at
LIMIT 100;
```

후보 인덱스는 다음이 될 수 있다.

```sql
CREATE INDEX idx_orders_status_expires
ON orders (status, expires_at);
```

이렇게 하면 `PENDING` 중 만료 시간이 지난 범위를 순서대로 찾을 수 있다.

하지만 `PENDING` row가 너무 많고 여러 worker가 동시에 실행된다면 같은 앞부분 range에 몰릴 수 있다.

이때는 batch chunking, `SKIP LOCKED`, shard key, 처리 상태 분리 등을 같이 봐야 한다.

인덱스만으로 모든 lock wait이 사라지지는 않는다.

### 7. 중복 인덱스를 줄인다

다음 인덱스들이 같이 있다면 중복일 수 있다.

```sql
INDEX idx_user (user_id)
INDEX idx_user_created (user_id, created_at)
```

`idx_user_created`는 leftmost prefix로 `user_id` 조건에도 사용될 수 있다.

따라서 `idx_user`가 별도로 필요한지 확인해야 한다.

하지만 항상 제거해도 된다는 뜻은 아니다.

작은 단일 컬럼 인덱스가 더 효율적인 쿼리도 있을 수 있고, unique 여부나 foreign key 제약 때문에 필요한 경우도 있다.

중복 여부는 실행 계획과 제약 조건, 실제 사용량으로 판단한다.

### 8. 배포 후 지표를 같이 본다

인덱스 추가 배포 후 조회만 보면 안 된다.

같이 봐야 한다.

```text
조회
  - p95, p99
  - rows examined/read
  - sort 발생 여부
  - table fetch 수

쓰기
  - insert/update latency
  - lock wait
  - deadlock
  - dirty page
  - redo log write
  - buffer pool hit rate

애플리케이션
  - timeout
  - retry rate
  - connection pool wait
```

인덱스는 읽기와 쓰기 양쪽에 영향을 준다.

조회 지표만 좋아졌다고 끝난 것이 아니다.

---

## 면접에서는 이렇게 답한다

질문은 이렇게 나올 수 있다.

> B-Tree 인덱스는 왜 조회를 빠르게 만들지만 쓰기를 비싸게 만드나요?

1분 답변은 이렇게 말할 수 있다.

B-Tree 인덱스는 key를 정렬된 page 구조로 저장하고, root에서 leaf까지 내려가며 원하는 key나 range의 시작점을 빠르게 찾습니다. 그래서 전체 테이블을 스캔하지 않고 필요한 범위로 바로 이동할 수 있어 조회가 빨라집니다.

하지만 쓰기에서는 이 정렬 구조를 계속 유지해야 합니다. row를 insert하면 clustered index뿐 아니라 모든 secondary index에 entry를 추가해야 하고, update로 인덱스 컬럼이 바뀌면 기존 entry 제거와 새 entry 추가가 필요할 수 있습니다. leaf page에 공간이 부족하면 page split이 발생하고, 변경된 page는 dirty page와 redo log 대상이 됩니다.

또 인덱스는 lock 범위에도 영향을 줍니다. InnoDB에서는 record lock이 인덱스 레코드 기준으로 잡히고, range 조건에서는 gap lock이나 next-key lock이 개입할 수 있습니다. 조건에 맞는 인덱스가 있으면 잠금 범위를 좁힐 수 있지만, key 순서가 맞지 않거나 selectivity가 낮으면 넓은 range를 스캔하며 lock wait이 커질 수 있습니다.

그래서 인덱스는 단순히 많을수록 좋은 것이 아니라, 실제 `WHERE`, `ORDER BY`, `LIMIT`, 반환 컬럼, 쓰기 빈도, lock wait을 함께 보고 설계해야 합니다.

핵심 채점 포인트는 다음이다.

- B-Tree가 정렬된 page 기반 자료구조라는 점
- root에서 leaf까지 내려가 조회 범위를 줄인다는 점
- secondary index가 쓰기마다 함께 갱신된다는 점
- page split, dirty page, redo log 같은 쓰기 비용을 아는 점
- 복합 인덱스의 leftmost prefix와 selectivity를 설명하는 점
- covering index의 장점과 비용을 함께 말하는 점
- 인덱스가 lock range에 영향을 준다는 점

꼬리 질문은 이렇게 이어질 수 있다.

### B-Tree 인덱스는 메모리에 있나요, 디스크에 있나요?

전체 인덱스는 디스크에 저장된다.

쿼리 실행 중 필요한 index page와 data page 일부가 buffer pool 또는 shared buffer에 올라온다. 이미 메모리에 있으면 디스크 I/O 없이 읽고, 없으면 디스크에서 page 단위로 읽어온다.

즉 B-Tree는 디스크에 있는 자료구조이고, 자주 쓰이는 page 일부가 메모리에 캐시된다고 보는 것이 맞다.

### secondary index의 leaf에는 실제 row 주소가 들어 있나요?

DB 엔진마다 다르다.

InnoDB 기준으로 clustered index leaf에는 row 데이터 자체가 들어 있다.

secondary index leaf에는 secondary key와 primary key 값이 들어 있다. DB는 secondary index에서 primary key를 얻고, 다시 clustered index를 탐색해 row를 찾는다.

그래서 InnoDB에서는 primary key가 길면 secondary index도 커진다.

### covering index는 일반 인덱스와 무엇이 다른가요?

covering index는 특별한 인덱스 종류라기보다, 특정 쿼리가 필요한 컬럼을 모두 포함해 table 접근을 생략할 수 있는 인덱스다.

같은 인덱스라도 어떤 쿼리에는 covering이 되고, 어떤 쿼리에는 covering이 아닐 수 있다.

장점은 table lookup을 줄이는 것이다.

단점은 인덱스 크기와 쓰기 비용이 늘어난다는 것이다.

### 인덱스가 lock wait을 줄이나요, 늘리나요?

둘 다 가능하다.

조건에 맞는 인덱스는 `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`가 잠글 row와 range를 좁혀 lock wait을 줄일 수 있다.

하지만 key 순서가 맞지 않거나 selectivity가 낮은 인덱스는 넓은 range scan을 만들 수 있다. InnoDB에서는 스캔한 인덱스 레코드와 gap에 lock이 걸릴 수 있으므로 lock wait이 커질 수 있다.

정확히는 "인덱스가 lock을 줄인다"가 아니라 "쿼리 조건에 맞는 인덱스가 lock 범위를 줄인다"다.

---

## 요약

B-Tree 인덱스는 정렬된 page 기반 자료구조다.

전체 인덱스는 디스크에 저장되고, 실행 중 필요한 page 일부가 buffer pool이나 shared buffer에 캐시된다.

조회는 root에서 leaf까지 내려가 원하는 key나 range의 시작점을 빠르게 찾는다. 그래서 full scan을 줄이고, 정렬과 limit을 인덱스 순서로 처리할 수 있다.

하지만 쓰기에서는 정렬 상태를 계속 유지해야 한다.

row 하나를 insert해도 primary 구조와 모든 secondary index가 갱신된다. leaf page에 공간이 없으면 page split이 발생하고, 변경된 page는 dirty page와 redo log 대상이 된다.

복합 인덱스는 key 순서가 중요하다.

`(user_id, created_at)`과 `(created_at, user_id)`는 같은 컬럼을 포함하지만 전혀 다른 접근 경로를 만든다. leftmost prefix, selectivity, range 조건, `ORDER BY`, `LIMIT`을 함께 봐야 한다.

covering index는 table lookup을 줄인다.

하지만 인덱스 크기를 키우고 쓰기 비용을 늘린다. 반환 컬럼이 작고 안정적인 hot path에 제한적으로 쓰는 편이 좋다.

인덱스는 lock range에도 영향을 준다.

적절한 인덱스는 쓰기 쿼리의 잠금 범위를 줄이지만, 잘못된 인덱스는 넓은 range scan과 lock wait을 만들 수 있다.

인덱스를 잘 쓰는 방법은 많게 만드는 것이 아니다.

실제 쿼리가 어디서 시작해 얼마나 읽고 얼마나 버리는지, 그리고 쓰기 때 어떤 인덱스를 유지해야 하는지 보는 것이다.

가장 중요한 질문은 이것이다.

> 이 인덱스는 어떤 쿼리의 읽기 비용을 줄이고, 대신 어떤 쓰기 비용과 잠금 범위를 새로 만드는가?

이 질문에 답할 수 있을 때 인덱스는 단순한 조회 최적화가 아니라, 읽기·쓰기·잠금 비용을 함께 조정하는 설계 도구가 된다.

---

## 참고 자료

- [MySQL Reference Manual - How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html)
- [MySQL Reference Manual - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [MySQL Reference Manual - The Physical Structure of an InnoDB Index](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)
- [MySQL Reference Manual - InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.4/en/innodb-buffer-pool.html)
- [MySQL Reference Manual - InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [PostgreSQL Documentation - Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL Documentation - Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [PostgreSQL Documentation - Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
- [PostgreSQL Documentation - Examining Index Usage](https://www.postgresql.org/docs/current/indexes-examine.html)
