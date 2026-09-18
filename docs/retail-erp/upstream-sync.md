# 私有主库与上游同步

本仓库 **`dingzl2004/XiHan.RetailERP` 是独立的私有主开发库**，不是 GitHub Fork。底座代码来自开源框架 [XiHanFun/XiHan.BasicApp](https://github.com/XiHanFun/XiHan.BasicApp)（MIT），零售业务只在本私有库演进。

公开 fork [`dingzl2004/XiHan.BasicApp`](https://github.com/dingzl2004/XiHan.BasicApp) 继续保留，**不要删除**；也不要把零售功能推回该 fork 或上游。

GitHub 不会把你本机的 `git remote` 存进仓库。每位协作者克隆后都要自己配远程。

## 远程约定

| 远程名 | URL | 用途 |
| --- | --- | --- |
| `origin` | `https://github.com/dingzl2004/XiHan.RetailERP.git` | 私有主库；日常 `push` / `pull` |
| `upstream` | `https://github.com/XiHanFun/XiHan.BasicApp.git` | 只读同步框架；只 `fetch`，再 merge/rebase |

克隆后执行：

```bash
git clone https://github.com/dingzl2004/XiHan.RetailERP.git
cd XiHan.RetailERP
git remote add upstream https://github.com/XiHanFun/XiHan.BasicApp.git
git remote -v
```

已有工作副本、只改远程名时：

```bash
git remote rename origin public-fork   # 若当前 origin 仍指向公开 fork，可先改名备份
git remote add origin https://github.com/dingzl2004/XiHan.RetailERP.git
git remote add upstream https://github.com/XiHanFun/XiHan.BasicApp.git
```

确认结果应类似：

```text
origin    https://github.com/dingzl2004/XiHan.RetailERP.git (fetch)
origin    https://github.com/dingzl2004/XiHan.RetailERP.git (push)
upstream  https://github.com/XiHanFun/XiHan.BasicApp.git (fetch)
upstream  https://github.com/XiHanFun/XiHan.BasicApp.git (push)  # 不要向 upstream push
```

不要执行 `git remote add origin` 时把它设成公开 fork；也不要把本库再「Fork」成 GitHub fork（fork 网络无法保持独立私有主库）。

## 为何是「私有主库 + upstream」

- **私有**：连锁零售业务、门店数据模型、收银与统配设计不能放进公开 fork。
- **独立仓库（非 GitHub Fork）**：GitHub 的 Fork 默认跟着上游网络；私有 fork 对上游组织还有额外限制。独立 `origin` 才能保持 private，同时仍可 `git remote add upstream`。
- **upstream 只读同步**：继续吸收 XiHan.BasicApp 的框架修复与内核能力，而不是复制一份永远分叉的底座。

## 日常开发

- 功能分支从 `origin/main` 拉出。
- 零售 / POS / 统配 / 加盟等业务 **只 `git push origin`**。
- **不要** `git push upstream`。
- **不要** 把零售代码、本目录架构说明里的业务细节、或本私有库 README 私有声明推到 `dingzl2004/XiHan.BasicApp`。

## 从 upstream 同步框架（3～5 步）

在干净工作区（或先 stash / 提交本地改动）操作：

```bash
# 1. 取上游
git fetch upstream

# 2. 回到主线
git checkout main
git pull origin main

# 3. 把上游 main 合进来（历史更直选用 rebase：git rebase upstream/main）
git merge upstream/main

# 4. 解决冲突：优先保留本库零售模块、docs/retail-erp、私有 README 声明；
#    内核/Saas/文档站公共部分按上游，再把本库导航补回去。

# 5. 推私有主库
git push origin main
```

建议单独开分支（例如 `chore/sync-upstream`）先合、跑构建，再合并进 `main`。冲突高发区：`.ai/skills`、`docs/.vitepress/config.ts`、根 README。

## 许可证（MIT）提醒

上游 [XiHan.BasicApp](https://github.com/XiHanFun/XiHan.BasicApp) 与本仓库根目录 [`LICENSE`](../../LICENSE) 均为 **MIT**。

私有化 **不等于** 去掉许可证。分发或基于本仓库再发布时，须保留 MIT 版权声明与许可文本（`Copyright (c) 2021-Present XiHanFun and contributors` 及完整 `LICENSE`）。自有零售业务代码可另附版权说明，但不得删除上游要求保留的声明。
