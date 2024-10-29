---
title: "Go开发中的测试"
date: 2024-10-27T15:00:00+08:00
tags: [Golang]
toc: true
---

近期接手新的 Go 项目，阅读代码的过程中，遇到了一些涉及测试的问题，所以重温了 Go 测试相关的一些内容，同时还查阅了一些资料，在这里一并记录，分享给或许面临同样问题的你。对于 Go 开发的老司机们，也是一个知识回顾的好机会。

问题具体背景是，当前我们正在推广云环境办公，所以研发同学都是用的华为云桌面，OS 是 Windows 10，为了环境兼容性和符合之前的研发习惯，团队总结出一套 Windows + WSL + VSCode/Goland + WSL Plugin 的开发方式，多数情况下这套环境是能正常工作的，但是在需要使用 delve 进行 Go 代码调试时，IDE 就会报错。后经查证（在 VSCode 上）是 delve 出于安全考虑禁止调试服务的 client 端 和 server 端属于不同用户，如果要解决这个问题，一是升级WSL 2，一是尝试直接启动一个 VMWare 之类的虚拟机做开发。但是二者都依赖于 Windows 10 能开启虚拟化功能。但是华为云桌面并不能支持二次虚拟化，直接否定了这两个解决方案。

剩下的方法是在开发过程中尽量使用 Debug Log 和 Unit Test 进行功能的验证和测试，至少组内的同学是这么跟我说的。这也是我在项目中发现了大量的 _test.go 文件和测试用例的原因（写测试是个好习惯！）。就当我暗自庆幸前人们留下这么多测试代码时，却又发现测试代码中存在各种异常：循环依赖、测试用例中依赖功能直接报错、测试代码能跑通但是依赖功能不能正常返回数据导致结果和期望不一致...如此种种，不一而足。

为了解决这些算不上棘手问题，我开始了回忆、查阅、重温、搜索、学习的过程。下图是我整理出的内容大纲，由于不是专业科普测试知识，这里不会按照惯例讲单元测试、集成测试、系统测试、UI测试、验收测试...，我们主要关注的是 Go 开发中经常用的一些内容，所以内容主要集中于 Unit Test 和 Mock 两点。

{{< figureCupper
img="outline.jpg"
caption="本文内容大纲"
command="Resize"
options="1080x" >}}

## 示例代码

