---
layout: single
title: "[티어리스트] 쿼리 개선기 - (3) 쿼리 실행 계획 분석 및 응답 시간 측정"
categories: cicd
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

# 쿼리 실행 계획 분석 및 응답 시간 측정

## 실제 쿼리

다음은 `Repository`코드로 인해 생성되는 실제 쿼리의 예시이다.

```sql
SELECT 
    t.id as id,
    t.title as title,
    t.thumbnail_image as thumbnailImage,
    t.created_at as createdAt,
    t.like_count as likeCount,
    t.comment_count as commentCount,
    CASE 
        WHEN v.email IS NOT NULL THEN 1 ELSE 0 
    END AS liked,
    t.is_published as isPublished,
    w.id as writerId,
    w.nickname as writerNickname,
    w.profile_image as writerProfileImage,
    tp.id as topicId,
    tp.name as topicName,
    c.id as categoryId,
    c.name as categoryName
FROM 
    tierlist t
INNER JOIN 
    topic tp ON t.topic_id = tp.id
INNER JOIN 
    category c ON tp.category_id = c.id
INNER JOIN 
    member w ON t.member_id = w.id
LEFT JOIN 
    tierlist_like tl ON tl.tierlist_id = t.id
LEFT JOIN 
    member v ON v.email = 'M00001@email.com' AND v.id = tl.tierlist_id
WHERE 
    t.is_published = true
GROUP BY 
    t.id
ORDER BY 
    t.like_count DESC, t.created_at DESC
LIMIT 20 OFFSET 0;

```

쿼리를 살펴보면 `JOIN`되는 테이블의 순서는 다음과 같다.

- `tierlist`
	- `topic`
		- `category`
	- `member` (작성자)
	- `tierlist_like`
		- `member` (조회 사용자)

위와 같은 테이블을 조인해서 조회하고 이를 `GROUP BY`로 묶고 `ORDER BY`로 정렬한다. `tierlist_like`의 `row`수가 1,000,000개에 가깝기 때문에 큰 연산이 발생할 것이 당연하다. 데이터가 많아지면 많아질수록 더욱 더 느려질 쿼리일 것이 분명하다.

## 쿼리 성능 측정

DBeaver를 통해 해당 쿼리를 실행 시키고 실행 시간을 살펴보았다.

