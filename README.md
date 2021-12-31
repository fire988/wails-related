# wails-related (本文档仅涉及Windows系统，wails 2.0)
  
     安装命令：go install github.com/wailsapp/wails/v2/cmd/wails@latest

## 1. 关于webview2

- WebView2是什么

    是微软开发的Windows系统的一个控件，方便使用HTML+CSS+JS进行本地桌面应用程序开发。

- 安装

    Windows 7,8,10 默认是未安装的，windows 11默认是安装的。

  >> 检测WebView2是否安装：

        64位程序检查：SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}，
        32位程序检查：SOFTWARE\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}，

        如果上述键值不存在，就是未安装，如果存在，下面代码判断是否安装并获取版本信息（摘自webview2runtime）：

``` go
        // Info contains all the information about an installation of the webview2 runtime.
        type Info struct {
            Location        string
            Name            string
            Version         string
            SilentUninstall string
        }

        // GetInstalledVersion returns the installed version of the webview2 runtime.
        // If there is no version installed, a blank string is returned.
        func GetInstalledVersion() *Info {
            var regkey = `SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`
            if runtime.GOARCH == "386" {
                regkey = `SOFTWARE\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`
            }

            k, err := registry.OpenKey(registry.LOCAL_MACHINE, regkey, registry.QUERY_VALUE)
            if err != nil {
                // Cannot open key = not installed
                return nil
            }

            info := &Info{}
            info.Location = getKeyValue(k, "location")
            info.Name = getKeyValue(k, "name")
            info.Version = getKeyValue(k, "pv")
            info.SilentUninstall = getKeyValue(k, "SilentUninstall")

            return info
        }
```

  >>下载地址：

    https://go.microsoft.com/fwlink/p/?LinkId=2124703

    下载后保存为：MicrosoftEdgeWebview2Setup.exe
 
  >>权限问题：

    安装需要管理员权限，Wails使用了github.com/leaanthony/webview2runtime包，但非管理员用户安装不会成功，需要对这个包进行改造，下面的例子通过runas重启自己从而获得了管理员权限：

``` go
package main

import (
  "fmt"
  "os"
  "strings"
  "syscall"
  "time"

  "golang.org/x/sys/windows"
)

func main() {
  // if not elevated, relaunch by shellexecute with runas verb set
  if !amAdmin() {
    runMeElevated()
  }
  time.Sleep(10 * time.Second)

}

func runMeElevated() {
    verb := "runas"
    exe, _ := os.Executable()
    cwd, _ := os.Getwd()
    args := strings.Join(os.Args[1:], " ")

    verbPtr, _ := syscall.UTF16PtrFromString(verb)
    exePtr, _ := syscall.UTF16PtrFromString(exe)
    cwdPtr, _ := syscall.UTF16PtrFromString(cwd)
    argPtr, _ := syscall.UTF16PtrFromString(args)

    var showCmd int32 = 1 //SW_NORMAL

    err := windows.ShellExecute(0, verbPtr, exePtr, argPtr, cwdPtr, showCmd)
    if err != nil {
    fmt.Println(err)
  }
}

func amAdmin() bool {
    _, err := os.Open("\\\\.\\PHYSICALDRIVE0")
    if err != nil {
      fmt.Println("admin no")
      return false
    }
    fmt.Println("admin yes")
      return true
}
```

  >>编译好的Wails应用程序可以在启动时检测WebView2是否安装，如果未安装，在用户同意的情况下可自动完成下载安装（非管理员用户安装不会成功，但程序可正常启动运行，下次启动还会提示安装），并正常启动程序。

## 2. 手动编译dev版本

     go build -tags dev -gcflags "all=-N -l"
     上述命令在项目根目录下生成一个changeme.exe，可运行，并且有控制台输出，鼠标右键有调试菜单，可以呼出DevTools。

## 3. 前后端双向通讯

     演示项目：https://github.com/fire988/wails-demo-01.git

## 4. 如何启动浏览器调试（不建议，建议使用手动编译dev版本）
     
     命令：wails dev -browser

## 5. 如何调试后端golang代码

    初始化项目的时候，命令行要加上 -ide 选项，如下：
    wails init -ide vscode -n testproj
    然后，就可以在vscode里愉快地调试了。
    注意：ide目前只能指定为vscode，GoLand还不支持。

## 6. 初始化一个React项目

    两个模板可选（推荐第一个）：
    wails init -n "your-project-name" -t https://github.com/flin7/wails-react-template
    wails init -n "your-project-name" -t https://github.com/kamilikamil/wails-react-template

## 7. Win7下不能运行的问题

    wails项目之所以在win7下不能运行，主要是有两个原因造成：
    1、wails项目编译的程序在启动的时候会检查WebView2的安装情况，如果未安装，会提示并可进行安装，但其检查的注册表项在win7下是不存在的。
    2、wails项目程序会使用微软的Shcore库，这个库在win7下是没有的，因此无法运行，但其只调用了这个库的一个API，因此，如果写个同名的shcore.dll文件，导出这个函数，就可以解决运行的问题了。

    FakeShcore项目地址：https://github.com/fire988/FakeShcore.git

    针对第一个原因，如果已经安装了WebView2，需要修正注册表项（实际上是增加），代码：

