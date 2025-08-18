---
tags:
  - StringUtils
  - String
---
# 자바에서 Base64 변환
## Base64 -> Utf8
```Java
import java.util.Base64;
import java.nio.charset.StandardCharsets;

public class Base64DecoderExample {
    public static void main(String[] args) {
        String base64Encoded = "c3Nzc3M="; // "sssss"의 Base64 인코딩 결과

        // Base64 디코딩
        byte[] decodedBytes = Base64.getDecoder().decode(base64Encoded);

        // UTF-8로 문자열 변환
        String decodedString = new String(decodedBytes, StandardCharsets.UTF_8);

        System.out.println(decodedString); // 출력: sssss
    }
}
```
## Utf8 -> Base64
```Java
import java.util.Base64;

public class Base64EncoderExample {
    public static void main(String[] args) {
        String originalString = "sssss";
        String base64Encoded = Base64.getEncoder().encodeToString(originalString.getBytes());
        System.out.println("Base64 인코딩된 문자열: " + base64Encoded);
    }
}
```
# String.isEmpty()와 String.isBlank()의 차이
## isEmpty
- 동작: 문자열의 길이가 0일 때만 `true`를 반환합니다.
- 즉, ""(빈 문자열)만 true이고, `" "`(공백 포함)이나 `"\t"`(탭) 등은 false입니다.
### 예시
```Java
"".isEmpty();      // true
" ".isEmpty();     // false
"\t".isEmpty();    // false
"abc".isEmpty();   // false
```
## isBlank
- 동작: 문자열이 비어 있거나, 공백 문자(스페이스, 탭, 개행 등)만으로 이루어진 경우에도 `true`를 반환합니다.
- Java 11 이상에서 사용 가능
- 아래 패키지를 사용해서 사용하자
	- `org.apache.commons.langs.StringUtils`
	- 비슷한 이름이 많아서 다른 비슷한 클래스를 import할 수 있음
### 예시
```Java
"".isBlank();      // true
" ".isBlank();     // true
"\t".isBlank();    // true
"abc".isBlank();   // false
```