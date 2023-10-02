---
title: "Page & Slice & No offset 분석 글"
seoTitle: "JPA, QueryDSL, JMeter..."
datePublished: Mon Oct 02 2023 15:08:52 GMT+0000 (Coordinated Universal Time)
cuid: cln910dp3000209mlhpfphofs
slug: page-slice-no-offset

---

# 기본적인 구현 방법

### Page

```kotlin
// DocumentRepository.kt
@Query(value = "select doc from Document doc order by doc.id desc")
fun findAllForPage(pageable: Pageable): Page<Document>
```

Pageable를 파라미터로 받고 `org.springframework.data.domain`의 Page를 Return Type으로 선언해주면 된다.

### Slice

```kotlin
// DocumentRepository.kt
@Query(value = "select doc from Document doc order by doc.id desc")
fun findAllForSlice(pageable: Pageable): Slice<Document>
```

Pageable를 파라미터로 받고 `org.springframework.data.domain`의 Slice를 Return Type으로 선언해주면 된다.

### No offset

```kotlin
@Repository
class DocumentRepositoryWithQueryDSL(
    private val queryFactory: JPAQueryFactory
) : QuerydslRepositorySupport(Document::class.java) {
    fun findAllForNoOffset(id: Long?, size: Long = 20): NoOffsetResponse<Document> {
        val dynamicLtId = BooleanBuilder()
        if (id != null) {
            dynamicLtId.and(document.id.lt(id))
        }
        val queryResult = queryFactory
            .selectFrom(document)
            .where(dynamicLtId)
            .orderBy(document.id.desc())
            .limit(size)
            .fetch()
        val nextId = queryResult.last().id
        return queryResult.toOffsetResponse(nextId)
    }

    fun <T> List<T>.toOffsetResponse(nextId: Long): NoOffsetResponse<T> =
        NoOffsetResponse.of(this, nextId)
}
```

```kotlin
class NoOffsetResponse<T>(
    val content: List<T>,
    val hasNext: Boolean,
    val nextId: Long? = null
) {
    companion object {
        fun <T> of(content: List<T>, nextId: Long): NoOffsetResponse<T> {
            return NoOffsetResponse(content, content.isNotEmpty(), nextId)
        }
    }
}
```

QueryDSL을 통해 동적 쿼리를 만들어 클러스터드 인덱스(Default; Primary Key)를 활용했다. NextId를 통해 다음 페이지를 검색할 수 있게 하기 위해 nextId도 제공하도록 구성한다.

이렇게 QueryDSL로 만들 수 있고, JdbcTemplate으로도 만들 수 있다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696257816830/53069936-c291-4f24-9be3-38c9cfa5a82c.png align="center")

Return Type(여기선 Slice와 Page만)별 관계는 위 다이어그램과 같다.

# Return Type 별 실제 쿼리

#### Page

```sql
/*
    컨텐츠들을 조회하는 쿼리
*/
select
    doc 
from
    Document doc 
order by
    doc.id desc
    select
        document0_.id as id1_0_,
        document0_.content as content2_0_,
        document0_.created_at as created_3_0_,
        document0_.title as title4_0_ 
    from
        document document0_ 
    order by
        document0_.id desc limit ?,
        ?
/*
    아이템 개수를 조회하는 카운팅 쿼리(Total Count가 필요할 때 사용)
*/
select
    count(doc) 
from
    Document doc
    select
        count(document0_.id) as col_0_0_ 
    from
        document document0_
```

컨텐츠 조회 쿼리 + Count 쿼리 발생

#### Slice

```sql
/*
    카운팅 쿼리 사용안하고 컨텐츠 n개 + 1개 조회(즉 limit + 1개 조회)
    + 1개는 다음 컨텐츠 확인용
*/
select
    doc 
from
    Document doc 
order by
    doc.id desc */ select
        document0_.id as id1_0_,
        document0_.content as content2_0_,
        document0_.created_at as created_3_0_,
        document0_.title as title4_0_ 
    from
        document document0_ 
    order by
        document0_.id desc limit ?,
        ?
```

컨텐츠 + 1 조회 쿼리 발생

#### List

