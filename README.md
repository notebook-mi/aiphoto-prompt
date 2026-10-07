# AIP 缩略图处理器

让 Windows 资源管理器直接显示 `.aip` 文件里**最新代图像**的缩略图，而不是千篇一律的图标 logo。
**换了文件夹位置怎么办**

`CodeBase` 是绝对路径。把整个 `thumbnail` 文件夹挪走后，重新双击一次 `register.bat` 即可
（它会重写 CodeBase 指向新位置）。
---

## 三步走

**省事版：直接双击 `install.bat`**（编译 + 注册 + 自检一条龙）。

**手动版：**

| 步骤 | 动作                                                         | 成功标志                                           |
| ---- | ------------------------------------------------------------ | -------------------------------------------------- |
| 1    | 双击 **`build.bat`**                                         | 窗口里出现 `[OK] 编译成功：…\AipThumb.dll`         |
| 2    | 双击 **`register.bat`**                                      | 屏幕闪一下（重启资源管理器），紧接着自动跑一遍自检 |
| 3    | 打开任意含 `.aip` 的文件夹，视图切到「**大图标**」或「**超大图标**」 | 直接看到图片缩略图                                 |

> 注册只写 `HKCU`（当前用户），**不需要管理员权限**，也不会改动 `.aip` 的打开方式。
> 想还原：双击 `unregister.bat`，`.aip` 立刻回到默认图标。

出任何问题，先双击 **`verify.bat`**，它会把「组件 / 加载 / 真实容器解析 / 注册表 / 系统开关」五项逐条打勾或打叉，把输出发我即可定位。

---

## 为什么上一版没生效（已修）

上一版有**五处硬伤**，任何一处都足以让缩略图永远出不来：

1. **`IThumbnailProvider` 的 GUID 写错了。**
   系统约定的接口 GUID 是 `e357fccd-a995-4576-b01f-234630154e96`，上一版写成了
   `…-bd01-234f3c5c74c0`。资源管理器是**按约定 GUID** 去 `HKCU\Software\Classes\.aip\ShellEx\`
   下面找处理器的，键名不对就等于没注册。
   → 现在 `.cs` 里的接口 GUID 与 `register.bat` 写的键名一致，且注册时会主动清掉旧错键。

2. **`IStream.Read` 的签名写错了。**
   托管签名里第三个参数是 `IntPtr`（写回实际读到的字节数），上一版写成了 `int[]`，
   于是 **`build.bat` 根本编译不过**，`AipThumb.dll` 从未生成。
   → 现在按 `IntPtr` 正确读写，并且 `build.bat` 会在失败时停下并保留报错原文。

3. **`WindowsBase.dll` / `PresentationCore.dll` 按名字引用必然失败。**
   这两个 WPF 程序集**不在编译器的默认搜索路径**（框架目录里没有，本机也没装
   Reference Assemblies），`/r:WindowsBase.dll` 会直接 CS0006。
   → 现在 `build.bat` 会先到 GAC 里把它们摸出来，用**完整路径**传给编译器；
   万一确实找不到，源码用 `#if WITH_WIC` 把 WIC 那段屏蔽掉，组件照样编得过（只是少掉 WebP）。

4. **脚本编码不对，把报错糊成了乱码。**
   上一版 `.bat` 是 UTF-8 无 BOM，而 cmd 按系统 ANSI（简中 = 936/GBK）解码，
   中文提示全变问号，`[X] 编译失败` 被淹没在乱码里。
   → 现在所有脚本按系统代码页落盘，中文提示正常可读。

5. **`Bitmap.GetHbitmap()` 返回的 DIB 里 alpha 不可靠**（常见结果是缩略图一片黑或全透明）。
   → 改为自己 `CreateDIBSection` 建 32bpp 顶向下 DIB 段，并按 Windows 缩略图约定做 alpha 预乘
   （与官方样例用的 `32bppPBGRA` 一致）。

顺带把解析路径也简化了：**不再解析 JSON**，直接读文件末尾 116 字节的 FOOTER 段表拿图像段
偏移与长度，去掉了 `System.Web.Extensions` 这个重依赖（少一个依赖就少一处编译失败的可能）。

---

## 它怎么工作

`.aip` 是自定义容器，**图像段永远是「最新代」的图像** —— 派生新代次、删除回退、就地编辑
都是原地重写同一个文件，图像段跟着指向当前代。所以处理器不需要任何「取最新代」的逻辑，
按段表读出图像段就是最新代图。

解析只读三个位置，不解析 JSON，没有额外依赖：

```
0x00      16 B   头    'AIP1' + u16 版本 + u16 标志 + u32 manifestLen + u32 footerOffset
...
末尾      116 B  尾    'AIPF' + ver + flags
                       + u32 manifestOff / manifestLen
                       + u32 promptOff   / promptLen
                       + u32 imageOff    / imageLen      ← 图像段偏移与长度
                       + 32 B SHA-256 + 'END!' + 48 B 保留
```

拿到 `imageOff / imageLen` 后按偏移切出原始图像字节 → 解码（GDI+ 主路径，WIC 兜底覆盖 WebP）
→ 等比缩放到资源管理器要的尺寸 → 返回 HBITMAP。

## 注册表写了什么

