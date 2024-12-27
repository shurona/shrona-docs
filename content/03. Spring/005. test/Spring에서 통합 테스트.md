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
