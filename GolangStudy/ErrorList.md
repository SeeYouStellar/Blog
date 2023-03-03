#### 1. 依赖获取及更新时访问不了依赖所在的仓库

![](https://content.markdowner.net/pub/VvWbXb-Bnp9e6d)

问题:

依赖获取时go module会根据环境变量GOPROXY默认的https://goproxy.org访问不了

解决方法:

将GOPROXY值改成国内代理https://goproxy.cn

```powershell
go env -w GOPROXY=https://goproxy.cn
```

#### 2. 同一个package内相互调用

![](https://content.markdowner.net/pub/N1Ow0o-0opdWj2)

问题:

同一个package内多个.go文件相互调用各自的函数

解决方法:

把两个文件放到两个package中, 并且需要严格按照package1, package2命名

参考[go 文件结构](https://go.dev/doc/modules/managing-source)

#### 3. 引用多package下的某个package时, go.mod文件依赖路径(module path)混淆

![](https://content.markdowner.net/pub/74B7an-EoR9X00)

go.mod

```
module example/hello

go 1.17

replace example.com/greetings/package1 => ../greetings/package1
```

hello.go

```go
package main
import (
    "fmt"
    "example.com/greetings/package1"
)
func main(){
    var name string = "lxy"
    fmt.Println(package1.Hello(name))
}
```

问题:

go.mod会先将replace内容替换, 然后添加伪版本号, 接着与hello.go中引入的依赖进行匹配, 发现hello.go中的"example.com/greetings/package1"路径下找不到go.mod文件

解决方法:

按照一下规则进行路径指定: **go.mod文件 require module path, .go文件import package path**

这里又说明了go项目中每一个package实现一个功能模块, 在代码中引用某个功能模块中的函数, 只需要引入这个package即可, 不用详细到package下的具体文件