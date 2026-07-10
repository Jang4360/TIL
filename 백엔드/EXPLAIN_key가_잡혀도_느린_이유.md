# EXPLAIN에서 key가 잡혔는데 왜 여전히 느릴까

로컬에서는 100ms 안에 끝나던 검색 API가, 데이터가 몇십만 건 쌓이자 3초까지 늘어났다.

인덱스는 이미 있다. `EXPLAIN`을 봐도 `key` 컬럼에 인덱스 이름이 잡혀 있다. 그런데도 느리다.

여기서 자주 놓치는 점이 있다.

> 인덱스를 탔다는 것과 인덱스를 잘 탔다는 것은 다르다.

`EXPLAIN`은 쿼리가 어떤 경로로 데이터에 도달하는지 보여준다. 하지만 그 경로가 실제로 충분히 좋은지는 `key` 하나만 보고 알 수 없다.

---

## 먼저 구조를 잡자

`EXPLAIN`은 옵티마이저가 세운 실행 계획이다.

쿼리를 파싱한 뒤 어떤 인덱스를 쓸지, 어떤 순서로 조인할지, 몇 행을 훑을 예정인지를 보여준다. 중요한 건 실제 실행 결과가 아니라 예측이라는 점이다.

예를 들어 다음 쿼리를 보자.

```sql
EXPLAIN
SELECT id, title, created_at
FROM articles
WHERE category = 'backend'
ORDER BY created_at DESC
LIMIT 20;
```

결과가 이렇게 나왔다고 하자.

```text
+----+-------------+----------+-------+---------------+--------------+-------+----------+-----------------------------+
| id | select_type | table    | type  | possible_keys | key          | rows  | filtered | Extra                       |
+----+-------------+----------+-------+---------------+--------------+-------+----------+-----------------------------+
|  1 | SIMPLE      | articles | range | idx_category  | idx_category | 12034 |    12.50 | Using where; Using filesort |
+----+-------------+----------+-------+---------------+--------------+-------+----------+-----------------------------+
```

이 한 줄이 말하는 것은 이렇다.

`idx_category` 인덱스를 `range` 방식으로 탔다. 하지만 1만 2천 행 정도를 훑을 예정이고, 그중 조건을 통과하는 비율은 12.5%로 예상된다. 게다가 `ORDER BY created_at DESC`는 인덱스 순서로 해결하지 못해서 `Using filesort`가 붙었다.

인덱스는 잡혔다.

하지만 비용은 여전히 새고 있다.

---

## 다섯 개만 먼저 본다

처음부터 모든 컬럼을 외우려고 하면 오래 못 간다.

먼저 다섯 개만 보면 된다.

| 컬럼 | 의미 | 봐야 할 신호 |
| --- | --- | --- |
| `type` | 테이블 접근 방식 | `ALL`, `index`, `range`는 비용을 의심 |
| `key` | 실제 선택된 인덱스 | null 여부보다 어떤 인덱스인지가 중요 |
| `rows` | 훑을 것으로 예상한 행 수 | 클수록 위험 |
| `filtered` | 조건을 통과할 것으로 예상한 비율 | 낮으면 많이 읽고 많이 버리는 상황 |
| `Extra` | 부가 작업 | `Using filesort`, `Using temporary`는 비용 신호 |

이 다섯 개를 한 줄로 읽으면 된다.

```text
어떤 방식(type)으로
어떤 인덱스(key)를 타서
몇 행(rows)을 읽고
얼마나 남기며(filtered)
추가 작업(Extra)을 하는가
```

`key`는 그중 하나일 뿐이다.

---

## type은 접근 비용의 층위다

`type`은 DB가 테이블에 접근하는 방식이다.

대략 다음 순서로 무거워진다.

```text
const -> ref -> range -> index -> ALL
```

`const`는 primary key나 unique key로 한 건을 찾는 형태에 가깝다.

`ref`는 인덱스로 여러 후보를 찾는다.

`range`는 인덱스의 특정 구간을 스캔한다.

`index`는 인덱스 전체를 훑는다.

`ALL`은 테이블 풀 스캔이다.

여기서 `range`와 `index`를 헷갈리면 안 된다.

`range`는 인덱스의 일부 구간을 읽는 것이다. `index`는 인덱스 전체를 읽는 것이다. 둘 다 인덱스가 보일 수 있지만 비용은 전혀 다르다.

