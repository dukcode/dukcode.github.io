---
layout: single
title: "[티어리스트] 쿼리 개선기 - (1) 쿼리 최적화 준비하기"
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

# 쿼리 최적화 준비하기

현재 티어리스트 프로젝트를 진행하며 대용량의 데이터를 고려하고 쿼리를 짜지 않고 동작이 되느냐 안되느냐에 치중해서 쿼리들을 짜왔다.

대용량 데이터를 삽입했을 시에 쿼리가 어떻게 느려지는지 검사하고 이를 개선해 볼 예정이다.


## DDL 소개

가장 먼저 티어리스트 프로젝트의 DDL을 살펴보자. 10개의 테이블이 존재한다.

```sql
CREATE TABLE member
(
    id            BIGINT AUTO_INCREMENT,
    email         VARCHAR(255) NOT NULL UNIQUE,
    nickname      VARCHAR(255) NOT NULL UNIQUE,
    password      VARCHAR(255) NOT NULL,
    profile_image VARCHAR(255),
    PRIMARY KEY (id)
);

CREATE TABLE category
(
    id             BIGINT AUTO_INCREMENT,
    name           VARCHAR(255),
    favorite_count INT DEFAULT 0 NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE category_favorite
(
    id          BIGINT AUTO_INCREMENT,
    category_id BIGINT,
    member_id   BIGINT,
    PRIMARY KEY (id)
);

CREATE TABLE item
(
    id          BIGINT AUTO_INCREMENT,
    name        VARCHAR(255),
    category_id BIGINT,
    PRIMARY KEY (id)
);

CREATE TABLE item_rank
(
    id          BIGINT AUTO_INCREMENT,
    `rank`      ENUM ('s','a','b','c','d','f','none'),
    image       VARCHAR(255),
    order_idx   INT NOT NULL,
    item_id     BIGINT,
    tierlist_id BIGINT,
    PRIMARY KEY (id)
);

CREATE TABLE tierlist
(
    id              BIGINT AUTO_INCREMENT,
    title           VARCHAR(255) NOT NULL,
    thumbnail_image VARCHAR(255),
    content         VARCHAR(255),
    is_published    BOOLEAN      NOT NULL,
    comment_count   INT          NOT NULL,
    like_count      INT          NOT NULL,
    member_id       BIGINT,
    topic_id        BIGINT,
    modified_at     TIMESTAMP(6),
    created_at      TIMESTAMP(6),
    PRIMARY KEY (id)
);

CREATE TABLE tierlist_comment
(
    created_at        TIMESTAMP(6),
    id                BIGINT AUTO_INCREMENT,
    modified_at       TIMESTAMP(6),
    parent_comment_id BIGINT,
    root_id           BIGINT,
    tierlist_id       BIGINT,
    writer_id         BIGINT,
    content           VARCHAR(255),
    PRIMARY KEY (id)
);

CREATE TABLE tierlist_like
(
    id          BIGINT AUTO_INCREMENT,
    member_id   BIGINT,
    tierlist_id BIGINT,
    PRIMARY KEY (id)
);

CREATE TABLE topic
(
    favorite_count INT DEFAULT 0 NOT NULL,
    category_id    BIGINT,
    id             BIGINT AUTO_INCREMENT,
    name           VARCHAR(255),
    PRIMARY KEY (id)
);

CREATE TABLE topic_favorite
(
    id        BIGINT AUTO_INCREMENT,
    member_id BIGINT,
    topic_id  BIGINT,
    PRIMARY KEY (id)
);
```

## 더미 데이터 삽입

이제 더미 데이터를 삽입해 보자. 개발 서버가 온프레미스로 되어있기 때문에 용량 문제가 발생할 수 있어 보수적으로 `row` 수를 잡았다.

삽입할 정보량은 다음과 같다. `comment`와 `item`, `item_rank`는 제외했다.

- member : 100,000 row

- category: 1,00 row
- category_favorite: 약 100,000 row (category 당 평균 1,000 row)

- topic: 2,000 row (category 당 평균 20 row)
- topic_favorite: 약 100,000 row (topic 당 평균 50 row)

- tierlist: 100,000 row (topic 당 평균 50 row)
- tierlist_favorite: 1,000,000 row (tierlist 당 평균 10 row)


