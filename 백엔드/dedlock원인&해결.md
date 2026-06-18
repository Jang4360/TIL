# 왜 서로 다른 요청은 각자의 락을 잡은 채 멈춰 버리는가

결제 요청은 주문을 먼저 수정하고 재고를 줄인다.

재고 보정 배치는 재고를 먼저 수정하고 주문 상태를 맞춘다.

둘 다 평소에는 빠르게 끝난다. SQL도 단순하고, DB CPU가 높은 것도 아니다. 그런데 어느 순간부터 두 요청이 동시에 멈춘 것처럼 보인다. 애플리케이션에서는 timeout이 늘고, DB에서는 lock wait이 쌓인다. 몇 초 뒤 하나의 트랜잭션이 강제로 실패하고, 실패한 요청은 다시 시도된다.

사용자는 이 현상을 이렇게 느낀다.

> 가끔 결제가 실패한다.

하지만 내부에서 벌어진 일은 단순한 timeout이 아니다.

서로 다른 요청이 필요한 락을 다른 순서로 잡았고, 트랜잭션이 끝날 때까지 이미 잡은 락을 놓지 않았기 때문에 생긴 일이다.

데드락은 락이 많아서만 생기지 않는다.

여러 트랜잭션이 같은 자원을 서로 다른 순서로 점유한 채, 다음 락을 기다릴 때 생긴다.

---

## 먼저 구조를 잡자

트랜잭션은 하나의 SQL만 보호하지 않는다.

트랜잭션이 시작된 뒤 수정한 행, 인덱스 엔트리, 조건 검색 범위, 외래키 검증 대상까지 여러 자원에 락이 걸릴 수 있다. 그리고 대부분의 쓰기 락은 트랜잭션이 `COMMIT` 또는 `ROLLBACK`될 때까지 유지된다.

주문과 재고를 함께 수정하는 요청을 보자.

요청 A는 주문을 먼저 수정하고 재고를 수정한다.

요청 B는 재고를 먼저 수정하고 주문을 수정한다.

둘은 같은 데이터를 다루지만, 락을 획득하는 순서가 다르다.

```text
1. 요청 A가 주문 행의 락을 잡는다.
2. 요청 B가 재고 행의 락을 잡는다.
3. 요청 A가 재고 행의 락을 기다린다.
4. 요청 B가 주문 행의 락을 기다린다.
5. DB는 순환 대기 상태를 감지한다.
6. DB는 둘 중 하나를 victim으로 선택해 rollback한다.
7. 애플리케이션은 실패를 받고 재시도하거나 오류를 반환한다.
```

여기서 중요한 점은 두 요청 모두 정상적인 로직이라는 것이다.

SQL 문법도 맞고, 비즈니스 규칙도 맞고, 단독 실행하면 빠르다. 문제는 동시에 실행될 때 자원을 점유하는 순서가 서로 충돌한다는 데 있다.

데드락을 이해하려면 세 가지 층을 같이 봐야 한다.

- 애플리케이션의 요청 흐름
- 트랜잭션 범위
- DB가 실제로 거는 row lock, index lock, range lock

코드에서 "주문 저장 후 재고 저장"처럼 보이는 흐름이 DB 내부에서도 단순히 두 개의 행만 잠근다는 보장은 없다.

`WHERE` 조건이 어떤 인덱스를 타는지, 외래키가 있는지, unique index를 갱신하는지, 격리 수준이 무엇인지에 따라 락의 대상과 범위가 달라진다.

---

## 내부에서는 무슨 일이 일어나는가

### 락은 요청 단위가 아니라 트랜잭션 단위로 유지된다

문제는 한 SQL이 끝났는데도 락이 바로 풀리지 않는다는 데서 시작한다.

개발자는 `UPDATE orders ...`가 끝났으니 주문 락도 끝났다고 생각하기 쉽다. 하지만 일반적인 트랜잭션에서는 쓰기 락이 트랜잭션 종료까지 유지된다.

원인은 정합성 보장이다.

주문 상태를 `READY`에서 `PAID`로 바꾸고 아직 재고 차감이 끝나지 않았는데, 다른 요청이 같은 주문을 취소해 버리면 데이터가 깨질 수 있다. DB는 이런 중간 상태를 보호하기 위해 트랜잭션이 살아 있는 동안 필요한 락을 유지한다.

내부 구조는 "SQL 실행 중 락"이 아니라 "트랜잭션 생존 기간 동안 락 보유"에 가깝다.

한 트랜잭션이 여러 테이블을 수정하면 먼저 잡은 락을 가진 채 다음 테이블의 락을 요청한다. 이때 다음 락이 이미 다른 트랜잭션에 잡혀 있으면 기다린다.

그래서 트랜잭션 안에 다음 작업이 들어가면 위험해진다.

- 외부 결제 API 호출
- 메시지 브로커 응답 대기
- 파일 업로드 또는 이미지 처리
- 긴 계산
- 대량 조회
- 사용자 입력 대기
- 다른 서비스의 HTTP 응답 대기

이 작업들은 DB 락과 직접 관련 없어 보인다. 하지만 트랜잭션 안에 들어가는 순간, DB 락을 잡은 채 외부 세계를 기다리는 구조가 된다.

트랜잭션은 짧을수록 좋다는 말은 단순한 성능 조언이 아니다.

락 충돌 면적을 줄이라는 운영 원칙에 가깝다. 짧은 트랜잭션은 실패해도 되돌릴 범위가 작고, 재시도 비용도 낮다.

### 같은 데이터를 다른 순서로 잡으면 순환 대기가 생긴다

문제는 요청 A와 요청 B가 모두 필요한 락을 잡으려 하지만, 획득 순서가 다를 때 생긴다.

A는 주문을 잡은 뒤 재고를 기다린다.

B는 재고를 잡은 뒤 주문을 기다린다.

둘 다 먼저 잡은 락을 놓지 않으므로 서로 양보할 수 없다.

```text
Transaction A
  holds: orders:100
  waits: inventory:sku-1

Transaction B
  holds: inventory:sku-1
  waits: orders:100
```

이 관계를 그래프로 보면 사이클이 생긴다.

```text
A -> B -> A
```

