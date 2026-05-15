
# macOS 上内存分配侧写（Allocation Profiling）所需的权限（Entitlement）

在 macOS 上，必须在签名 App Bundle 时附带 `com.apple.security.get-task-allow` Entitlement 才能让应用程序被 Instruments 的 Allocation / Leak profiler 附载，否则 Instruments 会无法附载到进程，并报错 `Failed to attach to target process`。

具体操作如下：

1. 新建一个 `ent.plist` 文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>com.apple.security.get-task-allow</key>
        <true/>
    </dict>
</plist>
```

2. 为你的 App Bundle 重新签名 Ad-Hoc 签名：

```
codesign --deep --force --sign - --entitlement ent.plist <AppBundle.app>
```

签名成功后即可。

3. 如果要验证 App Bundle 是否确实附带了此 Entitlement，可使用工具 [Apparency](https://www.mothersruin.com/software/Apparency/) 查看。

---

注：

- 来源：https://github.com/cmyr/cargo-instruments/issues/40#issuecomment-894287229
- 此博客编写于笔者尝试使用 Instruments 调查 KiCad 内存泄漏。KiCad 在构建后，其 App Bundle 的 Frameworks 目录中含有一个 `Python.framework` 目录，但在没有 `make install` 时，由于其中并没有完整的 Python 运行时库而只有 pcbnew 的 Python SWIG 动态链接库，所以 `codesign` 会报错

    ```
    $ codesign -s - -f --entitlements ../../ent.plist kicad/KiCad.app                                   
    kicad/KiCad.app: replacing existing signature
    kicad/KiCad.app: bundle format unrecognized, invalid, or unsuitable
    In subcomponent: /Users/rigoligo/contrib/kicad/kicad/build/kicad/KiCad.app/Contents/Frameworks/Python.framework
    ```
    
    解决方法也很简单，只需要先将 `Python.framework` 移出 App Bundle 之外，签名，再移回即可。此目录并不包含一个有效的需要签名的 Framework，安全机制不会拦截，在 Apparency 中也是看不到这个 Framework 的。
