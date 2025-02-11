---
tags:
  - restClient
  - restTemplate
  - webClient
---

# 외부 API를 위한 통신 방법
## RestTemplate
외부 통신을 위한 API로 동기방식으로 동작하며 HTTP 메서드별로 getForObject(), postForEntity() 등 구체적인 메서드를 호출하여 요청을 보내게 된다.
다양한 메시지 컨버터, 인터셉터, 에러 핸들러 등을 통해 세밀하게 커스터마이징할 수 있다.
[RestTemplate Deprecate 이슈](https://github.com/spring-projects/spring-framework/issues/32016)   
1월에 RestTemplate에 관련해서 Github에 물어본 질문이 있었는데 여기에서 답변은 7.x 버전까지는 지속적으로 유지 될 것이고 많은 사람들에게 혼돈을 주는 것 같아서 유지보수 모드도 제거했다고 적혀있다.
Spring 6.1 버전 부터 코드가 더 유연하고 가독성이 좋은 API를 제공하게 되는 것이 RestClient이다.
## WebClient
WebFlux에서 제공하는 HTTP 클라이언트로, 비동기적으로 Non-blocking 방식의 요청을 지원한다.

## RestClient
[RestClient 공식문서](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)   
스프링에서 외부 api를 호출함에 있어서 RestTemplate, WebClient, RestClient를 사용한다. 기존에는 RestTemplate이 사용이 되었으나 Template 클래스가 Http 기능들을 노출하게 됨에 따라서 너무나도 많은 메서드들을 오버로드 하는 문제가 있다고 판단되었다.

위의 문제를 해결하기 위해서 Spring 5.0에서 나온 것이 WebClient이다. 하지만 WebClient는 webflux아래에서 돌아가게 되어서 기본적으로 MVC로 동작하는 spring에서 WebClient를 사용하기 위해서 webflux의 모든 의존성을 추가해야 하는 단점이 있다.
### Http Interface
Http 요청을 위한 서비스를 자바 인터페이스와 어노테이션으로 정의 할 수 있도록 도와주는 역할이다.   
Http Interface와 연관하기 위해서 아래와 같이 Adpater를 만들어셔 연동시켜준다.

#### 적용해본 예시
```Java
// weatherConfig의 base URL을 사용하여 DefaultUriBuilderFactory를 생성하고,
// URL 인코딩을 수행하지 않도록 설정
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(
    weatherConfig.baseurl());
uriBuilderFactory.setEncodingMode(EncodingMode.NONE);

// 생성한 uriBuilderFactory와 로그용 인터셉터(logRequestInterceptor)를 
// 이용해 RestClient를 빌드
RestClient restClient = RestClient.builder()
    .uriBuilderFactory(uriBuilderFactory)
    .requestInterceptor(logRequestInterceptor())
    .build();

// RestClient를 어댑터로 감싸서 HttpServiceProxyFactory에 사용할 수 있도록 변환
RestClientAdapter adapter = RestClientAdapter.create(restClient);

// 어댑터를 기반으로 HTTP 서비스 프록시 팩토리를 빌드
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

// WeatherClient 인터페이스에 대한 HTTP 클라이언트 프록시를 생성하여, 
// 이후 WeatherClient의 메서드 호출 시 REST API 호출이 수행되도록 함
factory.createClient(WeatherClient.class);
```

위에서 Adapter와 함께 RestClient를 Http Interface와 연결을 시킨 다음에는 아래와 같이 함수 형식으로 선언이 가능하다. 또한 RequestParam이나 PathVaraible로 Request를 전송할 때 변수들을 선언해 줄 수 있다.
```Java
@HttpExchange
public interface WeatherClient {

    @GetExchange("?pageNo=1"
        + "&numOfRows=200"
        + "&dataType=JSON"
        + "&base_time=0550")
    WeatherResponse findWeatherInfo(
        @RequestParam("serviceKey") String serviceKey,
        @RequestParam("base_date") String baseDate,
        @RequestParam("nx") String nx,
        @RequestParam("ny") String ny
    );

}

```
이후 외부 API 호출이 필요한 지점에서 아래와 같이 의존성을 주입받은 이후에 메서드를 호출해주면 된다.
```Java
private final WeatherClient weatherClient;

// API 사용을 이용을 위한 메서드 호출
WeatherResponse weatherInfo 
= weatherClient.findWeatherInfo(weatherConfig.key(), today,  x, y);
```

## RestClient 적용 이후 문제 발생했던 점
### 보내는 URL 확인

처음 요청을 하였을 때 실패를 하여서 원인을 찾고자 하였음 하지만 어디서 문제가 발생했는지 확인하려고 했을 때 URL을 확인해 보고 싶었으나 baseURL로 넣어두고 코드로 쿼리파람을 입력하였는데 정작 보내는 URL확인이 로그로 안찍힘.

이를 확인하기 위해서 찾아본 결과 interceptor를 RestClient에 등록할 수 있는 것을 알게 되었고 여기서 로그를 찍어보기로 하였음

아래와 같이 ClientHttpRequestInterceptor를 생성해서 반환해주는 메소드를 생성 후 RestClient를 선언할 때 붙여주었다.

```java
RestClient restClient = RestClient.builder()
            .uriBuilderFactory(uriBuilderFactory)
            .requestInterceptor(logRequestInterceptor())
            .build();

private ClientHttpRequestInterceptor logRequestInterceptor() {
  return new ClientHttpRequestInterceptor() {
      @Override
      public ClientHttpResponse intercept(HttpRequest request, byte[] body,
          ClientHttpRequestExecution execution) throws IOException {
          log.info("Request URL: {}", request.getURI()); // 출력 url 확인
          return execution.execute(request, body);
      }
  };
}
```

### URL 인코딩 문제

[https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-uri-building.html](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-uri-building.html)

기상청에서 날씨 정보를 갖고 오려고 했을 때 기상청 API의 키에 %가 들어가 있어서 RequestParam으로 요청을 보내면 Encode가 되면서 Key가 변형이 되는 문제가 있었다. 여기서 Encode를 막는 방법을 알아보게 되었고 RestClient에서 Encode를 하지 않는 방법을 알게 되었다.

RestClient에서는 uri를 만들 때 아래와 같이 URI를 만들어 줄 수 있으며 여기서 EncodingMode를 설정해 줄 수 있다.

이번에는 아무런 인코딩을 하고 싶지 않으므로 NONE를 설정하엿다.

```java
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(
    weatherConfig.baseurl());
uriBuilderFactory.setEncodingMode(EncodingMode.NONE);
```

인코딩 종류로는 아래와 같이 4가지가 있다.   
기본값으로는 TEMPLATE_AND_VALUES이다
```Java
TEMPLATE_AND_VALUES
- non-ASCII 및 Illegal characters가 대체된다
VALUES_ONLY
- URI 템플릿에는 적용하지 않으나 대신 URI variables에 엄격한 Encoding을 적용 한다
URI_COMPONENT
- URI 변수를 먼저 확장하고 URI component 변수 결과를 인코딩한다.
NONE
```