---
categories:
  - python
date: 2024-11-09
draft: false
---
Python에서 클래스를 활용할 때 변수는 다양한 방식으로 활용한다.

클래스에서 간단한 변수 활용방법과 `Property`에 대해 다루어보았다.

<!-- more -->

## 클래스 변수, 인스턴스 변수

파이썬에서 클래스를 선언할 때, 클래스 변수와 인스턴스 변수를 선언할 수 있다.

```python
class Point:
    dimension = 2

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

    def __str__(self) -> str:
        return f"({self.x}, {self.y})"


point = Point(1, 2)
print(point)           # (1, 2)
print(point.dimension) # 2
```

위와 같이 `Point` 클래스에 클래스 변수로 `dimension`을, 인스턴스 변수로 `x`, `y`를 선언했다.

## Property [^1]

```python
class Point:
    dimension = 2

    def __init__(self, x: int, y: int) -> None:
        self._x = x
        self._y = y

    def __str__(self) -> str:
        return f"({self._x}, {self._y})"

    @property
    def x(self) -> int:
        return self._x

    @x.setter
    def x(self, x: int) -> None:
        self._x = x

    @property
    def y(self) -> int:
        return self._y

    @y.setter
    def y(self, y: int) -> None:
        self._y = y


point = Point(1, 2)
print(point)           # (1, 2)

print(point.x) # 1
point.x = 10
print(point.x) # 10
```

위의 코드에서 `property`를 사용하면 조금 더 개선할 수 있다.
인스턴스 변수로 선언한 `x`, `y`를 `_x`, `_y`로 변경하고, 대신 객체에서 `x`, `y`로 접근할 수 있는 `property`를 설정한다. 이렇게 설정하면 몇가지 장점이 있다.

1. 변수를 가져오거나 저장할 때 추가적인 검증 등의 코드를 추가할 수 있다.
2. 만약 `data`라는 인스턴스 변수가 딕셔너리 형태로 있고, 특정 키와 해당하는 값이 중요하게 사용될 때, `data`를 가져온 다음 별도로 키를 가져오는 것이 아니라 프로퍼티로 설정하여 빠르게 접근할 수 있도록 가능하다.

## 동적으로 프로퍼티 추가하기

```python
class Point:
    dimension = 2

    def __init__(self, x: int, y: int) -> None:
        self._x = x
        self._y = y

    def __str__(self) -> str:
        return f"({self._x}, {self._y})"

    @property
    def x(self) -> int:
        return self._x

    @x.setter
    def x(self, x: int) -> None:
        self._x = x

    @property
    def y(self) -> int:
        return self._y

    @y.setter
    def y(self, y: int) -> None:
        self._y = y


setattr(Point, "z", 0)
point = Point(1, 2)
print(point.z) # 0

setattr(Point, "distance", property(lambda self: (self.x**2 + self.y**2) ** 0.5))
print(point.distance) # 2.23606797749979
```

만약 `Point`클래스에서 선언하지 못한 추가적인 변수를 필요로 할 때, `setattr()` 함수를 사용해서 지정할 수 있다.
특정 값이 아니라 `property`로 생성하면서, lambda를 지정하면 원래의 클래스 변수나 인스턴스 변수를 활용한 값을 반환할 수 있다.

상황에 따라서 위와같이 활용할 필요가 있을 수 있지만, 일반적으로는 클래스 내부에서 선언하는편이 더 좋아보인다. 위와 같이 선언하게 되면 더 짧은 코드를 보고 더 많은 내용을 이해해야 한다. 예를 들어 `z`로 선언한 값은 기존 2차원 `Point`에서 어떠한 의미를 위해 추가했는지, distance는 어떤 의미를 하고 어떤 값을 가지는지와 같은 부분이다. 또한, IDE에 코드 자동완성이 동작하지 않는다. 아마 런타임 중에 코드에 입력되기 때문에, 코드를 실행하기 전에는 정확히 알 수 없어서인 것 같다.


[^1]: https://docs.python.org/ko/3/library/functions.html#property
