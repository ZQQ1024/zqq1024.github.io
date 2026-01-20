---
weight: 1
bookToc: true
title: "语法基本"
---

https://go.dev/play/

只写一些关联性不强的基本用法

### 变量/常量赋值

```golang
package main

import "fmt"

const Pi float64 = 3.14159265358979323846

func main() {

	var a = "Hello" + " World"
	fmt.Println(a)

	// int 零值为 0
	var b int
	fmt.Println(b)

	// bool 零值为 false
	var c bool
	fmt.Println(c)

    // 自行判定变量类型
	d := 5
	fmt.Println(d)

    var e, f int = 1, 2
    fmt.Println(e, f)

    // Pi = 3.4 error
}
```

### 数组/切片/结构体/Map

数组为值类型，长度固定
```golang
package main

import "fmt"

func main() {
	var n [10]int
	var i, j int

	for i = 0; i < 10; i++ {
		n[i] = i + 100
	}

	for j = 0; j < 10; j++ {
		fmt.Printf("Element[%d] = %d\n", j, n[j])
	}

	balance := [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
	for i = 0; i < 5; i++ {
		fmt.Printf("balance[%d] = %f\n", i, balance[i])
	}

	balance2 := [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
	for i = 0; i < 5; i++ {
		fmt.Printf("balance2[%d] = %f\n", i, balance[i])
	}

	//    将索引为 1 和 3 的元素初始化
	balance3 := [5]float32{1: 2.0, 3: 7.0}
	for k = 0; k < 5; k++ {
		fmt.Printf("balance3[%d] = %f\n", k, balance3[k])
	}
}
```

切片是引用视图，长度可变

```golang
package main

import "fmt"

func main() {
	// 不带长度，numbers := make([]int,0,5)
	numbers := []int{0, 1, 2, 3, 4, 5, 6, 7, 8}
	fmt.Println("numbers ==", numbers)
	fmt.Println("numbers[1:4] ==", numbers[1:4])
	fmt.Println("numbers[:3] ==", numbers[:3])
	fmt.Println("numbers[4:] ==", numbers[4:])

	numbers = append(numbers, 9, 10)

	/* 创建切片 numbers1 是之前切片的两倍容量*/
	numbers1 := make([]int, len(numbers), (cap(numbers))*2)
	copy(numbers1, numbers)
	fmt.Println("numbers ==", numbers)
}
```

```golang
package main

import "fmt"

type Books struct {
	title   string
	author  string
	subject string
	book_id int
}

func main() {
	var book Books

	book.title = "Go 语言"
	book.author = "zqq.name"
	book.subject = "Go 语言教程"
	book.book_id = 6495407

	fmt.Printf("Book title : %s\n", book.title)
	fmt.Printf("Book author : %s\n", book.author)
	fmt.Printf("Book subject : %s\n", book.subject)
	fmt.Printf("Book book_id : %d\n", book.book_id)
}
```

```golang
package main

import "fmt"

func main() {
	var siteMap map[string]string
	siteMap = make(map[string]string)

	siteMap["Google"] = "谷歌"
	siteMap["Baidu"] = "百度"
	siteMap["Wiki"] = "维基百科"
    siteMap["Facebook"] = "脸书"

	for site := range siteMap {
		fmt.Println(site, "站点是", siteMap[site])
	}

    delete(siteMap, "Facebook")

	name, ok := siteMap["Facebook"]
	if ok {
		fmt.Println("Facebook 的 站点是", name)
	} else {
		fmt.Println("Facebook 站点不存在")
	}
}
```

### 条件语句

```golang
package main

import "fmt"

func main() {
   var a int = 100;
 
   if a < 20 {
       fmt.Printf("a 小于 20\n" );
   } else {
       fmt.Printf("a 不小于 20\n" );
   }
   fmt.Printf("a 的值为 : %d\n", a);

   var grade string
   var marks int = 90

   switch marks {
      case 90: 
          grade = "A"
          fallthrough // 使用 fallthrough 会强制执行后面的 case 语句           
      case 80: grade = "B"
      case 50,60,70 : grade = "C"
      default: grade = "D"  
   }
   fmt.Printf("你的等级是 %s\n", grade ); // B
   
   var x interface{}
     
   switch i := x.(type) {
      case nil:  
         fmt.Printf(" x 的类型 :%T",i)
      case int:  
         fmt.Printf("x 是 int 型")                      
      case float64:
         fmt.Printf("x 是 float64 型")          
      case func(int) float64:
         fmt.Printf("x 是 func(int) 型")                      
      case bool, string:
         fmt.Printf("x 是 bool 或 string 型" )      
      default:
         fmt.Printf("未知型")    
   }
}
```

### 循环语句

```golang
package main

import "fmt"

func main() {
    for true  {
        fmt.Printf("这是无限循环。\n");
        break
    }

    var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}
    for i, v := range pow {
        fmt.Printf("2**%d = %d\n", i, v)
    }

    for i, c := range "hello" {
        fmt.Printf("index: %d, char: %c\n", i, c)
    }

    map1 := make(map[int]float32)
    map1[1] = 1.0
    map1[2] = 2.0
    map1[3] = 3.0
    map1[4] = 4.0

    for key, value := range map1 {
        fmt.Printf("key is: %d - value is: %f\n", key, value)
    }

    for key := range map1 {
        fmt.Printf("key is: %d\n", key)
    }

    for _, value := range map1 {
        fmt.Printf("value is: %f\n", value)
    }

    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    close(ch)

    for v := range ch {
        fmt.Println(v)
    }
}
```

### 其他

略