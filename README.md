# wails-related

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

## 3. 前后端双向通讯

  演示项目：https://github.com/fire988/wails-demo-01.git

## 4. 如何启动浏览器调试

   命令：wails dev -browser