
# stretchr testify单元测试框架介绍.md

参考：

- [stretchr testify GitHub官网](https://github.com/stretchr/testify/)

我们这里只简单介绍一下其中`mock` Package的用法及实现原理。

---

## 1. mock package的用法

我们可以使用mock包来产生一个`mock Object`以替换`real Object`来进行单元测试。

下面的示例中test函数进行单元测试，其通过`testObj`来创建expectations并断言它们是否真实发生:

```golang
package yours

import (
	"testing"

	"github.com/stretchr/testify/mock"
)

/*
  Test objects
*/

// MyMockedObject is a mocked object that implements an interface
// that describes an object that the code I am testing relies on.
type MyMockedObject struct {
	mock.Mock
}

// DoSomething is a method on MyMockedObject that implements some interface
// and just records the activity, and returns what the Mock object tells it to.
//
// In the real object, this method would do something useful, but since this
// is a mocked object - we're just going to stub it out.
//
// NOTE: This method is not being tested here, code that uses this object is.
func (m *MyMockedObject) DoSomething(number int) (bool, error) {
	args := m.Called(number)
	return args.Bool(0), args.Error(1)
}

/*
  Actual test functions
*/

// TestSomething is an example of how to use our test object to
// make assertions about some target code we are testing.
func TestSomething(t *testing.T) {
	// create an instance of our test object
	testObj := new(MyMockedObject)

	// set up expectations
	testObj.On("DoSomething", 123).Return(true, nil)

	// call the code we are testing
	targetFuncThatDoesSomethingWithObj(testObj)

	// assert that the expectations were met
	testObj.AssertExpectations(t)
}

// TestSomethingWithPlaceholder is a second example of how to use our test object to
// make assertions about some target code we are testing.
// This time using a placeholder. Placeholders might be used when the
// data being passed in is normally dynamically generated and cannot be
// predicted beforehand (eg. containing hashes that are time sensitive)
func TestSomethingWithPlaceholder(t *testing.T) {
	// create an instance of our test object
	testObj := new(MyMockedObject)

	// set up expectations with a placeholder in the argument list
	testObj.On("DoSomething", mock.Anything).Return(true, nil)

	// call the code we are testing
	targetFuncThatDoesSomethingWithObj(testObj)

	// assert that the expectations were met
	testObj.AssertExpectations(t)

}

// TestSomethingElse2 is a third example that shows how you can use
// the Unset method to cleanup handlers and then add new ones.
func TestSomethingElse2(t *testing.T) {
	// create an instance of our test object
	testObj := new(MyMockedObject)

	// set up expectations with a placeholder in the argument list
	mockCall := testObj.On("DoSomething", mock.Anything).Return(true, nil)

	// call the code we are testing
	targetFuncThatDoesSomethingWithObj(testObj)

	// assert that the expectations were met
	testObj.AssertExpectations(t)

	// remove the handler now so we can add another one that takes precedence
	mockCall.Unset()

	// return false now instead of true
	testObj.On("DoSomething", mock.Anything).Return(false, nil)

	testObj.AssertExpectations(t)
}
```

## 2. mock的基本工作原理


1） **mock.Mock数据结构**

`mock.Mock`对象对象保存所有注册的`Expectations`和已经执行的`Calls`:

```golang
// Mock is the workhorse used to track activity on another object.
// For an example of its usage, refer to the "Example Usage" section at the top
// of this document.
type Mock struct {
	// Represents the calls that are expected of
	// an object.
	ExpectedCalls []*Call

	// Holds the calls that were made to this mocked object.
	Calls []Call

	// test is An optional variable that holds the test struct, to be used when an
	// invalid mock call was made.
	test TestingT

	// TestData holds any data that might be useful for testing.  Testify ignores
	// this data completely allowing you to do whatever you like with it.
	testData objx.Map

	mutex sync.Mutex
}
```

2） **mock.Call数据结构**

保存一个Call的相关比较信息：

```golang
// Call represents a method call and is used for setting expectations,
// as well as recording activity.
type Call struct {
	Parent *Mock

	// The name of the method that was or will be called.
	Method string

	// Holds the arguments of the method.
	Arguments Arguments

	// Holds the arguments that should be returned when
	// this method is called.
	ReturnArguments Arguments

	// Holds the caller info for the On() call
	callerInfo []string

	// The number of times to return the return arguments when setting
	// expectations. 0 means to always return the value.
	Repeatability int

	// Amount of times this call has been called
	totalCalls int

	// Call to this method can be optional
	optional bool

	// Holds a channel that will be used to block the Return until it either
	// receives a message or is closed. nil means it returns immediately.
	WaitFor <-chan time.Time

	waitTime time.Duration

	// Holds a handler used to manipulate arguments content that are passed by
	// reference. It's useful when mocking methods such as unmarshalers or
	// decoders.
	RunFn func(Arguments)

	// PanicMsg holds msg to be used to mock panic on the function call
	//  if the PanicMsg is set to a non nil string the function call will panic
	// irrespective of other settings
	PanicMsg *string

	// Calls which must be satisfied before this call can be
	requires []*Call
}
```

### 2.1 向mock对象中注册expectations

使用`mock.On()`函数来向mock对象中注册expectations, 我们来看对应实现:

```golang
//	Mock.On("MyMethod", arg1, arg2)
func (m *Mock) On(methodName string, arguments ...interface{}) *Call {
	for _, arg := range arguments {
		if v := reflect.ValueOf(arg); v.Kind() == reflect.Func {
			panic(fmt.Sprintf("cannot use Func in expectations. Use mock.AnythingOfType(\"%T\")", arg))
		}
	}

	m.mutex.Lock()
	defer m.mutex.Unlock()

	c := newCall(m, methodName, assert.CallerInfo(), arguments, make([]interface{}, 0))
	m.ExpectedCalls = append(m.ExpectedCalls, c)
	return c
}
```

上面我们看到，会调用`newCall`来创建一个`mock.Call`对象，然后添加到`mock.ExpectedCalls`数组中保存起来。



### 2.2 mock.Called()检查expectations是否发生

`mock.Called`函数会检查相应的expectations是否发生，我们来看实现：

```golang
// Called tells the mock object that a method has been called, and gets an array
// of arguments to return.  Panics if the call is unexpected (i.e. not preceded by
// appropriate .On .Return() calls)
// If Call.WaitFor is set, blocks until the channel is closed or receives a message.
func (m *Mock) Called(arguments ...interface{}) Arguments {
	// get the calling function's name
	pc, _, _, ok := runtime.Caller(1)
	if !ok {
		panic("Couldn't get the caller information")
	}
	functionPath := runtime.FuncForPC(pc).Name()
	// Next four lines are required to use GCCGO function naming conventions.
	// For Ex:  github_com_docker_libkv_store_mock.WatchTree.pN39_github_com_docker_libkv_store_mock.Mock
	// uses interface information unlike golang github.com/docker/libkv/store/mock.(*Mock).WatchTree
	// With GCCGO we need to remove interface information starting from pN<dd>.
	if gccgoRE.MatchString(functionPath) {
		functionPath = gccgoRE.Split(functionPath, -1)[0]
	}
	parts := strings.Split(functionPath, ".")
	functionName := parts[len(parts)-1]
	return m.MethodCalled(functionName, arguments...)
}
```

`mock.Called()`会检查是否有匹配的`expectations`，如果没有则报错； 否则会将当前call加入到`mock.Calls`数组中，并返回所对应expectation call的`ReturnArguments`。
