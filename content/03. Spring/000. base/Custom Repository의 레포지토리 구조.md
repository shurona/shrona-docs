---
tags:
  - repository
---

[Custom Repository Implementations Spring 공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/custom-implementations.html)
## 개념
처음에 CustomRepository 구조를 갖고 가는 이유를 기존의 Spring Data JPA Repository 하나만을 Service에서 상속받고 싶어서 사용하는 구조로 이해를 하였다.

문서를 확인해 보니 실제로 Spring Data JPA에서 권장하는 구조였으며 실제로 구현하는 class는 Impl을 붙여야 한다고 한다.
```Java
// custom repositry interface
interface CustomizedUserRepository {
  void someCustomMethod(User user);
}

// custom repository 구현
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {

  public void someCustomMethod(User user) {
    // Your custom implementation
  }
}
```

구현 자체는 Spring data에 의존을 하지 않으며 일반적인 Spring bean일 수 있습니다. 따라서 일반적인 의존성 주입을 이용해서 위의 작업을 진행할 수 있습니다. 실제로 Impl에 Repository 어노테이션을 사용하면 문제없이 실행되는 것을 확인하였다.

하지만 아래의 방식으로 확장을 할 수 있다. 아래와 같이 확장을 하게 되면 CRUD 및 custom 기능을 제공할 수 있게 된다.
```Java
interface UserRepository extends CrudRepository<User, Long>, CustomizedUserRepository {

  // Declare query methods here
}
```

## 동작 방식

  

### Spring Data JPA 동작

• Spring Data JPA는 UserRepository와 같은 인터페이스를 런타임에 프록시로 생성합니다.
• 이 프록시는 기본 CRUD 메서드를 처리하는 동시에, Custom Repository 구현체(CustomizedUserRepositoryImpl)의 메서드를 위임합니다.

### Custom Repository 검색 규칙

• Spring Data JPA는 인터페이스 이름과 구현 클래스 이름을 기준으로 구현체를 자동으로 검색합니다.
• 구현 클래스는 반드시 **Impl 접미사**를 가져야 합니다. (CustomizedUserRepositoryImpl)

### 다양한 구현 방법
추가로 다양한 Repository들을 하나로 구현할 수 있는데 아래와 같이 구현할 수 있다. 
먼저 세부 구현을 아래와 같이 클래스로 선언을 한다. 
Interface를 Fragments라고 해당 페이지에서는 부르는 것 같다. ⇒ Fragments with their implements
```Java
interface HumanRepository {
  void someHumanMethod(User user);
}

class HumanRepositoryImpl implements HumanRepository {

  public void someHumanMethod(User user) {
    // Your custom implementation
  }
}

interface ContactRepository {

  void someContactMethod(User user);

  User anotherContactMethod(User user);
}

class ContactRepositoryImpl implements ContactRepository {

  public void someContactMethod(User user) {
    // Your custom implementation
  }

  public User anotherContactMethod(User user) {
    // Your custom implementation
  }
}
```

아래와 같이 두 정보를 갖는 하나의 Repository로 병합이 가능하다
```Java
interface UserRepository extends CrudRepository<User, Long>, HumanRepository, ContactRepository { 
	// Declare query methods here 
}
```

또한 Generic 방식으로의 확장도 가능하다
```Java
// 커스텀 Repository생성
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}

class CustomizedSaveImpl<T> implements CustomizedSave<T> {

  public <S extends T> S save(S entity) {
    // Your custom implementation
  }
}

// 일반 Repository로 연결
interface UserRepository extends CrudRepository<User, Long>, CustomizedSave<User> {
}

interface PersonRepository extends CrudRepository<Person, Long>, CustomizedSave<Person> {
}
```