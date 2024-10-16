# Gin-源码学习

```
//如果结构体Engine没有实现IRouter接口，编译将会报错
var_ IRouter= (*Engine)(nil)
```

```
//节选自engine结构体初始化，知道http请求类型共九个后，一次性把切片容量申请到位
trees: make(methodTrees, 0, 9)
```
