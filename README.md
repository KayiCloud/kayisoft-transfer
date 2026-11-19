# kayisoft-transfer

私有 GitLab 构建的 NuGet 包 → 本仓库 GitHub Packages → nuget.org 的中转仓库。

```
私有 GitLab (.ci-cd.yml)
  ├─ dotnet nuget push  → https://nuget.pkg.github.com/<OWNER>/index.json   (GH_PAT)
  └─ workflow_dispatch  → 本仓库 .github/workflows/ci-cd.yml
                            ├─ 跳过 nuget.org 已有的版本，删掉 GitHub Packages 副本
                            ├─ 逐个取包 (restore) + NuGet/login@v1 换 1 小时 key
                            ├─ 逐个 dotnet nuget push --skip-duplicate
                            └─ 推成功后删 GitHub Packages 副本
```

nuget.org 的 API key 现在最长 30 天，官方推 Trusted Publishing；它只认 gitlab.com 签的 OIDC
token，自建 GitLab 不在白名单，所以必须有这一跳。

## 两个 CI 文件，别混

| 文件 | 在哪 | 干什么 |
|---|---|---|
| `.github/workflows/ci-cd.yml` | 本仓库 | 收清单、取包、换 nuget.org 临时 key、推送、清副本 |
| `.ci-cd.yml` | 私有 GitLab 项目 | 打包、推 GitHub Packages、把清单 dispatch 过来 |

GitHub 只扫描 `.github/workflows/` 下的 `*.yml`，点开头的文件名不会被识别，所以本仓库这份叫
`ci-cd.yml`。

## 一、nuget.org 侧（一次性）

1. 账号 → Trusted Publishing → 新建策略
   - Repository Owner / Repository：本仓库
   - Workflow File：`ci-cd.yml`
   - Scope：`Push new packages and package versions`
   - Package：**这一项会限制包名**。前缀统一就写 `Kayisoft.*`；前缀很杂就写 `*`
   - 私有仓库策略只有 7 天临时激活期，7 天内发成功一次包才转永久
2. 本仓库 Settings → Secrets
   - `NUGET_USER`：nuget.org 用户名（不是邮箱）
   - `GH_PAT`：GitLab 推包 + 工作流取包 + 删副本都用它，见下
3. `FEED_OWNER` 取 `${{ github.repository_owner }}`，不用改
4. 本仓库默认分支是 `master`，工作流已对齐；GitLab 侧 dispatch 的 `ref` 也要传 `master`

## 二、包名到底受什么限制

- **nuget.org 策略的 Package glob**：真正拦人的就是这一层。清单里出现 glob 之外的包名，换不到
  能推它的 key。
- **NuGet 包 ID 规则**：最长 100 字符，字母/数字/`.`/`-`/`_`，以字母开头。中转工作流照这个校验。
- **GitHub Packages**：NuGet 不要求包名带组织前缀（npm 才要求），名字随便，和中转无关。

所以：包名前缀不统一就把策略 Package 写 `*`，风险靠本仓库权限和分支保护兜底。

## 三、私有 GitLab 侧要什么

只认一个东西：**GitHub PAT**（必须是 classic，GitHub Packages 的 NuGet feed 不支持 fine-grained PAT）。
GitLab 不需要、也拿不到 nuget.org 的任何密钥。

生成：GitHub → Settings → Developer settings → Personal access tokens (classic) → Generate new token (classic)

| Scope | 用途 |
|---|---|
| `repo` | dispatch 私有仓库的工作流 |
| `write:packages` | 推包 + 取包（含 `read:packages`） |
| `delete:packages` | 推成功后删 GitHub Packages 副本 |

其余一律不勾，有效期 90 天。

- GitLab 项目 CI/CD variables：`GH_PAT`，Masked + Protected
- 本仓库 Settings → Secrets：`GH_PAT`（同一个 token，工作流取包和删副本用它）

classic `repo` 覆盖你账号下所有私有仓库，所以最好用一个只对 relay 仓库有权限的机器账号来建这个 token。

`.ci-cd.yml`（被 `.gitlab-ci.yml` include）：

```yaml
publish_nuget:
  stage: publish
  image: mcr.microsoft.com/dotnet/sdk:8.0
  variables:
    GH_OWNER: OWNER
    FEED: https://nuget.pkg.github.com/OWNER/index.json
    RELAY_REPO: OWNER/kayisoft-transfer
    VERSION: 1.2.3                        # 本次发版版本
  script:
    - dotnet pack ...                     # 你们现有打包步骤，产出 artifacts/*.nupkg
    - |
      for f in artifacts/*.nupkg; do
        dotnet nuget push "$f" --source "$FEED" --api-key "$GH_PAT" --skip-duplicate
      done
    - |
      # 清单：从 nupkg 文件名去掉 .<版本>.nupkg 得到包 ID
      jq -n --arg v "$VERSION" '[]' > manifest.json
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

## 四、批量怎么跑

- 一次 dispatch 传一个 JSON 数组，工作流在**一次 run** 里循环处理，不再一个包一次 run。
- 单批上限 500 条（工作流里校验），每包约 3~5 秒，几百包 20~40 分钟；临时 key 1 小时够用。
- 已经发过 nuget.org 的版本会先被跳过并清理 GitHub Packages 副本，所以**重跑安全**、dispatch 重复触发也安全。
- 推成功后自动删 GitHub Packages 副本，几百个包不会在 Packages 里越堆越多。
- 单个 job 内循环，超过 6 小时会被 GitHub 杀掉；真要超，GitLab 侧拆成几批 dispatch 即可。

## 五、验证

本仓库 Actions 页面手动 Run workflow，`packages` 填 `[{"id":"Kayisoft.Core","version":"1.2.3"}]`：
GitHub Packages 能取到包、nuget.org 出现新版本、Packages 副本被删掉，即通。

## 六、边界

- 不落代码、不改包内容，签名哈希不变。
- 谁能触发本工作流，谁就能发任意包；闸门是 nuget.org 策略的包名 glob + 本仓库分支保护。
- 临时 key 1 小时有效，换完立刻推，别拆 job。
