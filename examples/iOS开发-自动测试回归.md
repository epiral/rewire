# 场景：iOS 开发每次改代码都要人肉跑测试、等 CI、看日志、回归

> **用户描述**：我是 iOS 开发，日常工作里有大量重复的开发-测试-回归流程。每次改完代码要手动跑测试、等 CI 出结果、看 crash 日志、改 bug、再提交再等。回归测试更烦，每次发版前要手动验一遍核心流程。我想减少人的介入，实现自动验收。

---

## 你现在的问题

> 你不是在做工程判断——你是在当 Xcode 和 CI 之间的人肉轮询器。改一行代码，Cmd+U 等 3 分钟编译，盯着进度条看测试跑完，红了点进去翻失败原因，切到 CI 网页再翻一遍日志，切到 Crashlytics 看崩溃堆栈。真正写代码可能只占你 30% 的时间，剩下 70% 你在等、在翻、在来回切换窗口搬运信息。发版前的回归更绝——打开模拟器，手动点 20 分钟核心流程，祈祷别出 bug。这些事没有任何判断价值，但你每天都在做。

---

## 闭环拆解

| 步骤 | 你现在怎么做 | 谁在干 | 问题 |
|------|------------|--------|------|
| **调研** | 改完代码，Cmd+U 跑一遍单测看会不会挂 | 人肉触发 + 人肉等待 | 意愿瓶颈：每次改一行都要等 2-5 分钟编译+测试，烦到不想跑 |
| **决策** | 肉眼看 Xcode 测试结果，红了点进去看原因；CI 挂了去网页翻日志 | 人肉 I/O | 吞吐量瓶颈：在 Xcode / CI 网页 / crash 平台之间来回切，信息散落三处 |
| **规划** | 脑子里排"先修哪个 test、再改哪个 crash" | 人（可辅助） | 没显性化，靠记忆 |
| **执行** | 改 bug → 跑测试 → 等结果 → 看日志 → 再改 → 再跑 | 人肉循环 | 一个 bug 要走 2-3 轮"改→跑→等→看"，每轮 5-15 分钟 |
| **反馈** | 发版前手动跑核心流程：登录→首页→下单→支付→订单列表 | 100% 人肉 | 意愿瓶颈 MAX：最无聊、最容易漏测、每次发版必做 |
| **复盘** | 缺失 | — | 哪些测试经常挂？哪些模块最容易 regression？没人统计 |

六步里五步人肉，一步缺失。你每天被卡在"等待 + 翻日志"的循环里，真正写代码的时间被反复打断。

---

## Claude Code 怎么驱动这一切

**机制**：Claude Code 运行在你的 Mac 上，跟你用同一套开发环境。核心武器是 Meta 开源的 **idb**（iOS Development Bridge）——一个统一的 iOS 自动化 CLI，能同时管理模拟器生命周期、操作 App GUI、跑测试、截图录屏。Claude Code 读项目目录里的 CLAUDE.md，知道你的项目结构和工作流，然后通过 idb + xcodebuild + gemini-vision 这套组合，替你完成从编译到验收的全流程。

核心工具链：

