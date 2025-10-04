## **Курс по дженерикам в Go**

### 📘 Введение в дженерики

**Цель:** понять, что такое дженерики и зачем они нужны.

- **Проблема без дженериков:**
   В Go до версии 1.18 нельзя было писать обобщённый код — приходилось дублировать функции для разных типов или использовать `interface{}`. Это приводило к потере типовой безопасности и необходимости кастов.
- **Что дают дженерики:**
   Позволяют писать функции, структуры и методы, которые работают с разными типами, сохраняя строгую типизацию.

**Пример простого дженерика:**

```go
package main

import "fmt"

func PrintSlice[T any](s []T) {
	for _, v := range s {
		fmt.Println(v)
	}
}

func main() {
	PrintSlice([]int{1, 2, 3})
	PrintSlice([]string{"a", "b", "c"})
}
```

Здесь:

- `T any` — типовой параметр, где `any` = `interface{}`.
- Мы пишем один код для разных типов.

### 📘 **Синтаксис дженериков**

- Типовые параметры определяются в квадратных скобках:

  ```go
  func FunctionName[T any](param T) { ... }
  ```

- Можно использовать несколько параметров:

  ```go
  func Swap[T any, U any](a T, b U) (U, T) {
      return b, a
  }
  ```

- Ограничения (Constraints) позволяют указать, какие типы можно использовать.

**Пример с ограничением:**

```go
package main

import "fmt"

type Number interface {
	int | int64 | float64
}

func Sum[T Number](a, b T) T {
	return a + b
}

func main() {
	fmt.Println(Sum(1, 2))       // int
	fmt.Println(Sum(1.5, 2.5))   // float64
}
```

------

### 📘 **Ограничения типов**

- Ограничения задают интерфейсы или набор допустимых типов.
- Они позволяют:
  - Ограничить операции (например `+`, `-`).
  - Использовать только определённые типы.

**Пример:**

```go
type Addable interface {
	int | float64 | string
}

func Add[T Addable](a, b T) T {
	return a + b
}
```

------

### 📘 **Дженерики и структуры**

Дженерики работают не только с функциями, но и со структурами.

```go
type Pair[T any] struct {
	First, Second T
}

func main() {
	p := Pair[int]{First: 1, Second: 2}
	fmt.Println(p)
}
```

------

### 📘 **Дженерики и методы**

Методы тоже могут быть дженериками или работать на дженерических типах.

```go
type Container[T any] struct {
	value T
}

func (c Container[T]) Get() T {
	return c.value
}

func main() {
	c := Container[string]{value: "Hello"}
	fmt.Println(c.Get())
}
```

------

### 📘 **Ограничения с методами**

Можно использовать интерфейсы с методами как ограничение.

```go
type Stringer interface {
	String() string
}

func PrintString[T Stringer](val T) {
	fmt.Println(val.String())
}
```

------

### 📘 ** Встроенные дженерические типы**

Go 1.18+ предоставляет встроенные дженерики в стандартной библиотеке, например `constraints.Ordered`.

```go
import "golang.org/x/exp/constraints"

func Min[T constraints.Ordered](a, b T) T {
	if a < b {
		return a
	}
	return b
}
```

------

### 📘 **Примеры реального применения**

- Дженерические коллекции.
- Дженерические утилиты (map, filter, reduce).
- Общие структуры данных (стек, очередь, список).

**Пример функции Filter:**

```go
func Filter[T any](slice []T, cond func(T) bool) []T {
	var result []T
	for _, v := range slice {
		if cond(v) {
			result = append(result, v)
		}
	}
	return result
}
```

------

### 📘 **Ограничения и производительность**

- Дженерики в Go компилируются под каждый конкретный тип, что даёт типовую безопасность без потери производительности.
- Нет "runtime cost" дженериков.