# 麒麟信安桌面操作系统 · 构建与测试镜像

对着麒麟信安公开 rpm 源自举出来的容器环境，用于**软件构建、打包与兼容性测试**。两个桌面大版本、三个档位，覆盖 amd64 / arm64 / LoongArch 新世界，公开在 GHCR。最近一轮 15 个镜像、637 项检查全部通过，零异常。

```bash
docker run --rm ghcr.io/distrotwin/kylinsec:v6-devel \
  bash -c 'grep PRETTY /etc/os-release; ldd --version | head -1; gcc -dumpfullversion'
```

## 这是什么，不是什么

镜像里没有内核——这是所有 Linux 容器镜像的常态，容器共享宿主内核。依赖内核模块的用户态组件（IMA 度量、可信计算）装在镜像里但不生效，只在启动时才有意义的东西（`initramfs-tools`、驱动）也不起作用，`systemd` 有二进制但不是 PID 1。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在麒麟信安上跑的二进制与 rpm；验证依赖闭包；检查产物需要的 glibc / libstdc++ 符号版本目标系统能否满足；复现只在这个系统上出现的编译问题。

**别用它**：当生产运行时基础镜像；复现内核相关行为；当作系统的完整替代品做验收测试。

## 先跑一遍

进容器，写个 A+B，编了跑，再看符号天花板。

```bash
docker run -it --rm ghcr.io/distrotwin/kylinsec:v6-devel /bin/bash
```

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

最后那行是这套镜像最有用的一句：**它直接告诉你产物需要目标系统多新的 glibc**。

## 选哪一个

| 版本 | glibc | gcc | libstdc++ | 架构 |
|---|---|---|---|---|
| `v6` | **2.38** | 12.3.1 | 6.0.30 / `GLIBCXX_3.4.30` | amd64 · arm64 · loongarch64 |
| `v3.4` | **2.28** | 7.3.0 | 6.0.24 / `GLIBCXX_3.4.24` | amd64 · arm64 |

`v6` 是当前在维护的桌面线，也是全线最新的 ABI。`v3.4`（完整版本号 3.4-4A）是上一代，2024-11 起停止更新，留着是因为 glibc 2.28 那一档在现场仍有存量。

**版本标识写 `v3.4` 而不是 `v34`**：后者容易被读成比 `v6` 更新的版本，而它是上一代。厂商的完整版本号是 3.4-4A，但 tag 上只写 `v3.4`——不额外发一套 `v3.4-4a-*` 别名，那会让 tag 数量翻倍而不增加信息。要确认手上镜像对应哪个具体版本，看 `rpm -q kylinsec-release` 或 `cn.internal.repo-bases` 这个 label（它记着取材的源地址，里面有 `3.4-4A`）。

三个档位：`micro` 只有 libc 与 shell，不带包管理器；`base` 加上 `dnf`、`python3`、网络工具；`devel` 再加 `gcc`、`gcc-c++`、`make`、`rpm-build`。

## LoongArch 是新世界

`v6` 的 loongarch64 是**新世界 ABI**，判据落在动态链接器上：

```
/lib64/ld-linux-loongarch-lp64d.so.1     ← 新世界（本镜像）
/lib64/ld.so.1                           ← 旧世界（Loongnix 血脉）
```

其 glibc 的依赖里写着 `ld-linux-loongarch-lp64d.so.1(GLIBC_2.36)`，而上游 glibc 正是 2.36 并入 LoongArch 支持。这一条决定了它能在上游 QEMU 下构建——旧世界还在调 syscall 79/80，上游把这两个调用去掉了，所以旧世界的镜像在托管 runner 上造不出来。

有个命名陷阱值得记：**rpm 世界里新旧两个世界都叫 `loongarch64`**，名字不携带世代信息。想确认手上的镜像是哪个世界，看链接器而不是看架构名。

`v3.4` 没有 loong：厂商源里那一支只有 x86_64 与 aarch64。

## 镜像是怎么造的

两个版本都**不经过 ISO**，直接从麒麟信安的公开 rpm 源 `mirrorlist.kylinsec.com.cn:8888/publicrepo` 解析依赖闭包再装。

不走 ISO 是被迫的，也记在这里省得别人重试：厂商放 ISO 的 `mirrorlists.kylinsec.com.cn`（注意主机名是复数）对 GitHub runner **全量 403**，而且真文件与不存在的路径同样 403——即那台主机不给存在性信号，「探不通」和「不存在」分不开。本机直连与走代理都一样。而公开 rpm 源那台（主机名单数、端口 8888）200/404 分明、秒级可达。

每个版本配两个源：`os/` 是冻结的发布树，`update/` 是滚动的维护树。合并时按 rpm 版本规则取最新，所以书写顺序不影响结果。两个版本的更新树层级不同——`v6` 在 `6/update/<arch>`，`v3.4` 在 `3.4-4A/os/<arch>/update`，厂商的目录约定并不统一。**loong 只有 `os/` 树**，没有维护树。

信任根是厂商公钥，取自官方 Desktop 6 ISO 根目录的 `RPM-GPG-KEY-kylinsec-release`，指纹钉在配置里，每个下载的 rpm 逐个验签。这个源没有 `repomd.xml.asc`，所以信任落在逐包签名上，比只验一个 repomd 更严。

## 镜像与安装介质的关系