DB는 대기 그래프를 추적하다가 순환을 발견하면 하나의 트랜잭션을 중단한다. MySQL InnoDB는 deadlock detection이 켜져 있으면 이런 순환을 감지하고, 보통 비용이 낮다고 판단한 트랜잭션을 victim으로 골라 rollback한다.

이 rollback은 DB의 복구 동작이다.

둘 다 계속 기다리게 두면 시스템이 멈추기 때문에, DB는 한쪽을 포기시켜 나머지 한쪽이 진행되도록 만든다.

그래서 데드락 오류는 "SQL이 틀렸다"는 뜻이 아니다.

동시성 제어 과정에서 트랜잭션 하나가 선택되어 취소됐다는 뜻이다.

### 인덱스가 없으면 생각보다 넓은 범위가 잠긴다

데드락을 이야기할 때 인덱스는 자주 과소평가된다.

인덱스는 조회를 빠르게 만드는 장치이기도 하지만, 쓰기 트랜잭션에서는 어떤 레코드와 어떤 범위를 잠글지 결정하는 요소이기도 하다.

InnoDB에서는 row lock이 실제로 인덱스 레코드 기준으로 잡힌다. `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` 같은 잠금 읽기는 실행 계획이 선택한 인덱스를 따라가며 만난 인덱스 레코드에 락을 건다.

다음 쿼리를 보자.

```sql
UPDATE orders
SET status = 'DONE'
WHERE user_id = 10
  AND status = 'READY';
```

`(user_id, status)` 복합 인덱스가 있다면 DB는 `user_id = 10 AND status = 'READY'`에 해당하는 범위를 좁게 찾을 수 있다.

반대로 적절한 인덱스가 없다면 더 많은 후보 row를 스캔해야 한다. 격리 수준과 실행 방식에 따라 더 많은 인덱스 레코드, gap, next-key 범위가 락 경합 대상이 될 수 있다.

즉 인덱스는 "락을 빨리 잡고 빨리 푼다"만의 문제가 아니다.

락을 잡는 후보 집합 자체를 줄인다.

### InnoDB의 record lock, gap lock, next-key lock

InnoDB 기준으로 자주 만나는 락은 다음과 같다.

| 락 | 의미 | 예시 |
| --- | --- | --- |
| Record Lock | 특정 인덱스 레코드 하나에 대한 락 | `WHERE id = 10`이 PK를 정확히 타는 경우 |
| Gap Lock | 인덱스 레코드 사이 빈 공간에 대한 락 | 특정 범위에 새 row가 끼어드는 것을 막음 |
| Next-Key Lock | Record Lock + 그 앞의 Gap Lock | 범위 검색에서 phantom row를 막기 위해 사용 |
| Insert Intention Lock | 특정 gap에 insert하려는 의도를 표시하는 락 | 서로 다른 위치 insert는 병행될 수 있지만 gap lock과 충돌 가능 |

예를 들어 이런 테이블이 있다고 하자.

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_user_status (user_id, status)
);
```

트랜잭션 A가 다음 쿼리를 실행한다.

```sql
BEGIN;

SELECT *
FROM orders
WHERE user_id = 10
  AND status = 'PENDING'
FOR UPDATE;
```

InnoDB는 `idx_user_status` 인덱스를 타고 조건에 맞는 인덱스 엔트리를 찾는다. 이때 테이블 row만 잠그는 것이 아니라, 해당 조건에 해당하는 인덱스 레코드와 필요한 범위를 잠글 수 있다.

그래서 트랜잭션 B가 같은 인덱스 범위에 영향을 주는 insert를 시도하면 대기할 수 있다.

```sql
INSERT INTO orders (id, user_id, status, created_at)
VALUES (100, 10, 'PENDING', NOW());
```

특히 MySQL의 기본 격리 수준인 `REPEATABLE READ`에서는 phantom read를 막기 위해 next-key lock이 중요해진다. 범위 조건이나 locking read가 섞이면 기존 row뿐 아니라 "그 범위에 새 row가 들어오는 것"까지 막을 수 있다.

정리하면 질문은 "row lock인가 table lock인가"에서 끝나면 안 된다.

더 중요한 질문은 이것이다.

> DB가 어떤 인덱스 범위를 스캔했고, 그 과정에서 어떤 record/gap/next-key lock을 잡았는가?

### lock wait timeout과 deadlock detection은 다르다

둘 다 애플리케이션에서는 실패처럼 보일 수 있다. 하지만 원인과 대응 방향은 다르다.

| 구분 | 의미 | 특징 |
| --- | --- | --- |
| lock wait timeout | 락을 기다리다가 설정 시간을 넘겨 실패 | 순환 대기가 없어도 발생 가능 |
| deadlock detection | 순환 대기를 감지해 한 트랜잭션을 rollback | timeout까지 기다리지 않고 중단 가능 |

lock wait timeout은 "너무 오래 기다렸으니 포기"에 가깝다.

앞 트랜잭션이 오래 걸리기만 해도 발생할 수 있다. 긴 트랜잭션, 느린 쿼리, 인덱스 누락, 대량 배치, 커넥션 풀 포화가 원인이 될 수 있다.

deadlock detection은 "기다려도 풀릴 수 없는 순환 구조를 발견했다"에 가깝다.

DB가 대기 그래프에서 사이클을 찾고, 하나의 트랜잭션을 victim으로 선택해 rollback한다.

둘 다 락 문제이지만 분석 방향은 다르다.

lock wait timeout은 "누가 오래 잡고 있었는가"를 봐야 한다.

deadlock은 "서로 어떤 순서로 잡고 기다렸는가"를 봐야 한다.

---

## 언제 문제가 되는가

데드락은 작은 테스트 환경에서는 잘 보이지 않는다.

요청이 적고, 데이터가 작고, API와 배치가 동시에 같은 자원을 건드리는 상황이 드물기 때문이다.

하지만 운영 중인 서비스에서는 사용자 요청, 스케줄러, 메시지 컨슈머, 관리자 기능이 한 DB를 공유한다. 같은 테이블을 다른 흐름이 수정하기 시작하면 데드락 조건이 자연스럽게 만들어진다.

### 주문과 재고

결제 API는 주문 상태를 바꾸고 재고를 차감한다.

재고 보정 배치는 재고 수량을 맞춘 뒤 영향을 받은 주문을 재검증한다.

```text
결제 API
  1. orders 잠금
  2. inventory 잠금

