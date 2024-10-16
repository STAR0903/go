# validator库

#### 翻译校验错误提示信息

`validator`库本身是支持国际化的，借助相应的语言包可以实现校验错误提示信息的自动翻译。下面的示例代码演示了如何将错误提示信息翻译成中文，翻译成其他语言的方法类似。

```
import (
	"fmt"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	enTranslations "github.com/go-playground/validator/v10/translations/en"
	zhTranslations "github.com/go-playground/validator/v10/translations/zh"
)

// 定义一个全局翻译器T
var trans ut.Translator

// InitTrans 初始化翻译器
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器

		// 第一个参数是备用（fallback）的语言环境
		// 后面的参数是应该支持的语言环境（支持多个）
		// uni := ut.New(zhT, zhT) 也是可以的
		uni := ut.New(enT, zhT, enT)

		// locale 通常取决于 http 请求头的 'Accept-Language'
		var ok bool
		// 也可以使用 uni.FindTranslator(...) 传入多个locale进行查找
		trans, ok = uni.GetTranslator(locale)
		if !ok {
			return fmt.Errorf("uni.GetTranslator(%s) failed", locale)
		}

		// 注册翻译器
		switch locale {
		case "en":
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		case "zh":
			err = zhTranslations.RegisterDefaultTranslations(v, trans)
		default:
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		}
		return
	}
	return
}
```

#### 自定义错误提示信息的字段名

错误提示中的字段并不是请求中使用的字段，例如：`RePassword`是我们后端定义的结构体中的字段名，而请求中使用的是 `re_password`字段。如何是错误提示中的字段使用自定义的名称，例如 `json`tag指定的值呢？

只需要在初始化翻译器的时候像下面一样添加一个获取 `json` tag的自定义方法即可。

```
// 注册一个获取json tag的自定义方法
v.RegisterTagNameFunc(func(fld reflect.StructField) string {
	name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
	if name == "-" {
		return ""
	}
	return name
})
```

以下是对这段代码的解释：

1. `v.RegisterTagNameFunc(func(fld reflect.StructField) string {...})`：这里是在为某个对象（通常是验证器对象）注册一个用于获取结构体字段标签名的函数。这个函数会在需要确定结构体字段的名称（通常是在进行数据验证或者绑定数据时）被调用。
2. 在注册的函数内部：
   * `name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]`：首先通过反射获取结构体字段的 `json`标签值，然后使用 `strings.SplitN`函数以 `,`为分隔符将这个标签值分割成两部分，并取第一部分作为字段的名称。这样做可能是为了处理 `json`标签中可能包含的额外选项（例如 `json:"fieldName,omitempty"`中的 `omitempty`）

     ```
     omitempty作用
     当对一个包含 omitempty 选项的结构体进行 JSON 编码时，如果该字段的值为其类型的零值（例如，对于整数类型是 0，对于字符串类型是 ""，对于指针类型是 nil 等），那么在生成的 JSON 数据中这个字段将被省略，不会出现在最终的 JSON 字符串中
     使用场景
     减少数据传输量：在网络通信中，尤其是在移动设备或带宽有限的环境下，减少不必要的零值字段可以降低数据传输的成本和提高传输效率。
     动态数据结构：当数据结构可能具有不同的状态，某些字段可能在某些情况下为空时，使用 “omitempty” 可以使生成的 JSON 更加简洁，只包含有实际意义的值。
     API 响应：在构建 API 响应时，可以使用 “omitempty” 来确保只返回有实际值的字段，使 API 的输出更加清晰和易于理解。
     ```
   * `if name == "-" { return "" }`：如果获取到的名称是 `-`，则返回一个空字符串。这通常表示该字段在进行数据绑定或验证时应该被忽略。
   * `return name`：如果名称不是 `-`，则返回获取到的字段名称。

#### 去掉结构体名称前缀的自定义方法

```
func removeTopStruct(fields map[string]string) map[string]string {
	res := map[string]string{}
	for field, err := range fields {
		res[field[strings.Index(field, ".")+1:]] = err
	}
	return res
}
```