### 더미 데이터 삽입 전 준비

명령어는 SQL 세션에서 Common Table Expressions 의 최대 재귀 깊이를 설정한다. SQL에서는 CTE를 사용하여 재귀 쿼리를 작성할 수 있습니다. 대부분의 데이터베이스 시스템에서는 재귀 쿼리의 깊이에 제한을 두어 무한 루프를 방지하는데, 이의 제한을 올려준다.

```sql
SET SESSION cte_max_recursion_depth = 100000000; 
```

### member 더미 데이터 삽입

10000명의 사용자 데이터를 삽입한다.

```sql
INSERT INTO member (email, nickname, password)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 10000
)
SELECT 
    CONCAT('M', LPAD(n, 5, '0'), '@email.com') as email,
    CONCAT('NICK', LPAD(n, 5, '0')) as nickname,
    '{noop}!@123123' as nickname
FROM cte;
```

### category 더미 데이터 삽입

100개의 카테고리 데이터를 삽입한다. 카테고리가 가장 상위의 항목이므로 적게 설정했다.

```sql
INSERT INTO category (name)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 100
)
SELECT 
    CONCAT('카테고리', LPAD(n, 3, '0')) as name
FROM cte;
```

### category_favorite 더미 데이터 삽입

티어리스트에서는 즐겨찾는 카테고리를 저장할 수 있는 기능을 제공한다. 100,000개의 카테고리 즐겨찾기를 생성한다. 사용자가 10,000명 이므로 한 사용자 당 평균 10개의 즐겨찾기를 가지게 된다.

지금 테이블에 어떤 인덱스도 걸려있지 않다. 따라서 `member_id`와 `category_id`에 유니크 제약조건을 걸어 데이터 정합성이 깨지지 않게 한 후, 더미 데이터를 삽입하고 이를 다시 풀어주었다.

또한 카테고리에 `favorite_count`를 추합해 업데이트 해주었다.

```sql
ALTER TABLE category_favorite
ADD CONSTRAINT unique_member_id_category_id UNIQUE (member_id, category_id);

INSERT IGNORE INTO category_favorite (member_id, category_id)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 100000
)
SELECT
    FLOOR(RAND() * 10000 + 1) as member_id,
    FLOOR(RAND() * 100 + 1) as category_id
FROM cte;


UPDATE category c
JOIN (
    SELECT category_id, COUNT(*) AS count
    FROM category_favorite
    GROUP BY category_id
) cf ON c.id = cf.category_id
SET c.favorite_count = cf.count;

ALTER TABLE category_favorite DROP INDEX unique_member_id_category_id;
```

