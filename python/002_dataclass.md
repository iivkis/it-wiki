# Иммутабельные dataclass в Python: безопасные ключи для dict и set

> В Python можно сделать экземпляры класса иммутабельными используя `@dataclass(frozen=True)`. Это гарантирует, что после создания никаких полей изменить нельзя – попытка присвоить новое значение вызовет `FrozenInstanceError`. А поскольку объект неизменен, его можно безопасно использовать в качестве ключа в `dict` (или элемента `set`).

```python
from dataclasses import dataclass, FrozenInstanceError

@dataclass(frozen=True)
class Point:
    x: int
    y: int

p = Point(1, 2)
# p.x = 3  # вызовет ошибку: dataclasses.FrozenInstanceError

d: dict[Point, str] = {}
d[p] = "Начальная точка"
print(d)  # {Point(x=1, y=2): 'Начальная точка'}
```

Иммутабельность открывает путь к более безопасным и предсказуемым структурам данных, особенно когда речь о кэшировании или ключах словарей.