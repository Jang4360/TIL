# TIL — 트랜잭션 격리 수준과 MVCC, Phantom Read, Gap Lock (MySQL InnoDB)

## 0. 한 줄 정의
- 트랜잭션 격리 수준(Isolation Level): 동시에 실행되는 트랜잭션들이 서로의 변경을 어느 범위까지 볼 수 있는지 정하는 정책이다.
- 낮은 격리 수준: 동시 처리량↑ / 정합성 오류(Dirty Read 등)↑
- 높은 격리 수준: 정합성↑ / Lock 경합·대기↑

---

## 1) 격리 수준 4단계와 대표 현상

### 1. READ UNCOMMITTED
- 트랜잭션의 변경 내용이 Commit/Rollback과 무관하게 다른 트랜잭션에 보인다.
- 문제: Dirty Read
  - 커밋되지 않은 값을 읽었다가 롤백되면, 읽은 값이 존재하지 않는 값이 된다.

### 2. READ COMMITTED
- 다른 트랜잭션의 변경 내용은 Commit 되어야만 조회 가능하다.
- 문제: Non-Repeatable Read
  - 같은 트랜잭션에서 같은 행을 두 번 읽었는데, 사이에 다른 트랜잭션이 커밋하면 값이 달라진다.

### 3. REPEATABLE READ
- 트랜잭션 내내 처음 잡은 스냅샷(읽기 기준)을 유지한다.
- 문제(이론): Phantom Read
  - 조회 결과의 행 개수가 달라지는 현상
  - 단, MySQL InnoDB는 MVCC + Next-Key Lock/Gap Lock 조합으로 Phantom을 상당 부분 방지할 수 있다.

### 4. SERIALIZABLE
- 트랜잭션들이 순차 실행된 것과 같은 결과를 보장한다.
- 보통 범위 잠금/락 경합이 커져서 처리량이 감소한다.

---

## 2) 핵심 전제: 트랜잭션 시작 시 스냅샷(읽기 기준)을 만든다
- 스냅샷은 데이터 전체 복사본이 아니다.
- 트랜잭션 시작 시점(또는 격리 수준에 따른 시점)에 맞는 버전만 보이도록 논리적으로 필터링한다.

---

## 3) MVCC (Multi-Version Concurrency Control)

### 3.1 MVCC란
- 데이터를 수정할 때 기존 데이터를 덮어쓰지 않고, 이전 버전을 Undo Log에 유지한다.
- 읽기는 내가 볼 수 있는 버전을 선택해서 보여준다.

### 3.2 스냅샷(Snapshot)과 Read View
- 논리적 스냅샷: 트랜잭션이 시작될 때 DB가 현재 실행 중인 다른 트랜잭션 ID 목록을 기록한다.
- 이 목록을 Read View라고 부른다.

### 3.3 버전 체인(Version Chain)
- 각 행(Row)은 수정 이력을 체인 형태로 가진다(Undo Log 활용).

예시
- 행 A: [버전3] <- [버전2] <- [버전1]
- 행 B: [버전5] <- [버전4]

### 3.4 스냅샷 읽기(Snapshot Read)
- 내가 읽는 시점에 현재 행 버전이 내 트랜잭션보다 최신이면,
  - Undo 영역을 따라 내려가서 내 Read View에 맞는 가장 최신 버전을 읽는다.

### 3.5 MVCC의 성질
- MVCC는 읽기/쓰기 충돌을 줄인다.
- 일반 SELECT는 쓰기(UPDATE/INSERT)와 서로 방해하지 않는 방향으로 동작한다.
  - 단, SELECT ... FOR UPDATE 같은 잠금 읽기는 예외다.

---

## 4) Read Committed vs Repeatable Read (스냅샷 시점 차이)

| 구분 | Read Committed | Repeatable Read |
| --- | --- | --- |
| 스냅샷 시점 | SELECT 쿼리마다 최신 스냅샷 갱신 | 트랜잭션 내 첫 SELECT 기준으로 스냅샷 고정 |
| 결과 | 다른 트랜잭션 커밋이 즉시 반영될 수 있음 | 트랜잭션 내내 같은 데이터(같은 버전)가 보임 |
| 대표 현상 | Non-Repeatable Read 가능 | 이론상 Phantom Read 가능 |

---

## 5) REPEATABLE READ는 값 변경은 막지만 행 개수 변화는 완벽 보장 못한다
- MVCC(스냅샷)는 기존에 존재하던 행의 버전 선택에는 강하다.
- 하지만 없던 행이 새로 INSERT 되는 사건은 스냅샷만으로 100% 통제하기 어렵다.
- 그래서 상용 DB는 MVCC만 믿지 않고 물리적 잠금(락)을 함께 사용한다.

---

## 6) Phantom Read와 물리적 잠금: Gap Lock, Next-Key Lock

