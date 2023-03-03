命令行输入go env查看所有go的环境变量

本机例子:

```powershell
GO111MODULE="on"  //1
GOARCH="amd64"
GOBIN=""
GOCACHE="/home/lixinyu/.cache/go-build"  // 2
GOENV="/home/lixinyu/.config/go/env"   //3
GOEXE=""
GOEXPERIMENT=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOINSECURE=""
GOMODCACHE="/home/lixinyu/go/pkg/mod"
GONOPROXY=""
GONOSUMDB=""
GOOS="linux"
GOPATH="/home/lixinyu/go" //4
GOPRIVATE=""
GOPROXY="https://goproxy.cn"  //5
GOROOT="/usr/local/go"  //6
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64" //7
GOVCS=""
GOVERSION="go1.17.6"
GCCGO="gccgo"
AR="ar"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD="/home/lixinyu/GolangStudy/test/go.mod"  //8
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build446377775=/tmp/go-build -gno-record-gcc-switches"
```

| 变量名         | 值                                       | 含义              |
| ----------- | --------------------------------------- | --------------- |
| GO111MODULE | on                                      | 是否开启go module   |
| GOCACHE     | "/home/lixinyu/.cache/go-build"         | 储存每次go编译的记录文件路径 |
| GOENV       | "/home/lixinyu/.config/go/env"          | 储存go环境变量文件路径    |
| GOPATH      | "/home/lixinyu/go"                      | go工作路径          |
| GOPROXY     | "https://goproxy.cn"                    | go依赖的代理地址       |
| GOROOT      | "/usr/local/go"                         | go安装路径          |
| GOTOOLDIR   | "/usr/local/go/pkg/tool/linux_amd64"    | go tool的路径      |
| GOMOD       | "/home/lixinyu/GolangStudy/test/go.mod" | 当前go.mod所在路径    |

待补充

附录:

[go env参数](https://cloud.tencent.com/developer/article/1650021)