재고 보정 배치
  1. inventory 잠금
  2. orders 잠금
```

이 구조는 단독 실행하면 정상이다. 하지만 같은 상품과 주문을 동시에 건드리면 순환 대기가 생길 수 있다.

### 사용자 잔액과 포인트 내역

결제 취소 요청은 포인트 내역을 먼저 만들고 잔액을 되돌린다.

정산 배치는 잔액을 먼저 잠그고 누락된 포인트 내역을 보정한다.

```text
결제 취소
  1. point_history insert
  2. user_balance update

정산 배치
  1. user_balance update
  2. point_history insert or update
```

여기서는 외래키, unique key, 보조 인덱스 갱신까지 함께 엮일 수 있다. 단순히 두 테이블만 보는 것으로는 부족하다.

### 대량 배치와 사용자 요청

배치는 보통 넓은 범위를 다룬다.

```sql
UPDATE coupons
SET status = 'EXPIRED'
WHERE expires_at < NOW()
  AND status = 'ACTIVE';
```

사용자 요청은 특정 쿠폰 하나를 사용한다.

```sql
UPDATE coupons
SET status = 'USED'
WHERE id = ?
  AND status = 'ACTIVE';
```

배치가 적절한 인덱스를 타지 못하면 넓은 범위를 스캔하며 락을 잡는다. 그 사이 사용자 요청이 같은 row 또는 같은 인덱스 범위를 건드리면 lock wait이 커진다.

배치가 여러 테이블을 함께 보정한다면 데드락으로 이어질 수도 있다.

### 외래키와 unique key 검증

외래키와 unique key는 데이터 정합성을 지켜준다.

하지만 쓰기 시점에는 검증을 위해 관련 인덱스와 부모/자식 row를 확인해야 한다.

예를 들어 자식 row를 insert할 때 부모 row 존재 여부를 확인해야 한다. 부모 row를 삭제하거나 수정하는 트랜잭션과 동시에 실행되면 대기 관계가 만들어질 수 있다.

unique key도 마찬가지다. 같은 unique 범위에 insert 또는 update가 몰리면 중복 검사를 위한 인덱스 접근에서 충돌이 생길 수 있다.

제약 조건은 필요하다. 다만 제약 조건이 있는 테이블은 쓰기 경로에서 어떤 인덱스를 추가로 확인하는지 같이 봐야 한다.

---

## 트레이드오프

데드락 대응은 하나의 정답이 아니라 선택의 연속이다.

락을 줄이면 동시성은 좋아질 수 있다. 대신 정합성 보장이 약해질 수 있다.

재시도를 넣으면 사용자는 성공을 더 자주 경험할 수 있다. 대신 부하가 증폭될 수 있다.

인덱스를 추가하면 락 범위는 줄어들 수 있다. 대신 쓰기 비용과 저장 공간이 늘어난다.

각 대응마다 무엇을 얻고 무엇을 잃는지 봐야 한다.

### 트랜잭션을 짧게 가져가는 선택

트랜잭션 범위를 줄이면 락 보유 시간이 짧아진다.

가장 먼저 할 일은 트랜잭션 안에서 DB 변경과 무관한 작업을 빼는 것이다.

```text
나쁜 흐름
1. BEGIN
2. 주문 상태 변경
3. 외부 결제 API 호출
4. 재고 차감
5. COMMIT
```

외부 API가 2초 지연되면 주문 락도 2초 동안 유지된다.

더 나은 흐름은 DB에서 먼저 "시도 상태"를 짧게 기록하고, 외부 API 호출 후 결과를 별도 트랜잭션으로 반영하는 방식이다.

```text
개선 흐름
1. BEGIN
2. 결제 시도 상태 저장
3. COMMIT

4. 외부 결제 API 호출

5. BEGIN
6. 결제 결과 반영
7. 주문/재고 상태 전이
8. COMMIT
```

트레이드오프는 상태 모델이 복잡해진다는 점이다.

`PAYMENT_REQUESTED`, `PAYMENT_APPROVED`, `PAYMENT_FAILED`, `STOCK_RESERVED`, `STOCK_FAILED` 같은 중간 상태가 필요해질 수 있다. 실패 보정 작업도 필요하다.

하지만 긴 트랜잭션 하나로 모든 일을 감싸는 방식보다 장애 지점을 분리하기 쉽다.

### 락 획득 순서를 통일하는 선택

같은 자원 집합을 수정하는 모든 경로가 같은 순서로 락을 잡아야 한다.

중요한 건 "규칙을 정한다"에서 끝내지 않는 것이다. 코드에서 그 규칙이 강제되어야 한다.

예를 들어 주문과 재고를 함께 수정한다면 다음처럼 순서를 명시한다.

```text
1. inventory를 product_id 오름차순으로 잠근다.
2. orders를 order_id 오름차순으로 잠근다.
3. order_items를 order_item_id 오름차순으로 잠근다.
4. 상태 변경을 수행한다.
```

여러 row를 잠글 때는 입력 순서를 그대로 쓰지 않는다.

사용자 요청에서 상품 ID가 `[3, 1, 2]`로 들어와도 DB 락은 `[1, 2, 3]` 순서로 잡는다.

```sql
SELECT id
FROM inventory
WHERE product_id IN (1, 2, 3)
ORDER BY product_id
FOR UPDATE;
```

이 방식은 API, 배치, 컨슈머, 관리자 기능 모두에 적용되어야 한다.

트레이드오프는 코드 자유도가 줄어든다는 점이다.

각 기능이 자기 흐름에 맞춰 자연스럽게 저장하던 방식을 버리고, 공통 락 획득 함수를 통과해야 한다. 대신 데드락 조건은 크게 줄어든다.

권장되는 형태는 다음과 같다.

```text
OrderInventoryLockService.lock(orderIds, productIds)
  - productIds 정렬
  - orderIds 정렬
  - inventory FOR UPDATE
  - orders FOR UPDATE
