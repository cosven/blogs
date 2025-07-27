---
title: AI编程哪家强？
date: 2025-07-26 10:35:00 +08:00
permalink: /blogs/10024
tags: [AI, coding, Aider, vscode]
categories: [稍微正经点的]
---

最后 AI 编码又火出了新高度，Claude Code、Gemini Cli 等命令行工具相继发布。


## Aider + deepseek-chat-v3-0324
Aider (openrouter/deepseek/deepseek-chat-v3-0324:free)

测试过程摘要
- 我：`解决一下这个问题：{problem_statement}`
- Aider：花了 10s 扫描了一遍整个项目
	> Scanning repo: 100%|███████████████████████████| 2580/2580 [00:09<00:00, 260.12it/s]
- Aider: 一波分析，比较准确的识别了问题所在的文件 `pylint/lint/expand_modules.py` 和 `pylint/lint/pylinter.py`，并且询问我是否要把这两个文件加入到聊天中。
  - 点评：Aider 分析的结果非常准确。可以看下 Aider 是怎样找到这两个文件的。
	> ```
	> Based on the bug report, the issue is with recursive file discovery not properly honoring ignore settings in Pylint. The key
	> files that likely need to be modified are:
	>
	>  1 pylint/lint/pylinter.py - This handles the core linting logic and would manage the recursive file discovery
	>  2 pylint/lint/expand_modules.py - This handles module expansion and file discovery
	>
	> These are the most likely files that need changes to fix the ignore pattern behavior during recursive mode.
	> ...省略一些分析...
	> ...省略一些分析...
	> ```
- 我：连续两个 Enter（代表 yes，把文件加入到聊天中）
- Aider: 输出了它的第一个解决方案
	> I'll help fix the ignore paths behavior in recursive mode. The issue is that _discover_files() in pylinter.py doesn't properly
	> apply the ignore patterns during recursive directory traversal. Here are the needed changes:
	- 点评：上面是 Aider 的一部分分析，Aider 这里又精确的找到了问题所在的函数。
- Aider: 自动执行了 lint，并询问我是否要修复这些 lint 错误
- 我：输入 No
	> 注：根据我过去的使用经验，Aider 的 lint 不一定能很好工作，可能会帮倒忙。几方面
	>   1. 代码库本来可能就没有 lint，它 lint 的时候，报了很多老代码的问题。这时候修复它们不是一个好主意
	>   2. 代码库的 lint 命令和 Aider 的 lint 命令可能不一样。导致结果帮倒忙。
	>
	> 但 Aider 自己在 benchmark 的时候，它在测试的过程中是会[自动执行 lint][swe-bench-lite] 的。
- 我：在另外一个终端中执行一下老的测试用例集(PASS-TO-PASS)，发现有两个之前能跑过的测试，现在跑不过了。于是把错误摘要信息贴给它。
   	> 有两个测试跑失败了：FAILED tests/test_self.py::TestRunTC::test_recursive - AttributeError: 'PyLinter' object has no attribute '_ignore_paths'
   	> FAILED tests/test_self.py::TestRunTC::test_recursive_current_dir - AttributeError: 'PyLinter' object has no attribute '_ignore_paths'
- Aider：又是一波操作
- Aider：又问我是否 lint
- 我：输入 No
- 我：再次执行一遍测试用例集，发现又有了新的报错。仍然简单的把错误贴给它。
	> 这两个测试还是报错，而且还是报类似的错，变量没有初始化：FAILED tests/test_self.py::TestRunTC::test_recursive - NameError: name '_is_in_ignore_list_re' is not defined
	> FAILED tests/test_self.py::TestRunTC::test_recursive_current_dir - NameError: name '_is_in_ignore_list_re' is not defined
