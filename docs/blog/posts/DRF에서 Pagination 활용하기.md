---
categories:
  - python
  - django
  - drf
date: 2025-05-25
draft: false
---
나는 처음 웹 개발을 배울 때, 페이지내이션 구현을 배우며 page num 방식으로 배웠다. 이건  하나의 반환에서 몇 개의 데이터를 보여줄 지 지정하고, 요청에서 몇번째 부분을 가져오는지 지정하는 방식이다. 예를 들어 페이지 크기를 10, 페이지를 2로 지정하면, 11번째 데이터부터 20번째 데이터까지 반환하는 방식이다. 이후 limit-offset 방식이 있다는 것을 들었지만, 어떠한 이유에서인지 잘 사용하지 않게 되었다. 기본적으로는 이 두가지가 큰 차이가 없다고 느꼈지만, page num 방식으로 구현하는게 조금 더 깔끔해 보인다고 느꼈기 때문이다.

그러다 Django, DRF의 공식문서를 살펴보던 중 DRF에서 기본 제공하는 페이지네이션 방식 중 하나로 cursor 페이지네이션 방식을 발견했고, 페이지네이션과 관련하여 조금 더 연구해보았다.

<!-- more -->

## 페이지네이션

페이지네이션은 많은 양의 데이터를 가져와서 보여주는 UI에서 데이터를 잘게 쪼개어 나누어 사용하는 방식이다.

예를 들어 게시판에서 구현을 한다고 하자. 게시판의 수많은 글을 많은 데이터를 한번에 모두 호출하는것은 여러모로 비효율을 초래한다. DB에서 많은 데이터를 가져오는데도 문제가 발생하며, 큰 데이터를 API 반환으로 포함하는 것 또한 좋은 생각은 아니다. 그래서 10개에서 20개 정도의 데이터를 순차적으로 반환하여 데이터를 보여주는 방식을 선택한다.

## page num 페이지네이션과 limit offset 페이지네이션

일반적으로 페이지네이션을 구현할 때 사용하는 두 가지 방식이다. 이 둘의 동작하는 방식은 거의 동일하고, 다만, 때에 따라 사용방식만 조금 달라질 수 있다.

page num으로 동작할 때는 지금 조회할 페이지의 수와 페이지 크기를 받는다. 예를 들어 페이지당 20개씩 1페이지를 조회하면 처음부터 20개의 데이터, 20개씩 2페이지를 조회하면 21번째부터 40번째를 받는 방식이다. limit offet은 순서대로 정렬된 데이터에서 앞에서 부터 제외할 갯수인 offset과 반환할 데이터의 수인 limit으로 구성된다. 그래서 limit 20, offset 0으로 지정하면, 처음부터 20개의 데이터를 반환 받고, limit 20, offset 20으로 지정하면 20개를 제외한 21번째부터 40번째 데이터를 받는다.

이 두방식 모두 SQL 쿼리나 Django queryset은 같은 방식으로 구현된다는 점이다. 예를 들어 `Artice`모델에서 최신순 `-created_at`로 조회하여 가져오는 경우 다음과 같은 Django queryset이 작성된다.

```python
Article.objects.all().order_by("-created_at")[20:40]
```

이러한 방식으로 구현을 하는 경우 offser을 해야할 양이 많아진다면, 즉 뒷 페이지를 조회할수록 성능이 저하될 수 있다.

## cursor 방식으로 페이지네이션 구현하기

cursor 방식은 조금 다르게 구현된다. Django queryset으로 표현하면 다음과 같이 나타낼 수 있다.

```python
Article.objects.filter(created_at__lt="cursor").order_by("-created_at")[:20]
```

특정 cursor 위치를 기반으로 그것보다 더 작은값(또는 필요에 따라서는 큰 값)을 필터 한 뒤 앞에서부터 특정 갯수만 조회하도록 하는 방식이다. 이렇게 되면 뒷페이지로 갈수록 많은 큰 값의 offset을 한 뒤 가져오는것이 아니라, 확실하게 사용하지 않을 데이터를 제외한 값 중 앞에서부터 특정 데이터만 가져오기때문에 더 효율적으로 동작 할 수 있다.