![](https://i.imgur.com/zRRc646.png)

평균 2000ms 정도의 소요시간을 가지고 있다. 이로써 프로젝트의 코드보다 쿼리의 성능 개선이 필요함을 확인할 수 있다.

## 쿼리 분석

이제 실행 계획 분석을 통해 이 쿼리가 늦는 실질적인 이유를 찾아보자.

### 실행 계획 분석 (`EXPLAIN`)

`EXPLAIN`을 해당 쿼리의 실행 계획을 조회해보자. 데이터베이스가 쿼리를 실행하기 위해 어떤 경로를 선택할지를 보여준다.

![](https://i.imgur.com/fyHpvPT.png)

![](https://i.imgur.com/wXkA6sg.png)


전체 결과는 다음과 같다. 한줄 한줄 살펴보자.

| id  | select_type | table | partitions | type   | possible_keys | key     | key_len | ref                     | rows   | filtered | Extra                                        |
| --- | ----------- | ----- | ---------- | ------ | ------------- | ------- | ------- | ----------------------- | ------ | -------- | -------------------------------------------- |
| 1   | SIMPLE      | t     |            | ALL    | PRIMARY       |         |         |                         | 99495  | 10.0     | Using where; Using temporary; Using filesort |
| 1   | SIMPLE      | tp    |            | eq_ref | PRIMARY       | PRIMARY | 8       | tierlist.t.topic_id     | 1      | 100.0    | Using where                                  |
| 1   | SIMPLE      | c     |            | eq_ref | PRIMARY       | PRIMARY | 8       | tierlist.tp.category_id | 1      | 100.0    |                                              |
| 1   | SIMPLE      | w     |            | eq_ref | PRIMARY       | PRIMARY | 8       | tierlist.t.member_id    | 1      | 100.0    |                                              |
| 1   | SIMPLE      | tl    |            | ALL    |               |         |         |                         | 996744 | 100.0    | Using where; Using join buffer (hash join)   |
| 1   | SIMPLE      | v     |            | eq_ref | PRIMARY,email | PRIMARY | 8       | tierlist.tl.tierlist_id | 1      | 100.0    | Using where                                  |

1. **`type` `ALL`**:
	- `table` 컬럼에서 `tierlist` 테이블과 `tierlist_like` 테이블의 `type`이 `ALL`로 표시되었다. 이는 풀 테이블 스캔을 의미한다. 이는 쿼리 성능에 큰 영향을 미친다.. `rows` 값이 99,495로 많은 데이터가 풀스캔되고 있어, 성능 저하가 나타난다.
	- `tierlist_like` 테이블에서 `Using join buffer (hash join)`, 즉 해시 조인이 사용되고 있다. 해시 조인은 메모리를 많이 사용하므로 성능 저하의 원인이 된다.
2. **`Using temporary` 및 `Using filesort`**:
    - `Extra` 컬럼에서 `Using temporary`와 `Using filesort`가 나타난다. 이는 정렬 작업을 위해 임시 테이블을 이용한다는 이야기이다. 성능 저하가 크게 발생하는 부분이고 데이터가 많아질 수록 임시 테이블의 크기가 더 커지기 때문에 많은 문제가 될 수 있다.

### 실행 계획 분석 (`EXPLAIN ANALYZE`)

`EXPLAIN ANALYZE`는 `EXPLAIN`보다 더욱 자세한 정보를 제공한다. 쿼리의 실행 계획과 함께 실제 실행 시간, 반복 횟수, 자원 사용량 등의 정보를 포함한다. 이를 통해 쿼리의 성능을 실질적으로 분석해보자.

한줄 한줄 분석해보며 병목의 원인을 상세히 찾아보자.

```
-> Limit: 20 row(s)  (actual time=2649..2649 rows=20 loops=1)
  -> Sort: t.like_count DESC, t.created_at DESC, limit input to 20 row(s) per chunk  (actual time=2649..2649 rows=20 loops=1)
```

전체 쿼리의 실행 시간이 2649ms로 아주 길다. 원인은 정렬 작업이다.

```
-> Table scan on <temporary>  (cost=2.98e+9..3.1e+9 rows=9.92e+9) (actual time=2594..2630 rows=100000 loops=1)
  -> Temporary table with deduplication  (cost=2.98e+9..2.98e+9 rows=9.92e+9) (actual time=2594..2594 rows=100000 loops=1)
```

임시 테이블에 대한 전체 스캔이 발생하고 있다. `cost`가 매우 높다. 그 원인은 `GROUP BY`를 처리하는 비용이 매우 높기 때문이다.중복 제거를 위한 임시 테이블 생성이 많은 시간을 소모하고 있다.

```
-> Nested loop left join  (cost=1.98e+9 rows=9.92e+9) (actual time=321..1237 rows=999500 loops=1)
  -> Left hash join (tl.tierlist_id = t.id)  (cost=992e+6 rows=9.92e+9) (actual time=321..913 rows=999500 loops=1)
```
 
여러 번의 조인이 이루어 지면서 중첩 루프 조인의 비용이 매우 높게 예상되었다.실제 처리하는 `row`의 수가 1,000,000 행에 달한다. 또한 해시 조인의 `cost`가 `992e+6`으로 매우 높고 처리하는 행의 수가 많다.

```
-> Filter: ((t.is_published = true) and (t.topic_id is not null) and (t.member_id is not null))  (cost=10102 rows=9950) (actual time=0.549..55.6 rows=100000 loops=1)
```

더미 데이터를 만들며 모든 티어리스트를 발행 상태로 생성했다. 실제 운영에 들어가더라도 발행 상태인 티어리스트가 대부분일 것이다. 하지만 MySQL은 이를 실제 `row`수 보다 훨씬 적인 9950개의 `row`로 예상하고 있다. 실제 행은 100,000개로 약 10배 많다.

```
-> Table scan on t  (cost=10102 rows=99495) (actual time=0.521..45.3 rows=100000 loops=1)
  -> Hash
    -> Table scan on tl  (cost=27.4 rows=996744) (actual time=0.0963..171 rows=999487 loops=1)
```

`tierlist` 테이블과 `tierlist_like` 테이블에 대한 전체 스캔이 발생하고 있다. 인덱스를 통해 이를 해결할 수 있을 것으로 보인다.

```
-> Filter: (v.email = 'M00001@email.com')  (cost=25.1e-6 rows=1) (actual time=226e-6..226e-6 rows=11e-6 loops=999500)
```

`email`이 특정 사용자의 이메일과 같은 `row`를 필터링 하고 있다. 중요한 점은 `loops`이다. 약 1,000,000회 반복되어 실행되고 있어 매우 비효율적이다.

### 정리

분석을 통해 정리해보면 해당 쿼리는 다음과 같은 문제가 있다.

1. 대규모 정렬 작업
2. 매우 큰 임시 테이블 사용
3. 비효율적인 조인 작업 (특히 `tierlist_like`와 `member` 테이블)
4. 인덱스 미사용으로 인한 풀 테이블 스캔
5. 반복되는 필터링 작업

이를 어떻게 개선할 수 있는지 다음 포스트에서 알아보자.