```

핵심은 개발자가 매번 순서를 기억하게 하지 않는 것이다.

락 순서는 코드 리뷰 규칙이 아니라 공통 경로로 강제되어야 한다.

### 인덱스로 락 범위를 줄이는 선택

인덱스는 읽기 성능뿐 아니라 잠금 범위에도 영향을 준다.

`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`가 느리거나 대기한다면 먼저 실행 계획을 확인해야 한다.

```sql
EXPLAIN
UPDATE orders
SET status = 'DONE'
WHERE user_id = 10
  AND status = 'READY';
```

봐야 할 것은 단순히 "인덱스를 탔는가"가 아니다.

- 어떤 인덱스를 탔는가
- 조건절 순서와 복합 인덱스 순서가 맞는가
- 스캔 row 수가 과도하지 않은가
- 범위 조건이 앞쪽 컬럼에 있어 뒤 컬럼을 충분히 줄이지 못하는가
- locking read가 예상보다 넓은 범위를 잠그는가

예를 들어 다음 두 인덱스는 다르게 동작할 수 있다.

```sql
INDEX idx_status_user (status, user_id)
INDEX idx_user_status (user_id, status)
```

`WHERE user_id = ? AND status = ?`가 주요 경로라면 일반적으로 `(user_id, status)`가 더 자연스럽다. 반대로 특정 상태의 모든 주문을 배치로 처리하는 경로가 주라면 `(status, created_at, id)` 같은 인덱스가 필요할 수 있다.

인덱스 추가의 트레이드오프도 분명하다.

- `INSERT`, `UPDATE`, `DELETE` 시 갱신해야 할 인덱스가 늘어난다.
- 저장 공간이 늘어난다.
- 옵티마이저가 잘못된 인덱스를 선택할 여지가 생긴다.
- 인덱스가 많아질수록 스키마 변경 비용이 커진다.

그래서 목적 없이 인덱스를 추가하면 안 된다.

목표는 "조회가 빨라진다"가 아니라 "수정 대상과 잠금 범위가 좁아진다"여야 한다.

### 격리 수준을 낮추는 선택

MySQL InnoDB의 `REPEATABLE READ`에서는 gap lock과 next-key lock이 데드락 분석에서 자주 등장한다.

`READ COMMITTED`로 낮추면 일부 gap lock이 줄어들 수 있다. MySQL 문서에서도 locking read, `UPDATE`, `DELETE`의 잠금 방식은 unique search인지 range search인지, 그리고 격리 수준에 따라 달라진다고 설명한다.

하지만 격리 수준을 낮추는 것은 단순한 성능 튜닝이 아니다.

트레이드오프는 정합성 모델이 바뀐다는 점이다.

- 같은 트랜잭션 안에서 같은 조회 결과가 달라질 수 있다.
- phantom read 가능성이 커질 수 있다.
- 애플리케이션이 기존에 기대하던 읽기 일관성이 깨질 수 있다.
- DB 종류와 버전에 따라 동작 차이를 확인해야 한다.

따라서 격리 수준 변경은 마지막에 검토하는 편이 좋다.

먼저 트랜잭션 범위, 락 순서, 인덱스, 배치 범위를 정리한다. 그래도 특정 범위 잠금이 과도하고 비즈니스 정합성이 허용한다면 격리 수준 변경을 검토한다.

### 재시도를 넣는 선택

데드락은 완전히 없애기 어렵다.

DB 공식 문서들도 데드락이 발생할 수 있으며, 애플리케이션은 트랜잭션 재시도를 처리해야 한다고 안내한다.

하지만 재시도는 조심해서 넣어야 한다.

```text
나쁜 재시도
1. 데드락 발생
2. 즉시 재시도
3. 또 데드락 발생
4. 즉시 재시도
5. 같은 자원에 요청이 더 몰림
```

재시도 대상은 "실패한 요청"이 아니라 "이미 충돌한 자원"이다. 같은 주문, 같은 재고, 같은 사용자 잔액을 두고 여러 요청이 동시에 다시 몰리면 충돌 확률은 줄지 않는다.

권장되는 재시도는 제한적이어야 한다.

```text
retry count: 1~3회
backoff: 50ms -> 100ms -> 200ms
jitter: 랜덤 지연 추가
deadline: 사용자 요청 전체 timeout 안에서만 수행
```

그리고 멱등성이 있어야 한다.

결제 승인, 쿠폰 사용, 포인트 적립처럼 외부 부작용이 있는 흐름은 DB 트랜잭션만 재시도하면 위험하다. DB는 rollback됐지만 외부 결제사는 이미 승인했을 수 있다.

이런 흐름에는 다음 장치가 필요하다.

- 요청 ID 또는 idempotency key
- DB unique constraint
- 상태 전이 검증
- outbox pattern
- 외부 API 호출 결과 저장
- 중복 요청에 기존 결과 반환

재시도는 실패를 숨기는 장치가 아니라, 일시적 충돌을 제한적으로 흡수하는 장치다.

### 큐로 직렬화하는 선택

특정 자원에 쓰기가 집중된다면 동시에 처리하지 않는 방식도 가능하다.

예를 들어 같은 `product_id`의 재고 변경을 하나의 queue partition으로 보내고, 같은 상품에 대한 작업은 순서대로 처리한다.

```text
partition key = product_id

product:1 -> worker A에서 순차 처리
product:2 -> worker B에서 순차 처리
product:3 -> worker C에서 순차 처리
```

이 방식은 DB 락 경합을 크게 줄일 수 있다.

하지만 트레이드오프가 있다.

- 처리 지연이 생길 수 있다.
- 큐 적체를 모니터링해야 한다.
- 작업 실패와 재처리 정책이 필요하다.
- 순서 보장이 필요한 범위를 잘못 잡으면 병렬성이 떨어진다.
- 큐에 넣은 뒤 DB 반영 전까지 중간 상태가 생긴다.

결제처럼 사용자 응답이 즉시 필요한 경로에는 모든 작업을 큐로 넘기기 어렵다. 대신 재고 예약, 포인트 정산, 집계 보정처럼 잠깐 늦어도 되는 작업부터 분리할 수 있다.

### 낙관적 락을 쓰는 선택

비관적 락은 먼저 잠그고 수정한다.

낙관적 락은 먼저 읽고, 수정 시점에 버전이 그대로인지 확인한다.

```sql
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = ?
  AND version = ?
  AND quantity > 0;
