---
layout: single
title: "[티어리스트] 쿼리 개선기 - (4) 쿼리 개선"
categories: dbms
tag: [query-optimization, explain-analyze, mysql, querydsl, querydsl-sql, ngrinder]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/BO2Io0W.png"
# search: false
---

# 쿼리 개선

이전 포스트에서 분석한 실행 계획을 가지고 어떻게 쿼리를 개선할 수 있을지 고민해보자.

1. 대규모 정렬 작업
2. 매우 큰 임시 테이블 사용
3. 비효율적인 조인 작업 (특히 `tierlist_like`와 `member` 테이블)
4. 인덱스 미사용으로 인한 풀 테이블 스캔
5. 반복되는 필터링 작업

위와 같은 문제가 존재했다.

## 개선 방안

1. **조인 버퍼의 크기를 늘리기**
	- 해시 조인은 메모리를 많이 사용한다. 따라서 조인 버퍼를 늘리는 것이 성능을 개선하는 방법이 될 수 있다. 하지만 더 많은 데이터가 생길 경우 다시 문제가 발생한다. 따라서 **근본적인 해결 방법이 될 수 없다.**

2. **인덱스 추가**:
    - `t` 테이블과 `tl` 테이블에서 인덱스를 추가해 전체 테이블 스캔을 방지할 수 있다. 하지만 인덱스를 통해 데이터를 읽어온다고 해도 한번에 많은 `row`를 불러오게 된다. 따라서 **근본적인 해결방법이라고 할 수 없다.**
	
3. **임시 테이블과 정렬 최적화**:
    - `Using temporary`와 `Using filesort`가 성능에 큰 영향을 미치므로, **데이터 정렬을 최소화할 수 있도록 쿼리를 변경하고 인덱스를 통해 효율적으로 정렬할 수 있다.**

4. **서브 쿼리를 통한 `row` 수 줄이기**
	- 많은 `row` 수를 불러옴과 동시에 임시 테이블을 통한 정렬 작업이 많은 성능 저하를 일으키므로 **서브쿼리를 통해 조인 작업을 제거해 처리할 데이터 양을 줄일 수 있다.**
	  -  `tierlist`와 `tierlist_like` 테이블을 조회하면서 `row`의 수가 뻥튀기 된다. 이를 **서브쿼리로 분리하고 `JOIN`해 불러오는 `row` 수 자체를 줄일 수 있다.**
	- 서브쿼리를 통해 **`GROUP BY`를 삭제**하고 **`DISTINCT` 서브 쿼리를 통해 필터링 작업을 줄이고 임시 테이블 생성을 회피**할 수 있다.

## 개선된 쿼리

위의 개선 방안에 맟춰 다음과 같이 쿼리를 개선할 수 있다.

### INDEX 추가

`member` 테이블에는 이미 `email`이 유니크 제약조건으로 걸려 있어 인덱스 생성이 되어있다.

다음은 `tierlist_like`를 살펴보자.

```sql
ALTER TABLE tierlist_like ADD INDEX idx_tierlist_id_member_id (tierlist_id, member_id);
```

`tierlist_like`는 `tierlist_id`로 많이 조회된다. 따라서 해당 컬럼에 대한 인덱스를 생성한다. 또한 조회자의 `member_id`가 존재하는지 자주 확인하기 때문에 복합 인덱스를 추가해 조회 성능을 최적화 할 수 있다.

### SQL