![](https://i.imgur.com/u2GXlQU.png)

위처럼 정합성이 깨지는 데이터들은 무시되고 약 95,000개의 더미 데이터가 생성 되었다.
### topic 더미 데이터 삽입

카테고리 1개 당 평균 토픽이 20개가 되도록, 2000개의 토픽을 만들었다.

```sql
-- 카테고리 1개당 토픽 평균 20개
INSERT IGNORE INTO topic (category_id, name, favorite_count)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 2000
)
SELECT 
    FLOOR(RAND() * 100 + 1),
    CONCAT('토픽', LPAD(n, 4, '0')) as name,
    0
FROM cte;

```

아래의 코드로 카테고리 1개 당 평균 토픽 갯수를 조회할 수 있다.

```sql
-- 카테고리 1개 당 평균 토픽 갯수 조회
SELECT AVG(cnt) FROM (SELECT COUNT(id) AS cnt FROM topic GROUP BY category_id) AS subquery;
```

아래와 같이 20개의 평균이 나오는 것을 확인할 수 있다.

![](https://i.imgur.com/xfwfkho.png)


### topic_favorite 더미 데이터 삽입

`category_favorite`과 같이 100,000개의 즐겨찾기를 추가 했다. 테이블의 제약조건을 건들지 않았기 때문에 임시로 제약조건을 추가하고 더미 데이터를 추가한다. 데이터 추가가 완료되면 제약조건을 해제한다.

또한 토픽에 `favorite_count`를 업데이트 해주었다.

```sql
ALTER TABLE topic_favorite
ADD CONSTRAINT unique_member_id_topic_id UNIQUE (member_id, topic_id);

INSERT IGNORE INTO topic_favorite (member_id, topic_id)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 100000
)
SELECT
    FLOOR(RAND() * 10000 + 1) as member_id,
    FLOOR(RAND() * 2000 + 1) as topic_id
FROM cte;


UPDATE topic t
JOIN (
    SELECT topic_id, COUNT(*) AS count
    FROM topic_favorite 
    GROUP BY topic_id
) tf ON t.id = tf.topic_id
SET t.favorite_count = tf.count;

ALTER TABLE topic_favorite DROP INDEX unique_member_id_topic_id;
```

정합성이 깨지는 데이터는 무시되었기 때문에 실제로는 100,000개의 데이터가 모두 들어가지는 않았다.

![](https://i.imgur.com/P8vTA0U.png)

### tierlist 더미 데이터 삽입

토픽 당 평균 티어리스트가 50개가 되도록 100,000개의 티어리스트 데이터를 추가했다.

```sql
INSERT IGNORE INTO tierlist (title, content, is_published, comment_count, like_count, member_id, topic_id, created_at)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 100000
)
SELECT
    CONCAT('티어리스트', LPAD(n, 6, '0')),
    '콘텐츠',
    true, 0, 0,
    FLOOR(RAND() * 10000 + 1),
    FLOOR(RAND() * 2000 + 1),
    TIMESTAMP(DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 3650) DAY) + INTERVAL FLOOR(RAND() * 86400) SECOND) AS created_at
FROM cte;
```

마찬가지로 아래의 쿼리로 평균 갯수를 확인해 볼 수 있다.

```sql
SELECT AVG(cnt) FROM (SELECT COUNT(id) AS cnt FROM tierlist GROUP BY topic_id) AS subquery;
```

토픽당 평균 50개의 티어리스트가 생성된다.

![](https://i.imgur.com/NgizvX9.png)

### tierlist_like 더미데이터 삽입

1,000,000개의 티어리스트 좋아요 데이터를 삽입한다. 마찬가지로 어떠한 제약조건도 걸려있지 않기 때문에 제약조건을 추가한 후 더미 데이터를 삽입하고 제거한다.

```sql
ALTER TABLE tierlist_like 
ADD CONSTRAINT unique_member_id_tierlist_id UNIQUE (member_id, tierlist_id);

INSERT IGNORE INTO tierlist_like (member_id, tierlist_id)
WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 1000000
)
SELECT
    FLOOR(RAND() * 10000 + 1) as member_id,
    FLOOR(RAND() * 100000 + 1) as tierlist_id
FROM cte;


UPDATE tierlist t
JOIN (
    SELECT tierlist_id, COUNT(*) AS count
    FROM tierlist_like 
    GROUP BY tierlist_id
) tl ON t.id = tl.tierlist_id
SET t.like_count = tl.count;

ALTER TABLE tierlist_like DROP INDEX unique_member_id_tierlist_id;
```

아래의 쿼리로 갯수를 조회할 수 있다.

```sql
SELECT COUNT(*) FROM tierlist_like;
```

조회해 보면 1,000,000개에 못미치는 데이터가 들어간 것을 확인할 수 있다.

![](https://i.imgur.com/znpxyUN.png)

## Target API 선정

문제가 되는 API를 선정해보자. 일반적으로 부하테스트 등을 걸어보고 문제가 되는 API를 선정하지만, 티어리스트 개발 서버에 접속해 본 결과 바로 어떤 API에 문제가 발생되었는지 확인할 수 있었다.


![](https://i.imgur.com/5zcn1iA.png)

주목할만한 티어리스트가 로딩되는데 아주 오랜 시간이 걸렸다.

![](https://i.imgur.com/S5USNeu.png)

개발자 도구에서 이를 확인해보니 무려 4000ms가 넘는 시간이 걸렸다. 성능 테스트로 문제 API를 알아볼 필요도 없이 아주 문제가 되는 응답시간이다.

티어리스트 목록 조회 API의 문제이다. 이제 nGrinder를 통해 해당 API에 부하테스트를 걸고 성능을 측정해보자.

