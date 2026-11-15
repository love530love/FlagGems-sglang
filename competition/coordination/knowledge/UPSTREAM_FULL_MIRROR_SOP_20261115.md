# 上游全量镜像 SOP：独立目录克隆 + 本地逐题分析

> 建立／验证：workbuddy2，2026-11-15 17:5x（GMT+8）。
> 目标：给「PR / 合并记录 / Release 复盘」一份**权威、只读、可随时丢弃重建**的全量基线，
> 不污染正在跑竞赛的主工作区。所有命令均可复算；本档不含任何凭据或鉴权 URL。

## 0. 为什么必须另建目录（三个实测理由）

| # | 主工作区现状 | 后果 |
|---|---|---|
| 1 | `K:/PythonProjects5/FlagGems-sglang` 有 **2500 项未纳入 git 的资产**（台账、ZIP、门禁、真机探针） | 全量 fetch / clone / 分支切换容易扫到或误提交 |
| 2 | 本地 `master` 与 `origin/master` 分叉：`ahead 1 / behind 570`，且历史被 graft 过 | 不宜作为对外引用或发版基线的历史源 |
| 3 | 本地 `v0.1.0` 标签指向 `30cd434…`，**上游真值是 `ef58dc0…`** | 在主工作区 `git fetch --unshallow --tags` 会报 `would clobber existing tag` 并拒收 |

⇒ 独立克隆同时解决：完整历史、权威标签、干净工作树、可丢弃重建。

## 1. 固定路径（后续一律用这两个）

- 主工作区（竞赛资产、写文档）：`K:/PythonProjects5/FlagGems-sglang`
- **只读全量镜像**：`K:/PythonProjects5/flagos-upstream/FlagGems-sglang-full`

⚠ **不要误用** `K:/PythonProjects5/_scratch_flaggems_sglang`：2026-11-15 实测它是
`shallow=true`、`1 commit` 的旧临时克隆，**不是**全量基线；同级 `_probe_sglangFL` 是另一个仓库。

## 2. 一次性建立（2026-11-15 已执行成功）

```bash
mkdir -p K:/PythonProjects5/flagos-upstream
git clone https://github.com/flagos-ai/FlagGems-sglang.git \
        K:/PythonProjects5/flagos-upstream/FlagGems-sglang-full
cd K:/PythonProjects5/flagos-upstream/FlagGems-sglang-full
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'
```

`git clone` 已带全部标签；第二条把 **382 个 PR head**（含未合并 PR）拉成本地引用，
这一步是 `clone` 本身不会给的。

## 3. 验收清单（本轮实测值，重建后逐项对）

```bash
cd K:/PythonProjects5/flagos-upstream/FlagGems-sglang-full
git rev-parse --is-shallow-repository          # false
git rev-list --count HEAD                      # 716
git rev-parse --short HEAD                     # 3503012
git rev-parse v0.1.0                           # ef58dc0a06ee4bdb7191f9ae7d0d96b310532e08
git for-each-ref --format='%(refname)' refs/remotes/origin/pr/ | wc -l   # 382
git status --short | wc -l                     # 0
du -sh .git                                   # 8.3M
git tag -l                                     # v0.1.0 + 3 个 rc.post 标签，共 4
```

## 4. 日常增量同步（只进不出）

```bash
cd K:/PythonProjects5/flagos-upstream/FlagGems-sglang-full
git fetch origin --prune
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'
git reset --hard origin/master     # ⚠ 仅当镜像工作树干净、无本地改动时执行
```

镜像是**只读基线**：不要在镜像里写算子、改代码或建提交；要改代码回主工作区。

## 5. 三类分析分别怎么做

### A. 已合并 PR：git log / git diff（镜像内离线可做）

```bash
# 找某 PR 的 merge commit
git log --format='%H %s' --merges origin/master | grep 'Merge pull request #324 '
sha=8fb0cb446842b4538e1618677656a1f7f60123ef
git diff "$sha^1" "$sha" --stat               # 该 PR 真正引入的内容（相对第一父）
git diff "$sha^1" "$sha" -- '*.py'            # 逐行 diff，用于提取优化 idiom
```

⚠ **坑 1**：`git log --merges` 在本仓只解析出 **268** 个 PR 号，而 GitHub API 的已合并 PR 是 **299**；
差的 31 个是 squash / rebase 合并，**没有 merge commit**。完整清单必须以
`gh pr list --state merged --json number,mergeCommit,additions,changedFiles` 为准，
对无 merge commit 的用 `git show <mergeCommit.oid>` 兜底。

### B. 未合并 PR：PR head 引用（镜像内离线可做）

```bash
git show --stat origin/pr/384                  # 11-14 提交的末批方案，master 里没有
git diff origin/master...origin/pr/384 --stat # 与 master 的差异
```

2026-11-15 实测：58 个 open PR，其中 44 个是算子 PR；最新 `#385`。

### C. 讨论区：只能走 GitHub API（git clone 永远拿不到）

```bash
R=flagos-ai/FlagGems-sglang
gh api --paginate "repos/$R/issues/comments?per_page=100"  > pr_comments.json
gh api --paginate "repos/$R/pulls/comments?per_page=100"   > pr_review_comments.json
gh pr view 324 --repo "$R" --json comments,reviews
```

2026-11-15 实测：全仓 **66 条** PR/issue 评论（54 个 PR）+ **26 条**行内评审（14 个 PR）。
主题分布：CLA 46 / 删除 solution-reference 18 / 正确性测试 14 / 重提关闭 9 / 命名 5 / **性能仅 4**。
⇒ 讨论区值得留档，但**技术手法主要从 diff 读**，别在评论区里找优化思路。

## 6. 本地正确性复现的边界（实测，勿越界声称）

上游公开入口：`tests/`（198 个 .py）、`benchmark/`、`conf/operators.yaml`、
`benchmark/core_shapes.yaml`、`baseline/`、`docs/benchmark_case_api.md`；README 给出
`pytest -q tests/test_silu_and_mul.py --quick` 与
`pytest -q benchmark/test_silu_and_mul.py --level core --iter 1 --warmup 1`。

⚠ **坑 2**：Windows 原生 Python 直接跑会被拦——
`tests/conftest.py:15 import fcntl` ⇒ `ModuleNotFoundError: No module named 'fcntl'`
（本轮以 `D:/A/python.exe` + pytest 8.3.4、torch/triton 均可用仍失败）。
本机已装 WSL Ubuntu（`wsl.exe -l -q` 可见），**正确性测试应在 WSL/Linux 容器里跑**；
本轮未绕过该限制，因此不得声称「本地正确性已复现通过」。

⚠ **坑 3**：本机 3090 的加速比 / 逐芯快慢**不能外推到弱芯**（项目既有定律），
Rank 分与加速比最终以平台判据为准，本地只能做正确性与机制排序。

## 7. 与主工作区的关系 / 引用纪律

- 主工作区本轮也已 `git fetch origin --unshallow`（`shallow=false`，追平 3503012），
  可临时用；但**对外引用、发版基线一律以独立镜像为准**（标签权威、工作树干净）。
- 引用格式：`PR #N <merge sha 前 10 位>`，无 merge commit 时写 `squash commit <oid 前 10 位>`。
- 本档路径与命令可直接复制执行；若镜像被误改，删目录按 §2 重建即可，无需备份。