```sql
SELECT 
    t.id,
    t.title,
    t.thumbnail_image AS thumbnailImage,
    t.created_at AS createdAt,
    t.like_count AS likeCount,
    t.comment_count AS commentCount,
    IF(v.id IS NOT NULL, 1, 0) AS liked,
    t.is_published AS isPublished,
    w.id AS writerId,
    w.nickname AS writerNickname,
    w.profile_image AS writerProfileImage,
    tp.id AS topicId,
    tp.name AS topicName,
    c.id AS categoryId,
    c.name AS categoryName
FROM 
    tierlist t
    INNER JOIN topic tp ON t.topic_id = tp.id
    INNER JOIN category c ON tp.category_id = c.id
    INNER JOIN member w ON t.member_id = w.id
    LEFT JOIN (
        SELECT DISTINCT tierlist_id 
        FROM tierlist_like 
        WHERE member_id = (SELECT id FROM member WHERE email = 'M00001@email.com' LIMIT 1)
    ) tl ON t.id = tl.tierlist_id
    LEFT JOIN member v ON v.email = 'M00001@email.com' AND tl.tierlist_id IS NOT NULL
WHERE 
    t.is_published = true
ORDER BY 
    t.like_count DESC, t.created_at DESC
LIMIT 20 OFFSET 0;
```

개선된 점은 다음과 같다.

1. 서브쿼리 사용 : `tierlist_like`와 `member`테이블을 복합적으로 조인하는 부분을 서브쿼리로 변경하며 처리할 `row`수 자체를 줄였다.
2. `GROUP BY` 제거 : `DISTINCT` 서브쿼리를 사용해 `GROUP BY`가 필요하지 않아졌다. `DISTINCT` 사용으로 `tierlist_like` 테이블에서 중복을 제거해 처리할 `row` 수를 줄일 수 있다.
3. 불필요한 `JOIN` 제거 : `tierlist_like`와 `member` 테이블을 조인하는 부분을 서브쿼리로 처리하여 처리해야할 `row`수를 줄이고 필터링에서 큰 `loops`를 가지던 구문을 제거할 수 있다.
4. `IF`문 사용: `CASE`문 보다 `IF`문이 소폭 성능을 개선할 수 있다고 한다.

## 쿼리 성능 측정

그러면 개선된 쿼리를 가지고 DBeaber를 통해 성능을 측정해보자.

