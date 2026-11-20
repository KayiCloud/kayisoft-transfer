# kayisoft-transfer

私有 GitLab 构建的 NuGet 包 → 组织 `KayiCloud` 的 GitHub Packages → nuget.org 的中转仓库。

## 为什么需要这一跳

nuget.org 已把 API key 寿命压到最长 30 天，并主推
[Trusted Publishing](https://learn.microsoft.com/nuget/nuget-org/trusted-publishing)。
后者只支持 GitHub Actions 和 gitlab.com 签发的 OIDC token，自建 GitLab 的 issuer
不在支持范围，换不到 nuget.org 的临时 key。

所以多一跳：GitLab 先把包推到本仓库的 GitHub Packages，再 dispatch 本仓库的
GitHub Actions，由它换 key 并推 nuget.org。

## 链路

```
私有 GitLab (.gitlab-ci.yml: publish-github → dispatch-relay)
  ├─ dotnet nuget push  →  https://nuget.pkg.github.com/KayiCloud/index.json   (GH_PAT)
  └─ workflow_dispatch  →  KayiCloud/kayisoft-transfer 的 .github/workflows/ci-cd.yml
                            ├─ 跳过 nuget.org 已存在的版本，删掉 GitHub Packages 副本
                            ├─ 逐个 restore 取包 + NuGet/login@v1 换 1 小时 key
                            ├─ 逐个 dotnet nuget push --skip-duplicate
                            └─ 整批推完、nuget.org 可服务后再删 GitHub Packages 副本
```

包属于**组织级 feed**（granular permissions），不是仓库级：REST 上要用
`/orgs/KayiCloud/packages/nuget/...`，`/repos/{repo}/packages` 对 NuGet 一律 Not Found。

## 两个 CI 文件，别混

| 文件 | 在哪 | 干什么 |
|---|---|---|
| `.github/workflows/ci-cd.yml` | 本仓库 | 收清单、取包、换 nuget.org 临时 key、推送、清副本 |
| `.gitlab-ci.yml` | 私有 GitLab 的 kayisoft 项目 | 打包、推 GitHub Packages（`publish-github`）、把清单 dispatch 过来（`dispatch-relay`） |

GitHub 只扫描 `.github/workflows/` 目录，所以本仓库这份放在那里、叫 `ci-cd.yml`；
GitLab 侧的 `.ci-cd.yml` 在私有项目里，是另一个文件。nuget.org 的 Trusted Publishing
策略绑定的是 `ci-cd.yml` 这个文件名，改名要先改策略。

## 一次性配置

### 1. nuget.org：建 Trusted Publishing 策略

账号 → Trusted Publishing → 新建策略：

| 字段 | 值 |
|---|---|
| Repository Owner / Repository | `KayiCloud` / `kayisoft-transfer` |
| Workflow File | `ci-cd.yml` |
| Scope | `Push new packages and package versions` |
| Package | 包名前缀统一就写 `Kayisoft.*`；很杂就写 `*` |

新建的策略只有 7 天临时激活期，7 天内成功发一次包才转永久；过期后重新激活即可。
仓库从账号迁到组织后，策略里的 Repository Owner 必须同步改成组织名，否则换不到 key。

### 2. 本仓库：配 Actions secrets

Settings → Secrets and variables → Actions：

| Secret | 值 |
|---|---|
| `NUGET_USER` | nuget.org 用户名（不是邮箱） |
| `GH_PAT` | 第 3 步建的 GitHub PAT |

同时确认：

- `FEED_OWNER` 取 `${{ github.repository_owner }}`，迁移后自动是 `KayiCloud`，不用改。
- 默认分支是 `master`，工作流已对齐；GitLab dispatch 的 `ref` 也传 `master`。
- 工作流文件必须已经存在于默认分支上，GitHub 才找得到 `workflow_dispatch` 入口。

### 3. GitHub：建 classic PAT

只认 **classic PAT**：GitHub Packages 的 NuGet feed 不支持 fine-grained PAT。

Settings → Developer settings → Personal access tokens (classic) → Generate new token (classic)

| Scope | 用途 |
|---|---|
| `repo` | dispatch 本仓库的工作流（`KayiCloud/kayisoft-transfer`） |
| `write:packages` | 推包 + 取包（含 `read:packages`） |
| `delete:packages` | 推成功后删 GitHub Packages 副本 |

其余不勾，有效期 90 天。token 的持有者必须是 `KayiCloud` 组织的成员，并对组织 Packages
有写权限（删包需要包的 admin）；组织开了 SSO 还要给 token 授权 SSO。

同一个 token 配置到两处：

- GitLab 项目 CI/CD variables：`GH_PAT`，Masked + Protected。
- 本仓库 Settings → Secrets：`GH_PAT`（工作流取包和删副本用它）。

### 4. GitLab：kayisoft 项目的 .gitlab-ci.yml

两个 job，`publish-github` 先推包，`dispatch-relay` 再分批触发本仓库工作流：

```yaml
publish-github:
  variables:
    GH_OWNER: KayiCloud
  script:
    - dotnet nuget push "$f" --source "https://nuget.pkg.github.com/$GH_OWNER/index.json" \
        --api-key "$GH_PAT" --skip-duplicate

dispatch-relay:
  variables:
    RELAY_REPO: KayiCloud/kayisoft-transfer
    RELAY_REF: master
    BATCH_SIZE: "100"
  script:
    - curl -fsS -X POST -H "Authorization: Bearer $GH_PAT" \
        "https://api.github.com/repos/$RELAY_REPO/actions/workflows/ci-cd.yml/dispatches" \
        --data-binary @payload.json
```

## 包名限制

哪些包发得出去，主要看 nuget.org 策略的 Package glob：

- **nuget.org Package glob**：真正拦人的一层。清单里出现 glob 之外的包名，换不到能推它的
  key，该版本会卡住。
- **GitHub Packages**：NuGet 不要求包名带组织前缀（npm 才要求），名字随便，和中转无关。
- **中转工作流自己的校验**：包 ID 必须匹配 `^[A-Za-z][A-Za-z0-9._-]{0,99}$`，版本必须形如
  `1.2.3` 或 `1.2.3-beta.1`。

工作流这条比 nuget.org 更严：nuget.org 允许数字或下划线开头的 ID。真有这类包，把工作流里的
正则放宽即可。

包名前缀不统一就把策略的 Package 写 `*`，风险靠本仓库权限和分支保护兜底。

## 批量、重跑与并发

- 一次 dispatch 传一个 JSON 数组，工作流在**一次 run** 里循环处理，不是一包一 run。
- 单批上限 500 条（工作流校验），每包约 3~5 秒，几百包 20~40 分钟；临时 key 1 小时够用。
- 已经发过 nuget.org 的版本会先被跳过并清理 GitHub Packages 副本，所以重跑和重复 dispatch
  都安全；全部已发布时连 key 都不会换。
- 某个包取不到或推送失败只记账、跳过，整批跑完后 job 失败；补跑同一份清单即可
  （已成功的会被跳过）。一个坏包不会拖累同批其余 99 个。
- 推成功后删 GitHub Packages 副本，但**要等整批推完**：包之间互相依赖，删早了后面包的
  restore 只能去 nuget.org 找还没索引的新版本（NU1102）。只有 nuget.org 已能取到该版本时才删，
  取不到就留着副本给后续批次用。
- 各批 dispatch 并行跑：GitLab 每批 100 条，批次间的包互不重叠，各自换自己的 1 小时 key，
  不会互相踩；重复版本靠 `--skip-duplicate` 兜底。
- 不要在工作流上加 `concurrency`：同一组 GitHub 只保留 1 running + 1 pending，第 3 批起
  前面的 pending 会被静默取消。
- 单个 job 超过 6 小时会被 GitHub 杀掉；真要超，GitLab 侧拆成几批 dispatch。

## 验证

Actions → Publish to nuget.org → Run workflow：

- 分支选 `master`（工作流只在 master 上执行）。
- `packages` 填 `[{"id":"Kayisoft.Core","version":"1.2.3"}]`。

跑完确认三点：run 绿了、nuget.org 出现该版本、GitHub Packages 里的副本没了。

## 边界

- 不落代码、不改包内容，签名哈希不变。
- 谁能触发工作流，谁就能发任意包；闸门是 nuget.org 策略的包名 glob + 本仓库分支保护。
- 临时 key 1 小时有效，换完立刻推，别拆 job。
- **包可见性**：GitLab 用 PAT 推到组织 feed 的包不会自动关联到仓库，默认仍是私有；
  只有用仓库的 `GITHUB_TOKEN` 在工作流里发布的包才跟随仓库可见性。本流程不需要公有——
  副本在中转结束后就被删掉，正式来源是 nuget.org。若要保留 GitHub Packages 作为公有镜像，
  得去掉工作流里的删副本，并手工把包关联到仓库。
- **迁移到组织后的必改项**：nuget.org 的 Trusted Publishing 策略、GitLab 的 `GH_OWNER` /
  `RELAY_REPO`、本仓库的两个 secret 都要对着 `KayiCloud` 核一遍；包属于组织 feed，
  删包走 `/orgs/KayiCloud/...`。
