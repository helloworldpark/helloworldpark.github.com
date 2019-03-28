---
layout: post
title:  "타베로그를 크롤링하기"
date:   2019-03-28 09:00:00 +0900
categories: jekyll update
---

# 조만간 삿포로에 가는데요
제가 곧 삿포로에 갑니다. 하루 정도 묵을 예정인데, 기왕이면 맛있는 걸 먹고 싶습니다. 그래서 [타베로그](https://tabelog.com)에서 무엇을 먹을지 찾아봅니다. 어이쿠, 죄다 평점들이 3점대입니다. 우리나라에서 평점 3점대이면 그저 그런 집들인데, 여기에선 4점대를 찾아보기 힘듭니다. 과연 3점대의 음식점들은 걸러야 할까요?

제 경험에 의하면 타베로그의 음식점들의 평점은 거의 대부분이 3점대입니다. 3점대 후반이면 정말 엄청난 맛집이고, 4점대 음식점들은 미슐랭 스타급의 음식점들인 경우가 많았습니다. 결국 집단의 평균 문제라고 생각하는데, 타베로그에 평점을 남기는 사람들이 약간 보수적으로 점수를 매기는 경향이 있는 것 같습니다.

이 문제를 확인해보기 위해서는 결국 데이터를 수집해서 확인해보는 수밖에 없어보였습니다. 그런데 안타깝게도 타베로그에서는 자사 정보 제공을 위한 API를 더 이상은 제공하지 않습니다. 그래서 어쩔 수 없이 크롤러를 만들어서 직접 정보를 수집해야 할 듯 합니다.

# Python으로 크롤러를 만들어봅시다
이런 크롤러 제작에는 역시 Python이 제격입니다. 문법도 간단하고, 속도도 적당히 빠르고, 무엇보다 크롤러 제작(뿐만 아니라 다른 어떤 목적이라도)에 필요한 오픈소스 라이브러리도 풍부해서 손쉽게 구현할 수 있습니다. 이미 많은 분들이 Python으로 크롤러를 구현하는 방법에 대해 기술해주셔서, 정보를 구하기도 쉬웠습니다. 저는 여기에서 코딩 관련한 팁들 정도만 첨가하는 식으로 기술하도록 하겠습니다. 전체 코드는 [여기](https://github.com/helloworldpark/matjip/tree/master/tabelog)에서 확인하실 수 있습니다.

## requests로 HTTP 요청 날리기
[requests](http://docs.python-requests.org/en/master/)는 Python에서 매우 유명한 HTTP 라이브러리로, 사용법이 편리하고 직관적입니다. 제공하는 기능도 파고들면 다양해서 활용하기에도 좋습니다. 다만, 크롤러를 구현할 때에는 **GET**요청만 날릴 것이기 때문에, ```requests```의 다양한 기능을 활용하지 못하는 것이 오히려 아쉽습니다.

HTTP 요청은 언제나 예외가 발생할 수 있고, 크롤러가 돌다가 예외가 생기면 크롤러를 처음부터 다시 돌려야 하는 불상사가 생길 수 있기에 예외를 **씹는** 코드를 구현하였습니다.

```python
import traceback
import sys
import requests

def handle_error(f):
    """
    예외를 출력하고 더 이상의 예외 전파는 막는다.
    :param f: 인자 없는 함수.
    :type f: Callable[[]]
    """
    try:
        f()
    except Exception:
        print(traceback.format_exc(), file=sys.stderr)

def get_html(url):
    """
    URL로부터 HTML을 string의 형태로 받는다. 만일 HTML을 받지 못했다면 ok는 False이다.
    :param url: HTML을 요청할 URL
    :type url: str
    :return: HTML, ok
    :rtype: str, bool
    """
    ok = False
    body = ''

    def request():
        res = requests.get(url=url)
        nonlocal body
        nonlocal ok
        ok = res.ok
        body = res.text

    handle_error(request)

    return body, ok
```

## BeautifulSoup으로 쉽게 파싱하기
[BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/)은 Python으로 작성된 꽤나 오래된 HTML 파싱 라이브러리입니다. 제가 한창 대학교에서 공부하던 2014년에도 이거 갖고 친구들이 과제를 하네마네 했으니까요. 사실 그래서 더 최근에 나온 다른 라이브러리를 써볼까 하다가, 그냥 이게 제일 낫다는 평을 듣고 저도 그냥 이걸로 크롤러를 구현했습니다.

```BeautifulSoup```은 오래된 라이브러리인만큼 우리말로 된 튜토리얼도 많습니다. 저는 [여기](https://beomi.github.io/gb-crawling/posts/2017-01-20-HowToMakeWebCrawler.html)에 나와있는 내용을 바탕으로 구현하였습니다. 

### URL을 편하게 뽑아내기
먼저, 크롤링을 위한 URL을 편하게 만들어낼 수 있도록 URL을 만들어내는 클래스를 하나 구현했습니다.

```python
class TabelogURL(URL):

    def __init__(self, ken, city, page):
        """
        :param ken: 일본의 도도부현
        :type ken: str
        :param city: 일본의 도시
        :type city: str
        :param page: 크롤링할 페이지
        :type page: int
        """
        self.ken = ken
        self.city = city
        self.page = page

    def url(self):
        return "https://tabelog.com/{:s}/C1100/rstLst/{:d}/?vs=2&sa={:s}&sk=&lid=hd_search1&vac_net=&svd=20190323&svt=1930&svps=1&hfc=1&sw=".format(
            self.ken, self.page, self.city
        )
```
보면 아시겠지만, 별거 없습니다. 나중에 도시 이름을 넣으면 도도부현(```Tabelog.ken```)을 자동으로 찾아주는 기능만 넣어주면 쓰기 더 편해지겠군요. URL은 보시다시피 2019년 3월 23일 오후 7시 30분에 식사할 수 있는 식당을 찾아주도록 되어 있습니다.

### 필요한 부분만 쏙쏙 찾아내기
웹 페이지 크롤링이라는 건 어쩔 수 없이 귀찮은 작업인 것 같습니다. 결국은 한 번은 웹 페이지를 직접 보고, 어느 부분을 어떻게 파싱해야할지 판단해야 하니까요. 다행히도 크롬에는 요소 검사 기능이 있고, 이 기능을 사용하면 HTML과 CSS 코드를 보면서 원하는 부분에 어떤 규칙성이 있나 확인할 수 있습니다. 심지어 복사도 되니 얼마나 편리합니까.

저에게 필요한 정보는 다음과 같습니다:
 - 점포 이름
 - 평점
 - 평가 갯수
 - 저녁 가격
 - 점심 가격

그래서 이를 저장하기 위한 클래스도 하나 선언해줍니다.
```python
class TabelogInfo(ExcelConvertible):

    def __init__(self, name, rating, reviews, price_night, price_noon):
        """
        :param name: 점포명
        :type name: str
        :param rating: 평점
        :type rating: float
        :param reviews: 평가 갯수
        :type reviews: int
        :param price_night: 저녁시간대 가격대
        :type price_night: float
        :param price_noon: 점심시간대 가격대
        :type price_noon: float
        """
        self.name = name
        self.rating = rating
        self.reviews = reviews
        self.price_night = price_night
        self.price_noon = price_noon

    def column_names(self):
        return ['name', 'rating', 'reviews', 'price_night', 'price_noon']
```
```ExcelConvertible```에 대해서는 뒤에서 설명하겠습니다.

이제 HTML이 주어졌다는 가정 하에, 파서를 구현해보겠습니다. 다음과 같은 순서로 진행됩니다.
1. HTML을 불러옵니다
2. ```BeautifulSoup```을 사용해서 HTML을 파싱합니다
3. 크롬을 통해 확인한 태그 및 CSS 요소를 뽑아냅니다
4. 뽑아낸 내용들은 전부 ```str``` 타입이므로, 이를 적절히 가공합니다
5. 가공한 내용들을 객체로 만들고, 리스트에 넣습니다

```python
def collect_info_sapporo(page):
    """
    삿포로의 맛집 정보를 수집합니다
    :param page: 맛집의 페이지. 1페이지부터 60페이지까지 조회 가능하다.
    :type page: int
    :return: HTTP 요청 성공 여부, 가공한 정보인 TabelogInfo의 리스트
    :rtype: bool, List[TabelogInfo]
    """
    url_sapporo = TabelogURL(ken='hokkaido', city='札幌市', page=page)
    body, ok = get_html(url=url_sapporo.url())
    if not ok:
        return False, []

    # start parsing
    soup = BeautifulSoup(body, features="lxml")
    shops = soup.select('#column-main > ul > li')

    info_list = []
    for shop in shops:
        name_soup = shop.select('div.list-rst__header > div > div > div > a')
        if not name_soup:
            continue

        rate_soup = shop.select('div.list-rst__body > div.list-rst__contents > div.list-rst__rst-data > div.list-rst__rate')[0]

        rating_soup = rate_soup.find_all('p', class_=re.compile("^c-rating"))
        review_soup = rate_soup.select('p.list-rst__rvw-count > a')
        price_night_soup = shop.select('div.list-rst__body > div.list-rst__contents > div.list-rst__rst-data > ul.list-rst__budget > li:nth-child(1) > span.c-rating__val.list-rst__budget-val.cpy-dinner-budget-val')
        price_noon_soup = shop.select('div.list-rst__body > div.list-rst__contents > div.list-rst__rst-data > ul.list-rst__budget > li:nth-child(2) > span.c-rating__val.list-rst__budget-val.cpy-lunch-budget-val')

        name = name_soup[0].text
        rating = rating_soup[0].text if rating_soup else '-1'
        review = review_soup[0].text if review_soup else '0件'
        price_night = price_night_soup[0].text if price_night_soup else '-'
        price_noon = price_noon_soup[0].text if price_noon_soup else '-'

        def reformat_str(s):
            return str(s).lstrip().rstrip()

        name = reformat_str(name)
        try:
            rating = float(reformat_str(rating))
        except:
            rating = -1

        try:
            review = int(reformat_str(review).rstrip("件"))
        except:
            review = 0

        if price_night == '-':
            price_night = -1
        else:
            price_night = reformat_str(price_night)
            price_night = price_night.split(sep='～')
            price_night = price_night[-1].lstrip('￥')
            price_night = price_night.replace(',', '')
            try:
                price_night = int(price_night)
            except:
                price_night = -1

        if price_noon == '-':
            price_noon = -1
        else:
            price_noon = reformat_str(price_noon)
            price_noon = price_noon.split(sep='～')
            price_noon = price_noon[-1].lstrip('￥')
            price_noon = price_noon.replace(',', '')
            try:
                price_noon = int(price_noon)
            except:
                price_noon = -1

        info = TabelogInfo(name=name, rating=rating, reviews=review, price_night=price_night, price_noon=price_noon)
        info_list.append(info)

    return True, info_list
```
몇 가지 최적화 요소를 확인할 수 있습니다. 첫째, 공통으로 묶을 수 있는 태그 및 CSS 요소들을 먼저 ```select```하고, 이를 재활용하면 조금 더 빠르게 파싱할 수 있을 것 같습니다. 즉, ```rate_soup```과 ```price_night_soup```, ```price_noon_soup```을 지금과 같이 뽑아내지 말고, 다음과 같이 뽑아냈다면 조금 더 좋은 코드가 되었을 것 같습니다.
```python
common_soup = shop.select('div.list-rst__body > div.list-rst__contents > div.list-rst__rst-data')
rate_soup = common_soup.select('div.list-rst__rate')
...
price_night_soup = common_soup.select('ul.list-rst__budget > li:nth-child(1) > span.c-rating__val.list-rst__budget-val.cpy-dinner-budget-val')
price_noon_soup = common_soup.select('ul.list-rst__budget > li:nth-child(2) > span.c-rating__val.list-rst__budget-val.cpy-lunch-budget-val')
...
```
둘째, ```BeautifulSoup```의 인자 중 하나인 ```features```에 기본적으로는 ```html.parser```를 제공하는데, Python의 기본 ```html.parser``` 대신 [lxml](https://lxml.de/)을 사용하면 조금 더 빠릅니다. 단점이라면, 이를 별도로 설치해줘야 한다는 것 정도가 있습니다만, macOS라면 설치가 그다지 어렵지 않으니 속도가 중요한 분들이라면 충분히 시도해볼 가치가 있다고 생각합니다.

## 이제 직접 돌려보자!
이제 열심히 구현한 함수를 호출만 해주면 될 것 같습니다! 대충 100페이지 정도만 크롤링해보도록 하겠습니다.

자 이제 이렇게 하시면 큰일납니다.
```python
restaurants = []
for page in range(1, 101):
    _, infos = collect_info_sapporo(page=page)
    restaurants += infos
```
왜 큰일날 수 있는가 하면, 자칫하다간 타베로그에서 IP에 밴을 먹일 수 있기 때문입니다. 크롤링과 DDoS 공격은 그 원리가 똑같습니다. 선량한 의도로 크롤링을 했을 뿐인데 해커로 오인당하면 억울하지 않겠습니까(제 친구들도 종종 저지르는 실수입니다).
그래서 한 페이지를 크롤링하고 나서 컴퓨터에게 적당한 휴식을 주는 것이 필요합니다.

```python
import time
restaurants = []
for page in range(1, 101):
    _, infos = collect_info_sapporo(page=page)
    restaurants += infos
    time.sleep(0.5)
```
위 코드처럼 스레드를 0.5초 정도 잠재우고 다시 크롤러를 돌리면 어지간해서는 문제는 생기지 않는 편입니다(사실 0.5초도 좀 길다고 생각합니다).

## Pandas로 Excel 파일에 저장하기
이제 수집한 자료를 저장해야 합니다. 기껏 열심히 크롤링했는데 파일로 기록해두지 않으면 크롤링에 소비한 시간이 너무 아깝지 않겠습니까. 이런 데이터는 XLSX 형식으로 저장해두면 편리할 것 같습니다. 엑셀이라는 마이크로소프트의 실수를 활용할 수도 있고, [R](https://www.r-project.org/)같은 통계 분석 패키지에서도 쉽게 읽어들일 수 있으니까요.

### Pandas Dataframe으로 변환하기
엑셀로 저장하는 것에는 다양한 방법이 있습니다만, 저는 [pandas](https://pandas.pydata.org/pandas-docs/stable/index.html)라는 Python의 데이터 분석 라이브러리를 활용하였습니다. 어차피 ```pandas```로 통계 분석을 할 것이기도 하고, ```pandas.DataFrame```으로 변환해두면 엑셀 파일로 저장하기도 편하거든요.

그런데, 제가 선언한 데이터 클래스인 ```TabelogInfo```를 그대로 활용하고 싶습니다. 어차피 ```pandas.DataFrame```은 지금은 엑셀 저장용으로밖에 안 쓸 거거든요. 즉, 다음과 같은 함수를 구현하고 싶은 거죠.
1. ```to_excel```을 구현한다
2. 그런데 ```to_excel```에 인자로 들어갈 녀석은 ```pandas.DataFrame```과는 전혀 엮이지 않았으면 좋겠다.

그렇다면, ```TabelogInfo``` 클래스는 ```pandas.DataFrame```과는 전혀 관련이 없으면서도 변환은 가능해야하군요. 이를 위해서라면 OOP의 인터페이스(Interface)가 제격입니다. 그런데 Python에는 인터페이스가 없습니다. 하지만 인터페이스의 핵심은 추상메소드(Abstract Method)만 선언되어있음에 있으니, 그런 클래스 하나만 만들어주면 되겠습니다.
```python
import abc


class ExcelConvertible(metaclass=abc.ABCMeta):

    @abc.abstractmethod
    def column_names(self):
        pass
```
이제 ```ExcelConvertible```을 구현하는 클래스들은 엑셀로 저장하고 싶은 멤버 변수들을 ```ExcelConvertible.column_names()```의 리턴 값에 넣어주면 됩니다. 위에서 이미 ```TabelogInfo``` 클래스는 ```ExcelConvertible```을 구현하고 있습니다.

이제 본격적으로 엑셀로 저장할 때가 되었습니다.
```python
def to_excel(convertible, filename):
    """
    엑셀로 변환할 수 있는 객체들을 엑셀 파일로 저장합니다. 저장 경로는 {working directory}/tmp/{filename}입니다.
    :param convertible: ExcelConvertible을 구현한 클래스의 객체 또는 그 리스트
    :type convertible: Union[ExcelConvertible, Iterable[ExcelConvertible]]
    :param filename: 저장할 파일의 이름.
    :type filename: str
    """
    df_dict = {}
    try:
        for x in iter(convertible):
            for col in x.column_names():
                if col in df_dict:
                    df_dict[col].append(x.__dict__[col])
                else:
                    df_dict[col] = [x.__dict__[col]]
    except TypeError:
        df_dict = {col: [convertible.__dict__[col]] for col in convertible.column_names()}

    writer = pd.ExcelWriter(os.path.join('tmp', filename))
    df = pd.DataFrame(data=df_dict)
    df.to_excel(writer, sheet_name='output')
    writer.save()
```
인자로 ```ExcelConvertible``` 또는 ```ExcelConvertible```의 ```Iterable```을 받도록 했습니다. 일단 ```Iterable```이라 가정하고 순회를 하며 ```Dictionary```에 데이터를 집어넣습니다. 만일 ```Iterable```이 아니라면 예외가 발생할 것이고, 그 예외를 캐치해서 마찬가지로 ```Dictionary```에 데이터를 집어넣습니다. ```Dictionary```에 데이터를 ```pandas.DataFrame```이 좋아하는 형태로 집어넣었으니 ```pandas.DataFrame```으로 변환하고, 이를 이용해서 엑셀 파일로 저장합니다.

## 멀티프로세싱으로 더 빠르게 크롤링하기
네트워크 I/O는 생각보다 느린 작업입니다. 그래서 시간이 좀 오래 걸리는 편입니다. 그렇기 때문에 동시 처리에 대한 욕구가 막 샘솟습니다. 다른 언어였다면 멀티스레딩을 고려했을 겁니다. 안타깝게도 여러분의 컴퓨터에 깔려있는 Python이라면 멀티스레딩은 그닥 효율적이지 않을 가능성이 높습니다. 굳이 직접 찾아보지 않는 이상 CPython이 설치될텐데, GIL(Global Interpreter Lock) 때문에 [CPython은 사실상 싱글스레드로 동작하기 때문입니다](https://dgkim5360.tistory.com/entry/understanding-the-global-interpreter-lock-of-cpython).
대안은 멀티프로세싱입니다. 스레드를 여러 개 만들어도 소용이 없다면, 프로세스를 여러 개 만들면 되지 않겠습니까. Python은 기본 라이브러리에서 이를 잘 지원해주고 있습니다. 그래서 Python의 [기본 예제](https://docs.python.org/ko/3/library/multiprocessing.html#multiprocessing-examples)에 나온 ```Queue```를 활용해 멀티프로세싱을 구현했습니다.

```python
from multiprocessing import Process, Queue
from time import sleep
from typing import Callable, Iterable


def __worker(work, time_sleep, task_queue, done_queue):
    """
    :type work: Callable[[object], (bool, object)]
    :type time_sleep: float
    :type task_queue: Queue
    :type done_queue: Queue
    :return: (Result of the work, Success, Task ID)
    """
    for task in iter(task_queue.get, 'STOP'):
        ok, result = work(task)
        done_queue.put((result, ok, task))
        if ok:
            print("OK   {}".format(task))
        else:
            print("FAIL {}".format(task))

        sleep(time_sleep)


def distribute_work(task_generator, func_work, time_sleep, pools=4):
    """
    :type task_generator: Callable[[], List]
    :type func_work: Callable[[object], (bool, object)]
    :type time_sleep: float
    :type pools: int
    :rtype: Iterable
    """
    # https://docs.python.org/ko/3/library/multiprocessing.html#multiprocessing-examples

    # Distribute pages to crawl
    tasks = task_generator()

    # Create queues
    queue_task = Queue()
    queue_done = Queue()

    # Submit tasks
    for task in tasks:
        queue_task.put(task)

    # Start worker process
    for _ in range(pools):
        Process(target=__worker, args=(func_work, time_sleep, queue_task, queue_done)).start()

    print("Started!")

    # Collect unordered results
    result_list = []
    success_once = set()
    failed_once = set()
    while len(success_once) + len(failed_once) != len(tasks) or not queue_task.empty():
        try:
            result, ok, task_id = queue_done.get()
        except:
            continue

        if ok:
            success_once.add(task_id)
            if result is not None:
                result_list.append(result)
        else:
            # Retry once
            if task_id not in failed_once:
                failed_once.add(task_id)
                queue_task.put(task_id)

    # Stop
    for _ in range(pools):
        queue_task.put('STOP')

    # Print failed ones
    for task_fail in failed_once:
        print("Failed: {}".format(task_fail))

    print("Stopped all!")

    return result_list
```
중요한 건 ```distribute_work(task_generator, func_work, time_sleep, pools)``` 함수입니다. 먼저, ```task_generator``` 함수를 이용해 프로세스에게 배분할 일을 만들어냅니다. 그리고, 작업 큐와 결과물 큐를 각각 만들어줍니다. 작업 큐에 작업들을 순차적으로 밀어넣어줍니다. 그리고 지정된 프로세스 갯수만큼 프로세스를 만들어줍니다. 프로세스를 만들 때에는 ```__worker```라는 함수를 ```target```으로 지정하게 되는데(즉, 데몬 프로세스는 ```__worker``` 함수를 실행하게 됩니다), ```__worker```의 인자로 다음이 넘어갑니다:
 1. 실제로 수행하게 될 작업: 아마도 크롤링 작업이 될 겁니다.
 2. 한 번 크롤링한 후에 스레드를 재우는 시간: 위에서 설명한 바 있습니다.
 3. 작업 큐와 결과물 큐: 메인 프로세스와 데몬 프로세스 간 데이터를 주고받는 통로가 됩니다.

프로세스가 돌기 시작하면, 작업 순서는 랜덤입니다. 즉, 어떤 페이지가 먼저 크롤링될지는 알 수 없습니다. 그렇더라도 상관없습니다. 어차피 순서는 중요하지 않으니까요. 동시성 문제도 걱정할 필요가 없는 것이 어차피 동시성 관련해서 잘 고려되어 설계된 큐에 들어가기 때문에 데이터가 꼬일 일도 없습니다. 

다만, 예제와 다른 점이 있다면 크롤링 실패 상황에 대해서 고려가 되어 있다는 것입니다. HTTP 통신은 본질적으로 불안하기 때문에 항상 성공한다는 보장이 없습니다. 페이지가 존재하지 않을 수도 있고, 서버가 갑자기 죽을 수도 있고, 우리집 인터넷 회선이 끊어져버릴 수도 있습니다. 이런 상황들에 모두 세심하게 대응한다면 정말 좋은 프로그램이 되겠지만, 제 생각엔 그 정도의 정성을 들일 프로젝트는 아닌 듯합니다. 그래서 어떤 상황이던지 1번 실패하면 1번 더 시도해보는 것으로 구현했습니다.

## 최종 정리
이제 ```main```함수는 다음과 같은 형태가 되었습니다.
```python
import time

def collect_info_sapporo_all(total_pages, pools=4):
    def task_generator():
        return range(1, total_pages+1)

    # Distribute crawling tasks
    tabelog_info_list = distribute_work(task_generator=task_generator,
                                        func_work=collect_info_sapporo,
                                        time_sleep=0.5,
                                        pools=pools)
    # Merge nested lists
    tabelog_info_list = [x for sub_list in tabelog_info_list for x in sub_list]
    # Save to excel
    to_excel(convertible=tabelog_info_list, filename='tabelog_sapporo_201903231930.xlsx')
    print("Saved to excel")

if __name__ == '__main__':
    start = time.time()
    parser.collect_info_sapporo_all(pools=4, total_pages=60)
    print("Elapsed: {}".format(time.time() - start))
```
60페이지까지만 크롤링한 이유는, 타베로그에서 61페이지부터는 접근을 막아서입니다. 그 이상의 데이터는 공개하지 않는 모양입니다.
이렇게 크롤링하니 제 컴퓨터(MacBook Pro 13-inch 2017년형)에서는 약 38초 걸렸습니다. 컴퓨터 환경에 따라서는 더 빨리 끝날 수도, 더 늦게 끝날 수도 있습니다.