- Aider：还是一波操作，修改了一波代码。接着就是问我是否 lint（我是不）
- 我：再次执行测试用例。发现这下测试都跑不起来了。由于错误信息不多，我就把错误信息完整贴给了它。
	> 现在测试都跑不了了： pytest tests/test_self.py
	> /Users/cosven/code/pylint/.venv/lib/python3.8/site-packages/_pytest/config/__init__.py:331: PluggyTeardownRaisedWarning: A plugin raised an exception during an old-style hookwrapper teardown.
	> Plugin: helpconfig, Hook: pytest_cmdline_parse
	> ConftestImportFailure: ImportError: cannot import name '_is_in_ignore_list_re' from partially initialized module 'pylint.lint.utils' (most likely due to a circular import) (/Users/cosven/code/pylint/pylint/lint/utils.py) (from /Users/cosven/code/pylint/tests/conftest.py)
	> For more information see https://pluggy.readthedocs.io/en/stable/api_reference.html#pluggy.PluggyTeardownRaisedWarning
	>   config = pluginmanager.hook.pytest_cmdline_parse(
	> ImportError while loading conftest '/Users/cosven/code/pylint/tests/conftest.py'.
	> tests/conftest.py:15: in <module>
	>     from pylint.lint import PyLinter
	> pylint/lint/__init__.py:21: in <module>
	>     from pylint.lint.parallel import check_parallel
	> pylint/lint/parallel.py:16: in <module>
	>     from pylint.lint.utils import _patch_sys_path
	> pylint/lint/utils.py:16: in <module>
	>     from pylint.lint.expand_modules import get_python_path
	> pylint/lint/expand_modules.py:8: in <module>
	>     from pylint.lint.utils import _is_in_ignore_list_re
	> E   ImportError: cannot import name '_is_in_ignore_list_re' from partially initialized module 'pylint.lint.utils' (most likely due to a circular import) (/Users/cosven/code/pylint/pylint/lint/utils.py)
- Aider：又是一波一样的操作。
- 我：再次执行测试用例。发现老的测试用例都可以跑过了
	- 点评：Aider 这里虽然解决了问题，但我人肉看了，它的解决方案不是最佳（它新增了一个文件来解决循环引用的问题，但其实不需要）。
- 我：用 swe-bench 里面用来评测这个修复的新测试用例(FAIL-TO-PASS)也跑了一遍，发现新测试用例也可以跑过了。至此，Aider 算是解决了这个问题。


[swe-bench-lite]: https://aider.chat/2024/05/22/swe-bench-lite.html

## Gemini Cli（第一轮测试）

测试过程摘要
- 我：`解决一下这个问题：{problem_statement}`
- Gemini: 上来说自己想按照问题描述的步骤来复现一下。问我是否同意？
- 我：不用复现了，直接修复吧 (我思考了一下，感觉复现比较复杂，所以让它直接修复。)
- Gemini：
	> 我将首先搜索代码库，找到处理文件收集和递归逻辑的部分，然后进行修复。我推测问题可能出在 pylint/lint/discover.py 文件中。
 	> ✔  FindFiles 'pylint/lint/discover.py ---> No files found
	> ✦ 好的，看来我猜错了文件名。
	>
	> 我将通过在代码库中搜索 recursive 来定位处理文件发现和递归逻辑的相关代码。这应该能让我找到正确的文件并进行修复。
	> ✔ SearchText 'recursive' within pylint ---> Found 32 matche(s)
	> pylint/lint/pylinter.py 中的 if self.config.recursive: 是一个很强的线索。我将检查该文件，了解是如何发现文件的，以及 ignore 逻辑是如何（或没有）被应用的。
	>
	> 我将阅读 pylint/lint/pylinter.py 的内容。
	> ✦ 我仔细阅读了 pylint/lint/pylinter.py 的代码。
	>
	> 问题似乎出在 _discover_files 方法中。当 recursive 标志被设置时，这个方法负责遍历目录并找到要检查的 Python 文件。
	> ...省略一些代码...
	> ...省略一些代码...
	> ...省略一些代码和一些分析...
	> 询问我是否同意修改？
- 我：同意
- Gemini: apply 了上面的修改建议，并问我是否需要运行 pytest 命令
- 我：不需要，并自己运行 swe-bench 中指定的测试用例 (PASS-TO-PASS)。
	- 点评：和 Aider 问我要不要跑 lint 类似，跑 pytest 有很多不确定性。我自己跑 pytest 肯定只跑特定的。
- 我：上面的测试用例都跑过了。尝试用 FAIL-TO-PASS 中的测试用例集合来跑一下。
- 我：发现这次测试的基准代码有问题。用了之前 Aider 修复后的代码。于是中断了这次测试。

## Gemini Cli (第三轮测试)

这里直接跳过了 Gemini Cli 的第二轮测试，是因为我在第二轮测试时，感觉 Gemini 似乎聪明了很多，
于是我找了下它的缓存，把缓存删了一把 `rm -rf ~/.gemini/tmp/xxxhashxxx/`。
删了之后，第三轮测试看起来是正常的。下面是测试记录

- 我：`解决一下这个问题：{problem_statement}`
- Gemini：（一波思考，过程忽略）
	- 备注：思考了两分钟，还在思考，我直接让它停止了。ℹ Request cancelled.
	- 点评：Gemini 和 Claude Code 在交互上有点相似，会把它当前在干什么动态展示出来，这个在体验上是个加分项。但是要补充的是，它展示的信息基本上就是一扫而过，并且我看不到过程记录，又有点“展示了个寂寞”的味道。
- 我：`解决一下这个问题：{problem_statement}`
	- 备注：由于它上一轮一直在思考，我给它 cancel 了，于是重新把问题发给它
- Gemini：
	> ```
	>  我将按以下步骤来修复这个问题：
	>  ...省略一些 Gemini 的分析...
	>
	>  首先，我需要浏览一下 Pylint 的代码，以便更好地了解项目结构。我将从 pylint/lint 目录开始，因为这里很可能包含了核心的 linting 逻辑。
	> ✔  FindFiles 'pylint/lint/**/*.py' ---> Found 10 matching file(s)
	>✦ pylinter.py 和 expand_modules.py 这两个文件看起来最可疑，一个负责 PyLinter 的主体逻辑，另一个负责模块的发现和过滤。
	>
	> 我先看看 pylint/lint/expand_modules.py，了解下它在递归模式下是如何搜寻文件以及应用 ignore 规则的。
	> ✔  ReadFile pylint/lint/expand_modules.py
	> 我注意到 PyLinter 类中有两个相关方法：_discover_files 和 _expand_files。
	>  ...省略一些 Gemini 的分析...
	>  ...省略一些 Gemini 的分析和修改思路...
	> 问我是否接收它的修改提议，问了好几个问题
	> ```
	- 点评：这一次 Gemini 比第一次测试的时候，表现得更聪明一些。思考两分钟还是有点效果的 :)
