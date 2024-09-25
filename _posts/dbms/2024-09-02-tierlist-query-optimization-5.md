---
layout: single
title: "[티어리스트] 쿼리 개선기 - (5) 쿼리 개선 적용 및 개선 후 성능 측정"
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

# 쿼리 개선 적용 (QueryDsl From절에 서브쿼리 적용하기)

## 개선된 쿼리

개선된 쿼리를 QueryDSL로 전환하고 프로젝트에 적용해보자. 개선 후의 쿼리는 다음과 같다.

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
        WHERE member_id = (SELECT id FROM member WHERE email = ? LIMIT 1)
    ) tl ON t.id = tl.tierlist_id
    LEFT JOIN member v ON v.email = ? AND tl.tierlist_id IS NOT NULL
WHERE 
    t.is_published = true
ORDER BY 
    t.like_count DESC, t.created_at DESC
LIMIT ? OFFSET ?;

```

## 문제점

QueryDsl에서는 `FROM`절에 서브쿼리를 지원하지 않는다. 이유는 JPA의 JPQL도 `FROM`절에 서브쿼리를 지원하지 않기 때문이다. QueryDsl은 JPQL에 기반을 두고 있기 때문에 역시 지원할 수 없다.

따라서 `FROM`절에 서브쿼리를 적용하려면 다음과 같은 방법을 고려해 볼 수 있다.

- 서브쿼리절을 `JOIN`으로 변경한다.
  - 이 방법을 적용하게되면 개선 전의 쿼리로 돌아가는 것이기 때문에 적용할 수 없다.
- 쿼리를 2번 보내 서브쿼리를 제거해 해결한다.
  - 이 방법을 적용하면 쿼리를 2번 날리기 때문에 성능상의 이점이 떨어진다. 성능을 개선하기 위해 이 작업을 진행하므로 맞지 않아보인다.

가능 한 두 방법으로 아래의 방법이 있다. 두 방법을 알아보고 비교해보자.

- QueryDsl-sql 적용하기
- JPASQLQuery 사용하기
  
## QueryDsl-sql

우리가 QueryDsl이라고 알고있는 라이브러리는 사실 QueryDsl-jpa이다. QueryDsl-jpa는 JPQL로 변환되기 때문에 `FROM`절에 서브쿼리를 지원하지 않는다. JPQL도 `FROM`절에 서브쿼리를 지원하기 않기 때문이다.

하지만 QueryDsl-sql은 Native SQL로 변환된다. 따라서 `FROM`절에 서브쿼리를 사용할 수 있다. 그렇다면 우리는 왜 QueryDsl-sql을 사용하지 않을까? 그 이유는 QueryDsl-jpa처럼 Q클래스를 지원하지 않아서 관리가 까다롭기 때문이다. 엔티티가 변경될 때마다 신경써야 할 점이 많아진다. 유지보수를 위해 다른 방법을 고안해보자.

## JPASQLQuery

JPASQLQuery는 JPQL을 위해서 만들어진 JPA Entity를 NativeSQL처럼 사용 가능하게 하는 기능이다. 이를 위해 QueryDsl-jpa 및 QueryDsl-sql 두 의존성이 모두 필요하다.

### 사용법

`build.gradle`에 다음과 같이 querydsl-sql 의존성을 추가한다.

```groovy
  implementation 'com.querydsl:querydsl-sql:5.1.0'
```

querydsl-jpa도 추가한다. 이는 스프링부트 및 querydsl-jpa의 버전마다 다르니 각자의 버전에 맞춰 의존성 추가와 설정을 한다.

다음과 같이 Bean을 설정한다.

```java
  @Bean
  public JPAQueryFactory jpaQueryFactory() {
    return new JPAQueryFactory(entityManager);
  }

  @Bean
  public SQLTemplates sqlTemplates() {
    return MySQLTemplates.builder().build();
  }