그러나 정렬기준이 더 많아지는 경우 등에서는 이러한 구현이 복잡하거나 성능적으로 부담될 수 있다.

### cursor 방식의 특징

cursor 방식으로 페이지네이션을 구현하면 다양한 장점이 있을 수 있다. 그러나 구현의 복잡성 말고도 몇가지 문제가 남아있다.

그러나, cursor 방식으로 진행하는 경우, 몇가지 문제가 있을 수 있는데, 각 필드의 유일성과 인덱스와 같은 문제이다.
cursor 방식으로 진행하면 보통 유일함을 보장하는 컬럼에 대하여 적용한다. 일반적으로 id, created_at과 같은 값이 있다. 그런데 만약 created_at으로 지정을 했는데, 이 값이 유일하지 않을 수 있다면, cursor를 제작하고 반환값을 만들어내는데 문제가 있을 수 있다. 이 경우, 다른 유일한 값, 특히 primary key와 함께 사용하도록 구현함으로 해결할 수 있다.
또 다른 문제는 인덱스와 관련한 문제이다. cursor로 접급하게 되면, 특정위치에서부터 탐색하게 된다. 예를 들어, created_at을 내림차순(시간 역순)으로 정렬하고, 특정시간 이전에 작성된 것 중에, 10개를 반환한다라는 방식으로 지정한다. 그런데 index가 없다면 정렬하는데에도 전체 탐색을 하는 비용이 발생하고, 특정 시간 이전에 작성된 경우를 찾기 위해서 전체 탐색이 추가로 필요하게 된다. 

### Django에서 cursor방식으로 구현해보기

limit offset 방식으로 구현하는걸 한번 생각해보면, 파라미터로 `?limit=20&offset=20`과 같이 입력받을 수 있다. 그러면 이를 queryset에 바로 대입함으로서 결과물을 받을 수 있다

```python
class Article(models.Model):
    created_at = models.DateField(auto_now_add=True)
```

다음과 같은 Article 모델이 있다고 하고, 필드로 PK 역할을 하는 id와 생성일 created_at이 있다고 하자. 그리고 데이터는 다음과 같이 있다고 하자. (예시를 나타내기 위한 편의를 위해 생성일자를 시간까지 표기하는 대신 날짜만 표기했다)

| id  | created_at |
| --- | ---------- |
| 1   | 2025-05-01 |
| 2   | 2025-05-02 |
| 3   | 2025-05-03 |
| 4   | 2025-05-04 |
| 5   | 2025-05-05 |
| 6   | 2025-05-06 |

그러면 최신순으로 첫 번째 페이지를 가져올 때는 다음과 같은 queryset을 작성할 수 있다. (여기서부터 한 페이지를 3개로 가정한다)

```python
articles = Article.objects.all().order_by("-created_at")[:3]
```

첫 번째 페이지를 가져온 후, 두번째 페이지를 가져올 때 id=3번 게시물부터 시작하여 더 과거에 생성된 게시물들을 가져와야한다. 그런데 위에서 얻은 `articles` queryset을 생각해보면, 2025년 5월 4일자 게시물이 가장 과거의 게시물이였고, 그 다음 게시물을 조회할때는 해당 날자 이전의 게시물부터 조회하도록 시작하여 가져올 수 있다. 따라서 다음과 같은 queryset을 작성할 수 있다.

```python
articles = Article.objects.filter(created_at__le="2025-05-04").order_by("-created_at")[:3]
```

그러면 첫 번째 페이를 조회했을 때, 그 다음 페이지를 조회하기 위해서는 다음과 같은 정보들이 필요하게 된다.

- 첫 번째 페이지의 가장 오래된 생성일 (위의 예시에서 `2025-05-04`)
- 정렬의 방식 (`-created_at`)

