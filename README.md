# Fincote/debs

Finx 编出来的 `.deb` 存放处，按 MacWave 的 `Bottle` 形式组织。

**不是安装源。** 它不能用来从零装 Finx —— 见下面「它不能做什么」。

## 存放形态：一个包一个 Release

```
tag    = 包名
资产名 = <包名>@<版本>-<修订>.darwin<Darwin>-<架构>.deb
```

同一个 tag 下，不同平台的构建并存：

```
tag: dpkg
  dpkg@1.19.7-1.3.darwin21-x86_64.deb
  dpkg@1.19.7-1.3.darwin24-arm64.deb
```

客户端拼出来的地址：

```
https://github.com/Fincote/debs/releases/download/dpkg/dpkg@1.19.7-1.3.darwin24-arm64.deb
                                                     └┬─┘ └───────────┬────────────┘
                                                     tag           资产名
```

每个 Release 还带 `SHA256SUMS`（`shasum -a 256 -c` 直接可用）与 `build.txt`
（记录用哪台机器、哪个 Darwin、哪个提交编的）。

### 为什么 Darwin 版本要进文件名

同一个 `dpkg@1.19.7-1.3` 在 macOS 12 与 15 上编出来是**两个不同的载荷**：每个 `.deb`
都带 `darwin (>= N-1)` 依赖，dpkg 会拒绝不符的那一个。只靠 tag 区分不够，因为
同一个包里两个平台要并存。

### 为什么镜像能用 rsync 原样提供

只有一条标准：**域名之后那段相对路径必须一致**。

```
默认：https://github.com/Fincote/debs/releases/download/dpkg/dpkg@1.19.7-1.3.darwin24-arm64.deb
镜像：https://mirror.example.cn/finx/debs/dpkg/dpkg@1.19.7-1.3.darwin24-arm64.deb
                                          └────────── 域名之后完全相同 ──────────┘
```

所以镜像站要的是普通文件树，不是容器 registry 那套按 digest 取字节的接口。
客户端换镜像只改一个环境变量：

```sh
export FINX_DEB_DOMAIN=https://mirror.example.cn/finx/debs
```

## 三条规则

**一、只增不删。** 已发布的资产永远可下载。删掉一个，等于让某个版本永久消失。

**二、同名不覆盖。** 资产一旦上传，字节不再改动。要重建同一个版本，**升 `.info` 里的
`Revision`**（`2.1` → `2.2`），文件名随之改变 —— 这与 Fink 的版本机制一致，不需要另造约定。
上传时不要用 `--clobber`。

**三、字节是构建产物，不是搬运。** 与 MacWave 的 Bottle 不同：那边搬的是上游原样文件，
校验值等于上游公布的 SHA256；这里是我们自己编出来的，所以校验值只能以本仓库的
`SHA256SUMS` 为准。

## 它不能做什么

**不能用来从零安装 Finx。** 装 `.deb` 需要 `dpkg`，而 `dpkg` 本身就在这批 `.deb` 里 ——
鸡生蛋。首次安装仍然走种子或源码自举，见 `Fincote/Finx` 的 `docs/OFFLINE.md`。

也因此，本仓库目前的作用是**先把产物存住**：macOS 12 上的一次完整自举要 54 分钟，
而 GitHub 已经移除了 `macos-12` runner，那一份只能由一台真实的旧 Mac 产出来。只留在
某个人的硬盘上，等于把唯一副本押在硬盘上。

## 现状

**52 个包，两个平台各 52 份，共 104 个 `.deb`。**

| 平台 | 数量 | 来源 |
| --- | --- | --- |
| darwin24 / arm64 | 52 | `Fincote/Finx` 的 Bootstrap CI（`macos-15`） |
| darwin21 / x86_64 | 52 | 一台真实的 MacBook Air（macOS 12.7.6） |

两个平台的构建时间不同步，这是正常的：每次 Bootstrap CI 跑完会刷新 52 个里的
arm64 那批，而 x86_64 那批只在老 Mac 上手工补。要判断某一份是不是最新，
看它的 `Revision` —— 不同步时对不上 `10.9-libcxx/<包>.info` 里的版本号。

`tree` 是最后一对补上的：CI 的「Collect the built packages」原本排在
「Build one package from source」之前，所以装完 `tree` 再也没人去收它 ——
artifact 里少了它，而整轮是绿的。两处都挪到末尾、并把 `if-no-files-found`
从 `warn` 改成 `error` 之后才收全。

## 怎么往里放

产物在 `/opt/finx/finx/10.9-libcxx/stable/main/binary-darwin-<架构>/base/`。改名与上传：

```sh
# 例：把 dpkg 的两个平台放进同一个 tag
gh release create dpkg --repo Fincote/debs --title dpkg --notes-file build.txt \
    dpkg@1.19.7-1.3.darwin21-x86_64.deb dpkg@1.19.7-1.3.darwin24-arm64.deb SHA256SUMS

# 追加新平台时不要 --clobber
gh release upload dpkg --repo Fincote/debs dpkg@1.19.7-1.3.darwin23-arm64.deb SHA256SUMS
```

## 追加一个平台时，两个坑

**一、`SHA256SUMS` 不会自动合并。** 它是资产，不是仓库文件，`gh release upload` 会撞名。
而 `gh release upload` 遇到**任一**资产名冲突就把**整批**回滚 —— 报错只说第一个冲突的名字，
看起来像"只失败了 SHA256SUMS"，实际上 `.deb` 也一个都没进去。

所以顺序必须是「先删旧的 `SHA256SUMS` → 再一次性传 `.deb` + 新 `SHA256SUMS`」，而且删之前
先把已有那份取下来合并：

```bash
p=dpkg
gh release download "$p" --repo Fincote/debs --pattern SHA256SUMS --dir cur --clobber
(cd cur && cat SHA256SUMS) > /tmp/new.sum
shasum -a 256 "$p"/*.deb >> /tmp/new.sum
sort -u /tmp/new.sum > "$p"/SHA256SUMS

# 删旧（见下：要数字 ID）
id=$(gh api "repos/Fincote/debs/releases/tags/$p" \
     --jq '.assets[] | select(.name=="SHA256SUMS") | .id')
gh api -X DELETE "repos/Fincote/debs/releases/assets/$id"

gh release upload "$p" --repo Fincote/debs "$p"/*.deb "$p"/SHA256SUMS
```

**二、两种 ID 不能混用。** `gh release view --json assets --jq '.assets[].id'` 给的是
**GraphQL node ID**（`RA_kwDO…`），而 `gh api -X DELETE .../releases/assets/<id>` 要的是
**数字 ID**。拿 node ID 去删只会得到 `404 Not Found`，且看着像权限问题。要数字 ID 就走
REST：

```bash
gh api "repos/Fincote/debs/releases/tags/<tag>" --jq '.assets[] | "\(.id)  \(.name)"'
```
