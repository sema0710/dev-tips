# Spring Boot Test + JUnit5 에서 테스트 초기 데이터 로딩

테스트 클래스 당 1번만 데이터를 로딩해서 테스트 내내 사용하고 싶다면 어떻게 해야할까?  
쉽게 말하면 테스트 용 데이터를 어떻게 만들고 읽어서 사용할 수 있을까?

일단 Spring Boot Test 시 데이터를 로딩하는 방법은 [직접 코드를 작성하는 방법](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/testing.html#testcontext-executing-sql-programmatically)과 [`@Sql`을 사용하는 방법](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/testing.html#testcontext-executing-sql-declaratively) 2가지가 있다.

결론부터 말하면 **테스트 클래스 당 1번만 데이터를 로딩해서 계속 사용하려면 `@Sql`로는 안되고 직접 코드를 작성해야 한다.**

```java
@SpringBootTest
@Transactional
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  //  (3)
class ProductReviewControllerTest {

    ...
    
    @Autowired
    private DataSource dataSource;


    @BeforeAll
    public void beforeAll() throws Exception {  // (2)
        System.out.println("BeforeAll");
        try (Connection conn = dataSource.getConnection()) {  // (2)
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/data/insert-customer-seller-product.sql"));  // (1)
        }
    }

    @AfterAll
    public void afterAll() throws Exception {
        System.out.println("AfterAll");
        try (Connection conn = dataSource.getConnection()) {
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/data/truncate-customer-seller-product.sql"));
        }
    }
    
    ...
}
```

코드 작성 시 특이한 놈은 3가지다.

(1)은 실제 SQL 스크립트 파일을 실행해주는 유틸인 것 외에는 크게 특이하지는 않다.  

재미있는 건 두 군데에 표시한 (2)다.  
일반적으로 `@BeforeAll`은 static 메서드로 사용된다. 그래서 `@Autowired`로 주입받은 변수를 참조해서 사용할 수 없는데, 여기에서는 `@BeforeAll`을 붙인 메서드가 static 도 아니고 그래서 주입받은 `dataSource`를 이용해서 DB connection 을 얻을 수 있다.  

어떻게 이게 가능했을까?  
해답은 (3)으로 표시한 `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`에 있다.

[Baeldung 문서](https://www.baeldung.com/junit-testinstance-annotation)에 정말 잘 설명돼있는데,  
짧게 얘기하면,
- 테스트 클래스의 인스턴스는 원래는 테스트 메서드가 수행될 때마다 계속 새로 생성되지만,
- `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`를 붙여주면 계속 새로 생성하지 않고 하나만 생성해서 테스트 클래스 내 테스트 메서드가 수행되는 동안 계속 사용할 수 있다.

테스트는 모두 독립적이어야 하므로 이렇게 테스트 클래스 인스턴스 하나를 계속 유지해서 테스트 메서드 사이에 상태를 공유하는 건 원칙적으로는 안티패턴일 수도 있는데,  
인스턴스 하나만 사용하면 여러 번 생성해서 사용하는 것보다 경제적인 것만은 확실히다. 그러니 도를 넘지 않는 수준으로만 상태를 공유 한다면 `@TestInstance(TestInstance.Lifecycle.PER_CLASS)`를 사용하는 것도 괜찮다고 본다.

그럼 도를 넘지 않는 수준의 경제적인 상태 공유란 뭘까?

- 테스트 데이터 구성
- 대용량 파일 로딩
- 기타 자원 로딩

위와 같이 어떤 한 인스턴스 변수를 테스트 사이에 공유하는 게 아니라 비용이 많이 드는 자원을 로딩하는 수준이라면 도를 넘지 않는 수준의 경제적인 상태 공유라고 할 수 있다고 본다.


아래 내용은 될 것 같지만 안 된다.

```java
@SpringBootTest
@Transactional  // 테스트 메서드 종료시마다 롤백
class ProductReviewControllerTest {

    private MockMvc mvc;

    @Autowired
    private WebApplicationContext ctx;

    private JacksonTester<ProductReviewIn> productReviewInTester;

    // 모든 테스트 시작 전 한 번 실행되는 @BeforeAll 메서드에 @Sql 을 추가해서 초기 데이터 로딩
    @Sql({"/data/insert-customer-seller-product.sql"})
    @BeforeAll
    public static void beforeAll() {
    }

    // 모든 테스트 종료 시 한 번 실행되는 @AfterAll 메서드에 @Sql 을 추가해서 초기 데이터 제거
    @Sql({"/data/truncate-customer-seller-product.sql"})
    @AfterAll
    public static void afterAll() {
    }

    @BeforeEach
    public void beforeEach() {
        mvc = MockMvcBuilders.webAppContextSetup(ctx)
                .addFilters(new CharacterEncodingFilter("UTF-8", true))
                .alwaysDo(print())
                .build();
        JacksonTester.initFields(this, new ObjectMapper());
    }

    @ParameterizedTest(name = "상품 {0} 에 대한 고객 {1} 의 리뷰 {2} 생성")
    @MethodSource("productReviews")
    public void create(Long productId, Long customerId, String comment) throws Exception {
        postNewProductReview(productId, customerId, comment)
                .andExpect(status().isOk())
                .andExpect(jsonPath("productId").value(productId))
                .andExpect(jsonPath("productName").exists())
                .andExpect(jsonPath("customerId").value(customerId))
                .andExpect(jsonPath("customerName").exists())
                .andExpect(jsonPath("comment").value(comment))
        ;
    }

    private static Stream<Arguments> productReviews() {
        return Stream.of(
                Arguments.of(1L, 1L, "1 신박한 상품이네여~"),
                Arguments.of(2L, 1L, "1 이거 사려고 20년을 기다렸습니다."),
                Arguments.of(3L, 1L, "1 이 가격 말이 되나요?"),
                Arguments.of(1L, 2L, "2 보기만 해도 가슴이 벅차오릅니다."),
                Arguments.of(2L, 2L, "2 사진과 너무 다릅니다.")
        );
    }

    private ResultActions postNewProductReview(Long productId, Long customerId, String comment) throws Exception {
        return mvc.perform(post("/v1/product-reviews")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .content(productReviewInTester.write(new ProductReviewIn(productId, customerId, comment)).getJson()));
    }
}

```

H2 용 truncate

```sql
SET REFERENTIAL_INTEGRITY FALSE;

truncate table seller;
truncate table customer;
truncate table product;

SET REFERENTIAL_INTEGRITY FALSE;

```