> Частенько замечал в различных модулях такую интересную штуку, но не предавал ей особого значения

Как оказалось - это очень полезная штука в условиях архитектуры кода
>[!note] Зачем собстна она нужна то?
>Мы используем конструкцию `if __name__ == "__main__"` в том случае, если нам необходимо отделить логику нашего приложения от всех остальных модулей. Тобишь подробнее - у нас есть условный модуль `start.py`, в котором у нас импортируются все наши модули сервиса и отправляются к нам на сервер. Вот пример:
>>[!example] Пример
>
```python
def make_print(your_string: str) -> None:
	print(your_string)

if __name__ == "__main__":
	print("Запускаем шарманку...")
	# Какие то дополнительные действия
```
В данном случае блок кода, написанный в `if __name__ == ..` будет выполнен лишь в том случае, если мы напрямую запускаем данный модуль, в котором есть этот блок кода, в ином случае - этот блок кода выполнен не будет и доступа к нему также не будет

[[Python]] [[IT]]