- 我：全部回答 yes（除了有一个问题：它想帮我加测试，这个我拒绝了）
- 我：手动执行一遍测试用例(PASS-TO-PASS)，发现报错
	> 有两个测试跑失败了：FAILED tests/test_self.py::TestRunTC::test_recursive - AttributeError: 'PyLinter' object has no attribute '_ignore_paths'
    > FAILED tests/test_self.py::TestRunTC::test_recursive_current_dir - AttributeError: 'PyLinter' object has no attribute '_ignore_paths'
	- 点评：这个报错和 Aider 第一次修复的报错很相似
- Gemini: 又来了一波修复，并询问我是否接受
	- 点评：Gemini 会把一次改动拆分成很多步骤，会提很多问题。而 Aider 则是一波流。
- 我：全部接受。然后执行(PASS-TO-PASS)。发现测试运行不了了。我把测试直接贴给了 Gemini
- Gemini: 又问了我三个问题
- 我：全部接受。
- 我：手动执行一遍测试用例(PASS-TO-PASS)，发现测试都可以过。然后执行(FAIL-TO-PASS)，发现测试用例都可以跑过了。

```
│  Cumulative Stats (4 Turns)      │
│                                  │
│  Input Tokens           474,114  │
│  Output Tokens            3,256  │
│  Thoughts Tokens         12,165  │
│  Cached Tokens  311,020 (63.5%)  │
│  ──────────────────────────────  │
│  Total Tokens           489,535  │
│                                  │
│  Total duration (API)    3m 32s  │
│  Total duration (wall)  20m 49s  │
```

## Claude Code + deepseek-chat-v3-0324 + deepseek-r1-0528

