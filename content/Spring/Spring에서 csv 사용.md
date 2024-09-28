# CSV처리를 진행 한 이유

라이더가 배달 구역을 선택할 때에는 위도 경도에 따른 범위 선택이 아니라 법정도를 기준으로 선택을 하기로 정하였고 이를 위해서 정부에서 제공해 주는 법정동 csv를 이용하기로 하였다.

[https://www.data.go.kr/data/15063424/fileData.do](https://www.data.go.kr/data/15063424/fileData.do)

### 고민 했던 사항

- 다른 모듈에서 사용하는 주소를 위해서 서버를 하나 새로 생성한다
- CSV 파일을 각 모듈에서 처리 한다.

이번 프로젝트를 MSA로 구성하게 되면서 서버의 갯수가 적지 않았고 공통 주소 처리를 위해서 서버를 하나 더 띄운다는 점이 부담으로 다가왔고 csv 파일이 4M 정도에 크지 않은 파일이였기 때문에 이를 해결 하기 위해서 다른 방안을 찾아보았다.

CSV파일을 각자의 모듈에서 나눠서 갖고 처리 하는 것은 추후 유지보수에서 불리할 것 같아서 다른 방안을 찾아보는 중에 멀티모듈의 공통 모듈에서 csv 처리를 진행할 수 있는 지 확인하였고 가능해 보여서 진행하게 되었다.

## open csv라이브러리

opencsv를 Gradle로 추가한 다음에 먼저 csv 파일을 `resource/csv` 폴더의 파위에 넣어놓았다. 이후 ResourceLoader를 사용해서 스프링의 resource의 위치로 부터 csv 파일 위치를 지정한다.

```java
private final ResourceLoader resourceLoader;

Resource resource = resourceLoader.getResource("classpath:" + ADDRESS_CSV);
```

아래와 같이 CSVReader를 이용하여서 첫 째 줄을 제외한 나머지를 읽어서 메모리에 저장한다.

```java
CSVReader reader = new CSVReader(
      new InputStreamReader(resource.getInputStream(), Charset.forName("EUC-KR")));

List<String[]> strings = reader.readAll();
this.addressSetList = strings.subList(1, strings.size());
```

### 1차 접근

처음에 위에서 읽어온 주소 목록을 가공해서 저장을 해보려고 아래와 같이 진행을 하였다.

```java
public void classifyAddress() {
    for (int i = 1; i < addressList.size(); i++) {
        String[] addressRow = addressList.get(i);
        String street = addressRow[2];

        if (!StringUtils.hasText(street)) {
            continue;
        }

        // streetInfo가 없는 경우에만 리스트를 생성하도록 최적화
        streetInfo.computeIfAbsent(street, k -> new ArrayList<>());

        // AddressDto 객체를 생성하고 리스트에 추가
        AddressDto addressDto = new AddressDto(
            addressRow[0],
            addressRow[1],
            addressRow[2],
            addressRow[3]
        );

        streetInfo.get(street).add(addressDto);
    }
}
```

주소의 목록은 대략 5만 라인이고 이를 City나 Street에 맞춰서 빠르게 쿼리할 수 있게 가공해 보려고 하였다. 하지만 위의 방식으로 진행을 하면 Map에 확인 후 List 초기화 및 객체 생성등을 통해서 가공되는 시간만 해도 20초가 걸리는 것으로 확인 되었다.

### 2차 접근

가공해서 데이터를 저장하는 것이 csv 파일을 저장하는 것과 메모리 차이가 크지 않을 것 같은 것으로 확인 되어서 csv 파일 목록을 그대로 메모리에 넣어 놓고 필요한 데이터를 함수로 갖고 오는 방식으로 변경

메모리에 올라가 있는 5만라인을 한 번 반복 자체는 크게 속도 상에서 문제가 없을 거라고 판단 하였고 로컬 환경에서도 0.1초도 안걸리는 것으로 확인 되어서 이 방식으로 진행하였다.

주소목록이 필요할 만한 시나리오로 계산해서 여러 함수로 나눠서 처리하였다

**도시에 속한 주소 목록을 조회하는 예시**

```java
public List<AddressDto> findAddressListByCity(String city) {
    checkAddressExist();
    return this.addressSetList.stream()
        .filter(addressSet -> addressSet[CITY_INDEX].equals(city))
        .map(AddressDto::from).toList();
}
```