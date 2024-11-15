## BuildGradle 설정

```gradle
// mongodb  
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
```
## Entity 설정
```Java
@Document(collection = "FoodCart")  
@Getter  
public class Cart {  
  
    @Indexed(unique = true)  
    String userId;
}
```

Document 어노테이션을 이용해서 MongoDB로 저장되는 entity를 지정해 줄 수 있으며
collection의 이름을 파라미터 값으로 넘겨 줄 수 있다.

MongoDB로 인덱스를 설정해 주기 위해서 위의 `@Indexed` 어노테이션을 이용해서 지정해 줄 수 있으며
Spring을 통해서 Index를 적용해 주기 위해서는 아래와 같이 설정을 해줘야 한다.
```yaml
mongodb:  
  auto-index-creation: true
```

## 저장 및 조회

### Repository
```Java
public interface CartRepository extends MongoRepository<Cart, String> {  
  
}
```
기존의 RDS에서 Jpa를 적용했던 것처럼 위와 같이 Repository를 설정해서 save 및 조회가 가능하다
##### Service에서 적용
``` Java
// 저장
Cart cart = Cart.createdCart(userId, product.getStore().getId(), product); 
cartRepository.save(cart);

// 조회
cartRepository.findByUserId(userId).orElse(null);
```

### MongoTemplate
```Java
Query query = new Query();  
query.addCriteria(Criteria.where("userId").is(userId));  
  
Update update = new Update();  
update.set("storeId", product.getStore().getId());  
update.set("productDto", ProductDto.createProduct(product));  
  
mongoTemplate.upsert(query, update, Cart.class);
```
#### MongoTemplate의 선택 이유
Cart를 간편한 저장 및 빠른 조회를 위해서 MongoDB를 사용해서 Cart 기능을 구현하고자 하였고
이 때 기본 ID를 유지하면서 userId를 키로 사용하고자 하였다.
#### 기존 아이디의 유지 이유
[ObjectId 공식 문서](https://www.mongodb.com/docs/manual/reference/bson-types/#objectid)
- 12바이트의 크기로 구성되어 상대적으로 적은 공간을 사용한다.
- 1초단위이지만 타임스탬프를 기반으로 생성되어서 정렬 및 필터링이 가능하다
- 고유성을 보장해준다.
위의 이유로 위와 같이 별도로 userId를 인덱스 하여 고유성을 보장해주었으며 간편한 저장을 위해서
`upsert`를 이용해서 저장을 데이터를 변경하고자 하였다

## Criteria를 사용한 쿼리

```Java
import static org.springframework.data.mongodb.core.query.Criteria.where;
import static org.springframework.data.mongodb.core.query.Query.query;

// ...

// 목록 조회
List<Person> result = template.query(Person.class)
  .matching(query(Criteria
  .where("age").lt(50).and("accounts.balance").gt(1000.00d)))
  .all();

// 단일 조회
Cart first = mongoTemplate.query(Cart.class)  
    .matching(query(where("userId").is(userId))).firstValue();

```
Spring data MongoDB에서 제공하는 Query 및 Criteria를 이용해서 위와 같이 조회가 가능하다.

```Java
Query query = new Query();  
query.addCriteria(where("userId").is(userId));
```
위와 이와 같이 동적으로 쿼리를 해야하는 경우 추가를 해줄 수도 있다.

[[MongoDB]]