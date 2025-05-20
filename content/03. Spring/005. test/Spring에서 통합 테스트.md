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

# 통합 테스트 중에 Mock 사용
## 예시 코드
```Java
@Transactional  
@SpringBootTest  
class MessageServiceImplTest {
	@Autowired  
	private MessageServiceImpl messageService;

	@MockitoBean
	private GroupServiceImpl groupService;

	@Test
	public void 테스트(){
		// Group 객체 mock 생성 예시
		Group mockGroup = new Group();
	    mockGroup.setId(1L);
	    when(groupService.findById(anyLong())).thenReturn(mockGroup);

		// List 생성 예시
		when(groupService.findGroupByIdList(anyList())).thenReturn(groupList);
	}
}
```
### 코드 설명
`@MockitoBean`으로 Service Bean을 등록하면, 이후 서비스 로직에서 해당 Bean의 실제 메소드 대신 Mockito로 Mock 처리한 메소드가 실행됩니다.
# 통합 테스트 중 @Transactional 동작
- `@Transactional` 어노테이션은 각 테스트 메서드마다 트랜잭션을 생성하고, 테스트 종료 후 자동 롤백한다.
- `@BeforeEach`에서 저장한 데이터도 같은 트랜잭션 내에 존재하지만, 테스트 종료 시 실제 DB에 반영되지 않는다.
