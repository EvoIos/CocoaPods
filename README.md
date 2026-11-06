# CocoaPods - External Source Fetch 控制版

基于 CocoaPods 1.17.0 的 fork，新增 `--force-fetch`，用于显式地把所有外部源（`:git` + `:branch`/`:tag`/`:commit`）按 Podfile 定义重新拉取，并同步更新 `Podfile.lock`。

## 背景

CocoaPods 的 `pod install` 是"按 `Podfile.lock` 复现安装"，而不是"求解最新"：

- `:tag` / `:commit` 是不可变引用，`pod install` 不会改变它们
- `:branch` 依赖在 `Podfile.lock` 的 `CHECKOUT OPTIONS` 里被钉成具体 commit，远端分支前进不会触发重新拉取
- `pod repo update` 只刷新 spec 仓库（`~/.cocoapods/repos`），对 `:git` 外部源完全无作用

原生要推进 branch 依赖只能 `pod update POD`。本 fork 增加 `--force-fetch`，让一次 `pod install` 即可把所有外部源重新解析一遍。

## 修改的文件

| 文件 | 修改内容 |
|------|---------|
| `lib/cocoapods/installer/analyzer.rb` | `--force-fetch` 下把所有外部源加入 fetch 列表、忽略 lockfile 的 checkout options、并解除版本锁定 |
| `lib/cocoapods/installer.rb` | 新增 `force_fetch` 属性，传递到 Config |
| `lib/cocoapods/config.rb` | 新增 `force_fetch` 配置项 |
| `lib/cocoapods/command/install.rb` | 新增 `--force-fetch` 命令行选项 |

## 使用方法

### 环境准备

```bash
# 1. 安装 Ruby 3.3.6（通过 rbenv，不影响系统 Ruby）
brew upgrade ruby-build
rbenv install 3.3.6

# 2. 在 iOS 项目 Gemfile 中指向本 fork
#    gem 'cocoapods', path: '<本仓库路径>'

# 3. 修改项目 .ruby-version
echo "3.3.6" > .ruby-version

# 4. 注释掉 Gemfile 中的 activesupport 固定版本（与 Ruby 3.3 不兼容）
#    gem "activesupport", "6.1.7.4"  → 注释掉这行

# 5. 删掉旧的 Gemfile.lock 重新解析
rm Gemfile.lock
bundle install

# 6. 让 git 忽略这些本地改动（不影响团队）
git update-index --assume-unchanged .ruby-version Gemfile Gemfile.lock
```

> `git update-index --assume-unchanged` 的作用：告诉 git "假装这些文件没改过"。
> `git status` 不会显示它们，`git add .` 也不会提交它们。纯本地操作，不影响远端。
>
> 恢复：`git update-index --no-assume-unchanged .ruby-version Gemfile Gemfile.lock`

### 正常使用

```bash
bundle exec pod install
```

未变化的 tag/commit/branch 依赖不会重新 clone（CocoaPods 原生行为），branch 依赖保持 `Podfile.lock` 里锁定的 commit。

### 强制重新拉取所有外部源

```bash
bundle exec pod install --force-fetch
```

- `:branch` 依赖 → 按 Podfile 的 branch 重新解析远端 HEAD，并更新 `Podfile.lock`
- `:tag` 依赖 → 重新 clone（tag 被 force push 过也能拿到新内容）
- `:path` 依赖 → 重新登记本地路径

```bash
# 只更新指定 pod（推荐，影响面更小，不会改动其他依赖）
bundle exec pod update POD_NAME
```

## 注意事项

- `pod install --force-fetch` 会改写 `Podfile.lock`（branch 依赖的新 commit），等同于对 git 依赖做了一次 update；它和 `--deployment` 语义冲突，不要一起用
- 版本号来源的 pod（`pod 'X', '1.2.3'`）不受 `--force-fetch` 影响，更新它们仍用 `pod update`
- 只更新单个/部分依赖时优先用 `pod update POD_NAME`
