---
tags:
  - test
---
#  기본 개념

```Java
@SpringBootTest
@ExtendWith({
	TestContainerConfig.class,
	MockitoExtension.class
})
class ServiceTest {

	//

}
```
`SpringBootTest` 어노테이션은 통합테스트를 제공하는 기본 어노테이션이다.   
해당 어노테이션을 사용하면 실제 구동되는 어플리케이션과 똑같이 `ApplicationContext`를 로드해서 테스트한한다.   
다만 모든 애플리케이션을 로드하기 때문에 일반적인 단위 테스트 보다 시간이 오래 걸린다.

# 통합테스트에서 verify 사용하기
처음에 통합테스트에서 내부 메서드가 호출되었는지 확인하기 위해서 Mockito의 verify 메서드를 사용하였다.
```Java
@Cacheable(cacheNames = "userCart", key = "args[0]")
public CartResponseDto findCartById(String userId) {
	...
}
```
위의 코드가 예제로 먼저 create로 캐시를 적용하면 아래 `findCardById`메소드가 호출되지 않는 것을 확인하기 위함이였다.    
하지만 에러가 발생하였고 자세한 해결 과정은 [[테스트 작성 중 만난 에러들]] 여기서 확인할 수 있다.
```Java
@SpyBean  
private MongoTemplate mongoTemplate;
```
위와 같이 SpyBean을 사용해서 원하는 클래스를 Mockito로 감싸주면 아래와 같이 `verify` 사용이 가능하다.
```Java
verify(mongoTemplate, times(0)).query(Cart.class);
```
