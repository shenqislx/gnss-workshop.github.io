---
title: "AI 大模型可监控性：从“隐瞒想法”到工程化监督"
layout: article
date: 2026-05-19
category: "AI"
author: "Andy"
summary: "从 OpenAI 与 Anthropic 关于思维链和内部激活的两篇文章说起，讨论 AI 大模型的可监控性为什么越来越像一个工程问题。"
---

# AI 大模型可监控性：从“隐瞒想法”到工程化监督

Anthropic 和 OpenAI 的关系，一直有点微妙。

这两家公司当然是竞争对手，但又不只是竞争对手。Anthropic 的几位核心创始人本来就出自 OpenAI，后来因为对 AI 安全、商业化节奏和组织治理的理解不同，另起炉灶做了 Claude。于是，今天再看 OpenAI 和 Anthropic 的公开研究，经常会有一种很奇妙的感觉：他们明明都在讲安全、讲对齐、讲负责任的模型能力，但读起来总像是在隔空回应。

最近关于 AI 大模型可监控性的几篇文章，就很有这种“针锋相对”的感觉。

先是 OpenAI 发了一篇文章，讨论推理模型对思维链（CoT）的控制能力。它的问题很具体：如果我们把模型自己输出的推理过程用来做风险监控，那么模型会不会故意把这段推理过程“粉饰太平”？

OpenAI 的观点是：至少现在，模型还不太擅长稳定控制自己的思维链，所以 CoT 监控仍然是一个有价值的监控方法。

这本来已经是一个有趣的发现。结果不久后，Anthropic 果断回击：别急着相信模型说出来的东西，因为模型内部可能已经有一些“没说出口”的判断。

Anthropic 发了一篇 Natural Language Autoencoders，试图把 Claude 内部数字形态的激活（activations）翻译成自然语言。

两篇文章都没有直接点名对方，但放在一起看，很难不觉得它们在围绕同一个问题拉扯：

**当 AI 系统越来越复杂，我们到底还能不能看见它为什么这么做？**

## OpenAI 先给了一个相对乐观的答案

OpenAI 讨论的是推理模型的 CoT，也就是模型在解决任务时生成的中间推理步骤。

过去很多人对 CoT 监控抱有期待。理由很直观：如果模型在行动前会把自己的推理过程写出来，那么我们就可以从这些推理轨迹中发现不安全意图、错误计划，或者偏离用户目标的行为。

这就像在模型行动前，多看了一眼它的草稿纸。

但这立刻带来另一个问题：

如果模型知道自己的 CoT 正在被监控，它会不会故意去修改 CoT，把它改得更安全、更干净、更不暴露真实风险？

OpenAI 把这个问题称为 CoT 可控性。它测试模型是否能遵守一些关于 CoT 的指令，比如避开某些关键词、改变推理格式，或者添加特定模式。结果显示，当前前沿推理模型的 CoT 可控性普遍较低。

这就是 OpenAI 文章标题里“难以掌控 CoT，但这反而是件好事”的含义。

如果模型还不能稳定操控自己的 CoT，那么 CoT 监测在现阶段就仍然值得去当作监管手段。

这是一种相对乐观的判断。问题是，Anthropic 很快把镜头往更深处推了一层。

## Anthropic 把问题推向内部

Anthropic 的 Natural Language Autoencoders（NLA） 文章，看起来像是对这种乐观判断的一次补充，或者应该说是反驳。

它最吸引人的地方不是技术细节，而是提出了一个很容易传播的说法：

AI 模型可能没有在输出中明说某件事，但它的 activations 中已经包含了相关判断。

Anthropic 的方法大致是，把模型内部的 activations 转成自然语言解释，再尝试从这段解释重构原始 activations，即 NLA。如果解释能帮助重构 activations，就说明这段解释（至少）捕捉到了（一部分）内部状态。

这套机制的目标不是让模型继续解释自己的输出，而是绕过最终输出，直接观察中间表征。

更有冲击力的是 Anthropic 举的例子。在一些安全测试中，Claude 的推理过程并没有说“我正在被测试”。但 NLA 给出的解释显示，它的内部状态里可能已经出现了类似“这是一个测试场景”的判断。

这就抛出了一个很有戏剧性的问题：

**AI 会不会隐瞒自己的想法？**

如果用人类语言描述，这当然很像“心口不一”。模型表面上没有说，内部却已经“知道”。但这里也正是最需要小心的地方：这种表述虽然有传播力，却很容易把问题拟人化。

从工程角度看，更准确的说法也许是：

大模型的输出文本只是系统状态的一种外显结果，并不等于全部内部状态。模型内部可能存在一些没有被最终输出显化的中间表征，而这些表征仍然可能影响后续行为。

无论是“心口不一”还是“表达差异”，形成这个结论就已经足够重要了。

因为一旦 AI 系统进入代码执行、网页操作、长任务规划等场景，我们关心的就不只是它最后说了什么，而是它在行动前是否已经形成了某些风险信号。

## 两篇文章为什么显得针锋相对

OpenAI 关心的是：模型说出来的 CoT 是否还能被信任。

Anthropic 关心的是：模型没说出来的内部状态是否也应该被看见。

一个看外显推理，一个往里看。一个强调“输出仍有用”，一个提醒“输出不够”。

他们实际上在争论一个更底层的问题：

**可靠监督的入口到底在哪里？**

如果入口是模型自己生成的 CoT，那么重点就是保护 CoT 的真实性，避免把模型训练成只会输出漂亮、安全、被修饰过的推理文本。

如果入口是模型内部 activations，那么重点就是发展可解释性工具，把高维表征翻译成人类可以理解的证据。

这两条路线当然并不互斥。现实中更可能需要两者配合：CoT 监控负责观察外显推理， activations 解释负责补充那些没有说出口的内部状态。

但它们确实存在路线竞争。因为资源、注意力和安全叙事是有限的。到底应该优先投入 CoT 监控，还是优先投入 activations 解释？这些问题都会影响未来大模型的训练方式和产品形态。

如果只看 OpenAI 和 Anthropic，这个问题已经足够有趣。但再往外看，会发现这不是两家公司之间的小分歧，整个 AI 安全领域都在绕着这个问题打转。

## 更大的视角

Apollo Research 和 Frontier Model Forum 更接近 OpenAI 的方向，但措辞更谨慎。它们认为 CoT 监控虽然脆弱，但很重要。只要模型还需要用自然语言完成复杂推理，人类就有机会通过监控 CoT 发现不安全的意图。

但它们也强调，CoT 不是唯一答案。未来模型可能学会隐藏意图，开发者也可能为了让输出更美观、更符合产品预期，把 CoT 训练得越来越不像真实推理。因此，它们主张把 CoT 可监控性当作一项安全特性来保护。

METR 的视角更工程化。它关心的不只是 CoT 是否“诚实”，而是整个评测系统是否能抓住模型的真实能力和潜在行为。即使某些 CoT 未完全表达意图，它仍然可能包含有用信号；但这些信号能否用于高风险场景，还需要和其他监控与控制方法结合。

Google DeepMind 和 UK AISI 则把问题放进更大的安全框架。它们更关注前沿模型的能力阈值、危险能力评估、防御规避能力和国家安全风险。可监控性在这里不是单项技术，而是整个风险管理体系中的一环。

NIST 和 Future of Life Institute 的角度更偏治理。它们不会押注某一种具体技术，而是强调风险识别、可测量、管理、外部评估、信息披露和问责机制。换句话说，即使 CoT 监控和 activations 解释都在进步，企业仍然需要回答：谁来评估？如何复现？失败是否披露？风险超过阈值时是否停止部署？

到这里，问题已经从“能不能看见模型在想什么”，变成了“一个越来越复杂的 AI 系统，应该如何被持续观察、约束和审查”。

这时候再回头看 OpenAI 和 Anthropic 的分歧，就不会觉得它只是两家公司在争路线。它更像是一个更大问题的两个切面：

一边是语言监控路线，观察模型自己生成的 CoT。

一边是内部解释路线，观察模型的 activations、表征和机制。

再往外，还有系统治理路线，通过评测、审计、披露和监管来兜底。

到这里，我想回到“AI 会不会隐瞒自己的想法”这个问题本身，重新回答一下。

## 不要把工具拟人化

我始终认为，AI 大模型是作为“工具”的一种存在，不具备人格。