文中示例代码可以在 [这里](https://github.com/op-y/mytest) 获取。

## Unit Test

### testing 包

对于 Go 开发者来说，单元测试不会陌生，几乎所有入门资料都会介绍 Go 官方提供了一个 [testing](https://pkg.go.dev/testing) 包，使用这个包能非常方便的在自己的代码中编写单元测试代码并通过 **go test** 命令运行单元测试（当然也可以使用 IDE 来运行，本质上也是执行 go test）。

#### 基础测试

如官方文档描述，编写一个基本的单元测试非常简单。在任何一个包内，创建一个如 **xxx_test.go** 这样以 _test.go 结尾的文件，文件中 **import testing** 包，然后在文件中编写 **func TestXXX(t *testing.T) {...}** 这样，名称以 Test 开头接收一个 *testing.T 对象的参数，且没有返回值的函数。这样 go test 就能识别该文件是测试代码文件（正常编译时会忽略该文件），文件中 TestXXX 函数时测试函数。

以示例仓库中代码为例

{{< code >}}
package math

import (
	"github.com/op-y/mytest/function"
)

func Add(a, b int) int {
	return a + b
}

func Sub(a, b int) int {
	return a - b
}

func Mul(a, b int) int {
	return a * b
}

func Div(a, b int) int {
	return a / b
}

func Rand10() int {
	// 返回10以内的整数
	return function.RandInt(10)
}
{{< /code >}}

math.go 中是一些简单的业务代码，一眼就能看出这里就是简单的整数加减乘除，暂时先忽略这里引入的 function 包，为了测试Add函数的功能，我在 math 包中创建一个 math_test.go 文件并编写相应TestAdd函数如下。

{{< code >}}
package math

import (
	"testing"
)

func TestAdd(t *testing.T) {
	a, b, expected := 1, 2, 3
	result := Add(a, b)
	if result != expected {
		t.Fatalf("%d add %d eq %d, but expect %d", a, b, result, expected)
	}
	t.Log("test pass")
}
{{< /code >}}

这样一个符合 go test 要求的测试用例就写好了。然后进入math目录，运行相应的命令执行该测试用例。

{{< cmd >}}
cd math
go test -v -count=1 -run=^TestAdd$ ./.
{{< /cmd >}}

{{< note >}}
想要了解更多 go test 选项和参数请参考 Go 语言官方文档。
{{< /note >}}

测试结果输出如下。可以看出这个测试用例顺利通过了。

{{< code >}}
=== RUN   TestAdd
    math_test.go:13: test pass
--- PASS: TestAdd (0.00s)
PASS
ok      github.com/op-y/mytest/math     0.671s
{{< /code >}}

现在一个最基础的单元测试就完成了。继续往下，是我在项目中遇到的第一个问题。

#### 包外测试

这个问题的报错非常简单粗暴: **import cycle not allowed in test** ！

顾名思义，代码中出现了循环依赖。在项目中这样的报错还不少😂。为了方便解释，这里模拟了一下这个场景。我先创建一个业务包名为 function，在包中创建一个 function.go 文件，里边有 RandInt 和 Fibonacci 两个函数，分别用于生成随机数和获取**斐波那切数列**中的第 N 项，然后再创建一个 function_test.go 测试文件，在文件中编写一个 TestFibonacci 测试函数。

{{< code >}}
package function

import (
	"math/rand"
)

func RandInt(upper int) int {
	// 返回upper以内随机整数
	return rand.Intn(upper)
}

func Fibonacci(num int) int {
	if num == 0 || num == 1 {
		return 1
	}
	return Fibonacci(num-1) + Fibonacci(num-2)
}
{{< /code >}}

{{< code >}}
package function

import (
	"testing"

	"github.com/op-y/mytest/math"
)

func TestFibonacci(t *testing.T) {
	fibs := []int{1, 1, 2, 3, 5, 8, 13, 21, 34, 55}
	idx := math.Rand10()
	expected := fibs[idx]
	result := Fibonacci(idx)
	if result != expected {
		t.Fatalf("the %d fibonacci number is %d, but expect %d", idx, result, expected)
	}
	t.Logf("the %d fibonacci number is %d, test pass", idx, result)
}
{{< /code >}}

在这个简单的场景下，能很直观的看出问题所在。测试函数 TestFibonacci 调用了 math.Rand10 随机生成 10 以内的整数，这非常正常（一个上层功能依赖一个底层的功能），然后回头看 math.go 文件中 Rand10，上一节被我们刻意忽略的对 function 的导入产生影响了，math 中又依赖了 function.RandInt（当然，这里是我有意为之的，项目代码中就是这种类似情况），这样在 math 和 function 之间就产生了循环依赖，而且这种循环依赖是因为测试代码产生的。

那如何解决这种问题呢？其实 Go 官方已经提供了一种方法：包外测试。Go 允许开发者在一个包下的 _test.go 文件中，定义个一个同名的以_test结尾的包用于测试，执行测试时 go test 会认为该包是一个外部包，循环依赖的问题迎刃而解。我们将 function_test.go 改写如下。

{{< code >}}
package function_test

import (
	"testing"

	"github.com/op-y/mytest/function"
	"github.com/op-y/mytest/math"
)

func TestFibonacci(t *testing.T) {
	fibs := []int{1, 1, 2, 3, 5, 8, 13, 21, 34, 55}
	idx := math.Rand10()
	expected := fibs[idx]
	result := function.Fibonacci(idx)
	if result != expected {
		t.Fatalf("the %d fibonacci number is %d, but expect %d", idx, result, expected)
	}
	t.Logf("the %d fibonacci number is %d, test pass", idx, result)
}
{{< /code >}}

此时已经可以正常运行测试。

{{< cmd >}}
go test -v -count=1 -run=^TestFibonacci$ ./.
{{< /cmd>}}

{{< code >}}
=== RUN   TestFibonacci
    function_test.go:18: the 3 fibonacci number is 3, test pass
--- PASS: TestFibonacci (0.00s)
PASS
ok      github.com/op-y/mytest/function 0.538s
{{< /code>}}

这里要注意的是，由于 function_test 已经是一个外部包，所以测试代码中不能直接调用 Fibonacci 函数了，需要 import function 然后以 function.Fibonacci 方式调用。

联想能力强的朋友会注意到一个问题，作为一个外部包，那原来 function 包中定义的一些 **非导出** 变量与函数就无法测试了。确实！一个常见的做法是，再编写一个 _export.go 文件例如 function_export.go ，通过一些中间的函数来获取 function 包中的非导出内容。

如此，遇到的第一个问题就解决了。

在项目中，我还看到了很多测试的不同方式。有些虽然没有被充分使用起来，但有总比没有强！继续往下，我做了一些整理和补充。

#### 测试子任务

测试函数 TestXXX 的另一种使用方法是使用方法是通过 testing.T.Run 方法，在一个测试函数中运行多个测试用例。这样做的用途是什么呢？官方示例的注释告诉我们，可将多个测试集中在一个函数中，然后统一执行一些前置和后置动作。

我们在 math_test.go 增加如下测试函数。

{{< code >}}
func TestSubtests(t *testing.T) {
	// setup code
	a, b := 100, 10
	t.Run("Add", func(t *testing.T) {
		expected := 110
		result := Add(a, b)
		if result != expected {
			t.Fatalf("%d add %d eq %d, but expect %d", a, b, result, expected)
		}
		t.Log("test pass")
	})
	t.Run("Sub", func(t *testing.T) {
		expected := 90
		result := Sub(a, b)
		if result != expected {
			t.Fatalf("%d sub %d eq %d, but expect %d", a, b, result, expected)
		}
		t.Log("test pass")
	})
	t.Run("Mul", func(t *testing.T) {
		expected := 1000
		result := Mul(a, b)
		if result != expected {
			t.Fatalf("%d mul %d eq %d, but expect %d", a, b, result, expected)
		}
		t.Log("test pass")
	})
	t.Run("Div", func(t *testing.T) {
		expected := 10
		result := Div(a, b)
		if result != expected {
			t.Fatalf("%d div %d eq %d, but expect %d", a, b, result, expected)
		}
		t.Log("test pass")
	})
	// tear-down code
}
{{< /code>}}

{{< cmd >}}
go test -timeout 30s -run ^TestSubtests$ github.com/op-y/mytest/math -v -count=1
{{< /cmd>}}

{{< code >}}
=== RUN   TestSubtests
=== RUN   TestSubtests/Add
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/math/math_test.go:25: test pass
=== RUN   TestSubtests/Sub
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/math/math_test.go:33: test pass
=== RUN   TestSubtests/Mul
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/math/math_test.go:41: test pass
=== RUN   TestSubtests/Div
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/math/math_test.go:49: test pass
--- PASS: TestSubtests (0.00s)
    --- PASS: TestSubtests/Add (0.00s)
    --- PASS: TestSubtests/Sub (0.00s)
    --- PASS: TestSubtests/Mul (0.00s)
    --- PASS: TestSubtests/Div (0.00s)
PASS
ok  	github.com/op-y/mytest/math	0.974s
{{< /code >}}

可以看到一个测试函数执行了四个测试子任务。

#### 表驱动法

提及一次执行多个测试任务（测试用例），通常针对一个功能，执行多个不同测试用例，更常用的是表驱动法（Table Driven Test）。它是在一个测试中，将多组测试输入参数和期待输出按照表格的形式组织（数据结构上通常是一个对象数组），然后在测试中遍历这组测试数据，执行测试，对比实际输出和期待输出，以验证功能是否符合预期。

在 math_test.go 中有这么一个测试函数。

{{< code >}}

func TestTableDriven(t *testing.T) {
	var tests = []struct {
		a        int
		b        int
		expected int
	}{
		{1, 2, 3},
		{2, 3, 5},
		{3, 5, 8},
	}
	for _, tt := range tests {
		result := Add(tt.a, tt.b)
		if result != tt.expected {
			t.Errorf("Add(%d, %d): got %d, want %d", tt.a, tt.b, result, tt.expected)
		}
	}
}
{{< /code >}}

这里就将三组测试用例集中在一个表中，然后在测试过程中遍历这些测试用例并执行。运行效果如下。

{{< cmd >}}
go test -v -count=1 -run=^TestTableDriven$ ./.
{{< /cmd >}}

{{< code >}}
=== RUN   TestTableDriven
--- PASS: TestTableDriven (0.00s)
PASS
ok      github.com/op-y/mytest/math     0.695s
{{< /code >}}

#### 测试钩子

前面介绍测试子任务时，提到它的一个作用是可以将测试集中，然后统一运行一些前置和后置逻辑。针对这种场景，其实 testing 包中还提供了一个 testing.M 对象和 TestMain 方法。只需要在 _test.go 文件中加入一个 **TestMain(m *testing.M)** 函数，并在函数中调用 m.Run() 方法，go test 在执行单个 TestXXX 方法时的逻辑就会发生变化，它将通过调用 TestMain 函数来执行该 TestXXX。如此一来我们就可以在 m.Run() 执行前后加上自己的处理逻辑了。

我们在 math_test.go 中追加 TestMain 函数。

{{< code >}}
func TestMain(m *testing.M) {
	fmt.Println("有些场景需要在测试用例运行之前做一些初始化工作如: 读取配置、建立数据库连接...")
	m.Run()
	fmt.Println("有些场景需要在测试用例运行之后做一些清理工作如: 关闭文件、关闭数据连接...")
}
{{< /code >}}

然后再执行前一个表驱动测试，效果如下。

{{< code >}}
有些场景需要在测试用例运行之前做一些初始化工作如: 读取配置、建立数据库连接...
=== RUN   TestTableDriven
--- PASS: TestTableDriven (0.00s)
PASS
有些场景需要在测试用例运行之后做一些清理工作如: 关闭文件、关闭数据连接...
ok  	github.com/op-y/mytest/math	0.717s
{{< /code >}}

#### 基准测试与模糊测试

testing 包还为我们准备了两类测试方法：基准测试和模糊测试。在开发过程中可能使用较少，但是在一些特定场景下用得上。

基准测试主要用来测试代码的性能，类似基本的单元测试，它使用 testing.B 对象，测试方法以 Benchmark 开头接受一个 *testing.B 入参且无返回值。特别的是执行基准测试需要在 go test 命令中加上 -bench 选项，如果加上 -benchmem 可以测试内存使用的情况，更多选项参考官方文档。

在 function_test.go 中有一个基准测试的用例。

{{< code >}}
func BenchmarkFibonacci1h(b *testing.B) {
	for range b.N {
		num := 10
		function.Fibonacci(num)
	}
}
{{< /code >}}

执行效果如下。

{{< cmd >}}
go test -benchmem -run=^$ -bench ^BenchmarkFibonacci1h$ github.com/op-y/mytest/function -v -count=1
{{< /cmd >}}

{{< code >}}
goos: darwin
goarch: arm64
pkg: github.com/op-y/mytest/function
BenchmarkFibonacci1h
BenchmarkFibonacci1h-8   	 3492006	       341.3 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/op-y/mytest/function	1.796s
{{< /code >}}

模糊测试官方说法是通过随机性的输入来测试功的一些边界情况是否正确，类似基本的单元测试，它使用 testing.F 对象，测试方法以 Fuzz 开头接受一个 *testing.F 入参且无返回值。

在 function_test.go 中有一个模糊测试的例子。

{{< code >}}
func FuzzFibonacci(f *testing.F) {
	for _, seed := range [][]int{
		{0, 1},
		{1, 1},
		{2, 2},
		{3, 3},
		{4, 5},
		{5, 8},
		{6, 13},
		{7, 21},
		{8, 34},
		{9, 55},
	} {
		f.Add(seed[0], seed[1])
	}
	f.Fuzz(func(t *testing.T, in, expected int) {
		result := function.Fibonacci(in)
		if result != expected {
			t.Fatalf("the %d fibonacci number is %d, but expect %d", in, result, expected)
		}
	})
}
{{< /code >}}

执行效果如下。

{{< cmd >}}
go test -timeout 30s -run ^FuzzFibonacci$ github.com/op-y/mytest/function -v -count=1
{{< /cmd >}}

{{< code >}}
=== RUN   FuzzFibonacci
=== RUN   FuzzFibonacci/seed#0
=== RUN   FuzzFibonacci/seed#1
=== RUN   FuzzFibonacci/seed#2
=== RUN   FuzzFibonacci/seed#3
=== RUN   FuzzFibonacci/seed#4
=== RUN   FuzzFibonacci/seed#5
=== RUN   FuzzFibonacci/seed#6
=== RUN   FuzzFibonacci/seed#7
=== RUN   FuzzFibonacci/seed#8
=== RUN   FuzzFibonacci/seed#9
--- PASS: FuzzFibonacci (0.00s)
    --- PASS: FuzzFibonacci/seed#0 (0.00s)
    --- PASS: FuzzFibonacci/seed#1 (0.00s)
    --- PASS: FuzzFibonacci/seed#2 (0.00s)
    --- PASS: FuzzFibonacci/seed#3 (0.00s)
    --- PASS: FuzzFibonacci/seed#4 (0.00s)
    --- PASS: FuzzFibonacci/seed#5 (0.00s)
    --- PASS: FuzzFibonacci/seed#6 (0.00s)
    --- PASS: FuzzFibonacci/seed#7 (0.00s)
    --- PASS: FuzzFibonacci/seed#8 (0.00s)
    --- PASS: FuzzFibonacci/seed#9 (0.00s)
PASS
ok  	github.com/op-y/mytest/function	0.463s
{{< /code >}}

我个人对模糊测试的作用目前还不太理解，需要各位更多的去查阅资料了。

#### 测试覆盖率

官方在 testing 包中还为开发者提供了测试覆盖率统计工具，它的实现原理可以参考[这里](https://go.dev/blog/cover)。从使用的角度来说比较简单。

我们新定义一个包 date，实现一个根据序号返回当前星期几的函数 GetWeekDayName。

{{< code >}}
package date

func GetWeekDayName(day int) string {
	switch day {
	case 0:
		return "星期天"
	case 1:
		return "星期一"
	case 2:
		return "星期二"
	case 3:
		return "星期三"
	case 4:
		return "星期四"
	case 5:
		return "星期五"
	case 6:
		return "星期六"
	}
	return "序号不在合理范围"
}
{{< /code >}}

然后为它准备一个测试文件 date_test.go 如下。

{{< code numbered="true" >}}
func TestGetWeekDayName(t *testing.T) {
	var tests = []struct {
		day      int
		expected string
	}{
		{0, "星期天"},
		{3, "星期三"},
		{8, "序号不在合理范围"},
	}
	for _, tt := range tests {
		result := GetWeekDayName(tt.day)
		if result != tt.expected {
			t.Errorf("GetWeekDayName(%d): got %s, want %s", tt.day, result, tt.expected)
		}
	}
}
{{< /code >}}

这里我们使用表驱动法，测试了8种情况中的3中，直观上看覆盖了 3/8 左右的代码。运行测试，这里加上 -coverprofile=coverage.out 选项。go test 会将测试覆盖率统计结果数据输出到当前目录 coverage.out 中。


{{< cmd >}}
cd date
go test -run=TestGetWeekDayName -coverprofile=coverage.out ./.        
{{< /cmd >}}

{{< code >}}
ok      github.com/op-y/mytest/date     0.685s  coverage: 44.4% of statements
{{< /code >}}

实际测试覆盖代码在 44.4% 左右。对于输出文件，我们还可以用浏览器查看。

{{< cmd >}}
go tool cover -html=coverage.out
{{< /cmd >}}

{{< figureCupper
img="coverage.png"
caption="测试覆盖率可视化"
command="Resize"
options="1080x" >}}

### testify 工具

古方提供的 testing 包对开中常见的测试场景而言已经非常够用了。不过总有追求极致的开发者们，所以社区里涌现出了更多更为便捷的测试工具。其中 [testify](https://github.com/stretchr/testify) 应该是目前开发者使用最多的第三方测试工具包了。按照其官方文档所描述

{{< blockquote >}}
testify 包含以下几个包：

* assert 提供了一个全面的断言函数集。
* http 包含了使你更容易进行http活动测试的工具。
* mock 提供了一个系统来模拟你的对象。
* suite 提供了一个基本测试结构，包含了setup/teardown等功能。

testify 是 Go 测试的强大工具，它可以帮助你写出更好的测试。
{{< /blockquote >}}

就目前观察而言，其中 http 和 mock 包由于还有细分领域的各种工具，所以使用的开发者相对较少，而 assert 包和 suite 包，由于其设计与使用更加符合测试惯例、更接近Java、Python 等面相对象语言的测试习惯，受到了广大 Go 开发者的欢迎。

我们在示例代码 math 包中创建了 assertion_test.go 和 suite_test.go 两个文件，做了简单的演示说明。

{{< code numbered="true" >}}
package math

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestMath(t *testing.T) {

	a, b, expected := 1, 2, 3
	result := Add(a, b)
	assert.Equal(t, result, expected,
		"Add(%d, %d) is %d, expected is %d", a, b, result, expected)

	a, b, unexpected := 2, 1, 0
	result = Sub(a, b)
	assert.NotEqual(t, result, unexpected,
		"Sub(%d, %d) eq %d, of course not %d", a, b, result, unexpected)

	assert.Nil(t, nil, "nil of course eq nil")

	a, b, expected = 100, 10, 10
	result = Div(a, b)
	if assert.NotNil(t, result, "result of Div must not be nil") {
		assert.Equal(t, expected, result)
	}
}
{{< /code>}}

{{< code >}}
=== RUN   TestMath
--- PASS: TestMath (0.00s)
PASS
ok  	github.com/op-y/mytest/math	0.829s
{{< /code >}}

如 assertion_test.go 中所示，[assert](https://pkg.go.dev/github.com/stretchr/testify@v1.8.1/assert#pkg-functions) 包提供了一系列的断言函数，开发者可以很方便地运行测试用例，如果结合表驱动测试方法则会更加便捷。此外，testify 中还有一个 [require](https://pkg.go.dev/github.com/stretchr/testify@v1.8.1/require#pkg-functions) 包，其功能与 assert 类似，不同之处在于 assert 在测试用例失败时使用 Errorf 方式输出信息，不会终止后续测试用例运行，而 require 使用 Fatalf 输出信息，会立即终止测试运行。

{{< code >}}
package math

import (
	"fmt"
	"testing"

	"github.com/stretchr/testify/suite"
)

type MathTestSuite struct {
	suite.Suite
}

func (suite *MathTestSuite) SetupSuite() {
	fmt.Println("在测试组运行之前总有些事情要做...")
}

func (suite *MathTestSuite) SetupTest() {
	fmt.Println("在测试运行之前总有些事情要做...")
}

func (suite *MathTestSuite) BeforeTest(suiteName, testName string) {
	fmt.Printf("要运行 %s 测试组里 %s 测试用例了...\n", suiteName, testName)
}

func (suite *MathTestSuite) AfterTest(suiteName, testName string) {
	fmt.Printf("运行完 %s 测试组里 %s 测试用例了...\n", suiteName, testName)
}

func (suite *MathTestSuite) TearDownTest() {
	fmt.Println("在测试运行之后总有些事情要做...")
}

func (suite *MathTestSuite) TearDownSuite() {
	fmt.Println("在测试组运行之后总有些事情要做...")
}

func (suite *MathTestSuite) TestAdd() {
	suite.Equal(Add(1, 2), 3)
}

func (suite *MathTestSuite) TestSub() {
	suite.Equal(Sub(2, 1), 1)
}

func (suite *MathTestSuite) TestMul() {
	suite.Equal(Mul(9, 9), 81)
}

func (suite *MathTestSuite) TestDiv() {
	suite.Equal(Div(100, 10), 10)
}

func TestMathTestSuite(t *testing.T) {
	suite.Run(t, new(MathTestSuite))
}
{{< /code>}}

{{< code >}}
=== RUN   TestMathTestSuite
在测试组运行之前总有些事情要做...
=== RUN   TestMathTestSuite/TestAdd
在测试运行之前总有些事情要做...
要运行 MathTestSuite 测试组里 TestAdd 测试用例了...
运行完 MathTestSuite 测试组里 TestAdd 测试用例了...
在测试运行之后总有些事情要做...
=== RUN   TestMathTestSuite/TestDiv
在测试运行之前总有些事情要做...
要运行 MathTestSuite 测试组里 TestDiv 测试用例了...
运行完 MathTestSuite 测试组里 TestDiv 测试用例了...
在测试运行之后总有些事情要做...
=== RUN   TestMathTestSuite/TestMul
在测试运行之前总有些事情要做...
要运行 MathTestSuite 测试组里 TestMul 测试用例了...
运行完 MathTestSuite 测试组里 TestMul 测试用例了...
在测试运行之后总有些事情要做...
=== RUN   TestMathTestSuite/TestSub
在测试运行之前总有些事情要做...
要运行 MathTestSuite 测试组里 TestSub 测试用例了...
运行完 MathTestSuite 测试组里 TestSub 测试用例了...
在测试运行之后总有些事情要做...
在测试组运行之后总有些事情要做...
--- PASS: TestMathTestSuite (0.00s)
    --- PASS: TestMathTestSuite/TestAdd (0.00s)
    --- PASS: TestMathTestSuite/TestDiv (0.00s)
    --- PASS: TestMathTestSuite/TestMul (0.00s)
    --- PASS: TestMathTestSuite/TestSub (0.00s)
PASS
ok  	github.com/op-y/mytest/math	0.339s
{{< /code >}}

suite 包与之前提到的官方 testing 包中 TestMain 函数相似，但是 suite 提供更加细粒度的控制方式。如 suite_test.go 中所示，我们定义了 MathTestSuite 结构体，它包含了suite.Suite 成员，在此基础上，可以为 MathTestSuite 实现一系列的测试 [hook](https://pkg.go.dev/github.com/stretchr/testify@v1.8.1/suite#pkg-types)，这样一来就可以实现在整体测试运行前后、每个测试函数运行前后执行特定的逻辑，然后再给 MathTestSuite 实现了一些测试方法，这里就是 TestAdd、TestSub、TestMul、TestDiv 了，最后通过一个标准的 TestXXX 方法调用 suite.Run(t, new(MathTestSuite)) 即可运行全部的测试了。当然也可以单独执行其中某一个测试方法，执行过程类似。

## Mock

介绍 testify 的时候，我们提到了其有一个 mock 包，不过不常用，更多的开发者会针对自己面临的场景选择更加合适和趁手的 Mock 工具。这里我就要结合遇到的第二个问题，来介绍测试中的 Mock 了。

### 什么是mock

通常在面向对象程序设计中，Mock（中文常称为模拟对象）是以可控的方式模拟真实对象行为的假的对象。程序员通常创造模拟对象来测试其他对象的行为，很类似汽车设计者使用碰撞测试假人来模拟车辆碰撞中人的动态行为。

我在项目中遇到的第个二问题就是这样一类场景。

* 其一是待测试的一个功能依赖内部系统返回用户的考勤数据和请假数据，业务代码根据这两个数据决定后续的执行逻辑，但是测试环境里这些外部系统都能正常访问，不过它们没有返回任何数据，会导致测试结果不符合预期；
* 其二是待测试的这个功能需要调用外部公司的OpenAPI，由于测试用例中的数据都是编造的，调用第三方的OpenAPI 必然会得到一个异常响应，这与我测试的出发点是不匹配的，也会导致测试结果不符合预期。

当时，我第一时间就想到了使用 Mock 方式模拟这两类调用，按照我的预期来返回数据。我在项目中找了很久，没有看到前人留下来的 Mock 代码，也不知道之前如何做到的。所以只能找合适的 Mock 工具现写了。

#### go-mock

我照常想到的是 Go 官方的解决方案。官方之前提供了一个 [mock](https://github.com/golang/mock) 工具，但是它在2023年就进入了归档状态，不少人说是因为这个工具太稳定了，没啥需要改进的，才被归档的😂。不过在此之后 [Uber](https://www.uber.com) fork 了这个项目，并一直在开发和维护这个 [mock](https://github.com/uber-go/mock) 工具。

Go/Uber 的这个 Mock 工具，从实现上说，是传统的面相对象方式，具体到 Go 项目中，它要求 **被mock** 的对象需要实现一个 **interface**，然后通过 Mock 工具分析这个 interface，自动生成 **mock对象**，后续的测试代码中就可以使用这个 Mock 对象模拟真实对象的行为了。

为了说明我在项目中遇到的问题，我在示例代码中模拟了这一过程。

首先，要使用这个 Mock 工具，我们先要安装它。

{{< cmd >}}
go install go.uber.org/mock/mockgen@latest
{{< /cmd >}}

项目中的业务代码是直接用函数写的，并没有实现某一个 interface，这里我姑且用一个 DepService 接口和一个 DepServiceImpl 结构体来替代。

{{< code >}}
package dep

type UserAttendanceInfo struct {
	UserId       int    `json:"user_id"`
	UserName     string `json:"user_name"`
	RealWorkTime string `json:"real_work_time"`
	VacationTime string `json:"vacation_time"`
}

// 这个接口不一定要包含真实Service所有方法
type DepService interface {
	GetUserAttendanceInfo(userId int) (*UserAttendanceInfo, error)
	ProcessOrder(orderId int) error
}
{{< /code >}}

{{< code >}}
package dep

import (
	"errors"
	"fmt"
)

type DepServiceImpl struct{}

func NewDepServiceImpl() *DepServiceImpl {
	return &DepServiceImpl{}
}

func (svc *DepServiceImpl) Init() {}

func (svc *DepServiceImpl) GetUserAttendanceInfo(userId int) (*UserAttendanceInfo, error) {
	fmt.Println("不用想了, 依赖的接口不会返回数据!")
	return nil, nil
}

func (svc *DepServiceImpl) ProcessOrder(orderId int) error {
	fmt.Println("不用想了, 三方系统无法正常处理测试数据!")
	return errors.New("参数异常")
}

func (svc *DepServiceImpl) Do(userId, orderId int) error {
	fmt.Println("一些前置工作")
	info, err := svc.GetUserAttendanceInfo(userId)
	if err != nil {
		fmt.Println("获取用户出勤信息异常")
		return err
	}
	if info.VacationTime != "0" && info.RealWorkTime < "8" {
		if err := svc.ProcessOrder(orderId); err != nil {
			fmt.Println("处理订单异常")
			return err
		}
	}
	fmt.Println("一些收尾工作")
	return nil
}
{{< /code >}}

参照项目中的情况，有一个测试文件，这里是 dep_test.go，其中的测试因为外部依赖接口问题是无法正常通过的。所以我们先使用 Mock 工具生成 Mock 对象到 dep_mock.go 中。

{{< cmd >}}
mockgen -source=foo.go -destination=dep_mock.go -package=dep
{{< /cmd >}}

{{< code numbered="true" >}}
// Code generated by MockGen. DO NOT EDIT.
// Source: interface.go
//
// Generated by this command:
//
//	mockgen -source=interface.go -destination=dep_mock.go -package=dep
//

// Package dep is a generated GoMock package.
package dep

import (
	reflect "reflect"

	gomock "go.uber.org/mock/gomock"
)

// MockDepService is a mock of DepService interface.
type MockDepService struct {
	ctrl     *gomock.Controller
	recorder *MockDepServiceMockRecorder
	isgomock struct{}
}

// MockDepServiceMockRecorder is the mock recorder for MockDepService.
type MockDepServiceMockRecorder struct {
	mock *MockDepService
}

// NewMockDepService creates a new mock instance.
func NewMockDepService(ctrl *gomock.Controller) *MockDepService {
	mock := &MockDepService{ctrl: ctrl}
	mock.recorder = &MockDepServiceMockRecorder{mock}
	return mock
}

// EXPECT returns an object that allows the caller to indicate expected use.
func (m *MockDepService) EXPECT() *MockDepServiceMockRecorder {
	return m.recorder
}

// GetUserAttendanceInfo mocks base method.
func (m *MockDepService) GetUserAttendanceInfo(userId int) (*UserAttendanceInfo, error) {
	m.ctrl.T.Helper()
	ret := m.ctrl.Call(m, "GetUserAttendanceInfo", userId)
	ret0, _ := ret[0].(*UserAttendanceInfo)
	ret1, _ := ret[1].(error)
	return ret0, ret1
}

// GetUserAttendanceInfo indicates an expected call of GetUserAttendanceInfo.
func (mr *MockDepServiceMockRecorder) GetUserAttendanceInfo(userId any) *gomock.Call {
	mr.mock.ctrl.T.Helper()
	return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "GetUserAttendanceInfo", reflect.TypeOf((*MockDepService)(nil).GetUserAttendanceInfo), userId)
}

// ProcessOrder mocks base method.
func (m *MockDepService) ProcessOrder(orderId int) error {
	m.ctrl.T.Helper()
	ret := m.ctrl.Call(m, "ProcessOrder", orderId)
	ret0, _ := ret[0].(error)
	return ret0
}

// ProcessOrder indicates an expected call of ProcessOrder.
func (mr *MockDepServiceMockRecorder) ProcessOrder(orderId any) *gomock.Call {
	mr.mock.ctrl.T.Helper()
	return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "ProcessOrder", reflect.TypeOf((*MockDepService)(nil).ProcessOrder), orderId)
}
{{< /code >}}

然后就可以改写测试用例，在其中使用 Mock 对象跑通测试了。

{{< code numbered="true" >}}
package dep

import (
	"testing"

	"go.uber.org/mock/gomock"
)

func TestServiceDo(t *testing.T) {
    // 创建 Mock 对象
	ctrl := gomock.NewController(t)
	m := NewMockDepService(ctrl)

    // 设置 Mock 对象的行为
	m.EXPECT().GetUserAttendanceInfo(gomock.Any()).
		DoAndReturn(func(userId int) (*UserAttendanceInfo, error) {
			if userId == 101 {
				return &UserAttendanceInfo{
					UserId:       101,
					UserName:     "张三",
					RealWorkTime: "6",
					VacationTime: "2",
				}, nil
			} else {
				return &UserAttendanceInfo{
					UserId:       999,
					UserName:     "其它人",
					RealWorkTime: "8",
					VacationTime: "0",
				}, nil
			}
		}).
		AnyTimes()

    // 设置 Mock 对象的行为
	m.EXPECT().ProcessOrder(gomock.Any()).Return(nil).AnyTimes()

	svc := NewDepServiceImpl()
	svc.Init()

	userId, orderId := 101, 101
	t.Log("一些前置工作")
    // 这里使用了 Mock 对象
	info, err := m.GetUserAttendanceInfo(userId)
	if err != nil {
		t.Fatal("访问依赖接口异常")
	}
	if info.RealWorkTime == info.VacationTime {
		t.Fatal("张三的测试用例中 RealWorkTime 与 VacationTime 不应该相等!")
	}

    // 这里也使用了 Mock 对象
	if err := m.ProcessOrder(orderId); err != nil {
		t.Fatal("访问三方依赖接口异常")
	}
	t.Log("一些收尾工作")
	t.Log("完成业务流程测试")
}
{{< /code >}}

正常运行结果如下。至此第二个问题得到了解决。

{{< code >}}
=== RUN   TestServiceDo
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/dep/dep_test.go:39: 一些前置工作
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/dep/dep_test.go:50: 一些收尾工作
    /Users/yezhiqin/gopath/src/github.com/op-y/mytest/dep/dep_test.go:51: 完成业务流程测试
--- PASS: TestServiceDo (0.00s)
PASS
ok  	github.com/op-y/mytest/dep	0.654s
{{< /code >}}

#### 第三方 Mock 工具

在官方的 Mock 工具之外，不乏一些实现路径不同的 Mock 工具。解决问题过程中，我通过查阅资料发现了这些各有特点的 Mock工具，将它们分享在这里。通过了解同一功能（mock）的不同实现途径，我也受到了不少启发。

首先是 **monkey patch** 方式，接触过 Python 的朋友一定不会陌生，在动态语言中，由于语言特性，使用一个对象替代另一个对象非常简单，那么在 Go 这种静态语言中能做到吗？Bouke 大神还真就做了，它的 [mock](https://github.com/bouk/monkey) 项目是在 Go 中通过动态修改函数 **代码段** 来实现 Mock 功能，其指导思想可以参考他的文章 [monkey-patching-in-go](https://bou.ke/blog/monkey-patching-in-go/)。后来国内开发者在此基础上改进，又实现了支持并发monkey patch 的工具 [mock](https://github.com/go-kiss/monkey)。我在示例代码 gokiss 目录中留下了 go-kiss/monkey 项目自己提供的 example。

上述这种 monkey patch 通过黑科技去修改汇编代码的方式虽然让 Mock 变的很简单，但也不是完全没有代价的。首先，在程序运行时修改函数的代码段这个操作有些操作系统本身就不支持。其中就有苹果的 M 系列芯片平台。苹果为了安全性，在系统层面禁止内存同时拥有可写和可执行权限。所以 monkey patch 没法在 M 系列芯片设备上原生运行。其次就是兼容性问题。因为底层实现依赖 Go 语言的 ABI，但 Go 语言只保证 API 下向兼容，从不保证 ABI 稳定不变。未来某个 Go 版本 ABI 大改，有可能导致之前写的单元测试都没法正常运行。

顺着修改代码的思路，又有高手 [xhd2015](https://blog.xhd2015.xyz) 创造了新的 Mock工具。不是直接修改底层汇编代码有平台不兼容的问题吗？那是不是可以参考 testing 实现 coverage 的 [原理](https://go.dev/blog/cover) 在编译期间直接修改源码呢？于是有了 [go-mock](https://github.com/xhd2015/go-mock) 项目。之后 xhd2015 大佬又通过编译期间修改 IR 代码的方式实现了另一个 [xgo](https://github.com/xhd2015/xgo) Mock工具。

那这种修改编译期间临时修改 Go 源码或者修改 IR 代码方式就完全没有缺点了吗？也不是，最大的不足是这种方式通常只能 Mock 我们自己编写的代码，而且由于其实现原理，运行开销也不小。

### Mock 中间件

有关项目问题中 Mock 的内容，到这里其实已经结束了。不过我还想简单的补充一些开源项目，它们主要是我这次项目学习中还没有遇到的场景，即 Mock HTTP服务和数据库，我将它们放在下边，以供各位参考。

* Mock HTTP
    * [net/http/httptest](https://pkg.go.dev/net/http/httptest)
    * [gock](https://github.com/h2non/gock)
    * [httpmock](https://github.com/jarcoal/httpmock)
* Mock MySQL
    * [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)
* Mock Redis
    * [miniredis](https://github.com/alicebob/miniredis)

## 后记

测试是开发过程中必不可少的一个环节，也是我们每个开发者应该熟练掌握的技能。经历这次解决新项目学习中遇到的各种测试问题，我对 Go 开发中的测试有了更加深入的理解。希望我分享的故事和提供的各种信息对你有帮助！