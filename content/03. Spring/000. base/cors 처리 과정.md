---
tags:
  - cors
---
# Cors 문제 발생
## 원인 
- spring security 설정에서 option이 그대로 Jwt filter로 들어와서 인증처리가 안됐음
- SecurityConfig에서도 Option이 예외 처리가 안되어 있어서 403 에러 발생
## 해결방법
### 기존의 WebConfig를 사용해서 cors 처리
- 아래의 과정을 거치면 Option 요청시 정상 200 응답이 오는 것을 확인할 수 있다.
#### `Jwt filter`에서 Option은 통과 처리
```Java
if (HttpMethod.OPTIONS.matches(request.getMethod())) {  
    filterChain.doFilter(request, response);  
    return; // OPTIONS 요청 처리 후 메소드 종료  
}
```
#### Security Config에서 Option은 통과 처리
```Java
http  
    .authorizeHttpRequests((authorizeHttpRequests) -> authorizeHttpRequests  
        .requestMatchers(HttpMethod.OPTIONS)  
        .permitAll()
```
#### WebConfig에서 cors 설정 추가
```Java
@Configuration  
public class SpringWebConfig implements WebMvcConfigurer {  
  
    @Override  
    public void addCorsMappings(CorsRegistry registry) {  
        registry.addMapping("/**")  
            .allowedOrigins("http://localhost:3000") // 허용할 출처 : 특정 도메인만 받을 수 있음  
            .allowedMethods("GET", "POST", "PATCH", "DELETE", "PUT") // 허용할 HTTP method            
            .allowCredentials(true);  
    }
}
```
### Spring Security 설정을 통해서 처리
[spring security cors 링크](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)
#### cors 설정을 추가한다.
```Java
/**  
 * Cors Global 처리  
 */  
UrlBasedCorsConfigurationSource corsConfigurationSource() {  
    CorsConfiguration configuration = new CorsConfiguration();  
    configuration.setAllowedOrigins(List.of("http://localhost:3000"));  
    configuration.setAllowedMethods(  
        List.of("GET", "POST", "PUT", "DELETE")); // 모든 메소드 허용  
    configuration.setAllowedHeaders(List.of("*")); // 모든 헤더 허용  
    configuration.setAllowCredentials(true); // 자격 증명 허용  
  
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();  
    source.registerCorsConfiguration("/**", configuration); // 모든 경로에 대해 적용  
    return source;  
}
```
#### Http security 설정 등록
```Java
@Bean  
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {  
  
    // CSRF 설정 및 시큐리티 기본 설정 비활성화  
    http.csrf(AbstractHttpConfigurer::disable);  
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
	
	... 추가 인증 정보 등록
}
```
### 주의 사항
- CorsFilterChain을 사용하는데 Filter Chain의 순서가 뒤집힐 수 있으니 주의하자.
- 처음에는 Bean으로 주입했었는데 그렇게 하면 동작하지 않으니 직접 함수로 주입해주자