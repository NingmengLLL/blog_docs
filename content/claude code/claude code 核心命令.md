```json
{
  "date": "2026.04.12 21:03",
  "tags": ["Claude Code", "Agent", "核心命令"],
  "description": "claude code常用命令。"
}
```

#### 1. /simplify

/simplify 做的事很简单：审查你刚写的代码，找出隐藏的问题，然后直接帮你改掉。现在官方文档已把 /simplify 列为 bundled skill。

工作机制：三步走  
第一步：确定审查范围。 通常围绕最近变更文件工作；不带参数时，它跑 git diff 拿增量变更；如果工作区没有未提交的修改，它会自动审查最近一次 commit。指定具体类名时（比如 /simplify MarketDataService），它会读取整个文件做全量审查。具体范围以当前 Claude Code 版本行为为准。  
第二步：并行启动三个审查 Agent。 不是串行地逐条检查，而是同时派出三个"审查员"，各自带着不同的视角去读同一份 diff：  
三个 Agent 各管一摊：  
1.Code Reuse Agent：看你的代码是不是在重复造轮子。比如你手写了一个 requireNonBlank()，它会在项目里搜一圈，发现已经有一个 InputValidator.requireNonBlank() 做了同样的事。  
2.Code Quality Agent：看代码设计有没有问题。比如同一个字符串硬编码写了三遍、两个方法长得几乎一样、一个类既管认证又管发邮件——该拆没拆、该抽象没抽象的地方，它都会指出来。  
3.Efficiency Agent：看代码跑起来会不会有性能问题。比如循环里反复创建同一对象，单线程场景非要用 ConcurrentHashMap、该用缓存的结果每次都重新算。  
第三步：汇总并修复。 三个 Agent 各自报告发现，Claude Code 会自动判断哪些是真问题、哪些是误报，然后直接动手改代码。


#### /review：代码审查
/review 和 /simplify 定位完全不同：/simplify 是自动清理工，找到问题直接改；/review 是资深审查员，找到问题列出来给你看，你自己决定改不改。

简单说，/simplify 关注可复用性、代码质量和效率，偏重清理与改进；/review 关注代码有没有写错，偏重正确性审查。

工作机制  
执行 /review 时，Claude Code 会做三件事：  
第一步：拿到变更。 它先跑 git diff 拿增量变更，或者根据你指定的 PR 读取远程变更。  
第二步：并行分析。 Claude Code 并行审查变更，结合置信度过滤来减少误报。  
第三步：输出分级报告。 最后你会拿到一份分级的问题清单（Critical / High / Medium / Low），每个问题带具体行号、原因和修复建议。

#### /loop

/loop 可以帮你定时跑任务，也可以帮你反复试错直到把活干完。

#### /debug

/debug 不是帮你 debug 业务代码，而是帮你排查 Claude Code 会话本身的问题。
比如 MCP 连接异常、工具调用失败、命令卡住、权限规则没生效、插件加载异常，

#### /batch
/batch 的核心本质是多任务并行编排器，它的强大之处在于它能将一个复杂的"大需求"自动拆解并并行执行。

1.任务拆解 (Task Decomposition)： 当你说一个大任务或者多条需求的时候，Claude 并没有胡乱开始，而是将其逻辑拆分成独立的 Unit（工作单元）。   
2.并行工作 (Parallel Workers)： Claude 会同时启动多个后台 Agent，分别处理不同的功能模块。  
3.独立工作区 (Independent Worktrees)： 为了防止多个 Agent 同时修改代码导致冲突，Claude 为每个 Worker 创建了独立的 Git Worktree。这意味着它们在物理隔离的环境中修改代码，互不干扰。

#### 其他辅助命令

/diff	    查看 Claude 到底改了什么	每次 /simplify、/batch 后必看
/context	查看上下文占用	            长任务开始变慢、变飘时先看
/compact	总结并压缩上下文	        长会话继续推进前用
/debug	    排查 Claude Code 会话问题	MCP、工具调用、权限异常时用
/permissions	管理工具权限	        跑 /loop、/batch 前先检查
/statusline	    配置状态栏	        想常驻看模型、目录、上下文、成本时用
/usage / /cost	查看用量和成本	        长任务前后看消耗