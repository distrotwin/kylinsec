# 开发指引

这个仓库只做一件事：把麒麟信安桌面操作系统的公开 rpm 源变成可用于软件构建与测试的容器镜像。它不是麒麟信安系统的替代品，回答的是「编出来的东西对不对」而不是「跑起来的系统对不对」。定位边界见 README 的前两节。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引里的元包版本

## 这个仓库放什么

```
kylinsec/
├── buildkit/                     # submodule，钉住一个 commit
├── distros/v6.conf  v3.4.conf    # 一个版本一个，文件名即 DID
├── keys/RPM-GPG-KEY-kylinsec-release   # 信任根，指纹钉在 conf 里
├── .github/workflows/build.yml   # 只定义矩阵、调用 buildkit 的可复用 workflow
├── README.md                     # 面向使用者
├── CLAUDE.md                     # 本文件
└── AGENTS.md -> CLAUDE.md        # 符号链接，不是副本
```

判断改动落在哪边只有一条：**只跟某个系统版本自己的事实有关**（源地址、种子包、ABI 基线、这一版特有的怪癖）就进 `distros/*.conf`；**跟怎么构建、怎么测、怎么发有关**就进 buildkit。第二类占绝大多数。写出了第五种文件就说明有东西该进 buildkit 了。

## 配置文件都有什么

字段完整契约见 `buildkit/docs/distro-conf.md`。这个仓库特有的几项：

- `METHOD=rpmrepo` —— 从远程 rpm 源取材。取材之后的装法与 `rpmmedia`（从 ISO 介质取材）完全共用，两条路径只差取材这一层
- `REPO_BASES` —— 空格分隔的源地址，可用 `$RPMARCH`。多个源合并时**按 EVR 取最新**，所以书写顺序不影响结果
- `RPM_KEY` / `RPM_KEY_FP` —— 信任根。指纹必须钉，不钉的话任何一把 key 都能让「验签通过」成立
- `RPM_DB_BACKEND` —— V6 必须写 `ndb`，V3.4 留空

`IMAGE` 必须按版本唯一（`kylinsec-v6`、`kylinsec-v3.4`），共用会让不同版本的镜像在同一台构建机上互相覆盖，症状是验收报「期望 2.38 实际 2.28」，读起来像构建错了，真因是 tag 撞车。

`EXPECT_*` 不许留空：留空会因「期望空 == 实际空」被判通过，等于这一项验收不存在。

按架构不同的基线必须条件覆写。本仓库当前两个版本的各架构工具链版本实测一致，所以没有基线覆写；但 `v6.conf` 有一处 `REPO_BASES` 的架构覆写——**loong 没有 `update/` 树**，不覆写会在 `repomd.xml` 上 404。

## 必知事实

- **DID 是 `v3.4` 而不是 `v34`。** `v34` 会被读成比 `v6` 更新的版本，而它是上一代。完整版本号 3.4-4A 通过发布时的别名 tag 体现
- **loong 是新世界。** 实测其 glibc 的 ELF 解释器为 `/lib64/ld-linux-loongarch-lp64d.so.1`、依赖里写着 `ld-linux-loongarch-lp64d.so.1(GLIBC_2.36)`，旧世界用 `/lib64/ld.so.1`。**rpm 世界里两个世界都叫 `loongarch64`，名字不携带世代信息**，所以世代判定要落在 `EXPECT_LOADER` 上而不是架构名上
- **ISO 拿不到，源拿得到。** 厂商放 ISO 的 `mirrorlists.kylinsec.com.cn` 对 GitHub runner 全量 403，且真文件与不存在的路径同样 403（即不给存在性信号，本机直连与走代理都一样）。公开 rpm 源 `mirrorlist.kylinsec.com.cn:8888`（主机名单数、端口 8888）则 200/404 分明、秒级可达
- **`os/` 冻结、`update/` 滚动。** V6 的 `os/` revision 是 2025-01-17，`update/` 是 2026-08-19。配上 `update/` 后镜像比 2025-07 那张盘新，方向安全：上游版本全不变，符号版本天花板不动
- **两个版本的更新树层级不同。** V6 在 `6/update/<arch>`，V3.4 在 `3.4-4A/os/<arch>/update`。厂商的目录约定不统一，只能逐版本核过再写
- **种子包名不能照抄。** 信安用 `shadow` 而非 RH 惯例的 `shadow-utils`、用 `pkgconf` 而非 `pkgconfig`；V3.4 没有 `dnf-data`。改种子后必须对着源的 `pkglist` 逐个核，否则会在切档位时才报「依赖未解析」

## 门禁

- 种子包必须逐个在源的 `pkglist` 里存在，**每个架构都要核**
- 取材阶段：公钥指纹必须匹配；每个 rpm 必须通过验签，判据是字面 `signatures OK`——未签名的包 `rpm -K` 会打印 `digests OK` 并返回 0，只看退出码或只找子串 `OK` 会让无签名的包蒙混过关（`NOT OK` 里也含 `OK`）
- 路径型依赖必须全部解析（靠 `filelists`），有残留就失败——留着会让镜像内 `dnf check` 报 missing requires
- `SOURCE_DATE_EPOCH` 取各源 `repomd` 的 revision，构建时断言它非空且晚于 2020。不能落到 `make_tarball` 的兜底常量 `1700000000`，那是假锚点

## 改动到跑通的完整回路

改 conf → 本地跑一个档位（**别拿 CI 当实验台**，一轮几十分钟）→ 本地 `verify.sh` 绿 → 推仓库跑 `publish=false` 的完整 CI → 看报告 artifact 确认零异常 → `publish=true` 发一轮 → 匿名视角验收 registry → 把 README 的数字对齐到这一轮实际结果。

## 本地构建与验证

本机没有 rpm 时，在 builder 容器里跑（它带 `rpm cpio zstd gpg`）：

```bash
docker build --build-arg http_proxy= --build-arg https_proxy= \
  -f buildkit/Dockerfile.builder -t ksbuild:latest buildkit/
docker run --rm --privileged -v "$PWD:/w" -e http_proxy= -e https_proxy= ksbuild:latest \
  bash -c 'ROOT=/w BK=/w/buildkit ARCH=amd64 /w/buildkit/build/build.sh v6 micro'
```

## 跑 CI

`gh workflow run build.yml --repo distrotwin/kylinsec -f publish=true -f include-loongarch=true`。`include-loongarch` 默认开着（新世界能跑），要快速验证时关掉它。

## 发布与验收

发布后清 GHCR 的无 tag 版本。注意日期钉版 tag 是**天粒度**，同一天重复发布会把 tag 移到新 digest，让上一次同日构建变成孤儿。删之前必须先把所有 tag 的 manifest 展开、收集成员 digest 做白名单——多架构 manifest 的各架构成员本身没有 tag，直接删「所有无 tag 版本」会把 tag 好的镜像掏空。

## 排错

先看构建环境记录那一步的输出：本 job 的输入、conf 解出的值、宿主信息都在那里。读日志读不出来时把制品下下来直接看，`docker load` 之后 `tar -tvf` 一览无余——照日志猜是明确记过的弯路。
