---
title: "Spring MVC에서 외부 서비스 호출 병목을 비동기로 개선하기"
seoTitle: "Spring MVC에서 외부 서비스 호출 병목을 비동기로 개선하기"
seoDescription: "@Async, @Retryable, @Recover, ThreadPoolTaskExecutor"
datePublished: Fri Jul 14 2023 10:37:06 GMT+0000 (Coordinated Universal Time)
cuid: clk2g2qfy000j0ajy18v72x1q
slug: spring-mvc
tags: programming-blogs, kotlin, springmvc, threads, programming-tips

---

# 들어가며

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689326380178/bf516b8f-6b1b-4e72-b623-ebcf52b3d389.png align="center")

서비스를 운영하며 알람을 보다 보니 한 서버의 특정 요청 로직에서 다른 서비스를 호출하는 로직에 응답속도 저하가 발생해 해당 특정 로직에도 영향을 받고 있음을 확인했다.

Tracing 된 블록들을 살펴보면 DB의 Connection을 획득하는 시간과 Query 하는시간으로만 보면 10ms 내외지만 다른 서비스를 호출할때 걸리는 시간이 73ms 정도를 차지하고 있었으며, 지연 작업 외 시간들을 다 합쳐도 훨씬 큰 수준이다.

지연되는 로직이 매번 이렇게 시간이 오래 걸리진 않았지만, 간혹가다 발생하는 이런 사례에도 이 API를 사용하는 사용자는 그만큼 응답을 늦게 받게 되어 사용성이 저하가 될 것이고, MVC에서 생성하는 Thread의 양도 한정적이기 때문에 이 API에 요청이 몰리게 되면 빠르게 처리되고 있었던 다른 API도 영향을 받을 것이다.

# 문제 해결 전략

해당 외부 호출의 응답 값을 활용해야 하는 사례가 아니였기 때문에 이 상황에서 세운 문제 해결 전략은 다음과 같다.

#### 응답 지연이 발생하는 로직을 개선하기 위해

ThreadPoolTaskExecutor를 활용해 해당 로직을 비동기로 다른 쓰레드에서 실행하기 위한 Thread Pool, 최대 Thread 수, Thread Pool이 꽉찼다면 잠시 대기시킬(Throttling) 용도의 Queue를 설정하고 이를 Async 어노테이션으로 해당 로직에 붙여 활용하는 것이다.

#### 해당 작업이 실패할 때 재실행하는 걸 보장하기 위해

해당 로직이 어느 정도는 재실행하도록 보장하기 위해서는 재실행할 수 있도록 구성해야 한다. 하지만 이 작업의 개수를 무한정으로 늘려 작업이 실패했을 때도 무한정 재실행하게 한다면 그만큼 해당 작업을 부여받은 Thread가 다른일을 하지 못하고 작업을 계속 진행하고 있기 때문에 한정된 자원에선 이렇게 할 수가 없다. 그래서 어느 정도 타협을 해, 재실행 횟수 3회, 재실행의 텀을 500ms으로 정했다.

#### 지정된 Exception이 발생해 작업을 복원하기 위해

해당 재실행(Retryable) 작업에서 특정 Exception이 발생한다면 특정 로직을 통해 복원할 수 있도록 제공 해주는 Recover 어노테이션을 활용했다. 해당하는 작업이 많이 발생하지 않았고, 실패하거나 특정 Exception이 throw 되어도 문제가 되는 상황이 없었기 때문에 향후 문제 정의를 위해 로그를 남기는 정도만 처리했다.

# 문제 해결 방법

`@Async`는 Spring Framework에 정의가 되어있기 때문에 따로 *dependencies*에 설정하지 않아도 된다.

스프링 프로젝트가 실행될 때 설정할 비동기 처리를 위한 ThreadPoolTaskExecutor의 정보를 Configuration Class를 만들어 정의 해주고 빈으로 등록한다.

```kotlin
@Configuration
@EnableAsync
class AsyncConfig(
    @Value("\${config.async.mail-service.core-pool-size}")
    private val mailServiceCorePoolSize: Int,

    @Value("\${config.async.mail-service.max-pool-size}")
    private val mailServiceMaxPoolSize: Int,

    @Value("\${config.async.mail-service.queue-capacity}")
    private val mailServiceQueueCapacity: Int
) {
    @Bean
    fun mailServiceThreadPoolTaskExecutor(): Executor {
        val taskExecutor = ThreadPoolTaskExecutor()
        taskExecutor.corePoolSize = blockKitCorePoolSize // 기본 스레드 수
        taskExecutor.maxPoolSize = blockKitMaxPoolSize // 최대 스레드 수
        taskExecutor.queueCapacity = blockKitQueueCapacity // Queue 사이즈
        taskExecutor.setThreadNamePrefix("Executor-")
        return taskExecutor
    }
}
```

`@Retryable`과 `@Recover`를 사용하려면 spring-retry dependency가 필요하니 불러와준다.

```kotlin
implementation("org.springframework.retry:spring-retry:1.3.3")
```

```kotlin
@Async("mailServiceThreadPoolTaskExecutor")
@Retryable(maxAttempts = 3, backoff = Backoff(delay = 500))
fun sendMail(...) {
    // ... 쓰레드 풀로 작업을 위임할 로직
}
```

간단하게 위처럼 문제를 해결할 수 있지만 더 상세한 설정 작업이 필요하다면 RetryTemplate, Listeners 등을 활용하는게 좋다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689330208786/2fe43a3f-5fa2-4f5c-ae3a-c887c5cb9c75.png align="center")

해당 로직의 응답 시간이 14ms 수준으로 줄어든 모습이다.

앞으론 ThreadPoolTaskExecutor나 Tomcat에서 생성한 Thread들이 어떻게 얼마나 작업을 처리하고 있는지 주기적으로 모니터링하며 자세한 설정값을 바꿔가면 된다.

# 호출을 무조건 보장해야 한다면

만약 해당 로직을 비동기로 전환해야하고 무조건 실행을 보장해야 한다면?

작업의 동시성이 보장되는 환경에 작업 정보를 넣어두고 다른 실행 가능한 주체(특정 Process, 특정 Thread 등)가 꺼내(혹은 마킹해) 실행하도록 하면 된다.

Redis나 MessageQueue(Kafka 등)서비스 위주로 알아보면 좋을 것 같다.