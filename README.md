# Fincote/debs

Finx 编出来的 `.deb` 存放处。**不是**安装源 —— 见下面「它不能做什么」。

## 为什么单独一个仓库

`.deb` 是构建产物，而且**编一次很贵**：macOS 12 上完整自举要 54 分钟，而 GitHub 已经
移除了 `macos-12` runner，那一份只能由一台真实的旧 Mac 产出来。把这些产物只留在某个人的
机器上，等于把唯一副本押在硬盘上。

放在这里还有两个好处：

1. **按 (Darwin, 架构) 组织。** 每个 `.deb` 都带着 `darwin (>= N-1)` 依赖，所以
   macOS 15 上编的包在 macOS 12 上一个都装不上。分组是硬需求，不是分类癖。
2. **镜像站能原样提供。** 一棵平铺的文件树，`rsync` 就够，不需要理解任何协议。

## 存放形态

一个 (Darwin, 架构) 一个 Release：

```
tag    = darwin<主版本>-<架构>        例：darwin21-x86_64
资产名 = <包>_<版本>-<修订>_darwin-<架构>.deb
```

客户端拼出来的地址：

```
https://github.com/Fincote/debs/releases/download/darwin21-x86_64/dpkg_1.19.7-1.3_darwin-x86_64.deb
                                                      └──────┬──────┘ └──────────┬──────────┘
                                                             tag              资产名
```

每个 Release 还带三个描述文件：

| 文件 | 内容 |
| --- | --- |
| `Packages` | 与 apt 的 `Packages` 同格式，`Filename` 已改成裸文件名（资产是平铺的，没有子目录） |
| `SHA256SUMS` | 每个 `.deb` 的 sha256，`shasum -a 256 -c` 可直接验 |
| `build.txt` | 这份是用哪台机器、哪个 Darwin、哪个版本编的 |

## 三条规则

**一、只增不删。** 已发布的资产永远可下载。删掉一个，就等于让某个版本永久消失。

**二、同名不覆盖。** 资产一旦上传，字节不再改动。要重建同一个版本，**升 `.info` 里的
`Revision`**（`2.1` → `2.2`），文件名随之改变 —— 这与 Fink 的版本机制一致，不需要另造约定。

**三、`Packages` 与 `SHA256SUMS` 跟着资产一起换。** 它们描述的是同一个 Release 里的文件，
分开更新会出现「索引说有、资产没有」。

## 它不能做什么

**不能用来从零安装 Finx。** 装 `.deb` 需要 `dpkg`，而 `dpkg` 本身就在这批 `.deb` 里 ——
鸡生蛋。首次安装仍然走种子或源码自举（见 `Fincote/Finx` 的 `docs/OFFLINE.md`）。

所以这里的作用是：

- 已有的前缀想加包时，不必重新编译；
- 为将来的二进制分发准备素材。

客户端怎么取这些字节（apt 0.5.4 没有 https method，这件事有约束）还没定，见
`Fincote/Finx` 的讨论。本仓库只负责**存得对、存得住**。

## 怎么往里放

产物由 `Fincote/Finx` 的 `scripts/pack_debs.sh` 生成，然后：

```sh
gh release create darwin21-x86_64 --title "debs-darwin21-x86_64" \
    --notes-file build.txt packages/Packages packages/SHA256SUMS packages/*.deb
```

已有 Release 追加包时用 `gh release upload <tag> ...`，不要 `--clobber` 覆盖同名文件。