```

## 쿼리 개선 적용하기

이제 메서드 내부에 `JPASQLQuery`를 인스턴스로 만들어 `FROM`절에 서브쿼리를 적용할 수 있다. Q클래스는 그대로 사용하면서 querydsl-jpa와 querydsl-sql이 적절히 섞인 상태로 사용할 수 있게 된다.

메서드를 개선된 쿼리가 작동하도록 변경해보자.

```java
  @Override
  public Page<TierlistResponse> loadTierlists(String viewerEmail, Pageable pageable,
      String query,
      TierlistFilter filter) {
    JPASQLQuery<?> jpaSqlQuery = new JPASQLQuery<>(entityManager, sqlTemplates);

    QMemberJpaEntity viewer = new QMemberJpaEntity("viewer");
    QMemberJpaEntity writer = new QMemberJpaEntity("writer");

    StringPath tierlistLike = stringPath("tl");
    NumberPath<Long> likedTierlistId = numberPath(Long.class, tierlistLike,
        "tierlist_id");

    JPQLQuery<Long> viewerIdSubQuery = JPAExpressions
        .select(memberJpaEntity.id)
        .from(memberJpaEntity)
        .where(memberJpaEntity.email.eq(viewerEmail))
        .limit(1);

    JPQLQuery<Long> likedTierlistSubquery = JPAExpressions
        .select(tierlistLikeJpaEntity.tierlistId)
        .distinct()
        .from(tierlistLikeJpaEntity)
        .where(tierlistLikeJpaEntity.memberId.eq(viewerIdSubQuery));

    List<TierlistResponse> tierlistResponses = jpaSqlQuery
        .select(Projections.constructor(TierlistResponse.class,
            tierlistJpaEntity.id,
            tierlistJpaEntity.title,
            tierlistJpaEntity.thumbnailImage,
            // JpaSqlQuery가 LocalDateTime을 가져오지 않아서 아래와 같이 해결
            Expressions.dateTimePath(Timestamp.class, tierlistJpaEntity, "created_at"),
            tierlistJpaEntity.likeCount,
            tierlistJpaEntity.commentCount,
            new CaseBuilder()
                .when(viewer.email.isNotNull()).then(true).otherwise(false).as("liked"),
            tierlistJpaEntity.isPublished,
            writer.id,
            writer.nickname,
            writer.profileImage,
            topicJpaEntity.id,
            topicJpaEntity.name,
            categoryJpaEntity.id,
            categoryJpaEntity.name
        ))
        .from(tierlistJpaEntity)
        .innerJoin(topicJpaEntity).on(tierlistJpaEntity.topicId.eq(topicJpaEntity.id))
        .innerJoin(categoryJpaEntity).on(topicJpaEntity.categoryId.eq(categoryJpaEntity.id))
        .innerJoin(writer).on(tierlistJpaEntity.memberId.eq(writer.id))
        .leftJoin(likedTierlistSubquery, tierlistLike)
        .on(tierlistJpaEntity.id.eq(likedTierlistId))
        .leftJoin(viewer).on(viewer.email.eq(viewerEmail).and(likedTierlistId.isNotNull()))
        .where(tierlistJpaEntity.isPublished.isTrue(), hasQuery(query))
        .orderBy(orderByFilter(filter))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    long count = jpaQueryFactory
        .selectFrom(tierlistJpaEntity)
        .where(tierlistJpaEntity.isPublished.isTrue(), hasQuery(query))
        .fetch().size();

    return new PageImpl<>(tierlistResponses, pageable, count);
  }