```

업데이트된 row 수가 0이면 누군가 먼저 수정한 것이다. 이때 애플리케이션은 다시 읽고 재시도하거나 사용자에게 실패를 반환한다.

낙관적 락은 락 보유 시간을 줄일 수 있다.

하지만 충돌이 많은 자원에서는 재시도가 늘어난다. 인기 상품 재고 차감처럼 같은 row에 쓰기가 몰리면 낙관적 락 실패가 폭증할 수 있다.

트레이드오프는 다음과 같다.

- 충돌이 낮으면 비관적 락보다 효율적이다.
- 충돌이 높으면 재시도 비용이 커진다.
- 업데이트 실패 처리를 애플리케이션이 명확히 해야 한다.
- 외부 부작용이 있는 작업과 함께 쓰려면 순서 설계가 필요하다.

낙관적 락은 "락을 안 쓰는 방법"이 아니다.

충돌 감지 시점을 뒤로 미루는 방법이다.

### 배치를 작게 나누는 선택

대량 배치는 데드락과 lock wait을 키우기 쉽다.

한 번에 10만 row를 수정하면 트랜잭션이 오래 유지되고, 락 범위도 넓어진다. 사용자 요청과 같은 테이블을 건드리면 지연이 전파된다.

배치는 작게 나누는 것이 좋다.

```sql
SELECT id
FROM coupons
WHERE status = 'ACTIVE'
  AND expires_at < NOW()
ORDER BY id
LIMIT 1000;
```

가져온 ID 목록을 정렬한 뒤 작은 단위로 수정한다.

```sql
UPDATE coupons
SET status = 'EXPIRED'
WHERE id IN (...)
  AND status = 'ACTIVE';
```

그리고 각 chunk마다 commit한다.

트레이드오프는 전체 처리 시간이 길어질 수 있다는 점이다.

하지만 한 트랜잭션이 너무 오래 락을 잡는 것보다, 작은 단위로 짧게 끝내는 쪽이 사용자 요청과 함께 사는 데 유리하다.

배치에는 다음 규칙을 둘 수 있다.

- PK 또는 고유한 정렬 기준으로 chunking한다.
- 매 chunk마다 commit한다.
- 같은 row를 중복 처리해도 안전하게 만든다.
- 처리량보다 락 보유 시간을 먼저 본다.
- 실패 지점을 기록해 이어서 재개할 수 있게 한다.

### `SKIP LOCKED`를 쓰는 선택

작업 큐성 테이블에서는 `SELECT ... FOR UPDATE SKIP LOCKED`가 도움이 될 수 있다.

이미 다른 트랜잭션이 잠근 row는 기다리지 않고 건너뛰는 방식이다.

```sql
SELECT id
FROM jobs
WHERE status = 'READY'
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

이렇게 하면 여러 worker가 같은 job 테이블을 읽어도 서로 기다리는 시간이 줄어든다.

하지만 트레이드오프가 있다.

- 잠긴 row가 계속 건너뛰어져 오래 남을 수 있다.
- 엄격한 순서 처리가 어렵다.
- 어떤 작업이 왜 늦게 처리되는지 추적해야 한다.
- 일반적인 주문/결제 정합성 처리에 무리하게 쓰면 안 된다.

`SKIP LOCKED`는 대기보다 처리량이 중요한 작업 큐에 잘 맞는다. 사용자 요청의 정합성 경로에서는 신중하게 써야 한다.

---

## 어떻게 분석할까

데드락을 볼 때는 "쿼리가 느린가"보다 "어디서 기다리는가"를 먼저 봐야 한다.

같은 SQL이 평소에는 빠른데 특정 시간대에만 느려진다면 락 대기를 의심해야 한다. CPU와 디스크가 여유 있어도 특정 row나 인덱스 범위에서 요청들이 서로 기다릴 수 있다.

### 1. 데드락 로그를 본다

MySQL InnoDB에서는 다음 명령으로 최근 데드락 정보를 볼 수 있다.

```sql
SHOW ENGINE INNODB STATUS\G
```

출력에서 `LATEST DETECTED DEADLOCK` 구간을 본다.

자주 발생하는 데드락을 에러 로그에 남기고 싶다면 `innodb_print_all_deadlocks`를 검토할 수 있다.

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

상시로 켤지는 신중하게 판단해야 한다. 데드락이 많은 상황에서는 로그가 빠르게 늘 수 있다.

봐야 할 내용은 다음이다.

- 어떤 트랜잭션이 victim이 되었는가
- 각 트랜잭션이 실행 중이던 SQL은 무엇인가
- 어떤 락을 보유하고 있었는가
- 어떤 락을 기다리고 있었는가
- 테이블과 인덱스 이름은 무엇인가
- 같은 시간대에 실행된 API, 배치, 컨슈머는 무엇인가

중요한 건 SQL 한 줄만 보는 게 아니다.

그 SQL이 어떤 비즈니스 흐름 안에서 실행됐는지 찾아야 한다.

### 2. lock wait을 본다

데드락은 순환 대기가 실패로 드러난 사건이다.

하지만 그 전 단계에는 lock wait이 있다. lock wait은 성공한 요청에도 지연으로 남는다.

MySQL 8에서는 `performance_schema`를 통해 데이터 락과 대기 관계를 볼 수 있다.

```sql
SELECT *
FROM performance_schema.data_lock_waits;
```

함께 보면 좋은 정보는 다음이다.

```sql
SELECT *
FROM performance_schema.data_locks;
```

DB 버전과 권한에 따라 볼 수 있는 컬럼은 다를 수 있다. 핵심은 기다리는 트랜잭션과 막고 있는 트랜잭션을 연결해서 보는 것이다.

### 3. 실행 계획을 본다

데드락에 등장한 SQL은 `EXPLAIN`으로 실행 계획을 확인한다.

```sql
EXPLAIN
SELECT *
FROM orders
WHERE user_id = 10
  AND status = 'PENDING'
FOR UPDATE;
```

특히 다음을 본다.

