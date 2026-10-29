# Cloud-Robotics 六类 GuideSeg 云边持续学习 Baseline

本工程在原有 LAECIPS 六类 `f/h` 云边系统上增加一个独立的 `F(x)`：整图 DeepSeek 场景描述 → SentenceTransformer/FAISS 规则检索 → DeepSeek Worker 定位 → SAM3 框提示分割 → DeepSeek Supervisor 审核与定位错误 → AiRC 决策 → SigLIP 核验 → SAM3/Worker 修正，最多四轮审核（含首次审核，最多三次修正）。六类及顺序固定为 `road, sidewalk, building, vegetation, sky, wall`。原项目和规则库保持只读，运行输出写在本目录的 `outputs/`。

## 已审查的复用边界

| 来源 | 直接复用 | 必须适配或新写的部分 |
|---|---|---|
| `/mnt/n0/guideline_seg/guideline-seg` | `src.utils.iter_controller.RefineAgent`：六状态 Q 表、动作、奖励更新与持久化 | 原 Worker/Supervisor 绑定 Gemini 和行人规则；本工程改成 DeepSeek 六类 JSON 调用。原 Retriever 的 SentenceTransformer+FAISS 算法保留，用现有六类规则做索引。原 SigLIP `cat` 对照判据不符合本实验，改为真实 logit 的 sigmoid。原 Smart Crop 不使用。 |
| `/mnt/n0/LAECIPS/cloud_robotics_six_class/sam3_only_incremental_v2` | `cloud/server.py` 的 HTTP 协议、`Store` Replay、`UpdateWorker`、训练/导出/RKNN 编译、板端 13 张量标定、配套版本下发与 `edge_bundle/run_edge.py` 安全切换 | 注入本工程 Teacher 与结果打包函数；独立端口、存储、device ID 和 teacher policy。原代码不改写。 |
| 现有六类规则库 `guideseg_rules_v1/rules.json` | 8 条固定规则与 12 条场景规则 | 兼容读取；默认从场景规则检索最多 6 条，固定规则另附。原文件不改写。 |
| 本工程 | — | DeepSeek 视觉适配、SAM3 交互分割、SigLIP 论文判据、多类实例与语义图、每轮审核归档、配置和测试。 |

原 Guideline-Seg 仓库采用 Apache-2.0 许可证。这里仅通过 Python import 复用其 AiRC 源码；未复制完整仓库。见原仓库的许可证及源码。模型权重、DeepSeek 服务和原 LAECIPS 目录仍是运行依赖。官方 Worker、Supervisor、SigLIP 及 SAM2.1 接口没有直接接入，因为分别与指定模型、六类任务或论文判据不兼容；这不是声称其原代码已被完整复用。

## 输入、输出和边界

`guideseg.pipeline.Teacher.run(image_path, case_folder, frame_id)` 接收整张原图。`DeepSeekVision` 产出 caption、带唯一 ID 的六类候选 `[ymin,xmin,ymax,xmax] / 1000`、三类审核问题和 boxgen 框。`RuleRetriever` 产出规则 ID、相似度、版本及本次规则集。`Sam3Segmenter` 在一次 `begin_image` 中缓存整图图像特征，逐框调用真实 SAM3 交互接口，保留每个实例原始二值掩码；必要时用已有低分辨率 logits 与负点重新预测。`SiglipVerifier` 对候选框带边距裁图，以实际 `logits_per_image` 计算 `sigmoid(logit) >= 0.5`；阈值可配置，不使用 `cat` 对照。漏检候选核验拟新增的六类；误检候选核验 Supervisor 指出的**实际可见材质**（`verification_text`），避免把错误掩码的类别当作真相核验。`AircController` 调用官方 Q 表动作策略，同时遵守其候选强制继续、前两轮优先继续、问题清零、停滞及最多四轮审核的调度边界。每次动作和 Q 值写入审核日志。