穷，使用 CCR 来为 Claude Code 提供模型 + API 服务
```json
  "Router": {
    "longContext": "openrouter,deepseek/deepseek-chat-v3-0324:free",
    "background": "openrouter,deepseek/deepseek-chat-v3-0324:free",
    "think": "openrouter,deepseek/deepseek-r1-0528:free",
    "default": "openrouter,deepseek/deepseek-chat-v3-0324:free"
  }
```

- 我：`解决一下这个问题：{problem_statement}`
- Claude Code: （思考了几十秒之后，就停止了）
	> ⏺ Update Todos
	>   ⎿  ☐ Search for the implementation of recursive file handling in Pylint to understand how ignore patterns are applied
	>      ☐ Investigate how ignore paths are processed in Pylint's codebase
	>      ☐ Propose a fix for the issue by modifying the relevant Pylint files
	>      ☐ Test the proposed fix to ensure it resolves the issue
	>
	> ⏺ <｜tool▁call▁begin｜>function<｜tool▁sep｜>Task
	>   json
	>   {"description": "Investigate ignore paths processing", "prompt": "Investigate how ignore paths are processed in Pylint's codebase, focusing on the handling of --ignore,
	>   --ignore-paths, and --ignore-patterns settings."}
	>
- 我：写代码解决问题呀
- Claude Code: （还是给我弹出来 tool_call_begin 的块，估计是模型和它不兼容，只能放弃）

试了几次，后面都卡“tool 调用不了”这个问题上了。有点可惜。

## Trae + Builder(Auto)

- 我：`解决一下这个问题：{problem_statement}`
- Trae(Builder):
	> 我需要查看Pylint的源代码，找出递归模式下忽略文件的实现逻辑，以解决忽略设置在递归模式下不生效的问题。
	>    Search codebase for 'recursive mode ignore files implementation'
	> ...省略它的若干代码搜索...
	> ...省略它的若干代码搜索...
	>
	> 我需要查看更多关于Pylint的文件收集和忽略规则实现的代码，特别是 expand_modules.py 文件，因为之前的搜索结果提到这个文件包含了与忽略列表和正则表达式匹配相关的关键函数。
	> 我需要查看Pylint的递归模式实现，特别是与命令行参数处理相关的部分。我将查看 pylint/lint/run.py 文件，因为它可能包含命令行参数处理和递归模式的实现。
	> ...省略它的若干文件浏览...
	> ...省略它的若干文件浏览...
	>
	> 我发现了问题所在。...blabla省略若干分析...
	> 最后它改了一行代码，并且加了一个测试用例
	- 点评：Aider 和 Gemini 基本都在第一轮就分析了问题对应的代码，而它没有。
	  它的行为和 Gemini 很相似，都是一波搜索加分析。可惜的是它分析的结果不对。
- 我：接受修改一行的修改，但决绝了添加测试用例
- Trae(Builder): 给上面的修改来了一波总结
- 我：（我知道修改一行肯定不行，于是直接告诉它不对）我大概看了下代码，你这个修复方法不对
- Trae(Builder): 开始发你，并且又进行了一波分析（这里就省略了），最后又改了几行代码
- 我：跑了一遍(PASS-TO-PASS)测试，也是两个用例失败了，然后就让它修改
- Trae(Builder): 查看了测试文件，然后又是一波分析，然后修复了一把
	- 点评：我发现 Trae 特别喜欢搜索代码、查看文件，而且把这些过程展示给我，我觉得这个非常繁琐。更无语的是它搜的东西都不对（要是搜对了，你展示一下，我可能还能开心一点）。
- 我：跑了一遍(PASS-TO-PASS)测试，过了。然后跑了一遍(FAIL-TO-PASS)测试，4个用例失败了。把4个用例贴给它。
- Trae(Builder): 修改了，并且持续思考，然后 Trae 报错了
	> Model thinking limit reached, please enter 'Continue' to get more.
	- 点评：我看了它改的代码，感觉它不行，基本陷入到它自己思维里面去了。感觉是 Auto 的模型不太行。

## Trae + Builder(Claude-4)

- 我：`解决一下这个问题：{problem_statement}`
- Trae：In queue：Too many current requests. Your queue position is 207.
	- 点评：吐了。难顶。没钱办事是真的难。还好的是这个 position 会实时刷新，让我能看到一丝希望。

Cursor 都测试完了，这个队还在排，才排到了 100+，吐了。直接换 Deepseek 吧。

## Cursor + Auto

- 我：`解决一下这个问题：{problem_statement}`
- Cursor: 和 Trae 有点像（有点倒反天罡），一顿 grep 加查看文件，最后得出结论
	> 问题根因
	> --recursive=y 时，Pylint 通过 _discover_files 递归遍历目录，但没有应用 ignore/ignore-patterns/ignore-paths 相关过滤，导致所有文件（包括应被忽略的）都被 lint。
	> 但在非递归模式下，Pylint 会通过 expand_modules，而该函数会正确应用 ignore 相关选项。
	> ...一堆分析...
	> ...一堆分析...
	> ...最后提出了一个代码修改方案，并问我接不接受...
	- 点评1：Cursor 的这个 grep 展示就比较简洁，我比较喜欢。
	  Trae 生怕我啥也不懂，把模型的弱智全给我展示出来 :)
	  我想大部分程序员，看到这个 grep keyword 就懂它在干嘛。
	- 点评2: cursor 这个根因分析也比较到位，和 aider/gemini 类似，很快定位到对应的两个函数。