![](https://i.imgur.com/a5aBQtJ.png)

개선 전 쿼리의 소요 시간이다. 약 2000ms의 소요 시간이 걸렸다.

![](https://i.imgur.com/vDTNS3L.png)

위는 쿼리 개선 후의 소요 시간이다. 단순 소요 시간으로만 봐도 10배 정도 개선됨을 확인할 수 있다.

## 개선된 쿼리 분석

### 실행 계획 분석 (`EXPLAIN` )

| id  | select_type | table         | partitions | type   | possible_keys             | key                       | key_len | ref                     | rows   | filtered | Extra                                 |
| --- | ----------- | ------------- | ---------- | ------ | ------------------------- | ------------------------- | ------- | ----------------------- | ------ | -------- | ------------------------------------- |
| 1   | PRIMARY     | t             |            | ALL    |                           |                           |         |                         | 99495  | 10.0     | Using where; Using filesort           |
| 1   | PRIMARY     | tp            |            | eq_ref | PRIMARY                   | PRIMARY                   | 8       | tierlist.t.topic_id     | 1      | 100.0    | Using where                           |
| 1   | PRIMARY     | c             |            | eq_ref | PRIMARY                   | PRIMARY                   | 8       | tierlist.tp.category_id | 1      | 100.0    |                                       |
| 1   | PRIMARY     | w             |            | eq_ref | PRIMARY                   | PRIMARY                   | 8       | tierlist.t.member_id    | 1      | 100.0    |                                       |
| 1   | PRIMARY     | <derived2>    |            | ref    | <auto_key0>               | <auto_key0>               | 9       | tierlist.t.id           | 10     | 100.0    | Using index                           |
| 1   | PRIMARY     | v             |            | const  | email                     | email                     | 1022    | const                   | 1      | 100.0    | Using where; Using index              |
| 2   | DERIVED     | tierlist_like |            | range  | idx_tierlist_id_member_id | idx_tierlist_id_member_id | 18      |                         | 100057 | 100.0    | Using where; Using index for group-by |
| 3   | SUBQUERY    | member        |            | const  | email                     | email                     | 1022    | const                   | 1      | 100.0    | Using index                           |

위는 `EXPLAIN`을 통한 실행 계획 조회 결과이다. 하나하나 살펴보며 어떤 부분이 개선되었고 어떤 부분을 더 개선할 수 있는지 생각해보자.

첫 째 행에서 `tierlist` 테이블에 대한 풀 테이블 스캔이 발생하고 있다. 약 1,000,000개의 행을 검사할 것으로 예상된다. WHERE 조건과 정렬에서 Using where; Using filesort이 발생한다.

`is_published`, `like_count`, `created_at`에 대한 복합 인덱스 추가를 고려 할 수 있지만, 해당 컬럼들은 `created_at`을 제외하고 자주 변경이 발생하기 때문에 복합 인덱스보다는 `created_at` 컬럼에만 인덱스를 걸어 더 개선할 수 있을 것으로 보인다.

두 번째 행과 세 번째, 네 번째 행은 `PRIMARY` 즉, 기본 키를 이용해 효율적으로 조인이 이루어 지고 있다. 각 조인마다 1개의 행만 조회하므로 효율적이다.

5번째 행은 서브 쿼리결과에 대한 참조가 이루어진다. 자동 생성된 키를 사용하고 있다. 커버링 인덱스를 사용하고 있어 매우 효율적으로 작업이 이루어진다.

여섯 번째 행은 `CONST`를 통해 상수 조회가 이뤄지고 있다. `email` 컬럼에 유니크 제약조건이 걸려있기 때문에 `email` 컬럼의 인덱스를 사용하고 있어 효율적이다. 또한 1개의 행만조회하게 된다.

일곱 번째 행은 RANGE 스캔을 하고 있다. 복합 인덱스를 추가한 덕에 효율적으로 탐색이 가능하다. 1,000,000개의 `row`에서 약 100,000개의 `row`로 크게 행의 수가 줄었다. 또한 `WHERE`와 `GROUP BY`가 인덱스를 통해 이루어져 효율적이다.

여덟 번째 행은 서브쿼리 내의 서브쿼리를 나타낸다. CONST 조회가 이루어지고 `email`인덱스를 사용해 효율적으로 작동한다.


### 실행 계획 분석 (`EXPLAIN ANALYZE` )

`EXPLAIN ANALYZE`를 통해 실행 계획 주요 부분을 자세히 분석해보자.

```
-> Limit: 20 row(s)  (cost=1.99e+9 rows=20) (actual time=201..201 rows=20 loops=1)
    -> Nested loop left join  (cost=1.99e+9 rows=9.96e+9) (actual time=201..201 rows=20 loops=1)
        -> Nested loop left join  (cost=996e+6 rows=9.96e+9) (actual time=201..201 rows=20 loops=1)
            -> Nested loop inner join  (cost=47412 rows=99495) (actual time=73.7..73.7 rows=20 loops=1)
                -> Nested loop inner join  (cost=34976 rows=99495) (actual time=73.6..73.7 rows=20 loops=1)
                    -> Nested loop inner join  (cost=22539 rows=99495) (actual time=73.6..73.7 rows=20 loops=1)
```

전체 쿼리 실행 시간이 201ms로 이전의 2.6초에서 크게 단축되었다. 중첩 루프 조인이 사용되었지만, 실제로는 20개의 행만 처리하기 때문에 빠르게 작동된다.

```
-> Sort: t.like_count DESC, t.created_at DESC  (cost=10102 rows=99495) (actual time=73.6..73.6 rows=20 loops=1)
    -> Filter: ((t.is_published = true) and (t.topic_id is not null) and (t.member_id is not null))  (cost=10102 rows=99495) (actual time=0.0342..32.3 rows=100000 loops=1)
```

정렬이 20개의 행에 대해서만 수행되기 때문에 매우 효율적이다. 또한 필터링 작업 개선 전위 쿼리와 달리 예상대로 수행되었다.

```
	-> Table scan on t  (cost=10102 rows=99495) (actual time=0.0295..25.8 rows=100000 loops=1)
-> Filter: (tp.category_id is not null)  (cost=0.25 rows=1) (actual time=0.00324..0.0033 rows=1 loops=20)
```

`tierlist` 테이블에 대한 풀 테이블 스캔이 여전히 발생하고 있다. 인덱스 추가를 고려할 수 있지만 `is_published`, `like_count`는 자주 변경되는 컬럼이므로 `created_at`에만 인덱스 추가를 고려해보는 것이 좋겠다.

```
-> Single-row index lookup on tp using PRIMARY (id=t.topic_id)  (cost=0.25 rows=1) (actual time=0.00304..0.00305 rows=1 loops=20)
	-> Single-row index lookup on c using PRIMARY (id=tp.category_id)  (cost=0.25 rows=1) (actual time=952e-6..975e-6 rows=1 loops=20)
		-> Single-row index lookup on w using PRIMARY (id=t.member_id)  (cost=0.25 rows=1) (actual time=0.00191..0.00195 rows=1 loops=20)
```

`category`, `topic`, `member` 테이블의 조회가 인덱스를 통해 효율적으로 이루어 졌다.

```
-> Covering index lookup on tl using <auto_key0> (tierlist_id=t.id)  (cost=48180..48182 rows=10) (actual time=6.37..6.37 rows=0 loops=20)
```

서브쿼리와 결과와의 조인 시 `tierlist_like`의 커버링 인덱스를 통해 효율적으로 수행되었다.

```
-> Materialize  (cost=48179..48179 rows=100057) (actual time=127..127 rows=101 loops=1)
	-> Filter: (tierlist_like.member_id = (select #3))  (cost=38174 rows=100057) (actual time=0.0281..127 rows=101 loops=1)
		-> Covering index skip scan for deduplication on tierlist_like using idx_tierlist_id_member_id  (cost=38174 rows=100057) (actual time=0.0244..127 rows=101 loops=1)
		-> Select #3 (subquery in condition; run only once)
			-> Limit: 1 row(s)  (cost=0..0 rows=1) (actual time=708e-6..750e-6 rows=1 loops=1)
				-> Rows fetched before execution  (cost=0..0 rows=1) (actual time=42e-6..42e-6 rows=1 loops=1)
-> Filter: (tl.tierlist_id is not null)  (cost=3.52e-6..3.52e-6 rows=1) (actual time=0.00104..0.00104 rows=0 loops=20)

```

서브쿼리의 실행계획이다. 서브쿼리 결과를 물리화 하는 `Materialize`가 효율적으로 한번 수행되었다. 커버링 인덱스를 사용해 서브쿼리가 실행되고 실제로 `DISTINCT`를 사용하였지만 내부적으로 `cost`가 낮은 `GROUP BY`를 통해 내부적으로 실행됨을 알 수 있다.

```
-> Constant row from v  (cost=3.52e-6..3.52e-6 rows=1) (actual time=804e-6..819e-6 rows=1 loops=20)
```

특정 이메일을 가진 사용자를 찾는 과정이 상수 시간에 처리되었다.

이전 쿼리에 비해 큰 성능 향상이 있었다. 세부 내용은 다음과 같다.

1. 정렬 작업이 20개의 행에 대해서만 수행되게 되었다.
2. 대부분의 테이블 조회가 인덱스를 사용하여 효율적으로 수행되었다.
3. 서브쿼리와 GROUP BY를 사용하여 중복 제거 작업이 효율적으로 수행되었다.
4. `tierlist` 테이블에 복합 인덱스, 또는 `created_at`에 인덱스를 추가하면 추가적인 성능 개선을 기대할 수 있다.