```sql
/*
    컨텐츠들을 조회하는 쿼리
*/
select
    doc 
from
    Document doc 
order by
    doc.id desc
    select
        document0_.id as id1_0_,
        document0_.content as content2_0_,
        document0_.created_at as created_3_0_,
        document0_.title as title4_0_ 
    from
        document document0_ 
    order by
        document0_.id desc limit ?,
        ?
```

컨텐츠들만 조회하는 쿼리 실행

#### No offset

```sql
/*
    클러스터드 인덱스인 PK(ID)로 빠르게 검색
*/
select
    document 
from
    Document document 
order by
    document.id desc 
        select
            document0_.id as id1_0_,
            document0_.content as content2_0_,
            document0_.created_at as created_3_0_,
            document0_.title as title4_0_ 
        from
            document document0_ 
        order by
            document0_.id desc limit ?
```

# Pageable 객체가 만들어지기 까지의 과정

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696234291615/0dc29e46-7767-48b2-88e2-67e078d47ef9.png align="center")

Request의 Pageable이 어떻게 만들어질까?

1. 사용자가 웹 브라우저를 통해 HTTP Request를 요청하면 DispatcherServlet이 요청을 처리하고
    
2. 요청에 매핑되는 URI를 HandlerMapping에서 검색
    
3. RequestMapping으로 구현한 API 검색
    
4. RequestMappingHandlerAdapter가 정보를 모두 갖고 있음
    
5. Intercepter들을 처리
    
6. Argument Resolver처리
    
    ...
    

천천히 찾아보자

![fd](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258211241/23de3423-ec57-4fc0-bd6b-cfc05da46897.png align="center")

RequestMappingHandlerAdapter의 AbstractHandlerMethodAdapter를 Override한 hadnlerInternal 메서드의 invokeHandlerMethod를 타고 들어가보자

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258425159/e5b2fde3-d834-4651-8a9c-9e1139f42698.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258492338/5b25331f-1847-4913-9f2f-f9f574777881.png align="center")

존재하는 RequestMappingHandlerAdapter에서 사전에 정의되어있는(!=null) argumentResolvers들을 설정해주는 모습이다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258570023/81c915ec-127a-4330-a3dd-25e9610086b9.png align="center")

HandlerMethodArgumentResolverComposite를 확인해보면 내부적으로 addResolver, AddResolvers를 제공한다.

여기서 argumentResolver의 interface를 확인해보면

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258667703/6b49fa46-c925-44e2-b001-e9b8a8665416.png align="center")

supportsParameter, resolveArgument라는 interface method가 정의되어 있는데 이들은 각각 적혀있는 대로

| supportsParameter: 지정된 메서드 매개 변수가 이 resolver에서 지원되는지 여부

| resolverArgyment: 매서드 매개 변수를 지정된 요청의 parameter 값으로 resolve되도록 하는 것

이다.

그렇담 HandlerMethodArgumentResolver interface를 구현한 PageableHandlerMethodArgumentResolverd에서 자세한 구현사항을 볼 수 있겠다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258874479/145b8c51-f536-45be-a73a-4a7951e3e22c.png align="center")

일단 먼저 확장한 PageableHandlerMethodArgumentResolverSupport 추상 클래스를 확인해보면

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696258947900/5c6daee5-1946-4013-95c3-20ecfdf3d2db.png align="center")

Pageable를 구성하기 위한 기본 상수 값들이 존재한다. 다시 PageableHandlerMethodArgumentResolverSupport로 돌아가면

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696259012758/57457849-7d16-4a52-a8c6-99edb8012882.png align="center")

앞서 설명한 resolveArgument를 볼 수 있는데 Pageable 객체와 이를 위한 속성들도 여기서 구성해준다.

# 성능 분석(TPS)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696235668945/f9a9e817-326d-40a2-a561-12ffe1ff48a2.png align="center")

로컬에서 서버를 띄우고 JMeter에서 테스트했다. 세부적인 테스트 환경은 사용자마다 너무 다를것이기 때문에 대략적인 수치만 확인하면 좋겠다.

Page 대략 TPS 140

Slice 대략 TPS 70

No offset(clustered index; pk) 대략 TPS 40