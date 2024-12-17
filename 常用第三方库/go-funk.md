# 介绍

`go-funk` 是基于反射(`reflect`)实现的一个现代 `Go`工具库，封装了对 `slice/map/struct/string`等的操作。

```
# 下载
go get github.com/thoas/go-funk
# 引入
import github.com/thoas/go-funk
```

# 判断是否存在包含关系

```
func Contains(in interface{}, elem interface{}) bool
func ContainsBool(s []bool, v bool) bool
func ContainsFloat32(s []float32, v float32) bool
func ContainsFloat64(s []float64, v float64) bool
func ContainsInt(s []int, v int) bool
func ContainsInt32(s []int32, v int32) bool
func ContainsInt64(s []int64, v int64) bool
func ContainsString(s []string, v string) bool
func ContainsUInt(s []uint, v uint) bool
func ContainsUInt32(s []uint32, v uint32) bool
func ContainsUInt64(s []uint64, v uint64) bool
```

### Contains

```
funk.Contains(father, target)
```

支持类型：String, Map, Slice, Array

##### map

```
对于map中是否含有某个元素，funk在底层逻辑是通过遍历in，获取key和value再进行判断，当遍历所得的键值对中有符合条件的时，返回true。所以，判断map是否包含某个元素时，每次调用函数只能判断一个键值对是否属于map
```

```
func mapContains() {
	in := map[string]int{
		"one":   1,
		"two":   2,
		"three": 3,
	}

	// 方式一：直接传入参数即可
	// 第一种情况：判断是否含有key值，传入的参数类型必须与map的key类型一致
	v0 := funk.Contains(in, []string{"one"})
	v1 := funk.Contains(in, "one")
	// 方式二：通过函数设定判断条件
	// 第一种情况：判断是否含有key值
	v2 := funk.Contains(in, func(key string, value int) bool {
		return key == "two"
	})
	// 第二种情况：判断是否含有value值
	v3 := funk.Contains(in, func(key string, value int) bool {
		return value == 3
	})
	// 第三种情况：判断是否含有这个键值对
	v4 := funk.Contains(in, func(key string, value int) bool {
		return key == "three" && value == 3
	})
	v5 := funk.Contains(in, func(key string, value int) bool {
		return key == "one" && value == 4
	})

	fmt.Println(v0, v1, v2, v3, v4, v5)
}
```

```
Output:
false true true true true false
```

##### string

```
v1 := funk.Contains("one you have three apples", "e y")
Output: true
```

##### slice

```
type person struct {
    Name string
Age  int
}

func sliceContains() {
    persons := []person{
       {Name: "ww", Age: 12},
       {Name: "PP", Age: 22},
    }
    p := person{Name: "ff", Age: 14}
    v := funk.Contains(persons, p)
    fmt.Println(v)
    persons = append(persons, p)
    v = funk.Contains(persons, p)
    fmt.Println(v)
}
```

```
Output: 
false
true
```

# 给出两个集合不相同部分

```
func Difference(x interface{}, y interface{}) (interface{}, interface{})
func DifferenceInt(x []int, y []int) ([]int, []int)
func DifferenceInt32(x []int32, y []int32) ([]int32, []int32)
func DifferenceInt64(x []int64, y []int64) ([]int64, []int64)
func DifferenceString(x []string, y []string) ([]string, []string)
func DifferenceUInt(x []uint, y []uint) ([]uint, []uint)
func DifferenceUInt32(x []uint32, y []uint32) ([]uint32, []uint32)
func DifferenceUInt64(x []uint64, y []uint64) ([]uint64, []uint64)
```

### Difference

```
result1, result2 := funk.Difference(group1, group2)
```

##### string

```
func Difference() {
    str1 := []string{"ww", "mm", "PP"}
    str2 := []string{"ww", "aa", "xx"}
    result1, result2 := funk.Difference(str1, str2)
    fmt.Println(result1, result2)
}
```

```
Output: 
[mm PP] [aa xx]
```

##### map

```
func Difference() {
    map1 := map[string]int{
       "one":   1,
       "two":   2,
       "three": 3,
    }
    map2 := map[string]int{
       "one":   1,
       "t":     2,
       "three": 4,
    }
    result1, result2 := funk.Difference(map1, map2)
    fmt.Println(result1, result2)
}
```

```
Output: 
map[three:3 two:2] map[t:2 three:4]
```

除非键和值对应相等，否则判定为不相同。

# 给出去除相同部分后的集合

```
result := funk.Join(group1, group2, funk.LeftJoin)
```