- 我：行
- Cursor：修改了代码，并让我确认
- 我：在 UI 上点了 accept，并且手动在终端执行了一波(PASS-TO-PASS)测试。结果有两个用例失败了。
	> 有两个测试失败了：FAILED tests/test_self.py::TestRunTC::test_recursive - AssertionError: expected output status 0, got 1. Below pylint output:
	> FAILED tests/test_self.py::TestRunTC::test_recursive_current_dir - AssertionError: expected output status 0, got 1. Below pylint output:
	- 点评：有趣的是，这些 AI 改了之后，失败的用例都是这两个
- Cursor: 从“失败原因分析”，到“解决思路”，最后“修复建议”
- 我：行
- Cursor：应用了上面这个修复建议对应的代码
- 我：重跑(PASS-TO-PASS)测试，结果失败的用例更多了，有4个。
	- 点评：我看了它的代码修改，它的代码修改是有问题的。
- Cursor: 又来了一个新的方案
	- 点评：我看了代码，知道它是不行的。算了，我感觉它也没救了。

## Trae + Builder(DeepSeek-V3-0324)

- 我：`解决一下这个问题：{problem_statement}`
- Trae：两次 grep 加两个文件查看，定位到了问题。并且给出了代码修复的建议。
	> I've analyzed the issue with Pylint not respecting ignore patterns in recursive mode. The problem occurs because the _discover_files method in pylinter.py doesn't apply the ignore patterns when recursively discovering files, while the ignore patterns are only applied later in _expand_files .
	- 点评：指定了一个模型之后，这个问题定位能力感觉变强了一点点。可惜这里不全对。
- 我：接受。并且手动去跑一下(PASS-TO-PASS)测试。也是两个测试报错了
	> 有两个测试报错了：FAILED tests/test_self.py::TestRunTC::test_recursive - NameError: name 'self' is not defined
	> FAILED tests/test_self.py::TestRunTC::test_recursive_current_dir - NameError: name 'self' is not defined
- Trae: 好家伙，它直接建议我修改我的测试用例。
- 我：我教育了一下它（其实我还挺怕教育它的，一般来说模型是纠正不过来的）
	> 不要改测试用例，测试没通过，说明你的解决方案、或者代码有问题。
- Trae: 表示理解我的诉求，并尝试继续改动 `_discover_files` 函数
- 我：接受。并且手动去跑一下(FAIL-TO-PASS)测试。结果这两个用例还是失败了。
	- 点评：我感觉它已经没戏了。除非我给它一些额外的 PROMPT。

经过这次测试，我感觉模型的影响特别明显，于是尝试用 claude-sonnet-3.7 来测试一把。
claude-sonnet-3.7 第一波分析的方向也不完全对，和 deepseek 差不多，就不继续测了。

再换一个 gemini-2.5-pro 试试，这样可以和 gemini-cli 对比。但看着也不对。