`result/` 包含 `original.png`、`overlay.png`、`semantic.png`、`masks/<id>.png`、`labels.txt`、`result.json`、`audit.json`、`bundle.json`。`s0` 至 `s3` 保存每轮叠加图、原始实例掩码和语义图；`context.json` 保存 caption/规则/初次 Worker 输出；顶层 `audit.json` 保存每轮问题、SigLIP 分数、动作、变更、耗时、未解决问题和模型版本。实例掩码是无损 PNG。跨实例或跨类重叠记录到每轮 `overlaps.json`；语义图按最新修正版本、再按 SAM3 分数选像素归属，原始实例掩码不覆盖。`255` 表示未标注/忽略区域。

云端 HTTP `/v1/infer` 继续使用原 LAECIPS ZIP 协议，请求号、图像 SHA256、模型哈希和结果包哈希按原协议核对。teacher policy 由规则库、模型配置和本工程实现源码哈希构成，边缘启动时核对。简单图本地保存；困难图上传整张图并串行等待结果落盘。边缘 `f/h` 配对检查、图片边界切换和回滚采用原 `model_manager.py`。结果成功生成后全部进入 Replay，不用审核通过与否筛除。原更新器在活动 Replay 数 **大于 15** 时触发（即至少 16 张），时间间隔触发也至少 16 张。随后冻结快照、固定 h 更新 f、导出和编译 RKNN、板端运行候选 f 并回传真实 13 路张量、固定新 f 更新 h、打包并下发配对版本，技术检查通过后在两张图片之间激活。训练依旧使用 Replay 的 `F(x)` 精确掩码；人工 GT 仅用于独立指标评估。服务器无法代替板端产生 RKNN 张量。

## 目录

```text
configs/         独立 cloud.json、edge.json
guideseg/        几何/Schema、DeepSeek、规则检索、SAM3、SigLIP、AiRC、主流程和 CLI
cloud/           现有 LAECIPS HTTP 服务注入、结果 ZIP 包
edge/            原边缘程序的独立配置入口
scripts/         配置生成、环境自检、真实组件冒烟测试
tests/           单元、真实检索、明确标注的 Mock 协议集成测试
outputs/         独立模型索引、Q 表、运行记录和冒烟测试产物
```

## 环境与配置

推荐使用已有 `/home/sdu/.conda/envs/sam3/bin/python`；安装 `requirements.txt` 中额外依赖，并保证该环境可加载现有 SAM3、PyTorch、Transformers 与 CUDA。已验证 `faiss-cpu`、`sentence-transformers` 和 `sentencepiece` 可在此环境安装。SAM3 源码、`/mnt/n0/sam3.pt`、Google SigLIP 权重、六类规则库、原 LAECIPS 项目、初始 f/h 和训练所需的原环境路径由 `configs/cloud.json` 配置。默认 SAM3 使用 `cuda:2`、SigLIP 使用 `cuda:3`，可按实际显存调整。`configs/edge.json` 中的初始 RKNN/ONNX 路径是板端现有 bundle 路径，需在复制到板子后核对。

DeepSeek 必须使用支持图像输入的模型和账号。默认 `https://api.deepseek.com` / `deepseek-flash`，可修改 `guideseg.deepseek.base_url`、`model`、超时、重试和 `max_tokens`。**不要把真实 Key 写到 JSON 或 Git。**启动前设置：

```bash
export DEEPSEEK_API_KEY='your-real-vision-key'
export LAECIPS_TOKEN='your-shared-random-token'
/home/sdu/.conda/envs/sam3/bin/python scripts/check_environment.py
```

板端使用相同的 `LAECIPS_TOKEN`。若通过 SSH 隧道连云端，板端 `cloud_url` 默认 `127.0.0.1:18086`；本机绑定 `127.0.0.1`，需按现有隧道部署。修改 `guideseg` 配置后执行 `python scripts/make_config.py` 重新计算 edge 预期 teacher policy；此脚本会覆盖**本工程**的两个生成配置，执行前保存自己的配置改动。

## 可执行入口

以下命令在本项目根目录执行，`PY` 可替换为适用环境：

