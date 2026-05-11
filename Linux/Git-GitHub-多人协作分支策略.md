# Git / GitHub：多人协作的本质与实操

**企业里多人往 GitHub 推送代码时真正关心的几件事**：谁在改什么、冲突在哪解决、主干怎么保持稳定、以及 Fast-forward 与 No-fast-forward 的差别与用法

---

## 1. HEAD、分支、远程

| 概念 | 本质 |
|------|------|
| **分支** | 一个可移动的「提交指针」，名字指向某次 commit |
| **HEAD** | **当前工作上下文**指向哪里：通常指向「当前检出分支」的最新提交；检出某次提交时也会 detached HEAD |
| **当前分支** | HEAD 所附着的那条分支（`git branch` 里带 `*` 的那条） 所有 **commit 都长在「当前分支」这条线上**，除非用了别的技巧 |
| **远程（如 `origin`）** | 别人的机器上的同名引用；`git push` / `pull` 只是在同步「本地引用」和「远程引用」以及对象库 |

多人协作的本质：**每个人在本地分支上推进提交，通过 Push / Pull Request 把变更汇合到团队约定的「稳定线」上**，并用规则避免多人同时破坏那条线 

---

## 2. 与你笔记对应的常用命令（速查）

```bash
git branch                    # 列出本地分支；* 表示当前分支
git branch 新分支名            # 仅创建分支，不切换
git checkout 分支名            # 切换 HEAD 到该分支（旧写法；等价心智可用 git switch）
git checkout -b 新分支        # 创建并切换
git merge 分支名               # 把「分支名」的历史并入「当前分支」
git branch -d 分支名           # 安全删除（已合并）
git branch -D 分支名           # 强制删除（未合并也会删；丢弃那条线上的未合并工作要谨慎）
```

**删除分支的注意点**：不能删除「当前所在分支」；先 `checkout` / `switch` 到其他分支再删 

---

## 3. 合并冲突：本质与「三板斧」

**冲突何时出现**：两个人改了同一文件的相近区域，或一人删文件另一人改文件，Git 无法自动裁决 

**处理流程（与你笔记一致）**：

1. 打开冲突文件，搜 `<<<<<<<` / `=======` / `>>>>>>>`，人工决定保留内容 
2. `git add <已解决文件>` —— 标记冲突已解决 
3. `git commit` —— 完成合并提交（若是 merge 产生的 commit，有时可用默认信息） 
4. `git push` —— 把结果推到远程（若在 PR 流程里，通常是推到你的功能分支，再在 GitHub 上合并） 

**原则**：冲突解决本身是一次「明确的提交」，不要留下冲突标记就 push 

---

## 4. Fast-forward（FF） vs No-fast-forward（--no-ff）

### 发生了什么（本质）

- **Fast-forward**：当前分支 `main` 的历史线只是「落后于」被合并分支 `feature`，合并时只需把 `main` 指针**向前挪**到 `feature` 的 tip，**不产生新的 merge commit**，历史上看起来像「main 一直线性跟在 feature 后面」 
- **No-fast-forward**（`git merge --no-ff 分支名`）：即使可以 FF，也**强制生成一个 merge commit**，明确留下「这一次是把 feature 合进来」的节点 

### 对比

| 维度 | Fast-forward | No-fast-forward |
|------|----------------|-----------------|
| 历史上是否总有「合并节点」 | 不一定；常常是直线 | 总有 merge commit |
| 可读性 | 历史更「干净」、线性 | **功能边界更清晰**：一眼看出哪次合并带进一整块功能 |
| 回滚 | 有时需要拣选多个 commit |  often 可整体 revert 那次 merge commit（视策略而定） |

### 怎么用（团队约定）

- **小改动、个人分支、习惯线性历史**：允许 FF（GitHub 上 「Squash and merge」也常用来保持 main 线性） 
- **需要保留「release / feature 合并」语义、审计、发布边界**：对重要分支使用 **`--no-ff`**，或在 GitHub 用 **「Create a merge commit」**（等价于产生合并节点） 

企业里常见做法：**main 上的合并方式在团队文档里写死**（例如：默认 squash，hotfix 用 merge commit），避免每人一种历史形状 

---

## 5. 分支策略

### 角色分工（概念）

- **`main`（或 `master`）**：**始终可发布或接近可发布**，只从经过验证的分支合入，不在上面做实验性开发 
- **`develop`（dev）**：日常集成线；功能分支从这里分出、测完再合回 dev，**再由发布流程合入 main** 

