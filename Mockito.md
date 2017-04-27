##  初始化
@Rule
public MockitoRule mockitoRule = MockitoJUnit.rule();
@Mock
private MockService mockService;
@InjectMocks
TestService testService = new TestService();

## mock
### mock有返回值的方法
Result result = new Result();
when([mockService].[method_name](any(String.class))).thenReturn(result);

### mock没有返回值（void）的方法
doNothing().when([mockService]).[method_name](anyInt(),anyBoolean()); 

### mock抛出异常
#### mock普通方法

when([mockService].[method_name](anyInt(),anyBoolean())).thenThrow(new RuntimeException());

#### mock void方法
doThrow(new RuntimeException("ERROR")).when(mockService).[method_name](anyInt(),anyBoolean());

