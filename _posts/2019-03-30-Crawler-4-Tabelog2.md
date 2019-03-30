---
layout: post
title:  "타베로그를 크롤링하기2 - Google Places API로 비교하기"
date:   2019-03-30 09:00:00 +0900
categories: jekyll update
---

1. [타베로그를 크롤링하기](https://helloworldpark.github.io/jekyll/update/2019/03/28/Crawler-4-Tabelog.html)
2. [타베로그를 크롤링하기2 - Google Places API로 비교하기](https://helloworldpark.github.io/jekyll/update/2019/03/30/Crawler-4-Tabelog2.html)
3. [타베로그를 크롤링하기3 - 데이터 까보기](https://helloworldpark.github.io/jekyll/update/2019/03/30/Crawler-4-Tabelog3.html)

# 그런데 구글에 검색하면 되잖아요
맞습니다. 비록 타베로그 한국어판 페이지가 있긴 하지만, 굳이 타베로그에 가서 힘들게 검색할 필요는 없습니다. 구글 지도로 훑어보던지, 식당 이름을 직접 검색해도 됩니다.

바로 그런 발상에서, 다음과 같은 일을 해보는 것도 의미가 있겠다 싶었습니다: 타베로그의 정보와 구글의 정보는 어떻게 다를까?

사실 이건 맛집 사이트들의 차이를 비교해보는 것과 동일한 문제입니다. 과연 A 맛집 사이트와 B 사이트는 다르긴 할까? 다르다면, 왜 다를까? 그렇다면, 어디가 더 신뢰할만한 곳일까? 하나하나가 답하기 굉장히 어려운 질문들이라고 생각합니다. 특히 신뢰도의 문제는 더더욱 어려운 질문입니다. 나한테는 A가 더 맞을 수도 있지만, 다른 사람들은 다 B가 더 좋다고 한다면 굉장히 난처해질 수 있을 것입니다. 그걸 증명하는 과정도 고도의 통계학과 데이터, 그리고 사회과학적 분석을 필요로 하기 때문에 답하기는 정말 어려운 질문입니다. 그렇기 때문에 저는 첫 번째 질문인 '타베로그와 구글이 제공하는 삿포로의 식당 평점 분포는 어떻게 다를까?'에 집중하도록 하겠습니다.

# Google Places API로 비교해보자
[Google Places API](https://developers.google.com/places/web-service/intro?hl=ko)는 이를 편리하게 해줄 서비스입니다. HTTP 요청만으로 편리하게 JSON을 얻을 수 있어서 이걸 썼는데... 유일한 단점은 가격입니다만 이는 나중에 설명하겠습니다.

API 문서를 읽어보니 저에게 필요한 건 [Places Search](https://developers.google.com/places/web-service/search?hl=ko)와 [Places Detail](https://developers.google.com/places/web-service/details?hl=ko) 이라는 세부 API들뿐이라 이 두 서비스만 소개하겠습니다.

## 서비스 시작하기
서비스를 사용해보겠습니다. **시작하기** 버튼을 누르면 아래와 같은 창이 뜹니다. 자기가 사용할 서비스만 신청합시다.
<p align="center">
<img src="/images/2019-03-30-01.png"><br>
</p>

프로젝트 이름을 지어줍니다.
<p align="center">
<img src="/images/2019-03-30-02.png"><br>
</p>

잠시 기다리면 결제 계정을 설정하라고 뜹니다. 저같은 경우에는 Google Cloud Engine을 사용하고 있어서 그 결제 계정에 연동했습니다만, 구글에 돈을 처음 헌납하는 분이시라면 신용카드 연동을 하셔야 합니다.
<p align="center">
<img src="/images/2019-03-30-03.png"><br>
</p>

이제 API 키를 생성할 순간입니다.
<p align="center">
<img src="/images/2019-03-30-04.png"><br>
</p>

API 키가 생성되었습니다. **소중히 보관**하셔야 합니다. 만약 유출되면 어마어마한 금액이 청구될 수도 있습니다.
<p align="center">
<img src="/images/2019-03-30-05.png"><br>
</p>

생성 후에는 다음과 같은 화면이 지원됩니다. API 사용 현황을 시각적으로 확인할 수 있습니다.
<p align="center">
<img src="/images/2019-03-30-06.png"><br>
</p>

## API 키를 안전하게
이제 API 키도 받았으니, 개발을 시작하면 될 것 같습니다. 먼저, API 키를 Github와 같은 소스 저장소에 노출시키지 않으면서 개발해야겠습니다. 그래서 저는 API 키는 별도의 파일에 저장해두고, Python 스크립트를 실행할 때마다 파일에서 API 키를 읽어들이도록 작업했습니다. 물론, API 키가 저장된 파일은 ```.gitignore```에 등록해두어야 합니다.
```python
__API_KEY = None


def load_api_key(path):
    with open(path, mode='r') as f:
        global __API_KEY
        __API_KEY = f.read()


if __name__ == '__main__':
    # Load API Key
    load_api_key('../apikey')
```

## Google Places Search
**Places Search**는 키워드를 넣으면 관련된 장소의 위치 정보를 위주로 알려주는 API입니다. 그렇다면 **Places Detail**과 뭐가 다르냐 하면, **Places Search**는 진짜 대충 알려줍니다. 즉, 그 장소에 대해 자세하게 알고 싶으면 **Places Detail**도 사용해야 합니다. 그러면 **Places Detail**을 바로 쓰면 안되냐! 하고 생각하실 수 있는데, 그게 불가능합니다. 이 부분은 뒤에서 설명하겠습니다.

사용법은 간단합니다. 쿼리 형식에 맞게 URL을 호출하면 끝입니다.
> ```https://maps.googleapis.com/maps/api/place/findplacefromtext/output?parameters```

이제 저 ```output```과 ```parameters```를 잘 채워넣어주면 되는데, ```output```에는 ```json``` 또는 ```xml``` 둘 중 하나만 넣을 수 있습니다. 취향껏 넣으시면 될 듯 합니다.
```parameter```가 핵심인데, 다음 3개 쿼리는 필수입니다.
 1. ```key```: API 키입니다.
 2. ```input```: 검색어입니다. 이름, 주소, 전화번호 등이 될 수 있습니다.
 3. ```inputtype```: 검색어의 타입입니다. ```textquery``` 또는 ```phonenumber``` 둘 중 하나여야 합니다. 전화번호라면 +로 시작하고, 국가번호가 뒤따르는 형식의 전화번호여야 합니다.

여기서부터는 옵션입니다. 자세한 건 문서를 참고하시면 될 듯 합니다. 제가 프로젝트에 사용한 쿼리는 다음 2개입니다.
 1. ```locationbias```: 특정 위치를 중심으로 검색하게 합니다. 이 프로젝트는 삿포로의 맛집을 조사하는 것인지라, 삿포로의 위경도를 value로 넘겨주었습니다.
 2. ```fields```: **얘가 핵심입니다**. 이 쿼리에 어떤 값을 넣느냐에 따라 **비용이 달라집니다**. 더 자세한 건 문서를 참고하시는 게 좋을 것 같습니다. 이 프로젝트에서는 다음의 항목들을 요청하였습니다.
     - ```name```: 장소의 이름입니다.
     - ```placeid```: 구글이 부여하는 그 장소의 고유 ID입니다. **Places Detail** API를 사용할 때 필요합니다.

최종적으로 요청하는 URL은 다음과 같은 형태가 되었습니다.
> ```https://maps.googleapis.com/maps/api/place/findplacefromtext/json?key=aaa&input=マジック バー TRICK&inputtype=textquery&fields=name,place_id&locationbias=point:43.075680,141.349305```

## Google Places Detail
**Places Detail**은 특정 장소의 디테일을 알려주는 API인데, 서버에 요청할 때 ```placeid``` 값을 요구합니다. 문제는 ```placeid``` 값은 미리 구해놓은 게 아닌 이상 **Places Search**를 사용하지 않으면 알 수가 없다는 점입니다. 즉, **Places Search** -> **Places Detail** 이 단계를 밟지 않을 수 없다는 것이지요. ~~제 연구에 비용이 두 배가 들게 되었습니다.~~

사용법 자체는 **Places Search**와 동일하니 제가 프로젝트에 사용한 ```fields```만 소개하겠습니다.
 - ```name```
 - ```place_id```
 - ```user_ratings_total```: 유저가 남긴 평가의 갯수입니다.
 - ```rating```: 유저가 남긴 평점의 평균입니다.
 - ```review```: 유저가 남긴 리뷰들 중 최대 5개까지 보여줍니다.

## 최종 구현
최종 구현은 타베로그 크롤러 구현 때와 크게 다르지 않습니다. 데이터 클래스를 정의하고, API 호출 함수를 구현하고, 이를 멀티프로세싱으로 좀 더 빠르게 하는 것뿐이지요. [여기](https://github.com/helloworldpark/matjip/tree/master/googleplaces)에서 확인하실 수 있습니다.

# 요금폭탄 조심하세요
사실 이게 진짜 중요한 건데요... **Google Places API**가 생각보다 비쌉니다. 요금은 [여기](https://developers.google.com/places/web-service/usage-and-billing?hl=ko)에서 확인하시면 되는데, 요약해드리면
 1. **Places Search - Text Search**: 1000건당 32달러
     - ```fields``` 중 Contact Data에 해당하는 필드 요청 시: 1000건당 3달러 추가
     - ```fields``` 중 Atmosphere Data에 해당하는 필드 요청 시: 1000건당 5달러 추가
 2. **Places Detatil**: 1000건당 17달러
     - ```fields``` 중 Contact Data에 해당하는 필드 요청 시: 1000건당 3달러 추가
     - ```fields``` 중 Atmosphere Data에 해당하는 필드 요청 시: 1000건당 5달러 추가
다행히도 첫 사용자에게는 200달러짜리 쿠폰을 주기는 합니다. 그래서 200달러어치는 무료로 사용해볼 수 있는데요, 제가 개발하면서 아무 생각없이 API를 마구 호출하는 바람에 230달러가 청구되었습니다. 200달러어치 쿠폰이 있기에 망정이지 그게 없었으면 24만원이 청구될 뻔했습니다. 12,153건이나 호출했으니 그 정도 나올 수밖에요. 여러분들은 유료 API 사용하실 때 현명하게 잘 사용하셔야겠습니다.
<p align="center">
<img src="/images/2019-03-30-07.png"><br>
</p>