测试过程中，发现 trae apply 代码有点慢，肉眼可见的那种慢。

## Claude Code + Qwen3-Coder-480B-A35B-Instruct

朋友说，它用 Claude Code + Qwen3 是可以的，于是我又测试了一把
```json
    "Router": {
        "default": "xxx,Qwen/Qwen3-Coder-480B-A35B-Instruct"
    }
```

- 我：`解决一下这个问题：{problem_statement}`
- Claude Code: 经过一波 "Considering Spelunking Philosophing Conjuring Musing Forging Wizarding ..."，给出了它的思路
	> ```
	> ⏺ I'll help you fix the issue where Pylint doesn't respect ignore settings in recursive mode. Let me first understand the codebase structure and locate the relevant files.
	>
	> ⏺ Search(pattern: "**/*.py")
	>   ⎿  Found 100 files (ctrl+r to expand)
	>
	> ⏺ Let me search for the main pylint module and recursive functionality:
	>
	> ⏺ Search(pattern: "recursive", type: "py")
	>   ⎿  Found 16 files (ctrl+r to expand)
	>
	> ⏺ Let me look at the main pylint module and the base options to understand how recursive mode is implemented:
	> ```
- 我：可以，继续
- Claude Code:（又向我发出请求确认，要读这读那的）
	> ⏺ Read(pylint/lint/base_options.py)
	>   ⎿  Read 572 lines (ctrl+r to expand)
	>
	> ⏺ Now let me look at the PyLinter class to understand how the recursive mode is implemented:
- 我：可以，继续
- Claude Code:（又请求要读这读那得，这里就不贴细节了）
- 我：可以，继续
- ...就这样持续了好几回合，终于...
- Claude Code:（它终于做出了它的结论，说定位到了问题）
	> ⏺ Now I understand the issue. Let me look at the specific problem. The issue is in the _discover_files method in
	>   pylint/lint/pylinter.py. Let me examine how the recursive mode is implemented and where the ignore patterns should be applied:
	>
	>   Based on my analysis, I can see the issue. In the _discover_files method, the ignore patterns are being applied, but there's a
	>    flaw in how they're checked. Let me create a fix for this issue.
	>
	>   First, let me create a test to reproduce the issue:
	> ...省略一些它的 TODO list...
	> ...它的 TODO list 第一项是写一个用例来复现...
- 我：不用写测试用例，直接修复（和之前测 Aider/Gemini 一样，直接让它改代码）
- Claude Code: （这里 claude code 主要做了几个事情）
	1. 仔细分析：它发现 `pylint/lint/pylinter.py` 文件中的 `_discover_files` 方法和这个问题有关。
		分析这个方法的调用关系。在分析调用关系的时候，又发现 `expand_modules` 这个函数，理解了一波。
		最后决定要修复 `_discover_files` 方法。
	2. 更新了一波 `pylint/lint/pylinter.py` 文件，修复了 `_discover_files` 方法。
	3. 自己 review 自己的修复，blablabla，然后还想继续修复，询问我是否继续
- 我：可以，继续
- Claude Code：修了一小段代码，问我是否继续
- 我：继续吧（我是不会干涉它的思考、修复流程的）
- ...就这样又持续了几个回合，终于...
- Claude Code: 给自己的修改搞了一个 Summary Of Changes, Blabla
- 我：跑了一波 PASS-TO-PASS 测试，发现有一个用例失败了。
	- 备注：我用 git diff 看了一下它的改动，我基本判断它没戏了。所以没继续后面的测试了。

记录一下第一印象：前面我已经测试过好几轮了，对各种工具的做事方式方法也大概有了些了解。
按照 Claude Code 的工作流程点评一下它：
1. 分析问题：这个过程会调用 `search keyword` / `read file` 等工具。然后让模型进行分析理解。
	这一步，它还是做的不错的。也定位到了这个问题的关键点：`_discover_files` 和 `expand_modules` 方法。
	但最可惜的是，它分析到最后，结论不对。(这可能是它后续失败的关键)。这里还有个问题是，它每次读一个新的问题，
	都让我给它确认，太繁琐了。（虽然听朋友说，可以配置，但咱们只测默认配置下的表现）
