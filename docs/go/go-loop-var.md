# Golang 循环变量引用问题

在Go for循环中，循环变量的生命周期被定义为整个循环，而不是每个迭代都有新一份的循环变量，导致了每一轮迭代产生的引用实际上都指向同一个值，而不是指向每一轮各自对应的值。

golang 的循环变量是 per loop 的，而不是 per iteration 的。如果对循环变量产生了引用（比如闭包 capture，或者取指针），不同次迭代取到的指针都是同一个。

如果这个指针/引用被逃逸出了一次迭代的范围内（比如 append 到了一个数组里，或者被go/defer后面的闭包capture了），因为所有 iteration 里取到的指针都是同一个，指向的对象也都会是同一个（最后一轮iteration的结果）。

闭包:
```GO
func main() {
	arr := []int{1, 2, 3, 4, 5}
	for _, v := range arr {
		go func() {
			fmt.Println(v)
		}()
	}
	time.Sleep(3 * time.Second)
}

// prints 5 5 5 5 5
```

隐蔽的指针取指针 implicit reference due to receiver mismatch:
```GO
type MyInt int

func (mi *MyInt) Show() {
	fmt.Println(*mi)
}

func main() {
	ms := []MyInt{1, 2, 3, 4, 5}
	for _, m := range ms {
		go m.Show()
		// implicitly converted to `go (&m).Show()`
		// thus creating a reference to loop variable.
		// but you would never know this without more context.
	}
	time.Sleep(100 * time.Millisecond)
}

// prints 5 5 5 5 5
```

直接取指针:
```GO
arr := []int{1,2,3,4,5}
for _, v := range arr {
	arr2 = append(arr2, &v) // all new elements &v are the same.
}

// arr2 == {v_arr, v_arr, v_arr, v_arr, v_arr}
// *v_arr == 5
```

解决方式一个是在函数参数中引用循环变量:
```GO
func main() {
	arr := []int{1, 2, 3, 4, 5}
	for _, v := range arr {
		go func(v int) {
			fmt.Println(v)
		}(v)
	}
	time.Sleep(3 * time.Second)
}
```

另一个方法是重新定义循环变量:
```GO
func main() {
	arr := []int{1, 2, 3, 4, 5}
	for _, v := range arr {
		v := v
		go func() {
			fmt.Println(v)
		}()
	}
	time.Sleep(3 * time.Second)
}
```

另一种很隐蔽的bug:
