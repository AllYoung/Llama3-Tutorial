# Llama 3 Agent 能力体验+微调（Lagent 版）

## 1. Llama3 ReAct Demo

首先我们先来使用基于 Lagent 的 Web Demo 来直观体验一下 Llama3 模型在 ReAct 范式下的智能体能力。我们让它使用 ArxivSearch 工具来搜索 InternLM2 的技术报告。
从图中可以看到，Llama3-8B-Instruct 模型并没有成功调用工具。原因在于它输出了 `query=InternLM2 Technical Report` 而非 `{'query': 'InternLM2 Technical Report'}`，这也就导致了 ReAct 在解析工具输入参数时发生错误，进而导致调用工具失败。 

![image](https://github.com/SmartFlowAI/Llama3-Tutorial/assets/75657629/f9e91a2e-3e46-478a-a906-4d9626c7e269)

## 2. 微调过程

接下来我们带大家使用 XTuner 在 Agent-Flan 数据集上微调 Llama3-8B-Instruct，以让 Llama3-8B-Instruct 模型获得智能体能力。
Agent-Flan 数据集是上海人工智能实验室 InternLM 团队所推出的一个智能体微调数据集，其通过将原始的智能体微调数据以多轮对话的方式进行分解，对数据进行能力分解并平衡，以及加入负样本等方式构建了高效的智能体微调数据集，从而可以大幅提升模型的智能体能力。

### 2.1 环境配置

我们先来配置相关环境。使用如下指令便可以安装好一个 python=3.10 pytorch=2.1.2+cu121 的基础环境了。

```bash
conda create -n llama3 python=3.10
conda activate llama3
conda install pytorch==2.1.2 torchvision==0.16.2 torchaudio==2.1.2 pytorch-cuda=12.1 -c pytorch -c nvidia
```

接下来我们安装 XTuner。

```bash
cd ~
git clone -b v0.1.18 https://github.com/InternLM/XTuner
cd XTuner
pip install -e .
```

### 2.2 模型准备

在微调开始前，我们首先来准备 Llama3-8B-Instruct 模型权重。

- InternStudio

```bash
cd ~
ln -s /root/share/new_models/meta-llama/Meta-Llama-3-8B-Instruct .
```

- 非 InternStudio

我们选择从 OpenXLab 上下载 Meta-Llama-3-8B-Instruct 的权重。

```bash
cd ~
git lfs install
git clone https://code.openxlab.org.cn/MrCat/Llama-3-8B-Instruct.git Meta-Llama-3-8B-Instruct
```

### 2.3 数据集准备

由于 HuggingFace 上的 Agent-Flan 数据集暂时无法被 XTuner 直接加载，因此我们首先要下载到本地，然后转换成 XTuner 直接可用的格式。

- InternStudio

如果是在 InternStudio 上，我们已经准备好了一份转换好的数据，可以直接通过如下脚本准备好：

```bash
cd ~
ln -s /root/share/new_models/internlm/Agent-Flan .
```

- 非 InternStudio

首先先来下载数据：

```bash
cd ~
git lfs install
git clone https://huggingface.co/datasets/internlm/Agent-FLAN
```

我们已经在 SmartFlowAI/Llama3-Tutorial 仓库中已经准备好了相关转换脚本。

```bash
cd ~
git clone https://github.com/SmartFlowAI/Llama3-Tutorial
python ~/Llama3-Tutorial/tools/convert_agentflan.py ~/Agent-Flan/data
```

在显示下面的内容后，就表示已经转换好了。转换好的数据位于 ~/Agent-Flan/data_converted

```bash
Saving the dataset (1/1 shards): 100%|████████████| 34442/34442
```

### 2.4 微调启动

我们已经为大家准备好了可以一键启动的配置文件，主要是修改好了模型路径、对话模板以及数据路径。

我们使用如下指令以启动训练：

```bash
mkdir -p ~/project/llama3-ft
cd ~/project/llama3-ft
xtuner train ~/Llama3-Tutorial/configs/llama3-agentflan/llama3_8b_instruct_qlora_agentflan_3e.py --work_dir ~/project/llama3-ft/agent-flan
```

在训练完成后，我们将权重转换为 HuggingFace 格式，并合并到原权重中。

```bash
# 转换权重
xtuner convert pth_to_hf ~/Llama3-XTuner-CN/configs/llama3-agentflan/llama3_8b_instruct_qlora_agentflan_3e.py
    ~/project/llama3-ft/agent-flan/iter_18516.pth \
    ~/project/llama3-ft/agent-flan/iter_18516_hf
# 合并权重
export MKL_SERVICE_FORCE_INTEL=1
xtuner convert merge ~/Meta-Llama-3-8B-Instruct \
    ~/project/llama3-ft/agent-flan/iter_18516_hf
    ~/project/llama3-ft/agent-flan/merged
```

## 3. Llama3+AgentFlan ReAct Demo

在合并权重后，我们再次使用 Web Demo 体验一下它的智能体能力吧~

可以看到，经过 Agent-Flan 数据集的微调后，Llama3-8B-Instruct 模型已经可以成功地调用工具了，其智能体能力有了很大的提升。

![image](https://github.com/SmartFlowAI/Llama3-Tutorial/assets/75657629/19a3b644-56b3-4b38-99c8-c6133d29f119)

