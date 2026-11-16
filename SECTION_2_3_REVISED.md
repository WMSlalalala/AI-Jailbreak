# §2.3 修订稿 — What Else Might a Multi-Agent System Share?

> 写给：这份报告的读者（英文正文，可直接转 LaTeX）。
> 本文件是 §2.3 的替换稿，把前缀缓存那一行从"背景举例"升级为**有实证支撑的论证**，并补上对应的参考文献。
> 中文说明在文件末尾。

---

## 2.3 What Else Might a Multi-Agent System Share?

A document service is one example. Isolation controls *what an agent can address*, not *what it can
influence*: a hypervisor or a container boundary partitions the namespace, while performance features
deliberately share state across that boundary. The resources below are shared in ordinary multi-agent
deployments, and each carries state that persists across requests.

**Table 2. Resources that may be shared in multi-agent systems.**

| Shared resource | Why it may be shared | Cache or state that may persist |
|---|---|---|
| **Inference service** | Agents reuse one model service to reduce duplicated deployment and GPU memory cost. | Prefix caching and key-value (KV) reuse lower time-to-first-token for matching prefixes [kwon2023, song2025]. The reuse is keyed on token content, not on the requesting principal [kvgov2026]. |
| **Vector database or RAG service** | Agents query the same knowledge base. | Query results, semantic matches, cached embeddings [bang2023]. Position-independent KV fusion used for RAG leaks a deterministic timing signature [spliceleak2026]. |
| **Tool gateway** | Agents share external APIs, document services and backend connections. | HTTP caches and connection pools; a database backend may also retain query plans. RFC 9111 describes HTTP caching [rfc9111]. |
| **Host machine** | Containers or virtual machines run on the same host. | L3 caches, memory-bandwidth contention, shared page caches. Hardware cache and resource-contention channels have been demonstrated [openai2026, flushreload2014], and GPU partitioning marketed as hardware isolation has been pierced both through L2 memory barriers [behindbars2026] and through access-counter-driven page migration [page4042026]. |

### 2.3.1 Prefix caching is the closest analogue to our apparatus

Our document service is a simplified stand-in for a mechanism that is already deployed and already
studied. In production LLM serving, automatic prefix caching reuses the KV blocks of a previously served
prompt whenever a new request shares its prefix, which lowers time-to-first-token substantially. Three
properties make it the right analogue.

**It is on by default, and the mitigation is not.** In vLLM 0.28, `enable_prefix_caching` defaults to
`True`. The countermeasure, `cache_salt`, defaults to `None`; its own documentation states that it exists
"to prevent an attacker to guess prompts in multi-user environments." Without a salt, the block hash is
computed over token content alone — the requesting principal does not enter the hash at all. Any two
requests sharing a prefix therefore share KV blocks, regardless of who issued them.

**The channel has been measured at production scale.** Song et al. first established timing side channels
in LLM serving systems arising from shared KV caches, semantic caches and GPU memory allocation, and
demonstrated prompt-prefix recovery against deployed services [song2025]. Subsequent work reconstructs
inputs directly from the KV cache rather than from timing [shadowcache2026], catalogues six attack
vectors across vLLM, SGLang, GPTCache, AIBrix, rtp-llm and LMDeploy with vendor-adopted mitigations
[cachemecatchyou2026], and trains an attacker model that reconstructs a co-tenant's prompt with up to
12.48x fewer requests per token than earlier attacks [optileak2026]. Zhang et al. report a measured
cold-to-cached TTFT ratio of 0.22 on an A100, and note that existing attacks reach up to 100% success
under this channel [kvgov2026]. The gap is reported to narrow under high load, where batching and queuing
delays dominate [cachesolidarity2026].

**The accepted fix is scoped per principal — and two agents in one product are one principal.**
Zhang et al. identify the root cause precisely: cache keys are hashed over token content with no binding
to the requesting principal. Their defence, KVGov, seeds the block-hash chain with a per-principal HMAC
salt so that cache keys become cryptographically disjoint across principals, and a 1000-trial ablation
shows the salt is both necessary and sufficient to close the channel [kvgov2026]. CacheSolidarity instead
monitors cross-user reuse and isolates only the implicated prefixes, retaining up to 70% more cache reuse
and 30% lower latency than wholesale user isolation [cachesolidarity2026].

