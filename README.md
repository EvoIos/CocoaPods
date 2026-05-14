# CocoaPods - Pre-download 优化版

基于 [CocoaPods](https://github.com/CocoaPods/CocoaPods) 的 fork，优化了 `pod install` 时对外部源依赖的 pre-download 行为。

## 问题

CocoaPods 对使用 `:git` + `:tag`/`:commit`/`:branch` 的依赖，每次 `pod install` 都会执行 pre-download（git clone），即使本地已有正确版本的缓存。这在有大量私有 git 依赖的项目中非常耗时。

## 解决方案

在 `fetch_external_sources` 阶段增加智能缓存判断：

- **所有外部依赖**：如果 lockfile 和 sandbox manifest 的 checkout options 一致且本地文件完整，跳过 pre-download
- **新增 `--force-fetch` 选项**：强制重新下载所有外部源
- **`pod update`**：仍然会强制重新下载

## 修改的文件

| 文件 | 修改内容 |
|------|---------|
| `lib/cocoapods/installer/analyzer.rb` | 新增 `skip_fetch_for_external_source?` 方法，在 `fetch_external_sources` 中判断是否跳过 |
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

# 输出示例：
# Fetching external sources
#   Using cache for `MMKV` (unchanged tag `2.0.2.3`)
#   Using cache for `MMKVCore` (unchanged tag `2.0.2.3`)
#   Using cache for `SomeLib` (unchanged branch `dev`)
```

### 强制重新下载

```bash
# 强制重新下载所有外部源
bundle exec pod install --force-fetch

# 更新特定 pod 到最新
bundle exec pod update MMKV
```

## 跳过条件

以下条件**全部满足**时才会跳过 pre-download：

1. 不是 `--force-fetch` 模式
2. 不是 `pod update`（全量更新）模式
3. 不是正在被 `pod update PodName` 指定更新的 pod
4. Lockfile 和 Sandbox Manifest 都存在
5. Sandbox 中 pod 目录存在
6. Sandbox 中 podspec 文件存在
7. Lockfile 中的 checkout options 与 Sandbox manifest 一致
8. 对于 tag/commit 依赖：Podfile 中的值与 Lockfile 记录一致

任何一个条件不满足，都会回退到标准 pre-download 流程。

## 注意事项

- 首次 `pod install`（无 Podfile.lock）不会跳过任何下载
- 从 `:path` 切换到 `:git` 后第一次需要下载，之后就会被缓存
- 要更新 branch 依赖到最新代码，用 `pod update PodName`
- Tag 被 force push 后，用 `--force-fetch` 或 `pod update PodName`
- verbose 模式（`--verbose`）会显示跳过原因的调试信息
