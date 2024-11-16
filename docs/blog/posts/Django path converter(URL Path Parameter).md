---
categories:
  - python
  - django
date: 2024-11-16
draft: false
---
백엔드 개발을 할 때, 가장 많이 다루기도 하고 중요한 부분 중 하나가 URL을 다루는 부분일 것이다. 백엔드 내부 구현만큼 고민을 많이하게 되고 중요한 부분이 URL과 관련된 부분이라 생각한다. 이 글에서는 URL을 구성하는 요소 중 경로 파라미터를 다루어 보았다.

<!-- more -->

## 시스템 구성

```python
# models.py
class Todo(models.Model):
    title = models.CharField(max_length=255, null=True, default=None)
    description = models.TextField(null=True, default=None)
    priority = models.IntegerField(default=0)

# views.py
class TodoAPI(APIView):
    def get(self, request):
        todos = Todo.objects.all()
        serializer = TodoSerializer(todos, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = TodoSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors)


class TodoPriorityAPI(APIView):
    def get(self, request, priority):
        todos = Todo.objects.filter(priority=priority)
        serializer = TodoSerializer(todos, many=True)
        return Response(serializer.data)

# urls.py
urlpatterns = [
    path("", TodoAPI.as_view(), name="todo_list"),
    path("priority/<int:priority>/", TodoPriorityAPI.as_view(), name="todo_priority"),
]
```

할일 목록을 위한 Model을 만들고, 할일을 만들고 조회하는 간단한 API를 만들었다. 이번 예시에서는 전체 목록 조회와 할일 우선순위 (`priority`)에 따른 조회를 하는 API를 추가하고, 간단한 테스트를 위한 할일 추가까지 가능하도록 하였다.

할일 관리를 구현하는 과정에서 우선순위를 0이나 양의 정수로 가능하게 하자, 그렇게 되면 자연스럽게 할일 추가 후 전체 조회나 우선순위에 따른 조회가 가능하다.

## 기획 변경

그러던 도중, 기획을 우선순위 없음을 0으로, 우선순위가 높을 수록 양수로, 낮을수록 음수로 지정한다고 가정하자. 

이렇게 변경한 후 우선순위에  동작하는지 확인하기 위해 API요청을 날려보니, 아래와 같이 404 반환을 받았다. API 구성상 조회되는 데이터가 없더라고 200을 반환할 것으로 생각되는 것과 다른 결과가 나온 것이다.

```
Not Found: /todo/priority/-1/
"GET /todo/priority/-1/ HTTP/1.1" 404 2717
```

## Django Path Converters [^1]

`urlpatterns`에 등록할 때 `path("priority/<int:priority>/", TodoPriorityAPI.as_view(), name="todo_priority")`로 등록했다.

여기서 priority값을 `int`로 등록을 했는데, 위에서 시도한 것처럼 음수의 경우에만 동작하지 않는다. 이에 대해서 Django 공식문서를 살펴보았다.

Django에서는 `<int:priority>`와 같이 path parameter를 등록하는 것을 Path converters 라고 부른다. 기본적으로 지원하는 형식은 str, in, slug, uuid, path가 있는데, int의 경우 0이나 양의 정수만 연결되어 View로 정수값으로 전달한다.

그렇다면 -1과 같은 음수를 전달할 수 있는 방법은 없을지 고민이 들었다.

```python
class TodoPriorityAPI(APIView):
    def get(self, request, priority):
        print("priority:", priority)
        todos = Todo.objects.filter(priority=priority)
        serializer = TodoSerializer(todos, many=True)
        return Response(serializer.data)
```

이를 조금 더 확인해보기 위해, view 함수의 가장 위에서 API로 입력된 priority를 출력을 하고 실행을 해보면 아래와 같은 결과를 얻을 수 있다.

```
priority: 1
"GET /todo/priority/1/ HTTP/1.1" 200 180

priority: 0
"GET /todo/priority/0/ HTTP/1.1" 200 180

Not Found: /todo/priority/-1/
"GET /todo/priority/-1/ HTTP/1.1" 404 2717
```