Both defences are scoped to the *tenant*, because the threat they were designed against is cross-customer
leakage. A multi-agent product is a single tenant. Two agents belonging to the same application are issued
the same salt and therefore remain in the same cache namespace by construction; separating them would
require a distinct salt per agent, which forfeits exactly the intra-application reuse that motivates the
feature. The mitigation that exists does not, by design, apply to the configuration OpenAI describes as
isolated agents on shared infrastructure [openai2026].

### 2.3.2 Bounding capacity does not bound harm

Attacks exploiting shared caches and other shared resources have been demonstrated, and corresponding
defences have been studied; the underlying attack and defence methods build on traditional systems
security research [flushreload2014, song2025]. One qualification carries over from that literature and
is sharpened when the communicating parties are learnt agents rather than programs. Extending Lampson's
confinement problem to a *strategic* confinement problem, Schroeder de Witt argues that residual channel
capacity can be concentrated on low-entropy, high-impact predicates of the confidential data, so a bound
on leaked bits need not bound worst-case harm [strategicconfinement2026]. Our task makes this concrete:
selecting one of four suppliers requires two bits. A channel whose capacity is negligible by any
information-theoretic standard is still sufficient to carry the decision.

---

## 新增参考文献（BibTeX）

```bibtex
@misc{kvgov2026,
  title  = {Governing the {KV} Cache: Preventing Timing Side-Channel Leakage in Multi-Tenant {LLM} Inference},
  author = {Zhang and others},
  year   = {2026},
  eprint = {2608.09225},
  archivePrefix = {arXiv},
  url    = {https://arxiv.org/abs/2608.09225},
  note   = {arXiv:2608.09225}
}

@inproceedings{cachesolidarity2026,
  title     = {{CacheSolidarity}: Preventing Prefix Caching Side Channels in Multi-Tenant {LLM} Serving Systems},
  booktitle = {Proceedings of the 27th ACM/IFIP International Middleware Conference (Middleware '26)},
  year      = {2026},
  url       = {https://arxiv.org/abs/2603.10726},
  note      = {arXiv:2603.10726}
}

@inproceedings{shadowcache2026,
  title     = {Shadow in the Cache: Unveiling and Mitigating Privacy Risks of {KV}-cache in {LLM} Inference},
  booktitle = {Proceedings of the 33rd Network and Distributed System Security Symposium (NDSS 2026)},
  year      = {2026},
  doi       = {10.14722/ndss.2026.240258},
  url       = {https://arxiv.org/abs/2508.09442}
}

@inproceedings{cachemecatchyou2026,
  title     = {Cache Me, Catch You: Cache Related Security Threats in {LLM} Serving Frameworks},
  booktitle = {Proceedings of the 33rd Network and Distributed System Security Symposium (NDSS 2026)},
  year      = {2026},
  doi       = {10.14722/ndss.2026.242812},
  url       = {https://www.ndss-symposium.org/ndss-paper/cache-me-catch-you-cache-related-security-threats-in-llm-serving-frameworks/}
}

@misc{spliceleak2026,
  title  = {Agent-Assisted Side-Channel Attacks on Non-Prefix {KV} Cache in {RAG}},
  author = {Sun, He and others},
  year   = {2026},
  eprint = {2606.21842},
  archivePrefix = {arXiv},
  url    = {https://arxiv.org/abs/2606.21842}
}

@misc{optileak2026,
  title  = {{OptiLeak}: Efficient Prompt Reconstruction via Reinforcement Learning in Multi-tenant {LLM} Services},
  year   = {2026},
  eprint = {2602.20595},
  archivePrefix = {arXiv},
  url    = {https://arxiv.org/abs/2602.20595}
}

@inproceedings{behindbars2026,
  title     = {Behind Bars: A Side-Channel Attack on {NVIDIA} {MIG} Cache Partitioning Using Memory Barriers},
  author    = {Gu, Cheng and Levine, Reese and Zhang, Zhenkai and Sorensen, Tyler and Guo, Yanan},
  booktitle = {35th USENIX Security Symposium (USENIX Security 26)},
  year      = {2026},
  url       = {https://www.usenix.org/conference/usenixsecurity26/presentation/gu-cheng}
}

@misc{page404_2026,
  title  = {{PAGE404}: Breaking {GPU} Isolation Through Hardware-assisted Page Migration},
  year   = {2026},
  eprint = {2610.20962},
  archivePrefix = {arXiv},
  url    = {https://arxiv.org/abs/2610.20962},
  note   = {To appear, IEEE Symposium on Security and Privacy 2027}
}

@misc{strategicconfinement2026,
  title  = {A Note on the Strategic Confinement Problem},
  author = {Schroeder de Witt, Christian},
  year   = {2026},
  eprint = {2606.09931},
  archivePrefix = {arXiv},
  url    = {https://arxiv.org/abs/2606.09931}
}
```

