
Они объединяют свойства сущности (в нашем случае — студента), чтобы работать с ней целостно и ничего не терять.

Объявление структуры выглядит так:

```go
type ИмяСтруктуры struct {
    // здесь переменные (поля) структуры
}
```

```go
type Student struct {
    name string
    age  int
}

func main() {
    student1 := Student{name: "vasya", age: 15}
    student2 := Student{name: "petya", age: 20}
    student3 := Student{name: "kolya", age: 25}
    student4 := Student{name: "masha", age: 30}
    student5 := Student{name: "dasha", age: 35}
    student6 := Student{name: "sasha", age: 40}

    //print students 
    fmt.Println(student1)
    fmt.Println(student2)
    fmt.Println(student3)
    fmt.Println(student4)
    fmt.Println(student5)
    fmt.Println(student6)
}
```

>[!info] Очень важно
>Структуры в Go могут быть экспортируемыми и неэкспортируемыми. Если имя структуры начинается с заглавной буквы, она экспортируется и доступна в других пакетах. Если же имя структуры или поля начинается со строчной буквы, она не экспортируется и доступна только внутри своего пакета.

[[Go]] [[IT]]