```
HKCU\Software\Classes\.aip
    PerceivedType            = image                    让资源管理器按图片类文件处理
    Content Type             = application/x-aip
    \ShellEx\{e357fccd-a995-4576-b01f-234630154e96}      ← IThumbnailProvider 约定的键
        (默认)               = {8C7E9F3D-2B5A-4E6D-9A1C-4F7B2D8E5C60}

HKCU\Software\Classes\CLSID\{8C7E9F3D-2B5A-4E6D-9A1C-4F7B2D8E5C60}
    (默认)                   = AIP Thumbnail Provider
    \InprocServer32
        (默认)               = mscoree.dll              托管 DLL 必须挂 .NET 的 COM shim
        ThreadingModel       = Both
        Class                = AipThumb.AipThumbnailProvider
        Assembly             = AipThumb, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
        RuntimeVersion       = v4.0.30319
        CodeBase             = file:///…/AipThumb.dll
```

`Assembly` 里的版本号必须和 DLL 实际版本一致（源码里用
`[assembly: AssemblyVersion("1.0.0.0")]` 钉住），否则 CLR 会因为「程序集版本对不上」拒绝加载
—— `verify.bat` 会专门检查这一条。

---

## 文件清单

| 文件                         | 作用                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| `AipThumbnailProvider.cs`    | 处理器本体（`IThumbnailProvider` + `IInitializeWithStream` + `IInitializeWithFile`） |
| `AipThumb.csproj`            | 给 Visual Studio / MSBuild 用的工程文件（可选）              |
| `install.bat`                | **一键**：编译 → 注册 → 自检                                 |
| `build.bat`                  | 编译，产出 `AipThumb.dll`（用 Windows 自带的 .NET Framework 4.x 编译器 + GAC 里找到的 WPF 引用） |
| `register.bat`               | 注册 + 重启资源管理器 + 清缩略图缓存 + 自动自检              |
| `unregister.bat`             | 卸载，恢复默认图标                                           |
| `verify.bat` / `verify.ps1`  | 自检：组件 / 加载 / 真实容器解析 / 注册表 / 系统开关         |
| `portable/`                  | **便携包**：拷到别的电脑上直接双击 `install.bat` 就能装（详见下节） |
| `AIP-thumbnail-portable.zip` | 便携包的压缩版，整包发给别人用这个                           |
| `_gen_scripts.py`            | 生成上面那些脚本（按系统代码页落盘，避免乱码），同时组装 `portable/` |
| `_check_cs.py`               | C# 静态体检：括号配平、关键 GUID、危险 API 残留              |
| `_sim_provider.py`           | 用 Python 一比一复现 C# 的解析路径，批量验证真实 `.aip`      |

### `install.bat` 引用了哪些文件

```
install.bat
├── build.bat ────────── AipThumbnailProvider.cs      ← 源码
│                         %WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe   ← 系统自带编译器
│                         GAC 里的 WindowsBase / PresentationCore                 ← 有则打开 WIC(WebP)
└── register.bat ─────── AipThumb.dll                 ← build.bat 的产物
                          verify.ps1                   ← register.bat 末尾自动跑它
```

另外两个入口不依赖别的脚本：`unregister.bat`（卸载）、`verify.bat`（只跑 `verify.ps1`）。

**开发目录里这几个文件与部署无关**，便携包里一律不带：`AipThumb.csproj`（给 Visual Studio 的工程文件）、
`_gen_scripts.py` / `_check_cs.py` / `_sim_provider.py`（生成与检查脚本）、`README.md`（本文件）。

---

## 便携包（换台电脑部署）

`thumbnail/portable/`（或 `AIP-thumbnail-portable.zip`）是给「在别的电脑上装一次」用的，
跟开发目录的区别只有两点：

1. **自带预编译的 `AipThumb.dll`** —— 目标机**不需要任何开发工具**，`install.bat` 先验证这个 dll
   能不能被 CLR 加载，能就直接注册；只有 dll 缺失或加载失败时才回退到现场编译。
   想强制重编（比如要打开 WebP 支持）：`install.bat rebuild`。
2. **开头多一句 `chcp 936`** —— 脚本是 GBK 落盘的，换到非简中区域设置的机器上先把控制台代码页
   钉成 936，中文提示才不会变乱码。

拷过去 → 双击 `install.bat` → 完事（不需要管理员，不需要联网）。

`portable/README.txt` 是给最终使用者的中文说明（UTF-8 带 BOM，任何区域设置下都认得）。

---

## 改完 `.cs` 之后建议跑一遍

```bash
python _check_cs.py AipThumbnailProvider.cs     # 结构与关键常量
python _sim_provider.py "../build/**/*.aip"     # 解析路径对 100+ 个真实容器验证
python _gen_scripts.py                          # 重新生成脚本
```

---

## 常见问题

**缩略图还是没出来**

1. 视图必须是「大图标 / 超大图标 / 内容」；「列表」「详细信息」视图本来就不显示缩略图。
2. 文件夹选项 → 查看，确认**没有**勾选「始终显示图标，从不显示缩略图」。
3. 重启一次资源管理器（`register.bat` 已经做了，手动时在任务管理器里重启「Windows 资源管理器」）。
4. 跑 `verify.bat`，看哪一项打叉。

**部分文件是图标，部分正常**

多半是图像格式系统解不了。`verify.bat` 会对样本报「解码失败，系统缺少该格式的编解码器」——
最常见的是 **WebP**。Win11 自带 WebP 解码；Win10 需要装一次商店里的「WebP 图像扩展」，
或者把素材换成 PNG/JPEG 再打包。

**只想临时看看，不想常驻**

`unregister.bat` 一键还原，不留尾巴。