```bash
PY=/home/sdu/.conda/envs/sam3/bin/python
$PY scripts/check_environment.py
$PY cloud/serve.py --config configs/cloud.json
$PY -m guideseg.cli single --image /path/to/image.png --output outputs/single_001
$PY -m guideseg.cli batch --images /path/to/images --output outputs/batch_001
$PY -m guideseg.cli audit --folder outputs/single_001
$PY -m guideseg.cli trigger-update --device-id orangePi_cloudrobotics6_guideseg_v1
$PY -m unittest discover -s tests -v
```

`trigger-update` 调用现有 Store 的条件判断，不强行跳过数量或时间条件；真正训练由运行中的云端后台 worker 执行。审计命令读取已经生成的 `audit.json`。单图/批量若无 DeepSeek Key 会明确报错，不生成伪造标注。批量逐张记录成功或失败到 `batch_records.json`。

板端**复用原六类 edge bundle**，把本工程的 `edge/run.py`、`configs/edge.json` 一同复制到板上；也可使用已生成的 `edge_adapter_guideseg.zip`（需要重新打包时运行 `python scripts/package_edge.py`，先移走旧 ZIP）。确保原 edge bundle 已有 `run_edge.py`、RKNN、h ONNX 和依赖。然后：

```bash
python3 edge/run.py --config configs/edge.json \
  --existing-edge /home/orangepi/orangePi/cloudrobotics6_sam3_incremental_v2 \
  --input /path/to/images --output /home/orangepi/orangePi/baseline_guideseg/results/run001 \
  --wait-updates-seconds 7200
```

边缘任务由原 edge 脚本完成；这台服务器当前不能代替 RK3588 做真实 RKNN/13 路回传，板端命令需在实际设备执行。边缘运行前先校验 `/health` 的 `teacher_policy` 与 edge 配置一致，运行结果及指标在原边缘输出目录。历史任务数据不会进入本工程独立 Replay 目录。

## 验证范围

本工程测试分层，避免把 Mock 当成真实模型：

* **真实组件**：已用实际 Cloud-Robotics 图像及真实 SAM3 权重完成框提示和负点修正；已用真实 SigLIP 权重计算图文 logit 与 sigmoid；SentenceTransformer + FAISS 已实际生成并查询六类规则索引。可用 `scripts/smoke_models.py --image ... --class-name road --box 500 0 1000 1000 --output outputs/my_smoke` 重跑（框只是用户提供的组件探针，不是 VLM 输出或 GT）。
* **Mock 集成**：`tests/test_integration_mock.py` 用标明的 Fake VLM/SAM3/SigLIP，验证 Supervisor→AiRC→修正→ZIP→Replay 接口；`tests/test_edge_mock.py` 验证原 LAECIPS 边缘循环先本地输出简单图、再上传困难图并等待结果及写入 Replay。这些测试不代表真实 RKNN 或多模态全链路准确性。
* **尚未真实验证**：本环境未配置 `DEEPSEEK_API_KEY`，因此 DeepSeek Enricher/Worker/Supervisor 的真实调用及整张图 `F(x)` 闭环尚未跑通。未连接实际 RK3588，所以候选模型 13 路输出、真实 f/h 增量训练及板端自动切换也未在本工程完成。任何正式实验前需逐项验证。

2026-10-29 在现有环境运行 `python -m unittest discover -s tests -v`，12 项通过；独立 HTTP 服务在 `127.0.0.1:18086` 启动并返回匹配的 teacher policy。环境自检显示规则、权重、原项目目录存在且可见 4 张 CUDA 卡；DeepSeek Key 未配置。当前环境的 SciPy 对 NumPy 版本发出兼容性警告，虽然本轮检索和测试通过，正式实验建议先在隔离环境中固定相容依赖版本。

真实 SigLIP 组件探针曾对一块 road 区域给出约 `0.065`、wall 区域约 `0.579`；在指定的 `0.5` 阈值下前者被拒绝，说明该阈值对现有六类并非天然校准。这里严格保留用户指定的论文判据，不将组件通过误述为伪标签质量已经改善。应使用人工 GT 独立评估最终 `F(x)` 和整套云边指标。