取出只在 group1，不在 group2 的元素。

```
result := funk.Join(group1, group2, funk.RightJoin)
```

取出只在 group2，不在 group1 的元素。

```
func f() {
	str1 := []string{"ww", "mm", "PP"}
	str2 := []string{"ww", "aa", "xx", "PP"}
	result1 := funk.Join(str1, str2, funk.LeftJoin)
	result2 := funk.Join(str1, str2, funk.RightJoin)
	fmt.Println(result1, result2)
}
```

```
Output: 
[mm] [aa xx]
```

# 给出两个集合相同部分

```
result := funk.Join(group1, group2, funk.InnerJoin)
```

**这个用法不接受map作为参数传入。**

```
func Inner() {
	str1 := []string{"ww", "mm", "PP"}
	str2 := []string{"ww", "aa", "xx"}
	result := funk.Join(str1, str2, funk.InnerJoin)
	fmt.Println(result)
}
```

# 获取首次匹配的索引

```
func IndexOf(in interface{}, elem interface{}) int
func IndexOfBool(a []bool, x bool) int
func IndexOfFloat64(a []float64, x float64) int
func IndexOfInt(a []int, x int) int
func IndexOfInt32(a []int32, x int32) int
func IndexOfInt64(a []int64, x int64) int
func IndexOfString(a []string, x string) int
func IndexOfUInt(a []uint, x uint) int
func IndexOfUInt32(a []uint32, x uint32) int
func IndexOfUInt64(a []uint64, x uint64) int
```

IndexOf 系列函数获取在数组中首次出现 value 的索引，如果找不到该值，则返回 -1。

### IndexOf

```
index := funk.IndexOf(group, target)
```

接受 string ，切片和数组作为 group 参数。

```
func f3() {
	str := []string{"ww", "mm", "PP", "ee", "ww", "1"}
	index1 := funk.IndexOf(str, "ww")
	index2 := funk.IndexOf(str, "star")
	fmt.Println(index1, index2)
}
```

```
Output：
0 -1
```

# 获取最后一次匹配的索引

```
func LastIndexOf(in interface{}, elem interface{}) int
func LastIndexOfBool(a []bool, x bool) int
func LastIndexOfFloat32(a []float32, x float32) int
func LastIndexOfFloat64(a []float64, x float64) int
func LastIndexOfInt(a []int, x int) int
func LastIndexOfInt32(a []int32, x int32) int
func LastIndexOfInt64(a []int64, x int64) int
func LastIndexOfString(a []string, x string) int
func LastIndexOfUInt(a []uint, x uint) int
func LastIndexOfUInt32(a []uint32, x uint32) int
func LastIndexOfUInt64(a []uint64, x uint64) int
```

LastIndexOf 系列函数获取在数组中找到 value 最后一次出现的索引，如果找不到该值，则返回 -1

### LastIndexOf

```
index := funk.LastIndexOf(group, target)
```

接受 string ，切片和数组作为 group 参数。

```
func f4() {
	str := "What! ww is the best girl ?"
	index1 := funk.LastIndexOf(str, "w")
	index2 := funk.LastIndexOf(str, "star")
	fmt.Println(index1, index2)
}
```

```
Output：
7 -1
```

# 结构体集合转换为map

```
result := funk.ToMap(group, 字段名)
```

返回以指定字段名为 key ，结构体为 value 的 map 。

如果结构体集合里指定字段有相同值，会覆盖，最后只输出一个结果。

```
func Tomap() {
    persons := []person{
       {Name: "ww", Age: 12},
       {Name: "mm", Age: 18},
       {Name: "PP", Age: 22},
       {Name: "ff", Age: 12},
    }
    result1 := funk.ToMap(persons, "Name")
    result2 := funk.ToMap(persons, "Age")
    fmt.Println(result1)
    fmt.Println(result2)
}
```

```
Output：
map[PP:{PP 22} ff:{ff 12} mm:{mm 18} ww:{ww 12}]
map[12:{ff 12} 18:{mm 18} 22:{PP 22}]
```

# 集合转换成map

```
result1 := funk.ToSet(group)
```

返回 `map[T]struct{}` ，T 是 group 的类型。


```
func Toset() {
	p1 := person{Name: "ww", Age: 12}
	persons := []person{
		p1,
		{Name: "mm", Age: 18},
		{Name: "PP", Age: 22},
		{Name: "ff", Age: 12},
	}
	result1 := funk.ToSet(persons).(map[person]struct{})
	result2 := funk.ToSet([]int{1, 2, 3, 4})
	fmt.Println(result1)
	fmt.Println(result2)
}
```

