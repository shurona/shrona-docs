---
tags:
  - filter
  - body
---
# Spring에서 Body처리 법
## `HttpServletRequest/InputStream` 직접 사용
- 바디를 직접 읽어서 처리 한다.
```Java
@PostMapping("/request-body-string")
public void requestBodyString(InputStream inputStream, Writer, responseWriter) throws IOException {
	String messageBody = StreamUtils.copyToString(inputStream, UTF_8);
	responseWriter.write("ok");
}
```
## HttpEntity 사용
```Java
public HttpEntity<String> requestBodyString(HttpEntity<String> httpEntity) {  
    String body = httpEntity.getBody();  
    return new HttpEntity<>("ok")
}
```
## RequestBody 사용법
- 들어오는 형식을 미리 정해놓으면 `RequestBody`의 어노테이션을 통해서 받을 수 있다.
```Java
public HttpEntity<String> requestBodyString(@RequestBody RequestBodyDto requestBodyDto) {  
      
    return new HttpEntity<>("ok");  
}
```
# 동작 원리
- Spring은 내부적으로 HttpMessageConverter를 사용해서 Http Body를 변환한다.
- Json 변환의 경우 Jackson 라이브러리를 통해서 객체 매핑이 이루어 진다.
- DTO 클래스는 기본 생성자와 `getter/setter`가 필요하다.
# Filter나 interceptor에서 읽으면 Controller에서 null이 되는 이유
## 원인
- Spring에서 Http 요청의 Body를 filter나 interceptor에서 한 번 읽으면 이후 Controller에서 `@RequestBody`나 `HttpServletRequest.getInputStream()`으로 다시 읽을 때 Body가 비어 있는 에러가 발생합니다.
### InputStream의 특성
- Http 요청의 Body는 네트워크에서 들어오는 스트림 데이터이다.
- Java Servlet의 `getInputStream`이나 `getReader()`로 Body를 읽을 수 있지만 이는 스트림 데이터이므로 단 한 번만 읽을 수 있다.
## 해결 방법
### ContentCachingRequestWrapper 사용
- Spring에서 제공하는 `ContentCachingRequestWrapper`는 요청 Body를 내부적으로 캐싱하여 여러 번 읽을 수 있게 한다.
- 다만 Body를 읽기전에 Filter에서 감싸줘야 한다.
  그 전에 이미 읽었으면 의미가 없다.
#### 예시 코드
- 필터 적용 방법
```Java
@Component
public class GlobalFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	    CachedBodyHttpServletRequest wrappedRequest = new CachedBodyHttpServletRequest(  
    servletRequest);  
  
	try {  
	    String header = servletRequest.getHeader("x-line-signature");  
	  
	    // Body 읽기  
	    String body = new BufferedReader(wrappedRequest.getReader())  
	        .lines()  
	        .collect(Collectors.joining());    
		
	        chain.doFilter(cachingRequest, response);
	        // 이후 cachingRequest.getContentAsByteArray()로 Body 재활용 가능
	    }
	    ...
	}
}
```
- 실제 CacheBody 코드
```Java
public class CachedBodyHttpServletRequest extends HttpServletRequestWrapper {  
  
    private final byte[] cachedBody;  
  
    public CachedBodyHttpServletRequest(HttpServletRequest request) throws IOException {  
        super(request);  
  
        // InputStream을 모두 읽어 캐싱  
        InputStream requestInputStream = request.getInputStream();  
        this.cachedBody = requestInputStream.readAllBytes();  
    }  
  
    @Override  
    public ServletInputStream getInputStream() {  
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(this.cachedBody);  
  
        return new ServletInputStream() {  
            @Override  
            public int read() {  
                return byteArrayInputStream.read();  
            }  
  
            @Override  
            public boolean isFinished() {  
                return byteArrayInputStream.available() == 0;  
            }  
  
            @Override  
            public boolean isReady() {  
                return true;  
            }  
  
            @Override  
            public void setReadListener(ReadListener readListener) {  
            }  
        };  
    }  
  
    @Override  
    public BufferedReader getReader() {  
        return new BufferedReader(new InputStreamReader(this.getInputStream()));  
    }  
}
```
# Body를 정렬하는 방법
## 사용 이유
Body 데이터를 검증에 이용하는 상황이 있었는데 들어오는 Body의 데이터의 Key 위치가 달라지면 검증에 문제가 발생하는 상황이 있었다.   
## 해결 코드
- 항상 key 순서(알파벳 오름차순)로 변환된 JSON 문자열을 만든다.
```Java
ObjectMapper objectMapper = new ObjectMapper();  

// 키 순서 정렬  
objectMapper.configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true); 

// String으로 변환
objectMapper.writeValueAsString(objectMapper.readTree(body));
```
## 용도
- JSON의 key 순서가 달라도 동일한 구조/값이면 항상 같은 문자열로 만들고 싶을 때 사용합니다.
- JSON의 key 순서가 바뀌어도 같은 데이터로 인식해야 할 때(예: API 요청 서명, 캐시 키 생성 등)에 유용합니다.