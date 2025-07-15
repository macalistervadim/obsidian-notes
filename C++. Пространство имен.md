Пространство имен позволяет нам использовать наши заголовки без необходимости писать их напрямую, как например в питоне: 
```python
import datetime 

print(datetime.datetime)
```

А вот с использованием некого "пространства имен":
```python
from datetime import datetime

print(datetime)
```

Примерно аналогичным образом все работает и в cpp:
```cpp
#include <iostream>
using namespace std;

def main() {
	cout << "Мы используем пространство имен, обращаясь напрямую к cout"

	cout << "Без него нам бы пришлось писать так: std::cout - некрасиво)"
	return 0;
}
```

[[C++]] [[IT]]