- `type`이 지나치게 넓은 접근 방식인지
- `key`가 기대한 인덱스인지
- `rows` 추정치가 과도한지
- `Extra`에 불필요한 filesort나 temporary가 있는지
- 조건절이 복합 인덱스의 왼쪽 prefix를 잘 활용하는지

락 범위는 실행 계획과 연결된다.

어떤 인덱스를 스캔했는지가 곧 어떤 인덱스 범위를 잠글 수 있는지와 연결된다.

### 4. 같은 자원을 수정하는 모든 경로를 찾는다

데드락은 두 SQL만 보고 끝나지 않는다.

같은 테이블을 수정하는 API, 배치, 컨슈머, 관리자 기능을 모두 찾아야 한다.

예를 들어 `orders`, `inventory`가 데드락 로그에 나왔다면 다음을 찾는다.

```text
orders 수정 경로
  - 결제 승인 API
  - 결제 취소 API
  - 주문 만료 배치
  - 관리자 주문 상태 변경
  - 배송 이벤트 컨슈머

inventory 수정 경로
  - 결제 승인 API
  - 재고 보정 배치
  - 입고 처리 API
  - 품절 처리 배치
```

그다음 각 흐름의 락 획득 순서를 적는다.

```text
결제 승인 API: orders -> inventory
재고 보정 배치: inventory -> orders
주문 만료 배치: orders -> order_items -> inventory
관리자 수정: inventory -> orders
```

이렇게 펼쳐놓으면 어디서 순서가 깨지는지 보인다.

### 5. 요청 timeout 예산을 본다

데드락은 DB 안에서만 끝나지 않는다.

애플리케이션 스레드는 DB 응답을 기다리고, DB 커넥션은 반환되지 않는다. 커넥션 풀이 포화되면 데드락과 무관한 조회 요청도 대기한다.

따라서 다음 지표를 같이 봐야 한다.

- deadlock count
- lock wait time
- p95, p99 latency
- retry rate
- final failure rate
- DB connection pool active count
- DB connection acquisition wait time
- slow query log
- long transaction count
- batch execution time

수치 기준은 서비스마다 다르다.

중요한 건 해당 요청의 전체 timeout 예산 중 락 대기가 얼마나 차지하는가다. 1초 안에 끝나야 하는 결제 API에서 300ms lock wait은 이미 큰 신호일 수 있다.

---

## 해결 전략

데드락 대응은 "재시도 추가"로 끝나면 안 된다.

재시도는 필요하다. 하지만 원인을 줄이지 않으면 같은 충돌을 반복할 뿐이다.

순서는 다음처럼 잡는 것이 좋다.

```text
1. 데드락 로그에서 테이블, 인덱스, SQL을 확인한다.
2. victim 트랜잭션의 요청 흐름을 찾는다.
3. 같은 자원을 수정하는 모든 경로를 찾는다.
4. 각 경로의 락 획득 순서를 비교한다.
5. 트랜잭션 안의 외부 작업을 제거한다.
6. UPDATE/DELETE/FOR UPDATE의 실행 계획을 확인한다.
7. 인덱스로 잠금 범위를 줄인다.
8. 제한적 retry와 멱등성을 추가한다.
9. 배치와 사용자 요청의 충돌 지점을 분리한다.
10. 그래도 부족하면 큐, 낙관적 락, 격리 수준 변경을 검토한다.
```

### 1. 트랜잭션 안에서 외부 작업을 제거한다

먼저 트랜잭션 내부 작업을 분류한다.

```text
트랜잭션 안에 남길 것
  - 반드시 함께 성공하거나 실패해야 하는 DB 변경
  - 상태 전이 검증
  - 중복 방지를 위한 unique key insert

트랜잭션 밖으로 뺄 것
  - 외부 API 호출
  - 메시지 발행 대기
  - 파일 처리
  - 긴 계산
  - 대량 조회
```

메시지 발행이 필요하다면 outbox pattern을 고려할 수 있다.

```text
1. BEGIN
2. 주문 상태 변경
3. outbox 테이블에 이벤트 저장
4. COMMIT
5. 별도 publisher가 outbox 이벤트 발행
```

이렇게 하면 DB 변경과 이벤트 기록은 같은 트랜잭션으로 묶고, 실제 메시지 브로커 호출은 트랜잭션 밖으로 뺄 수 있다.

트레이드오프는 outbox 테이블과 publisher 운영이 추가된다는 점이다. 대신 DB 락을 잡은 채 메시지 브로커를 기다리지 않아도 된다.

### 2. 락 획득 순서를 코드로 고정한다

순서를 문서로만 정하면 깨진다.

공통 함수나 공통 repository 경로로 락 획득을 모아야 한다.

```java
// 예시: 개념 코드
public LockedOrderInventory lockForOrderPayment(
    List<Long> orderIds,
    List<Long> productIds
) {
    List<Long> sortedProductIds = productIds.stream().sorted().toList();
    List<Long> sortedOrderIds = orderIds.stream().sorted().toList();

    List<Inventory> inventories =
        inventoryRepository.findAllByProductIdInForUpdate(sortedProductIds);

    List<Order> orders =
        orderRepository.findAllByIdInForUpdate(sortedOrderIds);

    return new LockedOrderInventory(orders, inventories);
}
```

여기서 중요한 점은 입력 순서가 아니라 정렬된 순서로 잠근다는 것이다.

배치도 같은 함수를 쓰거나 같은 SQL 순서를 따라야 한다. API만 고치고 배치가 반대 순서로 잠그면 데드락은 계속 남는다.

### 3. 여러 row는 항상 결정적인 순서로 잠근다

같은 테이블 안에서도 순서가 중요하다.

트랜잭션 A가 `id = 1 -> id = 2` 순서로 잠그고, 트랜잭션 B가 `id = 2 -> id = 1` 순서로 잠그면 같은 테이블 안에서도 데드락이 생길 수 있다.

그래서 여러 row를 잠글 때는 항상 정렬 기준을 둔다.

```sql
SELECT id
FROM orders
WHERE id IN (3, 1, 2)
ORDER BY id
FOR UPDATE;
```

단, SQL에 `ORDER BY`를 썼다고 항상 물리적인 락 획득 순서가 원하는 대로 보장된다고 단정하면 안 된다. 옵티마이저가 선택한 실행 계획에 영향을 받기 때문이다.

