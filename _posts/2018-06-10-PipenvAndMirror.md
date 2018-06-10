---
layout: post
title:  "Pipenv에 미러 설정으로 빠르게 환경 구축하기"
date:   2018-06-09 18:00:00 +0900
categories: jekyll update
---

## Pipenv 소개
**Pipenv**는 정말 매력적인 파이썬 환경 구축 툴입니다. 개발 잘 하는 친구를 통해 추천받아서 쓰게 되었는데요, **virtualenv**와 **pip***으로 각각 해줘야 했던 환경 구축이 한결 편리해졌습니다. 사용법도 엄청 간단합니다.
{% highlight bash %}
pipenv --python 3.6 # Python 3.6을 인터프리터로
pipenv install # 기본적인 환경을 만들어줘요
pipenv --help # 잘 모르겠으면 헬프
{% endhighlight %}

## Pipfile 쓰기
어떤 프로젝트를 구현할 때, 보통은 여러 개의 오픈 소스들을 섞어쓰게 마련입니다. 물론 밑바닥부터 다 치고 올리시는 분들도 계시지만, 그렇게 하는 건 여러모로 추천되지 않는 바입니다. **Pipenv**를 쓰면 이런 의존성 관리도 매우 쉽게 할 수 있습니다. **Pipfile**만 잘 써주면, 우리가 원하는 패키지를 손쉽게 가상환경에 설치할 수 있죠.
```toml
[[source]]
url = "https://pypi.python.org/simple"
verify_ssl = true
name = "pypi"

[packages]
flask = {version="==1.0"}
numpy = {version="==1.14.4"}
pandas = {version="==0.23.0"}
requests = {version="==2.18.4"}

[dev-packages]

[requires]
python_version = "3.6"
```

**Pipfile**을 이렇게 써준 후에, ```pipenv install```을 돌려주면 끝입니다. 문제는, 한국에서는 이렇게 하면 **Pipfile.lock** 파일을 생성하는 데에 너무 오래 걸립니다.

## 미러를 사용하여 Pipfile.lock 생성을 빠르게 하기
정확한 원인은 잘 모르겠지만, 제가 몇 번 경험해본 결과, **PyPI**에서 패키지를 다운로드하는 게 굉장히 오래 걸립니다. 아마 한국과 **PyPI** 서버 사이에 물리적인 거리가 제법 되어서 그런 것이지 않을까 싶습니다. 그러다보니 **Pipfile.lock**을 생성하는 데에도 시간이 엄청 오래 걸려서, 처음에는 시스템이 맛이 갔거나 데드락이 걸린 줄 알았습니다.

이 현상은 한국 내의 미러를 사용하니 해결되었습니다. 미러의 사용법은 [여기](https://docs.pipenv.org/advanced/)에서 찾았습니다. 여기에는 두 가지 해결책을 제시하는데요

1. 패키지별로 일일이 지정해준다
2. ```pipenv install --pypi-mirror <mirror_url>```로 해준다

저는 1번 방법을 사용하여 해결했습니다. 즉, 위의 **Pipfile**을 아래와 같이 수정했습니다.

```toml
[[source]]
url = "http://mirror.kakao.com/pypi/simple/"
verify_ssl = false
name = "kakao"

[packages]
flask = {version="==1.0", index="kakao"}
numpy = {version="==1.14.4", index="kakao"}
pandas = {version="==0.23.0", index="kakao"}
requests = {version="==2.18.4", index="kakao"}

[dev-packages]

[requires]
python_version = "3.6"
```

서버에 배포해야 할 떼에는 아무래도 2번 방법이 더 낫지 않나 싶습니다만, 개인 프로젝트 등의 간단한 경우에는 1번 방법도 충분히 고려해볼 만하지 않나 싶습니다.