`type: index`는 "인덱스를 탔으니 괜찮다"가 아니라 "테이블 대신 인덱스 전체를 훑고 있다"에 가까울 수 있다.

---

## rows와 filtered가 실제 비용을 드러낸다

`key`가 잡혔는데 느린 가장 흔한 이유는 많이 읽고 많이 버리기 때문이다.

예를 들어 다음처럼 나온다고 하자.

```text
rows: 100000
filtered: 1.00
```

옵티마이저는 대략 10만 행을 읽고, 그중 1%만 조건을 통과할 것으로 본다.

즉 인덱스를 타긴 했지만 대부분을 버린다.

이런 경우는 보통 세 가지를 의심한다.

```text
1. 인덱스 컬럼 순서가 WHERE 조건과 맞지 않는다.
2. 선택도가 낮은 컬럼만 인덱스로 잡혔다.
3. 복합 인덱스가 ORDER BY나 LIMIT까지 살리지 못한다.
```

예를 들어 검색 조건이 다음과 같다고 하자.

```sql
WHERE category = 'backend'
ORDER BY created_at DESC
LIMIT 20
```

`idx_category(category)`만 있으면 category 조건은 찾을 수 있다.

하지만 그 안에서 `created_at DESC` 정렬은 따로 해야 한다. category 안의 데이터가 많아지면 `rows`가 커지고 `Using filesort`가 붙을 수 있다.

이 경우 후보 인덱스는 다음처럼 달라질 수 있다.

```sql
CREATE INDEX idx_articles_category_created
ON articles (category, created_at DESC);
```

이 인덱스는 `category = 'backend'` 구간으로 들어간 뒤, 그 안에서 `created_at` 순서로 읽다가 `LIMIT 20`에서 멈출 수 있다.

핵심은 인덱스 유무가 아니라 쿼리 모양에 맞는 인덱스인지다.

---

## Extra는 숨은 비용 신호다

`Extra`는 쿼리의 부가 작업을 보여준다.

자주 보는 신호는 다음이다.

| Extra | 의미 | 해석 |
| --- | --- | --- |
| `Using index` | 커버링 인덱스로 처리 | 대체로 좋은 신호 |
| `Using where` | 읽은 뒤 조건 필터링 | 보통 함께 등장 |
| `Using filesort` | 인덱스 순서로 정렬 못함 | 정렬 비용 의심 |
| `Using temporary` | 임시 테이블 사용 | GROUP BY, DISTINCT 비용 의심 |

`Using index`는 쿼리가 필요한 컬럼을 인덱스만으로 해결했다는 뜻이다.

예를 들어 다음 쿼리가 있다.

```sql
SELECT id, title, created_at
FROM articles
WHERE category = 'backend'
ORDER BY created_at DESC
LIMIT 20;
```

인덱스가 다음처럼 되어 있다면 커버링이 될 수 있다.

```sql
CREATE INDEX idx_articles_category_created_cover
ON articles (category, created_at DESC, id, title);
```

테이블 row를 다시 읽지 않고 인덱스만으로 응답할 수 있다.

하지만 커버링 인덱스는 공짜가 아니다.

인덱스가 커지고, 쓰기 비용이 늘고, buffer pool 사용량도 늘어난다. 자주 호출되는 목록 API처럼 읽기 hot path에만 신중하게 적용해야 한다.

반대로 `Using filesort`는 정렬을 인덱스 순서로 해결하지 못했다는 신호다.

데이터가 적을 때는 티가 안 난다. 하지만 프로덕션 데이터가 커지면 정렬 비용이 갑자기 튄다.

`Using temporary`는 `GROUP BY`, `DISTINCT` 등을 처리하기 위해 임시 테이블을 만든다는 뜻이다. 메모리에서 끝나면 버틸 수 있지만, 디스크로 넘어가면 지연이 초 단위로 커질 수 있다.

---

## EXPLAIN은 예측이고 실제 실행은 다를 수 있다

MySQL 공식 문서 기준으로 `EXPLAIN`은 옵티마이저가 선택한 실행 계획을 보여준다.

여기서 `rows`와 `filtered`는 실제 측정값이 아니라 추정값이다. 옵티마이저는 테이블 통계, 인덱스 카디널리티, 히스토그램 같은 정보를 보고 판단한다.

