- [后端常见测试](#后端常见测试)
  - [repository层测试](#repository层测试)
    - [repository层---存](#repository层---存)
    - [repository层---查](#repository层---查)
  - [service层测试](#service层测试)
    - [不需要依赖repository的service](#不需要依赖repository的service)
    - [依赖repository的service](#依赖repository的service)
  - [controller层测试](#controller层测试)
  - [api测试](#api测试)
  - [测试注解](#测试注解)
  - [静态测试方法](#静态测试方法)
- [常见前端测试](#常见前端测试)
  - [普通UI测试](#普通ui测试)
- [工具网址](#工具网址)

# 后端常见测试
repository层、controller层和api测试，都需要spring框架的支持；而service层是单元测试，不依赖spring框架，但可以启动spring框架，也就是使用集成测试的容器来进行单元测试。单元测试和集成测试的区别在于测试的点而不是启动spring框架与否。

## repository层测试
repository层是屏蔽数据库的，也就是要mock数据库的返回。

### repository层---存
存方法是不需要@Sql方法去mock的。
```java
@Autowired
private OrderRepository orderRepository;

// Given
Order order = new Order("123", "Acg2345", "1234", "1234", "1234", "1234", LocalDateTime.now(), false, "1234", 0);
// When
Order saveMockOrder = orderRepository.save(order);
// Then
assertEquals(order.getId(), saveMockOrder.getId());
```

### repository层---查
这里mock了sql
```java
@Test
@Sql("/sql/insert_0_order.sql")
void shouldFindByOrderId() {
  // Given
  String orderId = "Acg2345";
  // When
  List<Order> mockOrderList = orderRepository.findByOrderId(orderId);
  // Then
  assertEquals("1000", mockOrderList.get(0).getId());
}
```

@Sql对应的数据insert_0_order.sql文件内容如下：
```sql
INSERT INTO order_info
(id, order_id, contact, sender, receiver, order_state, order_time, deleted)
VALUES(1000, 'Acg2345', 'steve', 'jobs', 'cooker', '运送中', '2022-08-10', 0);
```

- 当一个repo层测试文件有多个@Sql注入mock的sql时，需要注意的是，多个注入sql可能导致测试文件的scope被多次注入相同的sql的key，导致报错。解决办法要么在@sql中使用不同的sql文件注入，要么在class上使用@Transactional(rollbackFor = Exception.class)，来保证每个测试方法后会被事务性回滚。

## service层测试
service层需要隔离其依赖的repository。也就是将其mock处理。

### 不需要依赖repository的service
```java
@Test
void testGenerateOrderIdByUserId() {
    // Given
    String userId = "1234567890";
    // When
    String mockOrderId = "ACG" + OrderIdUtils.getTimeStamp() + OrderIdUtils.getLast4Num(userId) + OrderIdUtils.get4RandomNum();
    String orderId = orderService.generateOrderId(userId);
    // 去掉最后四位随机数
    String orderIdNoRandom = orderId.substring(0, orderId.length() - 4);
    String mockOrderIdNoRandom = mockOrderId.substring(0, mockOrderId.length() - 4);
    // Then
    assertEquals(mockOrderIdNoRandom, orderIdNoRandom);
}
```

### 依赖repository的service
这里的OrderService层需要依赖的repository代码都需要mock，然后通过构造器注入。
```java
private final AcgBusinessClient mockAcgBusinessClient = mock(AcgBusinessClient.class);
private final OrderRepository orderRepository = mock(OrderRepository.class);
private final CurrentUser currentUser = mock(CurrentUser.class);
private final OrderService orderService = new OrderService(mockAcgBusinessClient, orderRepository, null, currentUser,null);

@Test
void shouldReturnOrderListWhenInputCorrectlyUserId() {
    String userId = "123124";
    List<Order> orders = List.of(
            new Order("123", "Acg2345", "1234", "1234", "1234", "14213", LocalDateTime.now(), false, userId, 0),
            new Order("123", "Acg2345", "1234", "1234", "1234", "14213", LocalDateTime.now(), false, userId, 0));
    when(orderRepository.findByUserIdOrderByOrderTimeDesc(userId)).thenReturn(orders);
    when(currentUser.getUserId()).thenReturn(userId);
    List<Order> result = orderService.getOrders();
    assertEquals(2, result.size());
}
```

## controller层测试
controller层测试需要mock service层代码，然后接受API请求，需要Spring框架。在controller层不需要普通的断言assertXXX。值得一提的是，when方法中service调用的方法都入参中的```any(XXXDTO.class)```，这里使用的any方法可以保证在之后的```.content(new ObjectMapper().writeValueAsString(XXXDTO))```序列化和反序列化中的参数类型能保持一致，并且能成功获得body。多次测试中，如果前面不加any方法会导致接口200，但body返回为空。还有可以在controller层测试和api层测试的class上加上注释```@SuppressWarnings("PMD")```，以避免需要写一些无用断言才能提交的问题。
```java
@Autowired
private MockMvc mvc;

@MockBean
private OrderService orderService;

void testUpdateOrderStateByOrderStatusDto() throws Exception {
    OrderStatusChangeDto orderStatusChangeDto = BeanTestUtils.buildOrderStatusChangeDto();
    when(orderService.updateOrderStateByOrderStatusDto(any(OrderStatusChangeDto.class))).thenReturn(SUCCESS);

    mvc.perform(MockMvcRequestBuilders
                    .post("/orders/orderStatus")
                    .content(new ObjectMapper().writeValueAsString(orderStatusChangeDto))
                    .contentType(MediaType.APPLICATION_JSON_VALUE))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(MockMvcResultMatchers.jsonPath("$").value("success"))
            .andReturn();
}
```
对应get请求的测试，常见的如下所示。有普通的和restApi风格。
```java
.get("/examples/" + exampleId)
// ----
String userId = "123";
.get("/examples/{userId}, userId)
```

## api测试
api层测试不需要mock其他层，如果有需要跟数据库交互则需要mock一个SQL。但值得一提的是，如下提供的demo代码块，就mock了两个service，原因是第一个mock的service--AcgGetPriceRequest类是第三方调用，微服务多个模块之间本来就是需要相互通信的，这在api测试里也只能mock。第二个mock的service是CurrentUser这个类，这个类本身是另一个模块的鉴权和分配tokne功能，本身更贴切的做法是直接去在mvc中的header加入token，但这里成本很高所以直接mock第三方service也是合理的。
```java
@Autowired
private MockMvc mvc;
@MockBean
private AcgBusinessClient mockAcgBusinessClient;
@MockBean
private CurrentUser mockCurrentUser;

@Test
void saveOrderPackage() throws Exception {
  AcgGetPriceRequest acgGetPriceRequest = new AcgGetPriceRequest("AOG", "AQG");

  when(mockAcgBusinessClient.getPrice(acgGetPriceRequest)).thenReturn(new AcgGetPriceResponse(
          BigDecimal.valueOf(3978525),
          BigDecimal.valueOf(3057618),
          BigDecimal.valueOf(454822),
          BigDecimal.valueOf(762333233)
  ));
  when(mockCurrentUser.getUserId()).thenReturn("59dc9490-b283-4785-a7d2-a90aa97498bf");
  OrderPackageDTO orderPackageDTO = BeanTestUtils.getOrderPackageDTO();

  mvc.perform(MockMvcRequestBuilders
                  .post("/orders/orderInfo")
                  .contentType(MediaType.APPLICATION_JSON_VALUE)
                  .content(new ObjectMapper().writeValueAsString(orderPackageDTO)))
          .andDo(print())
          .andExpect(status().isOk())
          .andExpect(MockMvcResultMatchers.jsonPath("$.pickUpPrice").value(3978525));
}
```

下面的demo代码块是展示下需要DAO层返回数据的mock。
```java
@Sql("/sql/insert_b_example.sql")
public void should_return_content() throws Exception {
    mvc.perform(MockMvcRequestBuilders.get("/shopping_cart"))
        .andExpect(jsonPath("$.products").isArray())
        .andExpect(jsonPath("$.products.[0].id").value(1L))
        .andExpect(jsonPath("$.products.[0].price").value(BigDecimal.valueOf(20).setScale(1)));
}
```
值得一提的是，很多时候测试可以在开发代码中打断点debug，是非常好的调试测试的办法。

## 测试注解
- @JUnitWebAppTest 包含了@Target(ElementType.TYPE)//只能在class上使用、@Retention(RetentionPolicy.RUNTIME)//在运行期生效所以可以反射性地读取它们、@Inherited//允许子类继承、@ActiveProfiles("test")、@SpringBootTest、@AutoConfigureMockMvc。一般来讲集成测试直接使用这个注解在class上就可。
- @SpringBootTest // 使用spring框架跑测试，并且创建的测试类必须在主启动类所在包路径同级下或其子级下，否则无法扫描到bean，且无法使用那些自动注入的注解。不过单独使用一般需要结合其他注解，比如指定配置文件。
- @AutoConfigureMockMvc // 该注解表示 MockMvc由spring容器构建，你只负责注入之后用就可以了。这种写法是为了让测试在Spring容器环境下执行。
- @ActiveProfiles // 指定配置文件
- @Sql // 制定mock的DAO层的sql逻辑
- @MockBean // 避免真正去调用依赖的Bean
- @ExtendWith // 一般配置SpringExtension.class。JUnit5版本，一般不需要单独使用。
- @RunWith // 指定运行容器，如SpringRunner.class，避免依赖注入空指针错误。是JUnit4版本，一般不需要单独使用。
- @InjectMocks 等价于@Autowired 和其他MockBean自动注入。
- @Mock 在启动spring框架之下（也就是使用@SpringBootTest）条件下，能让字段等价于final Xxx xxx = mock(xxx.class);

## 静态测试方法
- mock() ，给予类成员字段（可以为非Bean）mock注入。
- when ， 指定Bean的行为，结合thenReturn，指定mock的Bean返回的内容。
- verify ， 确定某个Bean是否会调用某个方法。
- assertEquals ， 断言返回内容的相等。
- assertTrue ， 断言确定返回内容布尔值为真。
- assertThrows ， 断言确定抛出的异常。

# 常见前端测试

## 普通UI测试
```javascript
jest.mock('./service');
getOrderList.mockResolvedValue([]);
describe('checkOrderPage test', () => {
  test('should render check-order-page in screen', () => {
    render(<CheckOrderPage />);
    expect(screen.getByText('订单管理')).toBeInTheDocument();
    expect(screen.getByText('操作')).toBeInTheDocument();
  });
});
```
前端测试未完待续······

# 工具网址
- [丰富的React hook库](https://github.com/streamich/react-use)
- [JPA根据解析方法名自定义查询方法]https://blog.csdn.net/qq_38974638/article/details/120165125
- [Spring Data JPA官方英文文档](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories)
