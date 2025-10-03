[TOC]
# Задача из Ozon
Проброска счетчика параметром в горутины, чтобы в них были копии, а не указатели на одно значение счетчика.
```go
wg := sync.WaitGroup{}
wg.Add(5)
for i := 0; i < 5; i++ {
    go func(i int) {
        defer wg.Done() // Надо вызывать вначале, ичане по ходу дела можно нарваться на return к примеру
        fmt.Println(i)
    }(i)
} 

wg.Wait()
```
Нюанс задачи:
* `Wait group` в замыкание захватывается внутрь как указатель, поэтому мы не работаем с копией, если бы мы создавали отдельную функцию, то обязательно надо было бы передавать указатель, иначе бы работали со счетчиками копии.
* А вот в случае со счетчиком мы также работаем с указателем `i` , счетчик быстро пройдет итерации цикла, а горутины запустятся, чуть позже планировщиком и там у всех горутин в `i` по указателю будет лежать последнее записанное значение (5, потому что на `i=4` цикл закончится, в конце цикла будет `i++` но пять уже не пройдет проверку)
Второй вариант более запутанный, сделать локальную копию для захвата значения в замыкании:
```go
wg := sync.WaitGroup{}
wg.Add(5)
for i := 0; i < 5; i++ {
    i := i // В замыкание будет прокидывать локальное значение счетчика
    go func() {
        defer wg.Done()
        fmt.Println(i)
    }()
} 

wg.Wait()
```
# Задача "3 работника"
1. Есть 3 «работника» (`worker`), каждый выполняет какую-то работу случайной длительности (1–4 секунды).
2. Каждый `worker` должен отправить результат в **канал**.
3. Основная горутина должна собрать результаты всех `workers`, но **максимум ждать 3 секунды**.
4. Если кто-то не успел — выводим сообщение `Timeout!`.
5. Используем `WaitGroup`, чтобы дождаться завершения всех `workers`.

## Без контекста
```go
const (
	numWorkers    = 3               // Кол-во рабочих горутин
	workerTimeout = 5 * time.Second // Если какой-то worker будет долго тупить с записью в канал, то выводить предупреждение и ждать дальше.
)

type Worker struct {
	name  string
	value int
}

func (w *Worker) Work(channel chan<- int) {
	workTime := rand.IntN(4) + 1
	fmt.Printf("  %s start work (%d sec)...\n", w.name, workTime)
	time.Sleep(time.Duration(workTime) * time.Second)
	fmt.Printf("  %s write in channel.\n", w.name)
	channel <- w.value
	fmt.Printf("  %s stop work.\n", w.name)
}

func main() {
	workers := []*Worker{
		{name: "Worker #1", value: 1},
		{name: "Worker #2", value: 2},
		{name: "Worker #3", value: 3},
	}
    
    var wg sync.WaitGroup // Не обязательное держать в виде указателя (&sync.WaitGroup{} тоже работает)
	wg.Add(numWorkers)
    
    // Буферизованный канал на количество workers — чтобы они могли закончить
	ch := make(chan int, numWorkers)
    
	for _, worker := range workers {
		go runWork(worker, &wg, ch)
	}

	// Общий таймаут на сбор всех результатов
	timeout := time.After(workerTimeout)
	got := 0
	done := false
    result := 0

	for !done || got < numWorkers {
		fmt.Printf("Waiting value #%d...\n", got+1)
		select {
		case v := <-ch:
			result += v
			got++
		case <-timeout:
			fmt.Println("Timeout! Stop waiting for more results")
			done = true
		}
	}

	// Ждём всех workers
	wg.Wait()
	close(ch) // Теперь можно безопасно читать из канала до конца

	// Читаем всё, что осталось в буфере
	for v := range ch {
		fmt.Println("Late value from buffer:", v)
		result += v
	}

	fmt.Println("Result:", result)
}

func runWork(w *Worker, wg *sync.WaitGroup, ch chan<- int) {
	defer wg.Done()
	w.Work(ch)
}
```
## С контекстом
```go
func() {
	ctx, cancel := context.WithTimeout(context.Background(), workerTimeout)
	defer cancel()

	ch := make(chan int, numWorkers)
	var wg sync.WaitGroup

	workers := []*Worker{
		{name: "Worker #1", value: 1},
		{name: "Worker #2", value: 2},
		{name: "Worker #3", value: 3},
	}

	for _, w := range workers {
		wg.Add(1)
		go func(w *Worker) {
			defer wg.Done()
			w.WorkWithContext(ctx, ch)
		}(w)
	}

	// В отдельной горутине, чтобы цикл после читал канал
	go func() {
		wg.Wait() // Блокирует горутину, закрытие канала не будет, пока все workers не завершаться.
		close(ch) // Когда все workers закончили → закрываем канал (это сообщит v := range ch что можно уже не читать)
	}()

	result := 0
	for v := range ch {
		result += v
	}

	fmt.Println("Result:", result)
}

func (w *Worker) WorkWithContext(ctx context.Context, ch chan<- int) {
	workTime := rand.IntN(4) + 1
	fmt.Printf("  %s start work (%d sec)...\n", w.name, workTime)

	select {
	case <-time.After(time.Duration(workTime) * time.Second): // Рабочий доработал. Сработает, когда timeWork истечет
		fmt.Printf("  %s finished work, writing in channel.\n", w.name)
		// Если сделать без select, и канал никто не читает, тут будет блокировка.
		select {
		case ch <- w.value:
			fmt.Printf("  %s wrote result.\n", w.name)
		case <-ctx.Done():
			// Отменили пока ждал запись
			fmt.Printf("  %s canceled (timeout).\n", w.name)
		}
	case <-ctx.Done(): // Рабочий не успел доработать
		fmt.Printf("  %s canceled before finishing.\n", w.name)
	}
}
```

# Переполнение `int32`

```go
var t1 int32
t1 = math.MaxInt32
fmt.Println(t1) // 2 147 483 647
t1++
fmt.Println(t1) // -2147483648

t2 := math.MinInt32 + 1
fmt.Println(t2) // -2147483647

var t3 int32
t3 = math.MinInt32 + 1
fmt.Println(t3) // -2147483647
}
```