```

위와 같이 변경하면 의도된 쿼리가 나간다.

> `JpaSqlQuery`가  `LocalDateTime`을 가져오지 않고 `java.sql.Timestamps`를 가져온다. jpa는 JSR310을 지원하지만, querydsl-sql은 JSR310을 지원하지 않아서 생기는 문제라고 예상된다 (~~삽질 엄청해서 해결했다...~~). 위와 같이 `Expression.dateTimePath()`를 활용해 해결할 수 있었다.

> 또한 snake case로 변경을 지원하지 않으니 `@Table(name="tierlist_like")`, `@Colume(name = "profile_image")`와 같이 엔티티를 설정해야 한다.

아래는 쿼리가 찍힌 로그이다. 의도된 쿼리가 나가는 것을 확인할 수 있다.

```sql
[Hibernate] 
    select
        tierlistJpaEntity.id,
        tierlistJpaEntity.title,
        tierlistJpaEntity.thumbnail_image,
        tierlistJpaEntity.created_at,
        tierlistJpaEntity.like_count,
        tierlistJpaEntity.comment_count,
        (case 
            when viewer.email is not null 
                then ? 
            else ? 
        end) as liked,
        tierlistJpaEntity.is_published,
        writer.id as col_9,
        writer.nickname,
        writer.profile_image,
        topicJpaEntity.id as col_12,
        topicJpaEntity.name,
        categoryJpaEntity.id as col_14,
        categoryJpaEntity.name as col_15 
    from
        tierlist tierlistJpaEntity 
    inner join
        topic topicJpaEntity 
            on tierlistJpaEntity.topic_id = topicJpaEntity.id 
    inner join
        category categoryJpaEntity 
            on topicJpaEntity.category_id = categoryJpaEntity.id 
    inner join
        member writer 
            on tierlistJpaEntity.member_id = writer.id 
    left join
        (select
            distinct tierlistLikeJpaEntity.tierlist_id 
        from
            tierlist_like tierlistLikeJpaEntity 
        where
            tierlistLikeJpaEntity.member_id = (select
                memberJpaEntity.id 
            from
                member memberJpaEntity 
            where
                memberJpaEntity.email = ? 
            limit
                ?)) as tl 
            on tierlistJpaEntity.id = tl.tierlist_id 
    left join
        member viewer 
            on viewer.email = ? 
            and tl.tierlist_id is not null 
    where
        tierlistJpaEntity.is_published = ? 
    order by
        tierlistJpaEntity.like_count desc,
        tierlistJpaEntity.created_at desc 
    limit
        ? 
    offset
        ?
```

## nGrinder를 이용한 성능 테스트 (개선 후)

먼저 개선 전의 TPS 측정을 살펴보자.

![](https://i.imgur.com/I80ommE.png)

평균 TPS 0.7로 처참한 수준이었다..

이제 개선된 쿼리를 가지고 nGrinder를 통해 성능을 테스트 해보자. 이전 가상사용자가 10명일 때 뻗어버렸던 서버가 테스트를 받아낼 수 있게 되었다.

![](https://i.imgur.com/McssDnK.png)

**평균 TPS가 27.6으로 이전 0.7보다 약 4000%의 성능 개선이 있었다.** 아무래도 온프레미스(m1 mac mini 기본형)위에 vmware를 통해 ubuntu를 돌려 m1의 제한된 성능과 낮은 메모리 용량 때문인지 vuser의 수를 더 이상 늘릴 수 없었다. 일반적인 서버의 TPS를 기대할 수는 없었다.

하지만 쿼리 개선이 운영 서버에 적용된다면 수치적으로도 큰 개선일 것이라고 생각된다.

## 추후 개선점

추후 개선점으로는 다음을 redis 등의 캐시를 생각해 볼 수 있다.

### 캐시 사용

Redis 캐시를 통해 성능 개선을 생각해 볼 수 있을 것 같다. 해당 문제 쿼리였던 핫한 티어리스트 목록 조회는 사용자 입장에서 실시간성이 조금 떨어지더라도 빠른 응답을 받게된다면 페이지 체류 시간이 더욱 길어질 것으로 예상된다.

1시간이나 30분마다 해당 쿼리 조회 결과를 캐시해 DB조회 없이 빠른 응답을 구현할 수 있겠다. 추후 개발을 통해 적용해 볼 예정이다.