| 工具 | 角色 | 关键能力 |
|------|------|---------|
| **idb**（[Meta 出品](https://github.com/facebook/idb)，4.9k stars，6 小时前仍在活跃开发） | 主力——iOS 自动化统一入口 | 模拟器管理：`idb boot/shutdown/list-targets`；App 操作：`idb install/launch/terminate`；GUI 自动化：`idb ui describe-all`（读 UI 层级）、`idb ui tap X Y`（点击）、`idb text 'hello'`（输入）、`idb button HOME/LOCK`（物理按钮）；测试：`idb xctest run`；截图/录屏：`idb screenshot`/`idb record`；日志：`idb log` |
| **xcodebuild** | 编译和结构化测试 | `xcodebuild test` 跑单测/UI 测试输出 `.xcresult`；`-only-testing` 只跑相关测试；`-retry-tests-on-failure` 过滤 flaky test |
| **xcrun xcresulttool** | 测试结果解析 | `summary`/`tests`/`test-details`/`metrics`/`compare` 全部输出 JSON |
| **gemini-vision** | 视觉验收 | 截图后"看懂"界面——验证布局、文字、图片是否正常 |
| **atos** | Crash 符号化 | 还原崩溃堆栈到源码行号 |

**故事**：

你改完一个网络层的 bug，保存文件。在终端里跟 Claude Code 说一句："跑一下相关测试。"

它先看你改了哪些文件——`git diff --name-only` 拿到改动列表。发现你改了 `NetworkService.swift`，它知道这个文件关联的测试 target 是 `NetworkTests`。它不跑全量，只跑相关的：

```bash
xcodebuild test -workspace YourApp.xcworkspace \
  -scheme YourApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:NetworkTests \
  -retry-tests-on-failure \
  -resultBundlePath /tmp/test-result.xcresult
```

你继续写下一个功能的代码。两分钟后，它告诉你：

> "NetworkTests 跑完了。14 个 pass，2 个 fail。`testTokenRefresh` 失败：`#expect(response.statusCode == 200)` 实际值是 401。看了一下 `NetworkServiceTests.swift:87`，你改了请求头格式但 mock 里的 expected header 没同步更新。要我帮你改？"

你说："改。" 它更新 mock 数据，再跑一遍测试，全绿。然后帮你提交：`fix(network): update token refresh mock to match new header format`。

**发版回归更爽。** 你跟它说："跑一遍回归。"

它用 idb 接管整个流程。先 `idb boot` 启动一个干净的模拟器，`idb install` 装上最新编译的 App，然后 `idb launch` 启动 App。接下来它像人一样操作——`idb ui describe-all` 读取当前界面上所有元素和坐标，找到登录按钮的位置，`idb ui tap 187 400` 点击，`idb text '用户名'` 输入账号，一步步走完登录→首页→搜索→加购→下单→支付的完整流程。

每一步操作后，它用 `idb screenshot` 截图，然后用 `gemini-vision` "看"截图内容：

> "视觉验收：登录页 — 用户名和密码输入框正常显示，登录按钮可见，布局无异常。首页 — 商品列表加载完成，12 个商品卡片正常渲染。下单页 — 价格显示 ¥299.00，收货地址完整，提交按钮可用。"

这套组合的关键在于：**idb 负责操作，gemini-vision 负责"看懂"**。传统的 XCUITest 只能验证"元素存不存在"，这套组合能验证"界面看起来对不对"——文字截断、布局错位、图片加载失败这些人眼才能发现的问题，现在也能自动捕获。而且不需要提前写测试代码——Claude Code 直接像人一样操作 App。

跑完后它用 `xcresulttool compare` 对比两次回归结果，再看 `metrics` 里的性能数据：

> "回归完成。8 条核心流程全部通过，视觉验收无异常。第 5 条'下单流程'比上个版本慢了 1.2 秒（3.1s → 4.3s），建议排查。"

**Crash 日志也不用自己翻了。** 你把 `.crash` 文件丢到项目目录里，跟它说："帮我看看这个 crash。" 它用 `atos` 符号化堆栈：

> "崩溃在 `CartViewController.swift:142`，`viewDidLayoutSubviews` 里强解包了一个 nil 的 `selectedItem`。从堆栈看，是从后台回来时 `selectedItem` 被释放了。建议改成 `guard let`。要我改？"

全程你就像坐在一个高级工程师旁边——你说"跑测试"、"跑回归"、"看 crash"，它就做。你只在真正需要判断的地方介入。

---

## 你的工作会变成什么样

你改完代码，随手说一句"跑测试"。不用切 Xcode 等转圈。你继续写代码。两分钟后它告诉你结果——全绿继续，有红的它直接告诉你为什么、建议怎么改。

发版前说一句"跑回归"，去倒杯咖啡。回来看结果——它已经像人一样把核心流程走了一遍，每步截图并做了视觉验收。通过了直接发版，没通过它告诉你哪步断了、界面哪里不对。

**每天省下的不只是 1-2 小时，是被反复打断的专注力。** 你可以连续写两小时代码不中断——因为有人替你盯着。

---

## 怎么搭建

### 1. 安装工具

```bash
# idb（Meta 的 iOS 自动化 CLI）
brew install idb-companion               # idb 后端
pip install fb-idb                        # idb CLI
idb list-targets                          # 验证：应能看到可用模拟器

# Xcode 自带工具（无需额外安装）
xcodebuild -version                      # 编译和测试
xcrun xcresulttool --version              # 测试结果解析
which atos                                # crash 符号化

# 视觉验收
gemini-vision --version                   # GUI 截图理解
```

### 2. 准备 UI 测试（可选）

如果你已有 XCUITest，idb 可以直接跑。如果没有也没关系——Claude Code 可以直接用 idb 的 GUI 操作能力来做回归，不需要提前写测试代码。

但长期建议还是把核心流程写成 XCUITest，Claude Code 可以帮你写：

```
claude
# "帮我把登录→首页→下单这条核心流程写成 XCUITest"
```

### 3. 创建 Claude Code 项目配置

在你的 iOS 项目根目录创建 `CLAUDE.md`：

```markdown
# iOS 自动测试 Agent

## 项目信息
- Workspace：YourApp.xcworkspace
- Scheme：YourApp
- 测试 Target：YourAppTests（单测）、YourAppUITests（UI 测试）
- 模拟器：iPhone 17 Pro（iOS 26.2）

## 工具优先级
- GUI 操作和模拟器管理：优先用 idb
- 编译和跑测试：用 xcodebuild（输出 .xcresult 供解析）
- 视觉验收：idb screenshot + gemini-vision

## 文件-测试映射
改了以下文件时，只跑对应的测试：
- Sources/Network/** → NetworkTests
- Sources/Cart/** → CartTests, CartUITests
- Sources/Auth/** → AuthTests, LoginUITests
- Sources/Payment/** → PaymentTests, CheckoutUITests
- 其他改动 → 跑全量测试

## 回归测试流程
1. idb boot <UDID>（启动干净模拟器）
2. xcodebuild build（编译 App）
3. idb install YourApp.app（安装到模拟器）
4. idb launch com.yourapp.bundleid（启动 App）
5. 按以下核心流程操作 GUI（用 idb ui describe-all 读界面 → idb ui tap 点击 → idb text 输入）：
   - 登录：输入测试账号 → 点登录
   - 首页：验证商品列表加载
   - 搜索：输入关键词 → 验证结果
   - 下单：加购 → 填地址 → 提交订单
   - 支付：选择支付方式 → 确认（mock）
   - 订单：查看订单列表 → 订单详情
6. 每步操作后：idb screenshot → gemini-vision 视觉验收
7. 跟上次回归对比：xcresulttool compare

## Crash 分析
- dSYM 路径：~/Library/Developer/Xcode/Archives/ 下最新的
- 用 atos 符号化后给出：崩溃位置、原因、修复建议

## 提交规范
- commit message：<type>(<scope>): <summary>
- 测试全绿才允许提交
```

### 4. 启动验证

```bash
cd ~/your-ios-project
claude

# 第一步：验证 idb 连通
# "帮我启动模拟器，装上 App，截个图看看"

# 第二步：验证测试
# "帮我跑一下全量单测"

# 第三步：验证回归
# "帮我跑一遍回归测试"
```

---

## 元层：它会越来越好

**测试映射自动进化**：每次你手动指定"这次改动要跑这些测试"，Claude Code 会建议更新映射表。几周后它越来越精准地知道"改了这个文件该跑哪些测试"。

**GUI 操作脚本沉淀**：每次回归走过的 idb 操作序列，可以固化成 CLAUDE.md 里的标准流程。下次回归不需要重新探索界面，直接按流程走。随着 App 迭代，流程也跟着更新。

**视觉基线管理**：每次回归的截图保存为基线。下次回归用 gemini-vision 对比新旧截图，发现 UI 变化——是预期改动还是意外回归，一眼看出。

**失败模式积累**：每次测试失败的原因和修复方式积累下来。下次类似失败直接给修复建议。

**回归用例扩展**：线上出了 crash 或用户报 bug，修完后补进回归流程。覆盖面随事故自动增长。

**性能退化早发现**：每次回归记录关键流程耗时，突然变慢在发版前就能发现。
