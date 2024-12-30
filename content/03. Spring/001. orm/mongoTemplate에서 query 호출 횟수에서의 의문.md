# 원인 발생
MongoDB와 Cache를 활용해서 카트를 저장하고 조회하는 서비스를 만들었고 이후 이에대한 테스트를 작성하는 과정에서 문제가 발생하였다.

## 서비스 코드
```Java
@CachePut(cacheNames = "userCart", key = "args[1]")  
public CartResponseDto createCart(Long productId, String userId) {  
	// 일반적인 Product 생성
    Product product = productRepository.findById(productId)  
        .orElseThrow(() -> new ProductException(  
            ProductErrorCode.PRODUCT_NOT_FOUND)  
        );  
  
    Query query = new Query();  
    query.addCriteria(where("userId").is(userId));  
  
    Update update = new Update();  
    update.set("storeId", product.getStore().getId());  
    update.set("productDto", ProductDto.createProduct(product));  
    
    mongoTemplate.upsert(query, update, Cart.class);  
  
    Cart cart = mongoTemplate.query(Cart.class)  
        .matching(query(where("userId").is(userId))).firstValue();  
  
    return CartResponseDto.from(cart);  
}
```
`CachePut`어노테이션을 사용해서 메소드의 Output을 userId를 기준으로 캐시를 진행하도록 작성하였다.
내부적으로는 넘겨받은 userId 파라미터를 기준으로 데이터가 있으면 update 없으면 insert하도록 진행한다.
## 테스트코드
```Java
@DisplayName("카트 생성")  
@Test  
public void 카트_생성() {  
	// given  
	String name = "name";  
	Store store = mock(Store.class);  
	Long productId = 1L;  
	Product product = Product.of(store, name, 1200.0, "description");  
	String userId = "userOne";  

	// when  
	when(productRepository.findById(anyLong()))
	.thenReturn(Optional.of(product));  

	// then  
	CartResponseDto cartResponseDto 
		= cartService.createCart(productId, userId);  

	...

}  

@DisplayName("캐시 적용 유무 확인")  
@Test  
public void 캐시_적용확인() {  
	// given  
	String name = "name";  
	String userId = "userId";  
	Long productId = 1L;  
	Store store = mock(Store.class);  
	Product product = Product.of(store, name, 1200.0, "description");  

	// productDB는 mocking한다.          
	when(productRepository.findById(anyLong()))
	.thenReturn(Optional.of(product));  

	// 카트 생성  
	cartService.createCart(productId, userId);  

	// then        
	// find에서는 호출이 안되므로 create 할 때만 호출되어야 한다.  
	verify(mongoTemplate, times(1)).query(Cart.class);  

}
```
두 테스트 모두 동일하게 createCart를 실행을 한다.     

`verify(mongoTemplate, times(1)).query(Cart.class);`    
여기서 아래 `캐시 적용확인` 메서드만을 호출하는 경우 위의 호출 횟수가 2번 이지만
두 테스트를 동시에 실행하면 위의 호출 횟수가 1번으로 나온다.