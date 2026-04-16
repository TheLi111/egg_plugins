# Egg Sampling Daily Plugin

OpenClaw plugin for one-click daily egg sampling.

Current scope (Day 3):
- one tool orchestration: file -> egg-candling -> terminal -> document-summary
- step-level tracing and error localization
- compact summary in content and full raw results in details
- fallback markdown template when document-summary is unstable
- failure handling for auth failure, SSH timeout, insufficient images, and report persistence failure

## Install (local)

1. Build the package
2. Install with OpenClaw local plugin install command
3. Configure plugin values listed below

## Config

Required values:
- agentServiceBaseUrl
- apiToken

Core values:
- requestTimeout
- workflowTimeout
- defaultQueueSize

Common optional values:
- reportFilename
- reportDir
- terminalTimeCommand
- terminalTimeoutMs
- documentSummaryCallTimeoutSeconds
- documentSummaryWaitTimeoutSeconds

## Tool

Tool name:
- run_daily_egg_sampling_report

Tool behavior:
- default: execute 4 steps in order
	- system/file-agent
	- system/egg-candling-agent
	- system/terminal-agent
	- system/document-summary-agent
- 401: direct fail and stop workflow
- SSH timeout in egg detection: fail fast with reason_code=ssh_timeout and mitigation hint
- insufficient images: fail fast with reason_code=insufficient_images when queue_count < expected_count
- report persistence failure: fallback when report_file_id is missing
- 5xx/timeout: limited retry in HTTP client
- terminal failure: degrade and continue
- document-summary failure: switch to temporary markdown report template
- output shape:
	- content: compact summary for LLM context
	- details: step traces, raw snapshots, report_markdown, report_file_id, class_distribution

Debug params:
- dryRun=true: no backend calls, return request snapshots only
- skipEggDetection=true: skip step_egg_detect
- skipTerminal=true: skip step_time_context
- skipDocumentSummary=true: skip step_document_summary and force fallback report
- forceTemplateFallback=true: bypass document-summary and force fallback report

Run Day 3 regression:
- npm run build
- npm run regression:day3
- Output file: docs/day3-regression-results.json

Build installable package:
- npm run pack:plugin
- Output package: egg-sampling-plugin-0.3.0.tgz

## High-hit trigger examples

Use these prompts in OpenClaw to maximize tool selection:

1. 帮我查看今日蛋的抽样情况，并进行汇总，输出一份总结报告。
2. 执行今天的鸡蛋抽样照检流程，给我报告和分类统计。
3. 跑一遍每日蛋抽样：先抽样、再分类、再按当前日期生成报告。
4. 今天蛋的抽样检测结果是什么？给我报告文件和类别分布。
5. 生成今日鸡蛋抽样日报，包含每一步状态和最终总结。

## Notes

If SSH to the remote detection host is unstable, use skipEggDetection=true for pipeline wiring checks first.

If document-summary-agent is unstable, use skipDocumentSummary=true to force fallback markdown and keep demo flow unblocked.