2. 解决问题：Claude Code 有个特点是会自己修复，并且自己 review 自己的修复，然后提出进一步方案。
	它自己 review 这一点给我的印象还是深刻的。然并卵，它的分析就没有走在正确的方向上。
	如果我人工给它一点建议，或许会好一些吧。当 Claude Code 全部搞完后，它还会自我总结一下，来个
	Summary Of Changes，也挺有意思的。

## Claude Code 满血版（走 AICodeMirror）

- 我：`解决一下这个问题：{problem_statement}`
- Claude Code: 上来梳理了一个 TODO list。
	- 然后自动开始了第一步。就是一系列的搜索和读取文件
		```
		> ⏺ Search(pattern: "recursive", type: "py")
		>   ⎿  Found 16 files (ctrl+r to expand)
		>
		> ⏺ Search(pattern: "--recursive", type: "py")
		>   ⎿  Found 1 file (ctrl+r to expand)
		>
		> ⏺ Search(pattern: "ignore.*path", type: "py")
		>   ⎿  Found 8 files (ctrl+r to expand)
		>
		> ⏺ Read(pylint/lint/base_options.py)
		>   ⎿  Read 572 lines (ctrl+r to expand)
		>
		> ⏺ Read(pylint/lint/expand_modules.py)
		>   ⎿  Read 150 lines (ctrl+r to expand)
		> ```
	- 接着它继续理解代码。也是一些搜索和读取文件，这里就不粘贴了。
		> ```
		> ⏺ Now let me look at the main pylinter module to understand how recursive mode is implemented:
		> ...省略一堆它的搜索和读取文件...
		> ```
	- 然后它觉得自己找到了问题所在。自动进入 todo list 的第二项：写一个用例来复现，这时它来询问我
- 我： 跳过问题复现这一步，直接修复吧（和之前其它测试一样，让它跳过）
- Claude Code: 它按照要求跳过了这一步，进入了下一步
	> ⏺ Update Todos
	>   ⎿  ☒ Investigate current recursive mode implementation
	>      ☒ Reproduce the bug with test case
	>      ☐ Identify where ignore logic should be applied in recursive mode
	>      ☐ Fix the ignore logic implementation
	>      ☐ Add regression tests
	>      ☐ Run existing tests to ensure no regressions
	- 点评：这里特意把它的 todo list 贴了一下，我觉得得夸一下，它这个 UI 做的还不错，比较清晰的标记自己进入了第三步。（终端里面看是有颜色的，非常清晰）
	- 接着它就把自己的修复代码给输出出来了。
		> ⏺ Let me look at how the ignore logic is currently implemented by examining the expand_modules function:
		> ...省略...
		> ⏺ Now I need to modify the _discover_files method .... 省略...
		> ...省略...
		> ⏺ Let me check how the ignore settings are accessed in the config:
		> ...省略...
		> ...省略它的代码...
		> ⏺ Now I need to check if _ignore_paths is available ... 省略 ...
		> ...省略...
		> ⏺ I see that _ignore_paths is defined later in the code. I should use self.config.ignore_paths directly instead:
		> ...省略...
		> ...省略...
		> ⏺ Great! There are already some recursive tests. Now let me add a test for the ignore functionality:
		- 点评：值得一提的是，它输出代码的过程似乎也是分步的，先改一个代码文件；然后说接下来我要改第二个代码文件。但我觉得它应该是先总体规划好了，然后展示的时候，给人感觉是一步一步的。
- 我：（它想给我加测试用例，我没有回复它）我去跑了 PASS-TO-PASS 测试，发现符合预期。然后再去跑 FAIL-TO-PASS 测试，发现符合预期，问题已经解决。

相比 Qwen 版本，满血版(或者叫做原声版)效果明显要好很多，主要有几个方面
1. 没有每个步骤都让我确认（这里我也不太清楚为什么，命名运行参数都是一样的）。而该让我确认的，也让我确认了，比如它想复现，我就有机会拒绝它这个提议。
2. 定位问题更准，同样是一波搜索加文件读取，这次，它就准确的找到了问题，并且一次性修复。还是牛的。