그래서 priority에 따라 조회하기 위해 1, 0, -1을 입력했을 때, 1, 0은 Django 공식 문서처럼 200반환을 정상적으로 하는데 반해 -1을 입력한 경우 API 함수가 실행되기 전 404를 반환하고 있다.

이 결과는 path converter를 등록한 경우, 해당하는 조건에 맞지 않는다면 해당 리소스가 없는것으로 보아 Django가 404를 반환하는 것으로 보인다.

## Custom Path Converter [^2]

이제 문제를 알았으니 해결할 차례이다. Django에서 기본적으로 제공하는 path converter로는 해결되지 않는 상황이다. 그렇다고 항상 `str`으로 지정하여 입력받아 사용하는 것도 적절해 보이지는 않는다.

Django 공식 문서에서 이것을 해결하기 위한 간단한 예시를 알려주며 이를 "Custom Path Converter"라고 칭한다.

```python
from django.urls import path, register_converter

from . import converters, views

class FourDigitYearConverter:
    regex = "[0-9]{4}"

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return "%04d" % value

register_converter(FourDigitYearConverter, "yyyy")

urlpatterns = [
    path("articles/2003/", views.special_case_2003),
    path("articles/<yyyy:year>/", views.year_archive),
    ...,
]
```

공식문서에서 제공한 위의 예시를 해설하면 다음과 같다.

- converter로 등록하기 위한 클래스를 만들고, `regex` 클래스 어트리뷰트와 `to_python(self, valeu)`, `to_url(self, value)` 메서드를 가져야 한다.
- `to_python(self, value)` : value 값을 view로 전달하는 메서드이다. 여기서 `ValueError`를 일으키면 404을 반환한다.
- `to_url(self, value)` : python의 값을 url로 전달한다. `reverse()`를 사용해 url을 연결할 때 활용되는 것으로 보인다.

## Custom Path Converter로 문제 해결하기 [^3]

다시 돌아가서 처음 마주쳤던 문제를 해결해보자.

```python
from django.urls import path, register_converter

from todos.views import TodoAPI, TodoPriorityAPI


class NegativeIntConverter:
    regex = "-?\d+"

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return "%d" % value


register_converter(NegativeIntConverter, "negint")

urlpatterns = [
    path("", TodoAPI.as_view(), name="todo_list"),
    path(
        "priority/<negint:priority>/", TodoPriorityAPI.as_view(), name="todo_priority"
    ),
]
```

Django 공식 문서에서 확인한 바와 같이 converter 클래스를 만들고, 필요한 정보를 입력한다. 이렇게 입력한 후 API 조회를 해보면 200 반환을 받으며 성공함을 볼 수 있다.

```
priority: -1
"GET /todo/priority/-1/ HTTP/1.1" 200 181
```

## 조금 다르게 생각해보기

이전에 마주쳤던 문제를 해결한 기억을 바탕으로 유사한 예시를 만들기 위해 위와 같이 작성했다.

그런데 만약에 할일 관리 서비스를 만들기 위해, 할일 목록에 우선순위, 날짜, 완료 처리 등의 값들을 포함하도록 구현한 경우, 여러 값을 복합적으로 사용해서 질의하는 경우가 있을 것이다. 가령 우선순위는 3이고  내일 까지인 작업 중 완료되지 않은 작업을 조회하는 등의 행위이다.

그러면, 당연하게도 위의 예시처럼 경로 파라미터로 해결하는 방식보다는 쿼리 파라미터로 해결하는 편이 더 좋아보인다. 예를 들어 `toto/?priority=3&due=2024-11-17&done=false`와 같이 구현하는 것이다. 또는 일부 서비스에서는 이러한 조회를 할 때도 POST 메서드를 사용하여 쿼리들을 request body로 활용하는 것도 본적이 있다.



[^1]: https://docs.djangoproject.com/en/5.1/topics/http/urls/#path-converters
[^2]: https://docs.djangoproject.com/en/5.1/topics/http/urls/#registering-custom-path-converters
[^3]: https://stackoverflow.com/questions/48867977/django-2-url-path-matching-negative-value
