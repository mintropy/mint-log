---
date:
  created: 2024-10-15T21:39:00
  updated: 2024-10-15T21:39:00
categories: [python]
---

# Python Dynamic Property

어떤 클래스에 동적으로 속성을 추가하는 방법을 알아보자.

``` Py
# Python Dynamic Property

class MyClass:
    def __init__(self):
        self._data = {}

    def __getattr__(self, name):
        return self._data.get(name)

    def __setattr__(self, name, value):
        self._data[name] = value

    def __delattr__(self, name):
        del self._data[name]

obj = MyClass()
obj.name = 'John'
obj.age = 30

print(obj.name)  # John
print(obj.age)   # 30

```