그래서 통계가 오래됐거나 데이터 분포가 바뀌면 예측이 빗나간다.

예를 들어 원래는 `category = 'backend'`가 전체의 5%였는데, 지금은 60%가 됐다고 하자. 통계가 이를 반영하지 못하면 옵티마이저는 여전히 작은 범위라고 생각하고 잘못된 인덱스를 선택할 수 있다.

이때는 통계를 갱신한다.

```sql
ANALYZE TABLE articles;
```

MySQL 8.4 문서도 InnoDB optimizer statistics를 즉시 갱신하려면 `ANALYZE TABLE`을 사용할 수 있다고 설명한다.

실제 실행 비용까지 보고 싶다면 `EXPLAIN ANALYZE`를 본다.

```sql
EXPLAIN ANALYZE
SELECT id, title, created_at
FROM articles
WHERE category = 'backend'
ORDER BY created_at DESC
LIMIT 20;
```

`EXPLAIN ANALYZE`는 실제로 쿼리를 실행하고, 옵티마이저의 예상과 실제 실행 정보를 함께 보여준다. 단, 실제 실행이므로 운영 환경에서는 비용과 부작용을 고려해야 한다.

---

## 판단 순서

`EXPLAIN`을 볼 때는 다음 순서로 읽는다.

```text
1. type이 ALL 또는 index인가?
2. key가 기대한 인덱스인가?
3. rows가 너무 크지 않은가?
4. filtered가 너무 낮지 않은가?
5. Extra에 Using filesort나 Using temporary가 있는가?
6. slow query log의 실제 지연과 맞는가?
7. 통계가 오래되지는 않았는가?
```

상황별로 보면 이렇게 판단한다.

```text
type: ALL
  -> 인덱스 미사용. WHERE 조건과 인덱스를 먼저 확인한다.

type: index
  -> 인덱스 전체 스캔. 커버링이라 괜찮은지, 아니면 범위 조건이 빠진 건지 본다.

rows 큼 + filtered 낮음
  -> 많이 읽고 많이 버리는 상황. 복합 인덱스 컬럼 순서를 의심한다.

Using filesort
  -> ORDER BY를 인덱스로 처리할 수 있는지 본다.

Using temporary
  -> GROUP BY, DISTINCT가 인덱스로 처리되는지 본다.

EXPLAIN은 괜찮은데 slow query에 잡힘
  -> 통계 왜곡, 데이터 분포 변화, 실제 실행 계획 차이를 의심한다.
```

EXPLAIN을 읽는 힘은 컬럼을 외우는 데 있지 않다.

신호와 원인을 연결하는 데 있다.

---

## 인덱스를 추가하기 전에 볼 트레이드오프

인덱스를 추가하면 조회는 빨라질 수 있다.

하지만 쓰기는 비싸진다.

`INSERT`, `UPDATE`, `DELETE` 때마다 해당 인덱스도 함께 유지해야 한다. 인덱스가 커지면 저장 공간과 buffer pool 사용량도 늘어난다.

그래서 다음 질문을 먼저 한다.

```text
이 쿼리는 자주 호출되는가?
사용자 응답 시간에 직접 영향을 주는가?
rows와 filtered를 실제로 줄이는 인덱스인가?
ORDER BY나 LIMIT까지 같이 해결하는가?
커버링으로 만들 가치가 있는가?
이 컬럼은 자주 update되는가?
관리자 페이지처럼 호출 빈도가 낮은 경로인가?
```

조회 빈도가 낮은 관리자 페이지라면 인덱스 추가보다 쿼리 조건을 바꾸거나, 비동기 리포트로 분리하거나, 검색 전용 테이블을 두는 편이 나을 수 있다.

반대로 사용자 검색, 상품 목록, 종목 리스트처럼 자주 호출되는 hot path라면 복합 인덱스와 커버링 인덱스가 큰 효과를 낼 수 있다.

---

## 면접에서는 이렇게 답한다

질문은 이렇게 나올 수 있다.

> EXPLAIN에서 key 컬럼에 인덱스가 잡혔는데도 쿼리가 느립니다. 어디를 보시겠어요?

1분 답변은 이렇게 말할 수 있다.

인덱스를 탔다는 것과 잘 탔다는 것은 다르게 봅니다.

