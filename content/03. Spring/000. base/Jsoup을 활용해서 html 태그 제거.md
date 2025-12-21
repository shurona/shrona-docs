---
tags:
  - jsoup
---

Todo: 이거 완성하기 (\n을 유지하는 방법)

String s = Jsoup.parse(updBusinessProfileInfoDto.getProfileIntroduction()).wholeText();

Jsoup.clean(updBusinessProfileInfoDto.getProfileIntroduction(), "", Safelist.none(), new Document.OutputSettings().prettyPrint(false)),