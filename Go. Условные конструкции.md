
В гошке как и в си-подобных языках похохий синтаксис условий:
```go
var a int = 10

if a == 9 {
	fmt.Println("Hello world")
} else if {
	fmt.Println("rakamakafo")
} else {
	fmt.Println("umvacheseeeey")
}
```

Единственное отличие - здесь не нужны скобки вокруг условия а также блоки else и else if обязательно должны располагаться не на следующей строчке после блока кода (`{}`) а прям на этой же строчке после завершающей фигурной скобки `}`

Также можно использовать вложенные конструкции:

```go
if pet == "cat" {
	if age < 10 {
		fmt.Println("Кот")
	} else {
		fmt.Println("Тоже кот)")
	}
} else {
	fmt.Println("Hello world")
}
```

[[Go]] [[IT]]