가장 안전한 방식은 PK 또는 적절한 인덱스를 통해 정렬된 key 목록을 만들고, 그 key 기준으로 좁게 잠그는 것이다.

### 4. 조건절 인덱스를 잠금 관점에서 설계한다

다음 쿼리가 자주 대기한다고 해보자.

```sql
SELECT id
FROM coupons
WHERE user_id = ?
  AND status = 'ACTIVE'
  AND expires_at > NOW()
FOR UPDATE;
```

이때 `(user_id, status, expires_at)` 인덱스가 있으면 특정 사용자의 활성 쿠폰 범위를 좁게 찾을 수 있다.

하지만 `(status)`만 있으면 `ACTIVE`인 많은 쿠폰을 훑고 `user_id`를 필터링할 수 있다. 잠금 범위가 커지고, 다른 사용자 요청까지 영향을 받을 수 있다.

인덱스 설계는 다음 순서로 본다.

```text
1. 동등 조건 컬럼을 앞에 둔다.
2. 선택도가 높은 컬럼을 고려한다.
3. 범위 조건은 뒤쪽에 둔다.
4. 정렬이나 limit이 있다면 실행 계획으로 확인한다.
5. UPDATE/DELETE/FOR UPDATE에서 실제로 그 인덱스를 쓰는지 본다.
```

예시는 다음과 같다.

```sql
CREATE INDEX idx_coupons_user_status_exp
ON coupons (user_id, status, expires_at);
```

다만 인덱스가 많아지면 쓰기 비용이 늘어난다. 따라서 데드락 로그와 lock wait이 실제로 가리키는 쿼리부터 조정해야 한다.

### 5. 배치는 chunk 단위로 처리한다

대량 배치는 한 번에 끝내려 하지 않는 편이 좋다.

```text
나쁜 흐름
1. BEGIN
2. 만료 쿠폰 50만 건 update
3. 관련 주문 상태 보정
4. COMMIT
```

이 구조는 트랜잭션이 오래 살아 있고, 사용자 요청과 충돌하기 쉽다.

더 나은 흐름은 작은 범위로 나눠 처리하는 것이다.

```text
1. 처리 대상 PK 1000개 조회
2. BEGIN
3. 1000개 update
4. COMMIT
5. 다음 1000개 반복
```

배치가 중간에 실패해도 이어서 재개할 수 있도록 checkpoint를 남긴다.

```text
batch_checkpoint
  - job_name
  - last_processed_id
  - processed_at
```

트레이드오프는 전체 완료 시간이 길어질 수 있다는 점이다. 대신 각 트랜잭션의 락 보유 시간은 줄어든다.

### 6. 사용자 요청과 보정 작업을 분리한다

사용자 요청과 보정 배치가 같은 row를 반대 순서로 수정하면 데드락이 쉽게 생긴다.

보정 작업은 다음 중 하나로 바꿀 수 있다.

- 사용자 요청이 적은 시간대로 이동한다.
- 처리 대상을 작은 chunk로 나눈다.
- 사용자 요청과 같은 락 순서를 사용한다.
- 보정 대상 row를 먼저 snapshot 테이블에 적재하고 PK 기준으로 처리한다.
- 이미 사용자 요청이 처리 중인 row는 건너뛰고 다음 주기에 다시 처리한다.

작업 큐 성격이면 `SKIP LOCKED`를 사용할 수 있다.

```sql
SELECT id
FROM inventory_reconcile_jobs
WHERE status = 'READY'
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

이 방식은 기다리지 않고 처리 가능한 row부터 가져온다.

다만 "정확히 이 순서대로 처리되어야 하는 작업"에는 맞지 않는다. 건너뛴 작업이 늦게 처리될 수 있기 때문이다.

### 7. 재시도는 멱등성과 함께 넣는다

데드락 에러는 재시도 가능한 오류로 분류할 수 있다.

하지만 무조건 재시도하면 안 된다.

```text
재시도 조건
  - DB deadlock 오류인지 식별 가능
  - 전체 요청 timeout 예산이 남아 있음
  - 작업이 멱등적으로 설계됨
  - 외부 부작용 중복이 방지됨