``` go
package main

import (
  "fmt"
  "os"
  "runtime"
  "strings"
  "syscall"

  "golang.org/x/sys/windows"
  "golang.org/x/sys/windows/registry"
  ffmt "gopkg.in/ffmt.v1"
)

// Info contains all the information about an installation of the webview2 runtime.
type Info struct {
  Location        string
  Name            string
  Version         string
  SilentUninstall string
}

func getKeyValue(k registry.Key, name string) string {
  result, _, _ := k.GetStringValue(name)
  return result
}

// GetInstalledVersion returns the installed version of the webview2 runtime.
// If there is no version installed, a blank string is returned.
func GetInstalledVersion() (*Info, bool) {
  bInCurrUser := false
  var regkey1 = `SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`
  var regkey2 = `SOFTWARE\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`
  
  k, err := registry.OpenKey(registry.CURRENT_USER, regkey2, registry.QUERY_VALUE)
  if err != nil {
    if runtime.GOARCH == "386" {
      k, err = registry.OpenKey(registry.LOCAL_MACHINE, regkey2, registry.QUERY_VALUE)
    } else {
      k, err = registry.OpenKey(registry.LOCAL_MACHINE, regkey1, registry.QUERY_VALUE)
    }
    if err != nil {
      return nil, bInCurrUser
    }
  } else {
    bInCurrUser = true
  }

  info := &Info{}
  info.Location = getKeyValue(k, "location")
  info.Name = getKeyValue(k, "name")
  info.Version = getKeyValue(k, "pv")
  info.SilentUninstall = getKeyValue(k, "SilentUninstall")

  return info, bInCurrUser
}

func isAdmin() bool {
  _, err := os.Open("\\\\.\\PHYSICALDRIVE0")
  if err != nil {
    fmt.Println("not admin")
    return false
  }
  fmt.Println("admin yes!")
  return true
}

func runMeElevated() {
  verb := "runas"
  exe, _ := os.Executable()
  cwd, _ := os.Getwd()
  args := strings.Join(os.Args[1:], " ")

  verbPtr, _ := syscall.UTF16PtrFromString(verb)
  exePtr, _ := syscall.UTF16PtrFromString(exe)
  cwdPtr, _ := syscall.UTF16PtrFromString(cwd)
  argPtr, _ := syscall.UTF16PtrFromString(args)

  var showCmd int32 = 1 //SW_NORMAL

  err := windows.ShellExecute(0, verbPtr, exePtr, argPtr, cwdPtr, showCmd)
  if err != nil {
    fmt.Println(err)
  }
}

func main() {
  if !isAdmin() {
    runMeElevated()
  }

  info, bInCurrUser := GetInstalledVersion()
  if info == nil {
    fmt.Println("error getting WebView2 version info.")
    return
  }

  ffmt.Puts(info)

  if bInCurrUser {
    key, exists, _ := registry.CreateKey(registry.LOCAL_MACHINE, `SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`, registry.ALL_ACCESS)
    defer key.Close()

    // 判断是否已经存在了
    if exists {
      println(`键已存在`)
    } else {
      println(`新建注册表键`)
    }

    key.SetStringValue(`location`, info.Location)
    key.SetStringValue(`name`, info.Name)
    key.SetStringValue(`pv`, info.Version)
    key.SetStringValue(`SilentUninstall`, info.SilentUninstall)
    fmt.Println("创建完成")
  }   else {
    fmt.Println("不需要创建")
  }
}

```

## 8. 窗口不能用鼠标调整大小的问题

    勉强的解决方案：不要使用窗口标题，代码如下：

``` go
func main() {
	// Create an instance of the app structure
	app := NewApp()
	// Create application with options
	err := wails.Run(&options.App{
		Title:             "wails-t1",
		Width:             720,
		Height:            570,
		MinWidth:          720,
		MinHeight:         570,
		MaxWidth:          1920,
		MaxHeight:         1080,
		DisableResize:     false,
		Fullscreen:        false,
		Frameless:         true,
		StartHidden:       false,
		HideWindowOnClose: false,
		RGBA:              &options.RGBA{R: 33, G: 37, B: 43, A: 255},
		Assets:            assets,
		LogLevel:          logger.DEBUG,
		OnStartup:         app.startup,
		OnDomReady:        app.domReady,
		OnShutdown:        app.shutdown,
		Bind: []interface{}{
			app,
		},
		// Windows platform specific options
		Windows: &windows.Options{
			WebviewIsTransparent:  false,
			WindowIsTranslucent:   false,
			DisableWindowIcon:     false,
			EnableFramelessBorder: true,
		},
		Mac: &mac.Options{
			TitleBar:             mac.TitleBarHiddenInset(),
			WebviewIsTransparent: true,
			WindowIsTranslucent:  true,
			About: &mac.AboutInfo{
				Title:   "Vanilla Template",
				Message: "Part of the Wails projects",
				Icon:    icon,
			},
		},
	})

	if err != nil {
		log.Fatal(err)
	}
}
```