그러면 간단하게 첫 째 반황에서 다음 페이지를 알려줄 때, `?cursor=2025-05-04,-created_at`과 같은 방식의 파라미터를 추가함으로서 알려줄 수 있다. (여기서 정렬 방식은 기본정렬이 최신순이라면 별도로 알려줄 필요가 없겠지만, 기본정렬방식이 아니라면 추가로 기록이 필요할 수 있다.)

#### cursor의 중복 문제

만약에 생성일이 다음과 같이 중복이 발생했다고 하자

| id  | created_at |
| --- | ---------- |
| 1   | 2025-05-01 |
| 2   | 2025-05-02 |
| 3   | 2025-05-03 |
| 4   | 2025-05-03 |
| 5   | 2025-05-05 |
| 6   | 2025-05-06 |

그러면 위와 같은 방식으로 조회하면 문제가 된다. 이는 다른 방식의 페이지네이션에서도 문제가 되는데, 이러한 경우 주로 id와 같은 필드를 함께 사용하여 정렬한다.

```
articles = Article.objects.all().order_by("-created_at", "-id")[:3]
```

그러면, 첫 번째 페이지를 반환한 후, 다음페이지를 알려줄 때, `?cursor=2025-05-03,4,-created_at,-id`와 같은 방식으로 알려줄 수 있다.
그런데 이러한 방식으로 cursor에 대한 모든 정보를 반환에 추가하지 않기 위해 cursor을 암호화하여 반환하기도 한다.

### DRF에서 구현된 cursor 페이지네이션

DRF에서는 `pagination.CursorPagination`으로 구현이 되어있다. 상세한 구현은 다음과 같이 되어 있다.

```python
Cursor = namedtuple('Cursor', ['offset', 'reverse', 'position'])
```

DRF에서는 cursor를 offset, reverse, position이라는 세 개의 값을가지고 구현한다. [^1]

```python
def get_next_link(self):
    ...
    cursor = Cursor(offset=offset, reverse=False, position=position)
    return self.encode_cursor(cursor)
```

다음 페이지 링크를 만들 때, 위의 cursor를 생성하여 전달한다. [^2]

```python
def encode_cursor(self, cursor):
    tokens = {}
    if cursor.offset != 0:
        tokens['o'] = str(cursor.offset)
    if cursor.reverse:
        tokens['r'] = '1'
    if cursor.position is not None:
        tokens['p'] = cursor.position

    querystring = parse.urlencode(tokens, doseq=True)
    encoded = b64encode(querystring.encode('ascii')).decode('ascii')
    return replace_query_param(self.base_url, self.cursor_query_param, encoded)
```

마지막으로 cursor를 인코딩하여 링크의 파라미터로 제작하여 API반환에 추가한다.[^3]

```json
{
    "next": "...?cursor=cD0yMDI1LTA1LTAxKzA4JTNBMjglM0E1Ny4wNDc4ODYlMkIwMCUzQTAw",
    ...
}
```

그래서 마지막으로 이에대한 반환으로 cursor 파라미터가 다음와 같이 지정된 반환을 받게된다.
만약 DRF의 이런 기능을 사용하지 않고 구현한다고 하면, 특정위치를 판단할 수 있는 값을 기반으로 위와 같은 cursor를 생성 및 활용할 수 있어야 한다.

## 그렇다면 어떤 방식을 사용해야할까

그렇다면 나의 프로젝트에서는 어떠한 방식을 사용해야할까? 물론 정답은 프로젝트마다 모두 달라지겠지만, 어느정도 경우를 나누어 적합한 방법을 추론해볼 수 있다.

일반적인 경우에는 page num 방식이나, limit offset방식을 적용해도 충분할 것이다. 직관적이고, 구현에도 큰 어려움이 없다.

### 데이터 중복

그러나 위의 두 방식은 데이터의 중복이 발생할 수 있다.