```

결제 요청이라면 다음이 필요하다.

```text
payment_request_id unique
order_id + payment_attempt_no unique
외부 결제 승인 결과 저장
같은 요청 재진입 시 기존 결과 반환
```

재시도 간격은 jitter를 섞는다.

```text
1회차: 50~100ms
2회차: 100~200ms
3회차: 200~400ms
```

최대 횟수는 작게 둔다.

재시도 후에도 실패하면 사용자에게 모호한 성공을 보여주지 않는다. "처리 중" 상태를 저장하고 후속 조회나 보정 작업으로 수렴시키는 방식이 더 안전할 수 있다.

### 8. 설정 변경은 마지막에 검토한다

`innodb_lock_wait_timeout`을 늘리면 실패가 늦게 보인다.

하지만 커넥션 점유 시간도 늘어난다. 대기가 길어지면 애플리케이션 스레드와 DB 커넥션 풀이 먼저 막힐 수 있다.

격리 수준을 낮추면 일부 gap lock은 줄 수 있다.

하지만 정합성 모델이 바뀐다. 기존 코드가 같은 트랜잭션 안에서 반복 조회 결과가 유지된다고 기대했다면 문제가 생길 수 있다.

설정 변경은 원인을 이해한 뒤 보조 수단으로 써야 한다.

먼저 해야 할 일은 다음이다.

- 어떤 트랜잭션이 어떤 락을 보유했는지 확인
- 어떤 트랜잭션이 어떤 락을 기다렸는지 확인
- 락 순서가 다른 코드 경로 찾기
- 트랜잭션 안의 외부 작업 제거
- 인덱스와 실행 계획 확인
- 배치 범위 축소

그 다음에 timeout, isolation level, deadlock detection 설정을 검토한다.

---

## 면접에서는 이렇게 답한다

질문은 이렇게 나올 수 있다.

> 주문 결제 API와 재고 보정 배치가 동시에 실행될 때 간헐적으로 데드락이 발생합니다. 두 로직 모두 단독 실행하면 빠르고 정상입니다. 원인을 어떻게 설명하고, 어떻게 대응하겠습니까?

1분 답변은 이렇게 말할 수 있다.

데드락은 두 트랜잭션이 서로 필요한 락을 가진 채 상대 락을 기다릴 때 발생합니다.

이 시나리오에서는 결제 API가 주문을 먼저 잠그고 재고를 수정하는 반면, 재고 보정 배치가 재고를 먼저 잠그고 주문을 수정한다면 lock ordering이 달라져 순환 대기가 생길 수 있습니다.

DB는 deadlock detection으로 순환 대기를 감지하면 하나의 트랜잭션을 victim으로 선택해 rollback합니다. 그래서 애플리케이션에서는 간헐적 실패나 timeout처럼 보입니다.

대응은 세 단계로 봅니다.

첫째, 데드락 로그에서 어떤 트랜잭션이 어떤 락을 보유하고 어떤 락을 기다렸는지 확인합니다.

둘째, 같은 자원을 수정하는 API, 배치, 컨슈머의 락 획득 순서를 통일하고, 트랜잭션 안의 외부 API 호출이나 메시지 발행 대기를 제거해 락 보유 시간을 줄입니다.

셋째, `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`의 실행 계획을 확인해 조건절에 맞는 인덱스를 추가하거나 조정합니다. InnoDB는 인덱스 레코드와 gap, next-key 범위에 락을 걸 수 있으므로 인덱스는 조회 성능뿐 아니라 잠금 범위에도 영향을 줍니다.

단기적으로는 deadlock 오류에 대해 제한적인 retry와 backoff를 적용할 수 있습니다. 다만 결제처럼 외부 부작용이 있는 흐름은 idempotency key, unique constraint, 상태 전이 검증으로 중복 처리를 막아야 합니다.

핵심 채점 포인트는 다음이다.

- 데드락을 단순 timeout이 아니라 순환 대기로 설명하는가
- 트랜잭션 범위와 락 획득 순서를 함께 보는가
- victim rollback 이후 retry의 멱등성까지 말하는가
- 인덱스가 잠금 범위에 영향을 준다는 점을 아는가
- lock wait timeout과 deadlock detection을 구분하는가

꼬리 질문은 이렇게 이어질 수 있다.

### lock wait timeout과 deadlock detection은 무엇이 다른가?

lock wait timeout은 락을 기다리다가 설정 시간을 넘겨 실패하는 것이다. 순환 대기가 없어도 앞 트랜잭션이 오래 걸리면 발생할 수 있다.

deadlock detection은 DB가 대기 그래프에서 순환 대기를 발견하고 한 트랜잭션을 즉시 rollback하는 것이다. timeout까지 기다리지 않는다.

### 인덱스가 없을 때 UPDATE 데드락 가능성이 왜 커질 수 있는가?

조건절에 맞는 인덱스가 없으면 DB가 더 넓은 범위를 스캔한다.

InnoDB는 row lock을 인덱스 레코드 기준으로 잡기 때문에, 스캔한 인덱스 레코드와 범위가 넓어질수록 다른 트랜잭션과 충돌할 표면적이 커진다. 필요한 row만 좁게 찾는 인덱스가 있으면 잠금 범위와 대기 시간이 줄어들 수 있다.

### 데드락 발생 시 무조건 retry하면 왜 위험한가?

재시도는 같은 자원에 다시 요청을 보내는 것이다.

즉시 재시도하면 이미 충돌한 주문, 재고, 사용자 잔액에 트래픽이 다시 몰린다. 충돌이 줄지 않고 부하만 늘 수 있다.

그래서 retry는 횟수를 제한하고, backoff와 jitter를 넣고, 전체 timeout 예산 안에서만 수행해야 한다. 결제나 쿠폰 같은 부작용이 있는 흐름은 idempotency key와 unique constraint가 필요하다.

---

## 요약

데드락은 여러 트랜잭션이 같은 자원을 서로 다른 순서로 잡을 때 발생한다.

핵심은 락의 개수가 아니라 순서와 보유 시간이다.

트랜잭션 안에서 잡은 쓰기 락은 보통 `COMMIT` 또는 `ROLLBACK`까지 유지된다. 그래서 트랜잭션 범위가 길수록 lock wait과 데드락 가능성이 함께 커진다.

인덱스는 조회 성능뿐 아니라 잠금 범위에도 영향을 준다. InnoDB는 인덱스 레코드 기준으로 락을 잡고, 범위 조건에서는 gap lock과 next-key lock이 개입할 수 있다.

DB의 deadlock detection은 시스템 전체 멈춤을 막기 위해 victim rollback을 수행한다. 하지만 애플리케이션 요청은 실패하므로, 멱등성을 갖춘 제한적 retry와 backoff가 필요하다.

데드락을 줄이는 첫 번째 작업은 설정 변경이 아니다.

같은 자원을 수정하는 모든 코드 경로가 어떤 순서로 락을 잡는지 확인하고, 트랜잭션 범위를 줄이고, 조건절 인덱스로 잠금 범위를 좁히는 것이다.

가장 중요한 질문은 이것이다.

> 이 트랜잭션은 어떤 자원을 어떤 순서로 잠그고, 그 락을 얼마나 오래 들고 있는가?

이 질문에 답할 수 있으면 데드락은 운이 나쁜 간헐 오류가 아니라, 구조적으로 줄일 수 있는 동시성 문제로 보인다.

---

## 참고 자료

- [MySQL Reference Manual - InnoDB Locking](https://dev.mysql.com/doc/en/innodb-locking.html)
- [MySQL Reference Manual - Deadlocks in InnoDB](https://dev.mysql.com/doc/en/innodb-deadlocks.html)
- [MySQL Reference Manual - Deadlock Detection](https://dev.mysql.com/doc/refman/8.3/en/innodb-deadlock-detection.html)
- [MySQL Reference Manual - An InnoDB Deadlock Example](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-example.html)
- [MySQL Reference Manual - Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/9.1/en/innodb-locks-set.html)
- [MySQL Reference Manual - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.1/en/innodb-transaction-isolation-levels.html)
- [PostgreSQL Documentation - Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
