---
title: "SpringMVC와 SpringWebFlux 중 뭘 선택할 것인가"
datePublished: Mon Oct 02 2023 15:33:44 GMT+0000 (Coordinated Universal Time)
cuid: cln91wcns000408mka46h5c4g
slug: springmvc-springwebflux

---

뭘 선택할 것인가? 어떤 상황에서 어울리고 어떤 차이가 있는지 알아보자.

Spring Boot는 내장형 서버를 기본적으로 포함하여 주로 Tomcat을 내장 서버로 사용한다. 그리고 내장 서버는 어플리케이션을 실행하는 동안 Thread Pool을 관리하고 사용한다. 설정을 어떻게 하냐에 따라 Thread Pool 크기와 동작 방식을 조절할 수 있다.

### 그럼 Thread Pool 크기를 정하는 전략은?

어플리케이션의 작업 종류와 특성에 따라 Thread Pool 크기를 조절해야 한다. CPU-Bound 작업과 I/O-Bound 작업은 다른 Thread pool 크기를 필요로 할 수 있다.

* CPU-Bound 작업에서 적절한 Thread 수는 CPU core 개수 + 1 이다. ([자바 병렬 프로그래밍](https://product.kyobobook.co.kr/detail/S000000935083)) 이유는 Context Switching overhead 비용때문이다. 따라서 CPU-Bound Application은 CPU 자원을 많이 사용하기 때문에 CPU core 개수와 비슷한 수준 이상으로 늘려봤자 별 이점이 없으며, 오히려 각 Core에서 Thread가 많아질 수록 Context Switching 때문에 오버헤드만 더 많아져 성능에 안좋은 영향을 주게 된다.
    
* I/O-Bound 작업은 File I/O, Network I/O, Database 조회와 같은 I/O 작업이 많을 수록 Thread를 늘리는 것이 좋다. 이런 작업들은 I/O 장치의 응답 속도에 의존하는 것이 큰데, 예를 들어 File I/O는 File을 Read하기 위해 Disk에 데이터를 조회하는데 있어 하드웨어의 한계상 I/O 작업에 걸리는 시간이 상재거으로 오래 걸리므로 이 때 CPU는 I/O 작업이 완료될 때까지 idle 상태가 된다. 그래서 Single Thread일 경우 I/O 작업을 진행하는 동안 Blocking 되어 자원 낭비가 발생한다.
    
    하지만 Multi Thread일 경우 I/O 작업이 처리될 동안 다음에 이행할 작업들을 다른 Thread에게 자원을 할당하여 수행하도록 할 수 있다. Network I/O도 마찬가지로 Network 상황에 따라 전송 시간이 오래 걸릴 수 있으므로 Network 지연동안 thread를 재활용 시킬 수 있다. 따라서 I/O 작업이 많은 상황에서는 CPU Core 개수 보다 많게 2배, 3배 혹은 그 이상으로 Thread count를 늘려주는것이 Core들을 더 효율적으로 쓸 수 있기 때문에 성능 면에서 이점이 있다. 그러나 Thread를 늘리면 늘릴수록 잔여 Thread의 Context switching overhead와 Synchronization 등의 문제점이 동반 될 수 있기 때문에, I/O bound Application에 Multi-Thread model 대신 Async I/O 처리에 특화된 Event driven Programming Model(WebFlux)등을 쓸 수 있다.
    
    Event Loop를 통해 I/O Event를 감지하고 Event Handler를 호출하여 해당 Event를 처리하는 식이다. 만약 I/O 작업을 수행해야 하는 경우, Event Handler는 Async I/O를 사용해 I/O 작업을 수행하고 I/O 작업이 완료될 때까지 Event loop를 Blocking하지 않게 된다. 그래서 작업 처리 도중 다른 작업을 처리할 수 있어서 Multi-Thread Model에 비해 더 적은 resource를 사용하고 높은 처리량을 보여줄 수 있다. Main Thread가 Single-Thread이기 때문에 Multi-Thread model에서 발생할 수 있는 Synchronization 문제나 Race condition 등을 걱정할 필요 없이 CPU-bound 혹은 I/O-bound 작업이 발생하면 그때에만 Multi-Thread를 가져와 사용한다.
    

|  | Spring MVC | Spring WebFlux |
| --- | --- | --- |
| 기본 웹 서버 | Tomcat | Netty |
| 쓰레드 수 | 200 | CPU Core count |
| 워킹 스레드 | Multi Thread | Single Thread |
| I/O | Blocking I/O | Non-Blocking I/O |

# 현업에서 활용시 느낀점

# 상황별 성능 비교

SpringMVC와 WebFlux 서버를 구성하고 각각

1. 정적 페이지
    
2. CPU-Bound 작업
    
3. I/O-Bound 작업
    

을 구성하고 테스트 해본다. 추가로 WebFlux는 Reactor과 Kotlin Coroutine으로 각각 구성해보자.

CPU-Bound 작업은 행렬곱을 계한산다음 행렬의 모든 원소를 더하는 로직이고, I/O-Bound 작업은 파일 업로드 로직으로 구성했다.

### SpringMVC

```kotlin
@RestController
@RequestMapping("/")
class Controller {    // 정적 페이지
    @GetMapping("/empty")
    fun empty(): ResponseEntity<String> {
        return ResponseEntity.ok("empty")
    }
}

@RestController
@RequestMapping("/cpu-bound")
class CPUBoundController {    // CPU-Bound 작업
    @GetMapping("/matrix-multiplication")
    fun matrixMultiplication(): ResponseEntity<String> {
        val matrixA = createMatrix(100, 100)
        val matrixB = createMatrix(100, 100)
        val result = matrixMultiplication(matrixA, matrixB).sumOf { row -> row.sum() }.toString()
        return ResponseEntity.ok(result)
    }
}

@RestController
@RequestMapping("/io-bound")
class IOBoundController {    // I/O-Bound 작업
    @PostMapping("/file-upload")
    fun fileUpload(@RequestParam file: MultipartFile): ResponseEntity<String> {
        return if (!file.isEmpty) {
            try {
                val uploadedFileName = file.originalFilename
                ResponseEntity.ok("파일 업로드 성공: $uploadedFileName, 파일 사이즈: ${file.size}")
            } catch (e: Exception) {
                ResponseEntity.badRequest().body("파일 업로드 실패: ${e.message}")
            }
        } else {
            ResponseEntity.badRequest().body("업로드된 파일이 없습니다.")
        }
    }
}
```

행렬곱 코드는 아래와 같다. (WebFlux의 Reactor, Coroutine으로 구성한것도 코드는 같다.)

```kotlin
fun createMatrix(rows: Int, cols: Int): Array<IntArray> {
    val matrix = Array(rows) { IntArray(cols) }
    var value = 1
    for (i in 0 until rows) {
        for (j in 0 until cols) {
            matrix[i][j] = value
            value++
        }
    }
    return matrix
}

fun matrixMultiplication(matrixA: Array<IntArray>, matrixB: Array<IntArray>): Array<IntArray> {
    val numRowsA = matrixA.size
    val numColsA = matrixA[0].size
    val numRowsB = matrixB.size
    val numColsB = matrixB[0].size

    if (numColsA != numRowsB) {
        throw IllegalArgumentException("행렬 A의 열 수와 행렬 B의 행 수가 일치하지 않습니다.")
    }

    val result = Array(numRowsA) { IntArray(numColsB) }

    for (i in 0 until numRowsA) {
        for (j in 0 until numColsB) {
            var sum = 0
            for (k in 0 until numColsA) {
                sum += matrixA[i][k] * matrixB[k][j]
            }
            result[i][j] = sum
        }
    }

    return result
}
```

### Spring WebFlux

#### Reactor(Mono, Flux)

```kotlin
@RestController
@RequestMapping("/")
class Controller {
    @GetMapping("/empty")
    fun empty(): Mono<ResponseEntity<String>> {
        return Mono.just(ResponseEntity.ok("empty"))
    }
}

@RestController
@RequestMapping("/cpu-bound")
class CPUBoundController {
    @GetMapping("/matrix-multiplication")
    fun matrixMultiplication(): Mono<ResponseEntity<String>> {
        return Mono.fromSupplier {
            val matrixA = createMatrix(100, 100)
            val matrixB = createMatrix(100, 100)
            val result = matrixMultiplication(matrixA, matrixB).sumOf { row -> row.sum() }.toString()
            ResponseEntity.ok(result)
        }
    }
}

@RestController
@RequestMapping("/io-bound")
class IOBoundController {
    @PostMapping("/file-upload")
    fun fileUpload(@RequestPart("file") file: FilePart): Mono<ResponseEntity<String>> {
        return file.content()
            .reduce(ByteArray(0)) { acc, dataBuffer ->
                val bytes = ByteArray(dataBuffer.readableByteCount())
                dataBuffer.read(bytes)
                DataBufferUtils.release(dataBuffer)
                acc + bytes
            }
            .map { bytes ->
                val responseMessage = "파일 업로드 성공. 파일 데이터 길이: ${bytes.size} bytes"
                ResponseEntity.ok(responseMessage)
            }
            .switchIfEmpty(Mono.just(ResponseEntity.badRequest().body("업로드된 파일이 없습니다.")))
    }
}
```

#### Coroutine

Reactor로 구성하는 것은 코드가 직관적이지 않으며(평소처럼 절차적으로 코드를 이해하기 난해함), 콜백들로 구성되어있는 로직 때문에 디버깅이 비교적 힘들고 많은 Operator(switchIfEmpty...)등을 학습해야 한다던가, 백프레셔 연산자가 필요하다. 그래서 이를 해결해주는 것이 Coroutine으로 비즈니스 로직을 구성하는 것이다.

아래와 같은 로직은 Reactor과 별 다른 차이가 존재하지 않는데 Mono, Flux처럼 Reactor 구조체로 구성되어있는 로직들을 .awaitSingle 처럼 `org.jetbrains.kotlinx:kotlinx-coroutines-reactor` 에서 제공해주는 변환 메소드들로 Coroutine화 시킬 수 있다.

```kotlin
@RestController
@RequestMapping("/coroutine")
class ControllerByCoroutine {
    @GetMapping("/empty")
    suspend fun empty(): ResponseEntity<String> {
        return ResponseEntity.ok("empty")
    }
}

@RestController
@RequestMapping("/coroutine/cpu-bound")
class CPUBoundControllerByCoroutine {
    @GetMapping("/matrix-multiplication")
    suspend fun matrixMultiplication(): ResponseEntity<String> {
        return Mono.fromSupplier {
            val matrixA = createMatrix(100, 100)
            val matrixB = createMatrix(100, 100)
            val result = matrixMultiplication(matrixA, matrixB).sumOf { row -> row.sum() }.toString()
            ResponseEntity.ok(result)
        }.awaitSingle()
    }
}

@RestController
@RequestMapping("/coroutine/io-bound")
class IOBoundControllerByCoroutine {
    @PostMapping("/file-upload")
    suspend fun fileUpload(@RequestPart("file") file: FilePart): ResponseEntity<String> {
        return file.content()
            .reduce(ByteArray(0)) { acc, dataBuffer ->
                val bytes = ByteArray(dataBuffer.readableByteCount())
                dataBuffer.read(bytes)
                DataBufferUtils.release(dataBuffer)
                acc + bytes
            }
            .map { bytes ->
                val responseMessage = "파일 업로드 성공. 파일 데이터 길이: ${bytes.size} bytes"
                ResponseEntity.ok(responseMessage)
            }
            .switchIfEmpty(Mono.just(ResponseEntity.badRequest().body("업로드된 파일이 없습니다.")))
            .awaitSingle()
    }
}
```

```yaml
version: "3.7"
services:
  perform-spring-mvc:
    image: perform-spring-mvc:latest
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "256M"
    ports:
      - "8080:8080"
  perform-spring-webflux:
    image: perform-spring-webflux:latest
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "256M"
    ports:
      - "8081:8081"
```

# 결론