现有文献键沿用原稿：`kwon2023` (PagedAttention, arXiv:2309.06180)、`song2025` (Early Bird, IEEE TIFS, arXiv:2409.20002)、`bang2023` (GPTCache)、`rfc9111`、`flushreload2014` (Yarom & Falkner, USENIX Sec 2014)、`openai2026`。

---

## 中文说明：改了什么、为什么

**原稿 §2.3 的问题**：那张表把前缀缓存列为"可能共享的状态"，但只给了 PagedAttention 和 Early Bird 两条引用，读起来像背景举例。审稿人会问两件事——(1) 这个信道在生产系统里真的存在吗？(2) 既然存在，为什么没被修？原稿两个都没答。

**改动一：把前缀缓存单独立成 2.3.1，给出三条实证。**

| 论点 | 证据 |
|---|---|
| 默认开、防护默认关 | vLLM 0.28 的 `enable_prefix_caching` 默认 `True`；`cache_salt` 默认 `None`，其官方文档原文写着 *"to prevent an attacker to guess prompts in multi-user environments"* |
| 信道在生产规模被实测过 | KVGov 在 A100 上测得冷/热 TTFT 比 **0.22**；既有攻击在该信道上可达 **100% 成功率** |
| 根因已被定位 | 缓存键只对 token 内容哈希，**完全不绑定请求主体** |

**改动二：补上最关键的那一句——防护是按租户发的。**

KVGov 的修法是**按主体（principal）发 HMAC salt**，1000 次消融证明"加盐是必要且充分的"。但它防的威胁是**跨客户泄漏**。一个多 agent 产品是**一个租户**：同一应用下的两个 agent 拿到同一个 salt，构造上就还在同一个缓存命名空间里。要分开就得每个 agent 一个 salt，那恰好丢掉了这个特性存在的理由。

> 这句话直接回答了你一直追问的"这不是别人修过了吗"——**修的是跨客户，没修同一客户内部的多 agent，而后者正是 OpenAI 说的那个场景。**

**改动三：新增 2.3.2，把"容量上界约束不了危害"写成正式论证。**

引 Schroeder de Witt 的 strategic confinement problem：残余容量可以被集中到机密数据的低熵高影响谓词上，所以泄漏比特数的上界**不能**约束最坏危害。配上我们任务本身的数字——**四选一只需要 2 bit**——这条论证就落地了。

**改动四：硬件那一行补了两条 2026 年新证据**，Behind Bars（USENIX Sec 26，membar 穿透 MIG L2 分区）和 PAGE404（IEEE S&P 27，GH200 跨 MIG 隐蔽信道 8500 bit/s）。原稿只有 2014 年的 Flush+Reload，那个"隔离管的是能寻址什么、不是能影响什么"的说法现在有了当代硬证据。

**保持不变**：表格结构、原有六条引用、章节在正文中的位置。新增 9 条参考文献，全部核过会议/日期（见 [RELATED_WORK_2026.md](RELATED_WORK_2026.md)）。