### 6.1 Gap Lock
- 특정 인덱스 구간의 빈 공간(gap)에 락을 걸어 그 사이로 INSERT가 들어오는 것을 차단한다.

예시(개념)
- 키가 1과 5가 존재할 때, 1~5 사이 gap에 락을 걸면
  - 다른 트랜잭션이 그 범위에 INSERT하려고 할 때 내 트랜잭션 종료까지 대기한다.

### 6.2 Next-Key Lock
- 레코드 락(행 자체) + 갭 락(행 사이 범위)을 결합한 형태로 이해하면 된다.
- MySQL InnoDB는 Repeatable Read에서도
  - MVCC 위에 Next-Key Lock/Gap Lock을 얹어 Phantom을 방지하는 방향으로 동작한다.

---

## 7) Oracle은 READ COMMITTED, MySQL(InnoDB)은 REPEATABLE READ가 기본인 이유

| DB | 기본 격리 수준 | 이유(철학/배경) |
| --- | --- | --- |
| Oracle | Read Committed | 동시성 극대화. 커밋된 데이터는 즉시 공유 가능하다는 철학. MVCC로 읽기/쓰기 충돌을 줄이므로 기본을 RC로 둔다. |
| MySQL (InnoDB) | Repeatable Read | 전통적으로 복제(Replication) 안정성 관점이 컸다. 트랜잭션 순서/일관성 이슈를 줄이기 위해 RR을 기본으로 택했고 관례로 이어졌다. |

---

## 8) REPEATABLE READ에서 Phantom Read가 발생하는 핵심 트리거

### 8.1 스냅샷의 한계
- Read View는 이미 존재하는 행은 버전 선택이 가능하다.
- 하지만 없던 행이 INSERT로 생김은 스냅샷만으로 완벽한 차단 근거가 약해지는 지점이 있다.

### 8.2 결정적 트리거: 쓰기 작업(UPDATE/DELETE) 시점
- 트랜잭션 내에서 UPDATE/DELETE 같은 쓰기 작업을 수행하면,
  - DB가 단순 스냅샷만 보지 않고 최신 실제 레코드를 찾아가는 경로가 생긴다.
- 이때 다른 트랜잭션이 INSERT한 새 행이 범위에 들어와 있고,
  - 내가 그 행을 업데이트해버리면
  - 그 행이 내 트랜잭션 관점에서 갑자기 존재하게 되는 현상으로 나타날 수 있다(Phantom).

---

## 9) SELECT FOR UPDATE는 언제 쓰나

### 9.1 의미
- 조회 시점에 해당 레코드(및 조건에 따라 범위)에 쓰기 잠금(X Lock)을 건다.
- 읽는 동안 절대 바꾸지 마라를 DB에 선언하는 방식이다.

### 9.2 사용 시점
1) 재고/결제(잔액 차감)
- 조회 후 차감 로직에서 중간에 다른 트랜잭션이 값을 바꾸면 문제가 된다.
- Lost Update, 이중 차감 같은 문제를 방지하려면 잠금 읽기가 필요하다.

2) Phantom 방지(범위 INSERT 차단)
- Repeatable Read 환경에서 조회 범위에 대해 INSERT까지 막아야 하면 사용한다.
- 인덱스 범위 조건에서 FOR UPDATE가 걸리면 Gap Lock/Next-Key Lock이 적용되어
  - 해당 범위로 들어오는 INSERT가 대기하게 된다(조건·인덱스·옵티마이저에 영향).

### 9.3 한 줄 요약
- 정합성을 위해 읽는 순간부터 물리적으로 차단해야 하는 구간에 쓴다.

---

## 10) 실무 관점 정리
1. 격리 수준은 트랜잭션 간 가시성 정책이다.
2. 낮은 수준은 처리량에 유리하지만 정합성 오류가 발생한다. 높은 수준은 락 경합이 증가한다.
3. 실무는 성능 때문에 READ COMMITTED를 주로 사용한다. 정산/정확성 구간은 REPEATABLE READ + 비관적 락(SELECT FOR UPDATE)로 필요한 행/범위만 명시적으로 잠그는 전략을 쓴다.

---

## 11) 키워드 요약
- Dirty Read: 커밋되지 않은 값 읽기
- Non-Repeatable Read: 같은 행을 다시 읽었을 때 값이 바뀜
- Phantom Read: 같은 조건으로 다시 조회했을 때 행 개수가 바뀜
- MVCC: Undo Log 기반 다중 버전 관리
- Read View: 트랜잭션 시작 시점의 논리적 스냅샷 기준(트랜잭션 ID 집합)
- Gap Lock: 인덱스 구간 빈 공간 잠금(INSERT 차단)
- Next-Key Lock: 레코드 락 + 갭 락 조합(범위 정합성 강화)