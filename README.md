# ZEPHYR 提权（橘子6 · 临时 Root）

通过 Shizuku 拿到 shell(uid 2000)，把内核提权载荷推上去执行，拿到**当次开机有效**的 root，
再把 root 通道交给 KernelSU 等管理器。不依赖电脑。

- 包名：`com.zt.localroot`
- 版本：`OS6-v1.6`（versionCode 63）
- 界面：白/黑双主题，标题下可点「酷安：w97114514」
- 日志：`zt-run-MMDD-HHMM.log`，自动导出到 `/sdcard/Download/`，保留最近 10 份

## 它做了什么

1. 用 Shizuku 的 `newProcess` 起一个 shell(uid 2000) 子进程
2. 把 `assets/` 里的载荷推进 `/data/local/tmp/`
3. 跑提权脚本，逐行解析载荷输出的 stage 标记，界面实时显示进度和停止位置
4. root 通道起来后，按选定的管理器调用 `ksud late-load`

## 编译

需要 JDK + Android SDK build-tools（aapt2 / d8 / apksigner）。路径按自己机器改。

```sh
sh tools/build_shizuku.sh
# 产物：app/out/ZEPHYR-OS6-v1.6.apk
```

## 载荷

`app/assets/` 下四个二进制随仓库一起提供，开箱即可编译出能用的包：

| 文件 | 字节 | sha256 |
|---|---|---|
| `zt_run` | 603608 | `376d8fa69d1eb6b0cda9e95f36bc3131c0d99b0a01d7594d28a32f40a39a7f02` |
| `zt_su` | 603608 | `8cc31cc4fc36a3f5215216f39a2e152256f50c17ef8896deb196ee26001b786a` |
| `zt_fanout.zte` | 1020112 | `50ffe8f0d84e18c05b497ecfb8ffdde1f6843c51a7683e5738d827263f311bef` |
| `zt_su.zte` | 721904 | `d2dd5702b5e27dcc4bd18c104c7734d58ff872a301fb0b2e9b9e4f3f51d258b9` |

`Engine.java` 里的 `SHA256` 表就是上面这四个值，推送前会逐个校验。换载荷要同步改表。

`assets/zt_ksu.sh` 与 `assets/ksud_prep.sh` 是本项目自己的脚本。

## 管理器适配

| 管理器 | 包名 | ksud 调用方式 |
|---|---|---|
| KernelSU | `me.weishu.kernelsu` | `late-load --kmi <kmi> --allow-shell --package-name` |
| BakaSU | `com.resukisu.resukisu` | 同上 |
| SukiSU Ultra | `com.sukisu.ultra` | 参数不同：`late-load --package-name`（KMI 自动探测） |
| FinaSU | `me.yuki.fina` | 同上 |
| KSU-Next | `com.rifsxd.ksunext` | 同上 |

这几个管理器 APK 基本都是 `extractNativeLibs="false"`，so 不解压到 `lib/`，直接找 so 会全部 miss。
本项目改为：用 root 身份把管理器 APK 从 `/data/app` 拷出来，按内容特征
（`ksud::late_load`、`ksud::metamodule`、`kernelsu_patched_`）认出 ksud，
下发到 `/data/local/tmp/ksud`，再逐个试参数组合。日志里会打
`LATE_LOAD_RC` 和 `PKG_OK`，能直接看出哪一组被接受。

## 已知限制

- 载荷按「内核 build 号 + 内存容量」预编译，换机型/换内核要换载荷
- 目前只在 iQOO 13（V2408A，`6.6.127-abogki543627741`）上实测
- 临时 root，重启失效；提权过程中有小概率 panic 重启
- 只做授权设备上的研究，别拿去搞别人的机器

## ⚠️ 关于仓库里的 keystore.jks

`keystore.jks` 是**签名私钥**，本项目按作者要求公开，用于让任何人复现同签名的构建。

公开的影响要说清楚：**任何拿到这个文件的人都能签出与你官方包同包名、同签名的 APK，
安卓会把它当作正常升级装上去。** 也就是说，这个仓库发布之后，你分发的旧版本不再具备
"只能由我更新"的保证。

作者已知悉并接受。若之后要恢复更新链的可信性，只能换新密钥重新签、让用户卸载重装。

## 许可

本项目代码以 MIT 发布，见 LICENSE。

`app/assets/` 下的四个载荷二进制为第三方产物，随仓库一并提供，其权利归各自权利人，
不在本项目的 MIT 授权范围内。