```
Output：
map[{PP 22}:{} {ff 12}:{} {mm 18}:{} {ww 12}:{}]
map[1:{} 2:{} 3:{} 4:{}]
```

# 更多

### [官方文档](https://pkg.go.dev/github.com/thoas/go-funk)

### [参考笔记](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/)

[1. 介绍](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#1-%E4%BB%8B%E7%BB%8D)

[2. 下载](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#2-%E4%B8%8B%E8%BD%BD)

[3. 切片(`slice`)操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#3-%E5%88%87%E7%89%87slice%E6%93%8D%E4%BD%9C)

* [3.1 判断元素是否存在](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#31--%E5%88%A4%E6%96%AD%E5%85%83%E7%B4%A0%E6%98%AF%E5%90%A6%E5%AD%98%E5%9C%A8)
* [3.2 查找元素第一次出现位置](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#32-%E6%9F%A5%E6%89%BE%E5%85%83%E7%B4%A0%E7%AC%AC%E4%B8%80%E6%AC%A1%E5%87%BA%E7%8E%B0%E4%BD%8D%E7%BD%AE)
* [3.3 查找元素最后一次出现位置](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#33--%E6%9F%A5%E6%89%BE%E5%85%83%E7%B4%A0%E6%9C%80%E5%90%8E%E4%B8%80%E6%AC%A1%E5%87%BA%E7%8E%B0%E4%BD%8D%E7%BD%AE)
* [3.4 批量查找(都有则True)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#34--%E6%89%B9%E9%87%8F%E6%9F%A5%E6%89%BE%E9%83%BD%E6%9C%89%E5%88%99true)
* [3.5 批量查找(有一则True)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#35-%E6%89%B9%E9%87%8F%E6%9F%A5%E6%89%BE%E6%9C%89%E4%B8%80%E5%88%99true)
* [3.6 获取最后或第一个元素](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#36-%E8%8E%B7%E5%8F%96%E6%9C%80%E5%90%8E%E6%88%96%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%85%83%E7%B4%A0)
* [3.7 用元素填充切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#37-%E7%94%A8%E5%85%83%E7%B4%A0%E5%A1%AB%E5%85%85%E5%88%87%E7%89%87)
* [3.8 取两个切片共同元素结果集](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#38-%E5%8F%96%E4%B8%A4%E4%B8%AA%E5%88%87%E7%89%87%E5%85%B1%E5%90%8C%E5%85%83%E7%B4%A0%E7%BB%93%E6%9E%9C%E9%9B%86)
* [3.9 获取去掉两切片共同元素结果集](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#39--%E8%8E%B7%E5%8F%96%E5%8E%BB%E6%8E%89%E4%B8%A4%E5%88%87%E7%89%87%E5%85%B1%E5%90%8C%E5%85%83%E7%B4%A0%E7%BB%93%E6%9E%9C%E9%9B%86)
* [3.10 求只存在某切片的元素(除去共同元素)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#310--%E6%B1%82%E5%8F%AA%E5%AD%98%E5%9C%A8%E6%9F%90%E5%88%87%E7%89%87%E7%9A%84%E5%85%83%E7%B4%A0%E9%99%A4%E5%8E%BB%E5%85%B1%E5%90%8C%E5%85%83%E7%B4%A0)
* [3.11 分别去掉两个切片共同元素(两结果集)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#311-%E5%88%86%E5%88%AB%E5%8E%BB%E6%8E%89%E4%B8%A4%E4%B8%AA%E5%88%87%E7%89%87%E5%85%B1%E5%90%8C%E5%85%83%E7%B4%A0%E4%B8%A4%E7%BB%93%E6%9E%9C%E9%9B%86)
* [3.12 遍历切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#312-%E9%81%8D%E5%8E%86%E5%88%87%E7%89%87)
* [3.13 删除首或尾](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#313-%E5%88%A0%E9%99%A4%E9%A6%96%E6%88%96%E5%B0%BE)
* [3.14 判断A切片是否属于B切片子集](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#314-%E5%88%A4%E6%96%ADa%E5%88%87%E7%89%87%E6%98%AF%E5%90%A6%E5%B1%9E%E4%BA%8Eb%E5%88%87%E7%89%87%E5%AD%90%E9%9B%86)
* [3.15 分组](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#315-%E5%88%86%E7%BB%84)
* [3.16 把结构体切片转成map](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#316-%E6%8A%8A%E7%BB%93%E6%9E%84%E4%BD%93%E5%88%87%E7%89%87%E8%BD%AC%E6%88%90map)
* [3.17 把切片值转成 `Map`中的 `Key`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#317-%E6%8A%8A%E5%88%87%E7%89%87%E5%80%BC%E8%BD%AC%E6%88%90map%E4%B8%AD%E7%9A%84key)
* [3.18 把二维切片转成一维切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#318--%E6%8A%8A%E4%BA%8C%E7%BB%B4%E5%88%87%E7%89%87%E8%BD%AC%E6%88%90%E4%B8%80%E7%BB%B4%E5%88%87%E7%89%87)
* [3.19 打乱切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#319-%E6%89%93%E4%B9%B1%E5%88%87%E7%89%87)
* [3.20 反转切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#320-%E5%8F%8D%E8%BD%AC%E5%88%87%E7%89%87)
* [3.21 元素去重](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#321-%E5%85%83%E7%B4%A0%E5%8E%BB%E9%87%8D)
* [3.22 删除制定元素](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#322-%E5%88%A0%E9%99%A4%E5%88%B6%E5%AE%9A%E5%85%83%E7%B4%A0)

[4. 映射(`map`)操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#4-%E6%98%A0%E5%B0%84map%E6%93%8D%E4%BD%9C)

* [4.1 获取所有的 `Key`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#41-%E8%8E%B7%E5%8F%96%E6%89%80%E6%9C%89%E7%9A%84key)
* [4.2 获取所有的 `Value`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#42-%E8%8E%B7%E5%8F%96%E6%89%80%E6%9C%89%E7%9A%84value)

[5. 结构体(`struct`)切片操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#5-%E7%BB%93%E6%9E%84%E4%BD%93struct%E5%88%87%E7%89%87%E6%93%8D%E4%BD%9C)

* [5.1 取结构体某元素为切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#51-%E5%8F%96%E7%BB%93%E6%9E%84%E4%BD%93%E6%9F%90%E5%85%83%E7%B4%A0%E4%B8%BA%E5%88%87%E7%89%87)

[6. 判断操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#6-%E5%88%A4%E6%96%AD%E6%93%8D%E4%BD%9C)

* [6.1 判断相等](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#61-%E5%88%A4%E6%96%AD%E7%9B%B8%E7%AD%89)
* [6.2 判断类型一致](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#62-%E5%88%A4%E6%96%AD%E7%B1%BB%E5%9E%8B%E4%B8%80%E8%87%B4)
* [6.3 判断 `array|slice`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#63-%E5%88%A4%E6%96%ADarrayslice)
* [6.4 判断空](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#64-%E5%88%A4%E6%96%AD%E7%A9%BA)

[7. 类型转换](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#7-%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2)

* [7.1 任意数字转 `float64`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#71-%E4%BB%BB%E6%84%8F%E6%95%B0%E5%AD%97%E8%BD%ACfloat64)
* [7.2 将 `X`转成 `[]X`](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#72-%E5%B0%86x%E8%BD%AC%E6%88%90x)

[8.字符串操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#8%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%93%8D%E4%BD%9C)

* [8.1 根据字符串生成切片](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#81-%E6%A0%B9%E6%8D%AE%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%94%9F%E6%88%90%E5%88%87%E7%89%87)

[9.数字计算](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#9%E6%95%B0%E5%AD%97%E8%AE%A1%E7%AE%97)

* [9.1 最大值(`Max`)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#91-%E6%9C%80%E5%A4%A7%E5%80%BCmax)
* [9.2 最小值(`Min`)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#92-%E6%9C%80%E5%B0%8F%E5%80%BCmin)
* [9.3 求和(`Sum`)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#93-%E6%B1%82%E5%92%8Csum)
* [9.4 求乘积(`Product`)](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#94-%E6%B1%82%E4%B9%98%E7%A7%AFproduct)

[10. 其他操作](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#10-%E5%85%B6%E4%BB%96%E6%93%8D%E4%BD%9C)

* [10.1 生成随机数](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#101-%E7%94%9F%E6%88%90%E9%9A%8F%E6%9C%BA%E6%95%B0)
* [10.2 生成随机字符串](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#102-%E7%94%9F%E6%88%90%E9%9A%8F%E6%9C%BA%E5%AD%97%E7%AC%A6%E4%B8%B2)
* [10.3 三元运算](https://shershon1991.github.io/post/%E6%89%A9%E5%B1%95%E5%8C%85/29-go-funk/#103-%E4%B8%89%E5%85%83%E8%BF%90%E7%AE%97)