很多团队会简化为：**main + `feature/*` + `fix/*`**，用 PR 替代长期 develop；本质不变——**稳定线少动，改动在旁路完成** 

### GitHub 上多人协作的典型流程

1. **从最新约定基准开分支**（通常是 `main` 或 `develop`）：
   - `git fetch origin && git checkout main && git pull`
   - `git checkout -b feature/xxx` 或 `fix/xxx`
2. **本地开发与提交**：小步 commit，推送 **你自己的远程分支**：`git push -u origin feature/xxx` 
3. **开 Pull Request（PR）**：在 GitHub 上指向目标分支（main 或 dev）   
   - **Code Review**、CI 检查、讨论都在这里完成 
4. **同步上游避免巨大冲突**：别人也在往目标分支推；合并前在本地：
   - `git fetch origin && git merge origin/main`（或 `git rebase origin/main`，**团队需统一用哪一种**） 
5. **PR 被批准后合并**：由 Maintainer 或作者按策略点 Merge（merge commit / squash / rebase） 
6. **删除远程功能分支**（可选）：GitHub 上可勾选删除；本地 `git branch -d feature/xxx` 

**本质**：**协作发生在「分支 + PR + Review」上，而不是多人直接往 main 硬推** 

---

## 6. Bug 分支与 `git stash`

**场景**：正在 `feature` 上开发，`main` 线上发现紧急 bug 

推荐顺序：

1. `git stash push -m "wip feature"`（或简写 `git stash`）—— **把工作区（及可选暂存区）的未提交改动收起来** 
2. `git checkout main && git pull`，再 `git checkout -b hotfix/yyy` 
3. 修 bug → commit → **PR 合入 main**（或按规定直接 merge） 
4. 回到功能分支：`git checkout feature/xxx`，`git merge main`（或 `rebase main`，按团队规范）—— **先在功能分支解决与 main 的冲突**，再后续开发 
5. `git stash list` 查看，`git stash pop` 取出（若有冲突同样要手动解决后再 add/commit） 

**注意**：`stash pop` 成功后那条 stash 会删除；若要保留备份用 `git stash apply` 

---

## 7. 强制删除分支（`-D`）

**场景**：开了 `dev` 做功能，半途废弃 

- `git branch -D 分支名`：**不检查是否已合并**，直接删本地分支指针   
- **本质**：未 push 的 commit 若没有其他引用，日后可能被 GC；**已 push 的远程分支**要用 `git push origin --delete 分支名` 才能在远程删掉 

**规则**：`-D` 前确认没有别人依赖该分支；公共分支禁止随意强删 

---

## 8. 企业协作规则与注意事项（清单）

### 流程与权限

- **禁止 force-push 到 `main`/共享长期分支**（除非团队有明确豁免流程） 
- **默认通过 PR 合入 main**，并配置 **分支保护（Branch protection）**：必需 Review、必需 CI 通过、禁止直接 push 
- **合并前本地先拉远程**：减少「我以为基于最新 main」实则不是的情况 

### 冲突与集成

- **优先在自己的功能分支上合并 / rebase `main`**，解决冲突后再 push，**不要把半成品冲突 resolving 留在共享分支上** 
- 团队统一：**merge 还是 rebase**；对外共享分支上慎用 `rebase --force`，以免改写他人已基于的历史 

### 提交与沟通

- Commit message 可读；PR 描述写清「动机、行为变更、风险、如何测」 
- 大改动拆小 PR，降低 review 与冲突成本 

### 安全与仓库卫生

- **不要把密钥、大二进制、私钥** push 到 GitHub；用环境变量与密钥管理服务 
- `.gitignore` 维护好；误推敏感信息需按安全流程轮换密钥，仅靠删除 commit 不够 

---

## 9. 总结

- **HEAD / 当前分支**：决定了你的提交落在哪条线上   
- **`git merge`**：在当前分支上接入另一条线的历史；冲突本地解决后 **add → commit → push**   
- **FF vs `--no-ff`**：要不要留下「这次合并」的显式节点；按团队历史可读性与回滚需求选   
- **main 稳定 + dev 日常**：GitHub 上用 **分支保护 + PR** 强制执行   
- **Bug + stash**：打断当前工作 → 干净切到 hotfix → 回功能分支先对齐 main → `stash pop` 继续   
- **`-D`**：本地强制删分支；远程删分支是另一条命令，且要考虑他人是否仍在使用 

