# kayisoft-transfer

私有 GitLab 构建的 NuGet 包 → 本仓库 GitHub Packages → nuget.org 的中转仓库。

## 为什么需要这一跳

nuget.org 已把 API key 寿命压到最长 30 天，并主推
[Trusted Publishing](https://learn.microsoft.com/nuget/nuget-org/trusted-publishing)。
后者只支持 GitHub Actions 和 gitlab.com 签发的 OIDC token，自建 GitLab 的 issuer
不在支持范围，换不到 nuget.org 的临时 key。

所以多一跳：GitLab 先把包推到本仓库的 GitHub Packages，再 dispatch 本仓库的
GitHub Actions，由它换 key 并推 nuget.org。

## 链路

```
私有 GitLab (.ci-cd.yml)
  ├─ dotnet nuget push  →  https://nuget.pkg.github.com/<OWNER>/index.json   (GH_PAT)
  └─ workflow_dispatch  →  本仓库 .github/workflows/ci-cd.yml
                            ├─ 跳过 nuget.org 已存在的版本，删掉 GitHub Packages 副本
                            ├─ 逐个 restore 取包 + NuGet/login@v1 换 1 小时 key
                            ├─ 逐个 dotnet nuget push --skip-duplicate
                            └─ 推成功后删 GitHub Packages 副本
```

## 两个 CI 文件，别混

| 文件 | 在哪 | 干什么 |
|---|---|---|
| `.github/workflows/ci-cd.yml` | 本仓库 | 收清单、取包、换 nuget.org 临时 key、推送、清副本 |
| `.ci-cd.yml` | 私有 GitLab 项目 | 打包、推 GitHub Packages、把清单 dispatch 过来 |

GitHub 只扫描 `.github/workflows/` 目录，所以本仓库这份放在那里、叫 `ci-cd.yml`；
GitLab 侧的 `.ci-cd.yml` 在私有项目里，是另一个文件。nuget.org 的 Trusted Publishing
策略绑定的是 `ci-cd.yml` 这个文件名，改名要先改策略。

## 一次性配置

### 1. nuget.org：建 Trusted Publishing 策略

账号 → Trusted Publishing → 新建策略：

| 字段 | 值 |
|---|---|
| Repository Owner / Repository | 本仓库 |
| Workflow File | `ci-cd.yml` |
| Scope | `Push new packages and package versions` |
| Package | 包名前缀统一就写 `Kayisoft.*`；很杂就写 `*` |

新建的策略只有 7 天临时激活期，7 天内成功发一次包才转永久；过期后重新激活即可。

### 2. 本仓库：配 Actions secrets

Settings → Secrets and variables → Actions：

| Secret | 值 |
|---|---|
| `NUGET_USER` | nuget.org 用户名（不是邮箱） |
| `GH_PAT` | 第 3 步建的 GitHub PAT |

同时确认：

- `FEED_OWNER` 取 `${{ github.repository_owner }}`，不用改。
- 默认分支是 `master`，工作流已对齐；GitLab dispatch 的 `ref` 也传 `master`。
- 工作流文件必须已经存在于默认分支上，GitHub 才找得到 `workflow_dispatch` 入口。

### 3. GitHub：建 classic PAT

只认 **classic PAT**：GitHub Packages 的 NuGet feed 不支持 fine-grained PAT。

Settings → Developer settings → Personal access tokens (classic) → Generate new token (classic)

| Scope | 用途 |
|---|---|
| `repo` | dispatch 私有仓库的工作流 |
| `write:packages` | 推包 + 取包（含 `read:packages`） |
| `delete:packages` | 推成功后删 GitHub Packages 副本 |

其余不勾，有效期 90 天。classic `repo` 覆盖你账号下所有私有仓库，最好用一个只对
relay 仓库有权限的机器账号来建这个 token。

同一个 token 配置到两处：

- GitLab 项目 CI/CD variables：`GH_PAT`，Masked + Protected。
- 本仓库 Settings → Secrets：`GH_PAT`（工作流取包和删副本用它）。

### 4. GitLab：include .ci-cd.yml

```yaml
publish_nuget:
  stage: publish
  image: mcr.microsoft.com/dotnet/sdk:8.0
  variables:
    GH_OWNER: OWNER                          # 改成本仓库所属组织
    FEED: https://nuget.pkg.github.com/OWNER/index.json
    RELAY_REPO: OWNER/kayisoft-transfer
    VERSION: 1.2.3                           # 本次发版版本
  script:
    - dotnet pack ...                        # 现有打包步骤，产出 artifacts/*.nupkg
    - |
      for f in artifacts/*.nupkg; do
        dotnet nuget push "$f" --source "$FEED" --api-key "$GH_PAT" --skip-duplicate
      done
    - |
      # 清单：从 nupkg 文件名去掉 .<版本>.nupkg 得到包 ID
      jq -n '[]' > manifest.json
      for f in artifacts/*.nupkg; do
        id=$(basename "$f" .nupkg); id=${id%.$VERSION}
        jq --arg id "$id" --arg v "$VERSION" '. + [{id: $id, version: $v}]' \
          manifest.json > tmp.json && mv tmp.json manifest.json
      done
    - |
      curl -fsS -X POST \
        -H "Authorization: Bearer $GH_PAT" \
        -H "Accept: application/vnd.github+json" \
        "https://api.github.com/repos/$RELAY_REPO/actions/workflows/ci-cd.yml/dispatches" \
        -d "$(jq -n --slurpfile m manifest.json '{ref:"master",inputs:{packages:($m[0]|tojson)}}')"
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
- 某个包取不到或推送失败会中断该批，剩下的包补跑同一份清单即可（已成功的会被跳过）。
- 推成功后自动删 GitHub Packages 副本，Packages 里不会越堆越多。
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
