---
title: "Spring Data JPA에서 @Query + @Modifying을 통한 벌크연산 시 영속성 컨텍스트가 업데이트 되지 않을 때"
seoTitle: "Spring Data JPA에서 @Query + @Modifying을 통한 벌크연산 시 영속성 컨텍스트가 업데이트 되지 않을"
datePublished: Thu Mar 23 2023 14:37:25 GMT+0000 (Coordinated Universal Time)
cuid: clfl7wj2w000409i8d8dr9jyd
slug: spring-data-jpa-query-modifying

---

Spring Data JPA에서 `@Query`와 `@Modifying` 벌크 연산은 영속성 컨텍스트가 업데이트 되지 않는다.

JPA에서 조회할 때 1차 캐시를 확인해 해당 엔티티가 1차 캐시에 존재하면 DB를 통해 데이터를 받아오지 않고 1차 캐시에 있는 엔티티를 반환한다.

하지만 벌크 연산은 1차 캐시를 포함한 영속성 컨텍스트를 무시하고 바로 `@Query`에 명시된 쿼리를 실행하기 때문에 영속성 컨텍스트는 데이터 변경을 알 수 없다.

즉, <mark>벌크 연산 실행시 1차 캐시(영속성 컨텍스트)와 DB 데이터 싱크가 맞지 않게 된다.</mark>

하지만 Spring Data JPA는 `@Modifying`의 `clearAutomatically`라는 유용한 기능을 제공하기 때문에 이를 쉽게 해결할 수 있다.

<mark>clearAutomatically 옵션을 true로 변경해준다면, 벌크 연산 직후 자동으로 영속성 컨텍스트를 clear 해준다.</mark>

```kotlin
@Modifying(clearAutomatically = true)
@Query("update PollUser pu set pu.deletedAt = :now where pu.poll.id = :pollId")
fun softDeleteAllByPollIdQuery(pollId: Long, now: LocalDateTime)
```

그래서 조회를 실행하면 1차 캐시에 해당 엔티티가 존재하지 않기 때문에 DB 조회 쿼리를 실행하게 된다. 이렇게 하면, 데이터 동기화 문제를 해결할 수 있다.

JPA의 1차 캐시는 DB에 접근하는 횟수를 줄여 성능을 개선해주는 좋은 기능이지만, `@Query`와 `@Modifying`을 사용한 벌크 연산에서는 위와같은 이유로 예측하지 못한 결과가 나올 수 있으니 주의해서 사용해야 한다.