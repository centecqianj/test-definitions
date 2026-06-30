# stress_oom 经验总结

## 现象
在执行 `stress_oom` 1 小时测试时，日志显示 **确实触发了 OOM killer**，例如：

- `Out of memory: Killed process ... (stress-ng-vm)`

但最终任务仍然失败，并且 **没有正常把结果上报给 LAVA**。

## 根因分析

### 1. `stress_oom` 的目标就是触发 OOM
`stress_oom` 通过 `stress-ng --vm` 大量占用内存，主动把系统推到内存耗尽状态。

因此它和普通稳定性测试不同，它本身就是一个“破坏性”测试：

- 目标不是“系统不出事”
- 而是“系统在内存耗尽时能否触发 OOM killer”

### 2. OOM killer 可能杀掉的不只是 `stress-ng`
OOM killer 会根据内核的策略选择要杀掉的进程，不一定只杀 `stress-ng-vm`。

如果系统内存已经非常紧张，下面这些进程也可能受影响：

- `lava-test-shell`
- `sh`
- `tee`
- 结果收集脚本
- `send-to-lava.sh`

一旦这些关键进程被杀，测试链路就会中断，LAVA 就收不到完整结果。

### 3. 结果上报依赖脚本最后一步
脚本最后需要执行：

```sh
../../utils/send-to-lava.sh ./output/result.txt
```

但在 OOM 场景下，可能出现：

- 脚本没运行到最后
- `result.txt` 未完整生成
- `send-to-lava.sh` 没有机会执行
- 外层 runner 已经退出

这就是为什么即使 OOM 成功触发，任务仍然会失败，且结果无法正常上报给 LAVA。

### 4. 日志里的关键信号
你提供的日志中同时出现了：

- OOM killer 触发：`Out of memory: Killed process ... (stress-ng-vm)`
- runner 退出：`<LAVA_TEST_RUNNER EXIT>`
- LAVA 判断未完成：`Marking unfinished test run as failed`

这说明：

> **OOM 已经发生，但测试执行框架也被拖垮了。**

## 这个测试的通过条件
在当前脚本逻辑里，`stress_oom` 的判定标准是：

- 在 `stress_oom_kern.log` 中找到 `Out of memory: Kill process`
- 找到就认为通过
- 找不到就认为失败

也就是说，它本质上是要验证 **OOM killer 是否真的被触发**。

## 但为什么你这次会失败
这次不是“没有触发 OOM”，而是：

1. OOM 成功触发了
2. 但系统压力过高
3. 测试框架和上报链路也受到了影响
4. 最终 LAVA 只看到“未完成”，所以把任务标记为失败

## 建议改进

### 1. 放宽 OOM 日志匹配
脚本现在匹配的是：

```sh
grep -c "Out of memory: Kill process"
```

但实际日志里常见格式也可能是：

```text
Out of memory: Killed process ...
```

建议兼容两种写法，避免误判。

### 2. 降低对测试框架进程的影响
可以考虑：

- 给关键上报进程更低的 OOM 风险
- 尽量让 `lava-test-shell`、`tee`、结果脚本存活
- 让日志和结果尽早落盘

### 3. 把结果上报前置
不要把唯一的结果收尾放在测试结束之后。
可以考虑：

- 先写入阶段性结果
- 或增加 watchdog/cleanup 机制
- 或在单独的保护进程里保留最小上报能力

### 4. 将 `stress_oom` 作为独立测试运行
不要和必须完整返回结果的常规测试混跑。
它更适合：

- 单独执行
- 在专用环境中执行
- 允许一定程度的 runner 风险

## 一句话总结
`stress_oom` 的成功标准是“**触发 OOM killer**”，但你的任务失败是因为 **OOM 触发得太狠，把 LAVA 的测试执行和结果上报链路也一起打断了**。

