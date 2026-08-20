# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

数据库中一条 completed 运行的 `started_at` 与 `completed_at` 相差 30 分钟，可详情接口读出来后两个时间完全一样，执行时长也成了零。本次仅诊断而不改代码，生产代码、测试和配置都不作调整。请逐项核对原始行、扫描后的对象和中间的时间归一化，找出时间被折叠的证据。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-08
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-08.git
- parent SHA：60ea084fc598145cc2844e7474188541cd3e4d0c

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-08.git bug-repro
cd bug-repro
git checkout --detach 60ea084fc598145cc2844e7474188541cd3e4d0c
go test ./internal/storage/sqlite -run ^TestInferenceRunReadPreservesDistinctLifecycleTimes$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/storage/sqlite -run ^TestInferenceRunReadPreservesDistinctLifecycleTimes$ -count=1
--- FAIL: TestInferenceRunReadPreservesDistinctLifecycleTimes (0.08s)
    annotation_core_behavior_test.go:39: restored start/completion = 2026-08-18 08:40:00 +0000 UTC / 2026-08-18 08:40:00 +0000 UTC, want 2026-08-18 08:10:00 +0000 UTC / 2026-08-18 08:40:00 +0000 UTC
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/storage/sqlite	0.080s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/storage/sqlite -run ^TestInferenceRunReadPreservesDistinctLifecycleTimes$ -count=1
--- FAIL: TestInferenceRunReadPreservesDistinctLifecycleTimes (0.32s)
    annotation_core_behavior_test.go:39: restored start/completion = 2026-08-18 08:40:00 +0000 UTC / 2026-08-18 08:40:00 +0000 UTC, want 2026-08-18 08:10:00 +0000 UTC / 2026-08-18 08:40:00 +0000 UTC
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/storage/sqlite	0.517s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断必须定位 internal/domain/inference_run.go 的 InferenceRun.NormalizeLifecycleTimes，并证明 internal/storage/sqlite/read_core.go 的 scanInferenceRun 在分别解析 started_at 与 completed_at 后调用该方法；该方法选取最晚时间覆盖所有非空生命周期字段，导致相差 30 分钟的原始值折叠为同一时刻、时长变为零。需对照数据库行、扫描对象和接口结果，且目标仓库零改动。
