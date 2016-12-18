---
layout: post
title:  "Github로 블로그 만들기 + LaTeX 적용하기"
date:   2016-12-18 22:15:27 +0900
categories: jekyll update
---

# Github로 블로그 만들기 + LaTeX 적용하기
개발자로 커리어가 시작된 지 이제 1년이 다 되어가는 뉴비이긴 한데, 블로그를 하나 만들어서 내가 공부한 내용을 정리하고 남들과 공유할 수 있으면 좋겠다고 생각했다. 네이버 블로그도 간편해서 좋긴 한데, 코드가 예쁘게 안 나와서 참 마음에 안 들고, 그렇다고 유료 블로그를 쓰기에는 괜히 돈이 아깝고 해서, 소스 관리도 할 겸 Github에다가 블로그를 만들기로 했다(Github 정책상 돈 안 내면 내 소스는 다 까발려져야 하는데, 어차피 근본없는 개발자인 거 스파게티 소스 남들한테 다 까발려져서 죽어라 까이는 게 내 연봉에 훨씬 더 도움된다).

그런데 말입니다, 제가 괜히 또 욕심이 생겨서 한 가지 요구사항이 생겼지 말입니다. 수식 입력이 꼭 잘 됐으면 좋겠다는 작은 바람이 있었다. 거의 가라로 배우긴 했지만 그래도 배운 게 수식 갖고 노는 것들이었고, 지금 하고 있는 일도 수식 갖고 노는 일인지라 수식이 예쁘게 안 나오면 눈에 굉장히 거슬린다. 그리고 나에게 있어 수식의 아름다움은 LaTeX 뿐이다. 입력에 들어가는 수고 따위야 내 손이 고생하는 거고, 결국 나와 나와 남들 눈에 계속 보일 텐데, 가독성 떨어지는 건 죽어도 보기 싫다. 그런데 의외로, Github에서 제공하는 마크다운은 수식 렌더링이 안 된다. 앞으로도 개선할 생각은 없어보이기는 하는데, 개발하는 사람에게 수식이 뭐 그리 중요하겠나. 범용성이 더 핵심인지라 수식입력 같은 사소한 기능은 내가 알아서 확보하면 되는 일이다.

사이트는 다음 네 곳을 참조했다.

- [GitHub의 Pages를 이용하여 개인 사이트 구축하기기](http://blog.saltfactory.net/note/create-personal-web-site-using-with-github-pages.html)
- [GitHub 블로그 빠르게 시작하기!](http://thdev.net/653)
- [지킬로 깃허브에 무료 블로그 만들기](https://nolboo.kim/blog/2013/10/15/free-blog-with-github-jekyll/)
- [Jekyll 기반의 GitHub Pages에 블로그 만들기](https://xho95.github.io/blog/github/jekyll/git/2016/01/11/Make-a-blog-with-Jekyll.html)

나는 몇 번의 삽질 끝에 다음과 같이 만들었다. 삽질은 OS X 10.11 El Capitan에서 했다.

### 리포지토리 생성
Github에서 Repository를 하나 만든다. Repository의 이름은 반드시 **\[내 계정 이름\].github.com** 이어야 한다. 예를 들어, 내 아이디는 helloworldpark이므로, helloworldpark.github.com 이어야 한다. 이 때 귀찮으니 README.md를 자동으로 만들어준다.

### 디렉토리 만들기
이제 로컬로 돌아와서, 적절한 위치에 디렉토리를 하나 만들자. 나는 ~/Documents/githubProjects/에서 터미널을 열었다.

### Jekyll 설치하기
[Jekyll](http://jekyllrb.com/)을 깔아주자. 엘 카피탄 유저들은 ```gem install jekyll -n /usr/local/bin``` 을 쳐주자. ```rm -rf /```로 시스템 날리는 거 막으려고 애플이 도입한 SIP 때문에, 저 ```-n /usr/local/bin```이 있어야 한다고 한다. 만일 Bundler가 없다면 똑같은 방식으로 Bundler도 깔아주자.

### Jekyll로 페이지 만들기
터미널에 ```jekyll new 계정이름.github.com```을 쳐주자. 그러면 폴더가 하나 생기고, Jekyll에 의해 만들어질 웹 페이지의 파일들도 생성된다.

### 오류 해결
터미널에 ```cd 계정이름.github.com``` 후에 ```jekyll serve --watch```를 쳐주자. 그러면 뭔가 에러가 막 뜰 수 있는데, 디펜던시 걸린 패키지가 없다는 뜻이다. 이름 뜨는 거 잘 보고 해당하는 거 깔아주면 된다. 나같은 경우에는 [minima](https://github.com/jekyll/minima)가 없다고 떴었는데, ```gem install minima -n /usr/local/bin```을 쳐주니 해결되었다.

### MathJax 설치하기
만일 LaTeX으로 수식을 입력할 예정이 아니라면, 설치 과정은 여기에서 끝난다. 하지만 나는 LaTeX을 설치할 예정이므로, 여기에서 멈추지 않고 계속 간다. 검색 결과, Github의 마크다운 엔진 자체는 LaTeX을 지원하지 않지만, Jekyll에 [MathJax](http://docs.mathjax.org/en/latest/start.html)를 연동하면 수식을 입력할 수 있다. 들어가서 다음 스크립트를 ```_layouts/post.html``` 의 ```<article>``` 태그 바로 다음에 붙여넣도록 하자.
{% highlight HTML %}
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
{% endhighlight %}

### Minima 수동설치하기
그런데 ```_layouts``` 라는 폴더가 없다! 사실 최신 버전의 Jekyll은 기본 테마로 [Minima](https://github.com/jekyll/minima)를 쓰는데, 이걸 gem으로 관리한다. 수동으로 뭔가 해주려면 이 파일이 있어야 하니, 위의 링크로 들어가서 Zip 파일을 받은 다음에 ```_includes, _layouts, _sass, assets``` 디렉토리를 우리 디렉토리에 옮겨주자.

### 푸시하기
이제부터는 다른 사이트에 나와 있는 것과 같다. Github 리포지토리에 푸시하면 끝난다.

찾아보니 MathJax는 몇 가지 안 되는 기능들이 있기는 하다. 사실 근데 크게 신경쓸 문제는 아니고, 더 중요한 팁은 다음과 같다.

- 수식을 입력할 때에는 ```$$```로 시작하고, ```$$```로 끝내야 한다.
- 마크다운으로 블로그를 쓸 때에는 백슬래시를 두 번 써줘야 할 때가 있다.

LaTeX 문법은 [스택 익스체인지](http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference)의 한 스레드에 잘 정리되어 있어서, 참고하면 좋을 듯 하다.