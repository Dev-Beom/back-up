---
title: "Jpa 대용량 배치작업 개선하기"
datePublished: Sun Jul 16 2023 08:52:13 GMT+0000 (Coordinated Universal Time)
cuid: clk577jny000h09laci4xdezv
slug: jpa
tags: jdbctemplate

---

## 개요

맡고있는 서버 서비스 중 배치 작업을 담당하는 서비스가 있다. 해당 배치작업은 한번 작업시 대략 100만건의 row가 추가되는 작업다.

이 작업은 시간이 오래 걸릴뿐더러 당연히 트랜잭션 내에서 DB Connection을 잡고있어 DB에 부하를 준다.

오래걸리는 작업이 거의 다 끝나가고 있을 때 예기치 못한 오류로 거대 작업의 Transaction이 Rollback 되면 이 오래걸리는 작업을 다시 해야하고(recover), 실시간으로 서비스가 되고있는 환경에서는 아무리 사용자가 없는 시간에 작업한다 한들 정상적으로 서비스 되고있는 환경에 피해를 줄 수 있다.

## 기존의 문제점

이 작업은 기존에 JPA에서 제공해주는 saveAll 추상 메소드를 사용해 100만건의 대량 데이터를 DB에 insert 하고 있다.

```kotlin
batchRepository.saveAll(entity)
```

saveAll은 어떻게 동작할까?