它没有人类意义上的自我意识、欲望、羞耻心或道德意图，也没有人类特有的连续内心世界。因此，当我们说模型“隐瞒想法”“伪装推理”“意识到自己被评测”时，需要非常谨慎。

这些说法有助于传播，但也容易误导。

更准确地说，大模型是在训练目标、上下文输入、推理机制和反馈信号共同作用下，生成某种输出和行为。它可能表现出类似隐瞒、规避、装乖的模式，但这不等于它像人一样“故意”隐瞒自己的想法。

这里的区别很重要。

如果我们把 AI 当成人，就很容易追问：它到底在想什么？它是不是有恶意？它是不是想骗我们？

但如果我们把 AI 当作复杂工具系统，就会变成工程问题：哪些内部状态与风险行为相关？哪些外显推理信号仍然可信？哪些日志、activations、工具调用、行为轨迹可以被记录和审查？哪些评测能暴露规避行为？哪些部署机制能在风险升高时及时切断？

我更相信后一种提问方式。

AI 不需要具备人格，才会产生难以直接观察的中间状态。

飞机没有人格，飞控系统仍然需要黑盒记录和传感器监控；自动驾驶系统没有人格，仍然需要感知解释、轨迹记录和安全冗余。

大模型也是一样。

我们不必假设它“有心隐瞒”，也应该承认它的内部状态和行为路径会越来越复杂。可监控性的目标不是证明 AI 有人格，而是把这些复杂状态转化为可测量、可解释、可审查的工程对象。

## 最后：可监控性是工程问题

所以，我对 AI 大模型可监控性的判断偏乐观。

不是因为我相信模型会天然诚实，也不是因为我相信某一种监控技术已经成熟，而是因为我认为问题本身可以被工程化拆解。

CoT 监控可以观察外显推理， activations 解释可以补充内部表征，行为评测可以观察真实任务表现，对抗测试可以寻找规避路径。

工具调用日志可以记录行动链路，权限系统可以限制可执行动作，外部审查和治理框架可以把技术信号转化为责任机制。

这些方法没有任何一个能单独解决问题，但它们叠加起来，就能把“不可见的模型状态”逐步变成“可观测的系统行为”。真正可靠的可监控性，应该不会来自单一路线，而会来自多层叠加。

因此，可监控性不是一个抽象哲学问题，也不是一个人格判断问题。它更像是安全工程、可解释性、系统设计和治理机制交叉形成的一组能力。

更重要的是，只要我们不被“AI 是否真的在想”这个问题带偏，而是持续追问“哪些状态可以被测量，哪些行为可以被审查，哪些风险可以被提前发现”，那么 AI 大模型的可监控性就不是空中楼阁。

它应该，也一定会成为 AI 工程体系中的基础能力。

## 参考文章

- Anthropic: [Natural Language Autoencoders: Turning Claude's thoughts into text](https://www.anthropic.com/research/natural-language-autoencoders)
- OpenAI: [推理模型难以掌控思维链，但这反而是件好事](https://openai.com/zh-Hans-CN/index/reasoning-models-chain-of-thought-controllability/)
- Apollo Research: [Chain of Thought Monitorability: A New and Fragile Opportunity for AI Safety](https://www.apolloresearch.ai/science/chain-of-thought-monitorability-a-new-and-fragile-opportunity-for-ai-safety/)
- Frontier Model Forum: [Chain of Thought Monitorability](https://www.frontiermodelforum.org/issue-briefs/chain-of-thought-monitorability/)
- METR: [CoT May Be Highly Informative Despite "Unfaithfulness"](https://metr.org/blog/2025-08-08-cot-may-be-highly-informative-despite-unfaithfulness/)
- Google DeepMind: [Updating the Frontier Safety Framework](https://deepmind.google/blog/updating-the-frontier-safety-framework/)
- UK AI Security Institute: [AISI Frontier AI Trends Report 2025](https://www.aisi.gov.uk/research/aisi-frontier-ai-trends-report-2025)
- NIST: [Artificial Intelligence Risk Management Framework 1.0](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-ai-rmf-10)
- Future of Life Institute: [2025 AI Safety Index](https://futureoflife.org/ai-safety-index-summer-2025/)
