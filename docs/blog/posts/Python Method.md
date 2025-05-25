---
categories:
  - python
date: 2025-05-25
draft: true
---
Python 클래스에서 메서드는 객체 지향 프로그래밍의 핵심 개념 중 하나로, 클래스와 연관된 객체의 동작을 규정하기 위한 함수들이다. 그 중 유사해보여 궁금했던 instance method, static method, class method에 대해 조금 더 고민해 보았다.

이 글에서는 dunder method 등 모든 클래스의 모든 메서드에 대해 다루기 보단, 일반적으로 메서드라 부르는 instance method와 데코레이터로 장식하는 특수한 메서드인 static method, class method에 대해서만 집중해보았다.

<!-- more -->

## 1. 인스턴스 메서드(instance method)

인스턴스 메서드는 클래스 인스턴스에 대해 작동하는 메서드이다. 일반적으로 클래스의 "메서드"라고 하면 이 인스턴스 메서드를 부르는 경우가 많다.

```python
class Pizza:
    def __init__(self, name):
        self.name = name

    def cook(self):
        print("피자를 굽습니다")


pizza = Pizza()
pizza.cook()  # 피자를 굽습니다.
```

인스턴스 메서드는 인스턴스의 상태를 읽거나, 변경할 때 등 다양하게 사용된다.

## 2. 스태틱 메서드(정적 메서드, static method)

```python
class Pizza:
    def __init__(self, name):
        self.name = name

    @staticmethod
    def cook(self):
        print("피자를 굽습니다")


pizza = Pizza()
pizza.cook()  # 피자를 굽습니다.
```

정적 메서드는 클래스, 인스턴스와 관계없이 동작하는 메서드로 특정 상태에 의존하지 않는 기능에 주로 사용된다.
동작 자체는 인스턴스 메서드와 동일하다고 볼 수 있지만, 

## 3. 클래스 메서드(class method)

```python
class Pizza:
    menu = []

    @classmethod
    def menu(cls):
        return cls.menu
```

클래스 메서드는 클래스 외부에서 호출할 때 사용하여, 클래스 레벨에서 상태를 조작하거나 팩토리 메서드 패턴 구현 등에 사용

## 4. 메서드를 상송하여 다양하게 사용해보기