| id  | created_at |
| --- | ---------- |
| 1   | 2025-05-01 |
| 2   | 2025-05-02 |
| 3   | 2025-05-03 |
| 4   | 2025-05-04 |
| 5   | 2025-05-05 |
| 6   | 2025-05-06 |

위의 데이터를 최신순으로 3개씩 조회한다고 하면, 처음에는 `6, 5, 4`번 글을 조회하게 된다. 이 때 새로운 글이 작성되어 id=7, created_at=2025-05-07로 만들어진 상태에서 2페이지를 조회하는 경우, 1페이제는 `7, 6, 5`가 조회되어야 하기 때문에 2페이지에는 `4, 3, 2`번이 조회된다. 그러면 1페이지에서 봤던 5번 글이 2페이지에서도 중복으로 조회된다.

### cursor 방식으로 해결해보기

이러한 경우, cursor방식으로 지정하게 되면, 게시글의 중복없이 목록을 만들어 낼 수 있다. 특히 무한스크롤 방식으로 구현한 경우 목록이 계속 중첩되어 나올 수 있어 조금 더 유용해 보일 수 있다.

그렇다고 하여 cursor방식이 만능인 것은 아니다. cursor를 다루는데 있는 추가적인 비용이 있을 수 있고, 데이터 양이 적은 경우, 단순 limit-offset으로 진행하는게 더 빠를 수 있다.

또한, cursor를 사용하는 경우 특정 페이지부터의 조회가 쉽지 않을 수 있다. DRF의 경우 등에서 cursor는 보통 암호화 한 키를 사용하고, 클라이언트에서 이 암호를 알 수 없다면, 1페이지가 아니라 특정 페이지를 호출하는것이 쉽지 않다는 것이다. 또한 curosr에 정렬, 필터 등이 모두 포함되다보니, 유동적인 페이지 크기를 적용하는것도 쉽지 않다. (유동적인 페이지 크기를 지정하려 한다면 그러한 값이 cursor에 포함되어야 하는데, 이는 추가적인 비용을 필요로 하기 때문에 좋지 않다.)

로컬에서 DRF로 페이지네이션을 테스트 한 경우, 인덱스가 없는 경우에 10만개의 데이터를 시간순 정렬로 1000페이지를 조회하는 경우에 대해 테스트를 해본 결과, 1000페이지를  순서대로가져오는데 page num, limit offset 방싱 모두 6.5초 정도로 유사하게 나온 반면, cursor방식으로는 14.3초가 걸렸다. 반면 인덱스가 추가되면 page num, limit offset방식은 모두 3.1초로 더 빨라졌고, cursor방식은 이보다 더 빠른 2.7초 정도가 걸렸다. 그래서 여러 필드에 대해 정렬을 하는 경우에는 인덱스가 있는지 등을 잘 고려해야한다. (물론 이 경우는 임의의 테스트 중 하나이고, 실제 경과는 다를 수 있다. 그러나 일반적으로 정렬 결과의 앞 쪽 데이터는 유사하지만, 뒷페이지로 갈수록 cursor 방식이 빠른 편이다.)

### 요약

그래서 상황에 따라 적합한 방식으로 적용이 필요하다. 
만약 많은 데이터양때문에 cursor방식을 해야 하는 것이라면, 한번에 조회가능한 데이터의 최대 양을 제한하는 방식도 가능하다. 예를 들어 최대 1년단위로 페이지네이션을 만들어가는 방식도 적용할 수 있다.


[^1]: [DRF pagination.py Cursor](https://github.com/encode/django-rest-framework/blob/985dd732e058644f6e875de76205a392ce4241dd/rest_framework/pagination.py#L130)
[^2]: [DRF pagination.py get_next_link()](https://github.com/encode/django-rest-framework/blob/985dd732e058644f6e875de76205a392ce4241dd/rest_framework/pagination.py#L698-L749)
[^3]: [DRF pagination.py encode_cursor()](https://github.com/encode/django-rest-framework/blob/985dd732e058644f6e875de76205a392ce4241dd/rest_framework/pagination.py#L871-L885)