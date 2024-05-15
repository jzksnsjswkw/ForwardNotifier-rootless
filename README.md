# ForwardNotifier-rootless

仅实现部分功能的 rootless 版 [ForwardNotifier](https://github.com/Greg0109/ForwardNotifier)  
A rootless version of [ForwardNotifier](https://github.com/Greg0109/ForwardNotifier) with only partial functionalities implemented.

## 不支持的功能 / Unsupported Features

-   Set Device as Receiver
-   Blacklisted Apps
-   SSH
-   原项目的`Crossplatform Server`实现 / The original project's `Crossplatform Server` implementation

## 如何使用 / How to Use

https://github.com/jzksnsjswkw/ForwardNotifier-rootless/assets/69855421/1a45c1d9-8665-49ef-846a-69869c3c30bd

<img src='https://github.com/jzksnsjswkw/ForwardNotifier-rootless/assets/69855421/a23188b7-b04d-414b-b429-5f95834395c9' width='300px'/>

相较原版，当你在设置中关闭应用的通知权限，则不会转发该应用的推送  
Compared to the original version, if you disable notification permissions for an app in the settings, its notifications will not be forwarded.

你只能使用`Crossplatform Server`，并在其中填入完整的地址，如`http://ip:port`，数据将以如下`json`发送。需要注意的是，原项目将所有值都采用`base64`编码，而本项目没有，你只需要 base64 解码图片即可（如果有的话）  
You can only use `Crossplatform Server`, and enter the complete address in it, such as `http://ip:port`. The data will be sent in the following `json` format. It is worth noting that the original project encodes all values in `base64`, but this project does not. You only need to decode the base64 encoded images (if any).

```json
{
	"appName": "appName",
	"title": "Test Notification",
	"subtitle": "Test Subtitle",
	"message": "Test Message",
	"icon": "Base64 Image"
}
```

之所以做此改动是因为我原先使用[Pusher](https://github.com/NoahSaso/Pusher)的自定义服务作为推送，但我无法成功编译:(  
The reason for this change is that I was originally using the custom service of [Pusher](https://github.com/NoahSaso/Pusher) for notifications, but I could not compile it successfully :(

你可以按照原项目的指引，配置相应的通知服务到其他设备上，但你需要适配新的格式。这有一个 Go 实现的，在 macOS 上使用[terminal-notifier](https://github.com/julienXX/terminal-notifier)进行推送的程序可供参考。要使用它，你只需要 `brew install terminal-notifier`  
You can follow the original project's guide to configure the notification service on other devices, but you will need to adapt to the new format. Here is a Go implementation for reference, which uses [terminal-notifier](https://github.com/julienXX/terminal-notifier) on macOS for notifications. To use it, you only need to `brew install terminal-notifier`.

```Go
package main

import (
	"encoding/base64"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/exec"
	"strings"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.POST("/", Local)

	if err := r.Run(":25835"); err != nil {
		panic(err)
	}
}

func Local(ctx *gin.Context) {
	s := struct {
		AppName  string `json:"appName"`
		Title    string `json:"title"`
		Subtitle string `json:"subtitle"`
		Message  string `json:"message"`
		Icon     string `json:"icon"`
	}{}
	if err := ctx.ShouldBindJSON(&s); err != nil {
		s := fmt.Sprintf("ShouldBindJSON err: %v", err)
		log.Println(s)
		ctx.String(http.StatusBadRequest, s)
		return
	}

	cmd := []string{"-title", escape(s.AppName), "-message", escape(s.Message)}
	if s.Title != "" && s.Subtitle == "" {
		cmd = append(cmd, "-subtitle", escape(s.Title))
	} else if s.Title == "" && s.Subtitle != "" {
		cmd = append(cmd, "-subtitle", escape(s.Subtitle))
	} else if s.Title != "" && s.Subtitle != "" {
		cmd = append(cmd, "-subtitle", escape(fmt.Sprintf("%s: %s", s.Title, s.Subtitle)))
	}

	if s.Icon != "" {
		switch {
		default:
			b, err := base64.StdEncoding.DecodeString(s.Icon)
			if err != nil {
				log.Printf("decode base64 icon err: %v\n", err)
				break
			}
			f, err := os.CreateTemp("", s.AppName)
			if err != nil {
				log.Printf("CreateTemp err: %v\n", err)
				break
			}
			defer func() {
				f.Close()
				os.Remove(f.Name())
			}()
			if _, err := f.Write(b); err != nil {
				log.Printf("write icon content err: %v\n", err)
				break
			}
			cmd = append(cmd, "-contentImage", f.Name())
		}
	}

	c := exec.Command("/usr/local/bin/terminal-notifier", cmd...)
	fmt.Printf("c.String(): %v\n", c.String())
	if err := c.Run(); err != nil {
		log.Printf("run command %s err: %v\n", c.String(), err)
	}

	ctx.String(http.StatusOK, "OK")
}

func escape(s string) string {
	buf := strings.Builder{}
	buf.WriteByte('"')
	buf.WriteRune('\u200B')
	for _, r := range s {
		switch r {
		case '"', '\\':
			buf.WriteByte('\\')
			buf.WriteRune(r)
		default:
			buf.WriteRune(r)
		}
	}
	buf.WriteByte('"')
	return buf.String()
}

```

## 如何构建 / How to Build

-   安装[theos](https://theos.dev/docs/installation-macos) / Install [theos](https://theos.dev/docs/installation-macos)
-   克隆项目源码 / Clone the project source code
-   在项目根目录执行`make package` / Execute `make package` in the project root directory.

编译完成后，你将在`packages`目录看到构建的`deb`文件  
After compilation, you will find the built `deb` file in the `packages` directory.

## 废话 / Remarks

这是本人第一次接触插件开发，零基础自学`Objective-C`及`Theos`一天时间改出来的，有许多东西尚未完善，例如使用`AltList`替代`AppList`实现黑白名单。欢迎提交 PR :)  
This is my first time developing a plugin. I taught myself `Objective-C` and `Theos` from scratch and made modifications within a day. There are many things that are not yet perfect, such as using `AltList` to implement a blacklist/whitelist instead of `AppList`. PRs are welcome :)
****