![JpaRepository interface의 saveAll implementation 정보](https://blog.kakaocdn.net/dn/nz0wU/btrPCCJ9j1Z/8jVFuKwj0qkkKimyzzbyE0/img.png align="left")

해당 구현체는 `@Transactional` 로 감싸여있으며 파라미터로 받아온 entities를 for문으로 순회하며 save로 저장을 하고있다.

그럼 save가 어떻게 구현되어있냐면

![JpaRepository interface의 save implementation 정보](https://blog.kakaocdn.net/dn/dWZ7wX/btrPDkvrV1x/esrRg2UkFpjh5HbsnKnlH1/img.png align="left")

해당 구현도 `@Transactional`로 묶여있고 entity 정보를 조회한 후 해당 entity가 새롭게 생성된 entity인지 확인하고, 이에따른 entity manager에서 제공해주는 persist / merge 로직으로 이어진다.

이(persist와 merge 로직으로 분기)는 현 상황과 크게 관련이 없고 결국 entity manager가 영속성컨텍스트를 바탕으로 실제로 어떻게 쿼리가 되냐가 중요하다. (참고로 persist를 한다해도 쓰기지연때문에 쿼리는 전송되지 않고 commit 시점에 발송된다)

한번 query를 로그로 찍어보면 다음과 같은 결과가 나온다.

![](https://blog.kakaocdn.net/dn/LqQbR/btrPCEnJbVx/zt7xTISq7Q2OnaGFV1KIq0/img.png align="left")

결국엔 JPA의 saveAll은 한건마다 insert into ~ values (?, ? ...)를 쿼리하는 것이다.

즉, 하나의 커넥션을 통해 insert into 작업 전 후의 불필요한 작업이 매번 진행되기 때문에 saveAll로 대량의 data를 insert하는 시간이 늦어질 수밖에 없다.

그렇다면 어떤 방법으로 개선해볼 수 있을까?

다음 이야기를 하기 전에 정리할겸 기존 로직으로 인해 야기될 수 있는 문제를 다시 정리해봤다.

* DB Connection을 오래 잡고있어 긴시간동안 발생하는 DB 부하(deadlock으로도 이어질 수 있음)
    
* 긴 시간동안 작업을 했는데 예기치 못한 오류로 rollback이 됐을 때의 재작업 부담
    

## 해결하기 까지의 과정

일단 문제를 점진적으로 개선하기 위해 다양한 아이디어로 접근해봤다.

* saveAll을 사용하는 대신 chunk를 나눠서 구현하자.
    
* chunk로 나눠서 배치작업을 진행하며 회복을 위한 정책으로 Spring Batch 작동 방식에 아이디어를 얻어 중간마다 작업내역을 저장하자.
    
* ThreadPoolTaskExecutor를 활용해 비즈니스 로직을 Async하게 진행되도록 하고 즉시 응답하자.
    
* Async하게 작업을 진행하면 작업 정보(Status)를 Redis로 클러스터링해 작업 정보 API를 제공하자.
    
* 즉시 작업이 필요한 경우가 아니라면 MessageQueue(RabbitMQ, Kafka, Redis 등)에 담아서 스케줄링해 사용자 시간대가 적은 시간에 Queue를 소비하자.
    

등이 나왔다. 테스트 해본 후 성능이 작게나마 상승하고, 기존의 안정성이 동일하게 보장된다면 점진적인 개선 관점으로 도입하는게 맞다고 생각한다. 그래서 위 아이디어를 고려하고 적용한 이후의 결과는 다음과 같다.

비슷하거나 연관된 내용은 한꺼번에 정리했다.

* saveAll을 사용하는 대신 chunk를 나눠서 구현하자.
    
* chunk로 나눠서 배치작업을 진행하며 회복을 위한 정책으로 Spring Batch 작동 방식에 아이디어를 얻어 중간마다 작업내역을 저장하자.
    

chunk를 나눠 작업한다는 것은 매 chunk마다 Connection을 새롭게 잡기 때문에 하나의 Connection을 길게 잡아서 생기는 이슈는 해결되는것 같아보였습니다. 그리고 예외처리를 위한 작업내역 기반 회복 정책을 적용했으니 충분히 괜찮아보였다. 하지만 결과적으로 DB 부하는 있었지만 기존의 부하가 조금이나마 분산되는 효과를 얻을 수 있었다.

앞선 내용을 통해 큰 효과를 얻진 못했지만 batch insert 시에 수반되는 부가적인 작업들이 많이 생겼습니다. 그리고 이를 모니터링해야할 필요도 있었고, API에서 동기적으로 작업을 실행후 응답해주고 있었기 때문에 HTTP time out이 발생하기 일수였다.

이를 해결하기위해 아래처럼 작업했다.

* ThreadPoolTaskExecutor를 활용해 Async하게 작업을 진행하고 즉시 응답하자.
    
* Async하게 작업을 진행하면 작업 정보(상태)를 Redis로 클러스터링해 작업 정보 API를 제공하자.
    

또 

* 즉시 작업이 필요한 경우가 아니라면 MessageQueue(RabbitMQ, Kafka, Redis 등)에 담아서 스케줄링해 사용자 시간대가 적은 시간에 Queue를 소비하자.
    

이런 아이디어까지 나왔다. 하지만 위 과정들을 통해 실제 배치 API를 이용하기 위한 추가적인 작업도 많아졌고 결과적으로 크게 개선이 됐다고 할 수 없는 상황이였다.

Async를 활용한 눈속임으로 Time out만 안나지 Thread와 DB는 뒤에서 고통을 받고 있었고 근본적으로 해결은 되지 않았다.

## 해결

여기서 Bulk insert에 대하여 되짚어볼 필요가 있다.

```kotlin
insert into ~ values (?, ? ...)

insert into ~ values (?, ? ...)

insert into ~ values (?, ? ...)

...
```

작업을  

```kotlin
insert into ~ values (?, ? ...), (?, ? ...), (?, ? ...), (?, ? ...)
```

이렇게 진행하는 것이 쿼리의 오버헤드가 더 적기 때문에 훨씬 효율적이다.

```kotlin
@Repository
class BatchRepositoryImpl(
    private val jdbcTemplate: JdbcTemplate
) : BatchRepository {

    @Transactional
    override fun bulkSave(payload: List<CustomProperty>) {
        batchInsert(1000, payload)
    }

    private fun batchInsert(batchSize: Int, subItems: List<Property>) {
        val query =
            "INSERT INTO properties (속성들) values (?, ? ...)"
        jdbcTemplate.batchUpdate(query, subItems, batchSize, ::setArgument)
    }

    private fun setArgument(ps: PreparedStatement, argument: Property) {
        ps.setDate(1, zdtToDate(1번에 들어갈 속성이자 zdt))
        ps.setString(2, 2번에 들어갈 속성)
        ps.setLong(3, 3번에 들어갈 속성)
        ps.setObject(4, zdtToDate(4번에 들어갈 속성이자 zdt))
        ps.setString(5, argument.5번에 들어갈 속성)
        ps.setLong(6, argument.6번에 들어갈 속성)
        ps.setLong(7, argument.7번에 들어갈 속성)
        ps.setLong(8, argument.8번에 들어갈 속성)
        ps.setString(9, argument.9번에 들어갈 속성)
        ... 
    }

    private fun zdtToDate(arg: ZonedDateTime): Date {
        val localDate = arg.toLocalDate()
        return Date.valueOf(localDate)
    }
}
```

위와같이 JdbcTemplate 에서 제공해주는 baatchUpdate를 통해 inline query를 만들 수 있다.

쿼리를 만들어놓고 여기에 대항되는 argument를 하나씩 set해주는 작업을 바탕으로 정의한 사이즈(위에선 1000) 만큼 하나의 쿼리로 만들어져 발송하게 된다. 이 수치는 테스트를 통해 조절해나가면 된다.

# 마무리

그냥 생각나는 과정들을 의식의 흐름대로 정리했기 때문에 이야기가 산으로가기도 했는데요

양해해주세요 🙄