먼저 `type`, `rows`, `filtered`, `Extra`를 같이 봅니다. `key`가 잡혀도 `rows`가 크고 `filtered`가 낮으면 인덱스로 많은 행을 읽고 대부분을 버리는 상황이라 여전히 느릴 수 있습니다.

그다음 `Using filesort`나 `Using temporary`가 있는지 봅니다. `ORDER BY`나 `GROUP BY`를 인덱스 순서로 처리하지 못하면 데이터가 커질수록 지연이 크게 늘 수 있습니다.

또 `EXPLAIN`은 통계 기반 예측이므로 실제 실행과 다를 수 있습니다. 데이터 분포가 바뀌었거나 통계가 낡았다면 `ANALYZE TABLE`로 통계를 갱신하고, 필요하면 `EXPLAIN ANALYZE`로 실제 실행 정보를 봅니다.

마지막으로 인덱스를 추가할 때는 쓰기 비용도 같이 봅니다. 복합 인덱스로 `WHERE`, `ORDER BY`, `LIMIT`을 함께 줄일 수 있는지 확인하되, 호출 빈도가 낮은 쿼리라면 인덱스 추가보다 쿼리나 화면 요구사항을 바꾸는 게 나을 수도 있습니다.

꼬리 질문은 이렇게 이어질 수 있다.

### EXPLAIN에서 `key`가 null이면 무조건 문제인가요?

항상 그렇지는 않다.

테이블이 아주 작으면 풀 스캔이 더 빠를 수 있다. 조건 컬럼의 선택도가 낮아도 옵티마이저가 인덱스 이점이 없다고 판단할 수 있다.

판단 기준은 `key` null 자체가 아니라 실행 시간, slow query log, rows, 데이터 크기, 호출 빈도다.

### `Using filesort`는 항상 나쁜가요?

항상 나쁜 것은 아니다.

정렬 대상이 작으면 큰 문제가 아닐 수 있다. 하지만 사용자 요청 hot path에서 정렬 대상이 계속 커진다면 위험 신호다.

`ORDER BY`를 복합 인덱스에 포함해 정렬을 인덱스 순서로 처리할 수 있는지 확인한다.

### `filtered`가 낮으면 무조건 인덱스를 바꿔야 하나요?

무조건은 아니다.

반환 데이터가 작고 호출 빈도가 낮으면 괜찮을 수 있다. 하지만 hot path에서 `rows`가 크고 `filtered`가 낮다면 복합 인덱스 컬럼 순서나 조건식을 다시 봐야 한다.

---

## 요약

`EXPLAIN`에서 `key`가 잡혔다고 쿼리가 잘 최적화된 것은 아니다.

`key`는 실제 선택된 인덱스일 뿐이다. 같이 봐야 할 것은 `type`, `rows`, `filtered`, `Extra`다.

`rows`가 크고 `filtered`가 낮으면 많이 읽고 많이 버리는 구조다. 이때는 복합 인덱스 컬럼 순서와 선택도를 의심한다.

`Using filesort`는 정렬을 인덱스로 해결하지 못했다는 신호다. `Using temporary`는 그룹핑이나 distinct 과정에서 임시 테이블을 만든다는 신호다.

`EXPLAIN`은 예측이다. 통계가 낡거나 데이터 분포가 바뀌면 실제 실행과 다를 수 있다. 필요하면 `ANALYZE TABLE`로 통계를 갱신하고 `EXPLAIN ANALYZE`로 실제 실행 정보를 확인한다.

인덱스 추가는 공짜가 아니다.

조회는 빨라질 수 있지만 쓰기 비용, 저장 공간, buffer pool 사용량이 늘어난다.

가장 중요한 질문은 이것이다.

> 이 인덱스는 쿼리가 읽는 행 수와 버리는 행 수를 실제로 줄이는가?

이 질문에 답할 수 있을 때 `EXPLAIN`은 단순한 표가 아니라 병목을 찾는 도구가 된다.

---

## 참고 자료

- [MySQL Reference Manual - EXPLAIN Output Format](https://dev.mysql.com/doc/en/explain-output.html)
- [MySQL 8.4 Reference Manual - EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [MySQL 8.4 Reference Manual - ANALYZE TABLE Statement](https://dev.mysql.com/doc/refman/8.4/en/analyze-table.html)
- [MySQL 8.4 Reference Manual - Optimizer Statistics](https://dev.mysql.com/doc/refman/8.4/en/optimizer-statistics.html)
