 >Данная структура данных чаще всего используется для каких-то частоиспользуемых классов, которые постоянно вызываются и хранят данные, к которым нужен быстрый доступ
 
 По сути своей, датаклассы являются стандартными классами, но с некоторыми улучшениями
 
>[!example]
```python
from dataclasses import dataclass

@dataclass(slots=True, frozen=True)
class Coordinates:

    longitude: float
    latitude: float

def get_gps_coordinates() -> Coordinates:
    return Coordinates(longitude=10, latitude=20)
```

[[IT]] [[Python]]