镜像的包来自公开 rpm 源，而客户装的是厂商 respin 的安装介质，两者不是同一批构建。拿官方 Desktop 6 ISO（`6-release-250704`，2025-07-04）的包清单与 `v6-devel` 逐包对过账：

- 2710 个共有包名里 **117 项版本不同，只有 5 项上游版本不同**
- `gcc` / `gcc-c++` / `libstdc++` / `libstdc++-devel` / `binutils` / `rpm` / `bash` / `coreutils` / `openssl` / `ca-certificates` / `python3` / `tzdata` / `filesystem` 与介质**逐字符相同**
- `glibc` 只差厂商 release（源 `2.38-47` 对介质 `2.38-54`），**上游 2.38 相同，ABI 底线一致**
- 224 个「介质有而源没有」的包全是桌面应用（`atril`、`audacity`、`caja-actions`），不是构建相关

配上 `update/` 之后镜像在若干包上比那张 2025-07 的盘**更新**（`glibc` 2.38-65、`openssl` 3.0.12-47 对盘上的 3.0.12-15）。这个方向是安全的：上游版本全部不变，符号版本天花板不动，而 `openssl` 那一跳是一大批 CVE 修复。

要与某张具体介质完全一致，只能拿那张 ISO 切片——而那条路在 CI 里走不通，原因见上一节。

## 认出自己在哪个系统上

```bash
cat /etc/kylinsec-release        # 版本自述
rpm -q kylinsec-release          # 发布包版本
rpm -E %{dist}                   # 构建标记：.ks6 / .ky3
```

## tag 与钉版

- 滚动 tag：`v6`、`v6-devel`、`v3.4-base`、`latest`（指向 `v6-devel`）
- 日期钉版 tag：`v6-devel-YYYYMMDD`，内容不变，用于复现

钉版 tag 是**天粒度**。同一天重复发布会把这个 tag 移到新 digest。

## 镜像自带的溯源信息

```bash
docker inspect ghcr.io/distrotwin/kylinsec:v6-devel --format '{{json .Config.Labels}}' | python3 -m json.tool
```

`cn.internal.repo-commit` 是构建时这个仓库的 commit；`cn.internal.build-method` 是 `rpmrepo`；`cn.internal.tier` 是档位；`cn.internal.repo-bases` 是取材的源地址（含厂商的完整版本号）；`cn.internal.rpm-key-fp` 是验签用的公钥指纹。

## 已知的怪癖与期望失败

- **`micro` 档没有包管理器**，这是设计：它的种子里不含 `dnf`，出厂也不写 `/etc/yum.repos.d`
- **rpm 数据库后端是 `ndb`**（V6）。厂商的 rpm 4.18.2 编译时默认就是它；不显式设置的话装完镜像内 `rpm -qa` 返回 0 行而退出码 0，与「空镜像」不可区分
- **厂商公钥的 UID 写的是 `Kylin OS <support@kylinos.com.cn>`**，是麒麟软件的域名而不是信安自己的。这是厂商现状，如实记录
- **同一张 ISO 的两份自述互相矛盾**：`.treeinfo` 写 `variant = Server`，`.kylinsec-info` 写 `name = KylinSec-Desktop` / `installclass = desktop-environment`。桌面 respin 时没同步 treeinfo
- **通用漏洞扫描器对这个系统没有有效覆盖**。trivy 判不出发行版族（`Family: none`），于是没有任何包进入版本比对。SBOM 完整不能推出漏洞扫描有效

## 本地构建

本机没有 rpm 时在 builder 容器里跑（它带 `rpm cpio zstd gpg`）：

```bash
git clone --recurse-submodules https://github.com/distrotwin/kylinsec.git && cd kylinsec
docker build --build-arg http_proxy= --build-arg https_proxy= \
  -f buildkit/Dockerfile.builder -t ksbuild:latest buildkit/
docker run --rm --privileged -v "$PWD:/w" -e http_proxy= -e https_proxy= ksbuild:latest \
  bash -c 'ROOT=/w BK=/w/buildkit ARCH=amd64 /w/buildkit/build/build.sh v6 micro'
```

跨架构构建还要宿主装 `qemu-user-static` 与 `binfmt-support`。LoongArch 的用户态模拟是 QEMU 7.1 才加的，Ubuntu 22.04 只带 6.2。不要引入 `tonistiigi/binfmt` 容器，实测它会破坏本来可用的 binfmt 注册。

## CI

`gh workflow run build.yml --repo distrotwin/kylinsec -f publish=true -f include-loongarch=true`

构建、测试、报告、发布四个阶段。测试在**干净机器**上装载并真正启动镜像——构建阶段的机器状态会掩盖镜像自身的缺陷。最近一轮 15 个镜像、637 项检查：全部通过、零异常、零期望失败。报告与完整日志按系统打包在每次 run 的 artifact 里。

## 仓库结构

```
distros/v6.conf  v3.4.conf   # 源地址、种子包、ABI 基线、这一版特有的怪癖
keys/                        # 厂商公钥，指纹钉在 conf 里
.github/workflows/build.yml  # 只定义矩阵，调用 buildkit 的可复用 workflow
buildkit/                    # submodule，构建与测试的全部实现
```

改动落在哪边只有一条判据：只跟某个系统版本自己的事实有关就进 `distros/*.conf`，跟怎么构建怎么测怎么发有关就进 buildkit。
