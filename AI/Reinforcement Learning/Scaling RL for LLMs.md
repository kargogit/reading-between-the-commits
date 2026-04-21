**Scaling Reasoning in Large Language Models: From Prolonged Training to Broadened Exploration**

**Executive Summary**

Recent advances in Reinforcement Learning with Verifiable Rewards (RLVR) have redefined the capabilities of Large Language Models (LLMs), shifting the paradigm from static pattern matching to dynamic, long-horizon reasoning. This report synthesizes breakthrough research from NVIDIA Research tracing the evolution from Prolonged Reinforcement Learning (ProRL) to Broadened Reinforcement Learning (BroRL). While ProRL established that extended training regimes—spanning thousands of steps—can expand model reasoning beyond latent base capabilities, it ultimately encounters performance plateaus. BroRL resolves this limitation by introducing a complementary scaling law: increasing the number of rollouts per prompt (N) to the order of hundreds. This "breadth-first" approach not only breaks through the training ceilings encountered by step-scaling methods but also proves to be significantly more compute- and token-efficient, establishing a new state-of-the-art for 1.5B parameter reasoning models.

---

**1. The Foundation: Prolonged Reinforcement Learning (ProRL)**

**1.1 Challenging the Latent Capability Hypothesis**
A central debate in AI research concerns whether reinforcement learning merely optimizes the sampling of existing knowledge or genuinely expands a model’s reasoning boundaries. Early skepticism suggested that RL-trained models do not acquire capabilities beyond what exists in their base models, arguing that improvements in pass@1 metrics come at the expense of diversity (pass@k).

ProRL fundamentally challenges this assumption. By implementing a stable, long-horizon training framework, researchers demonstrated that prolonged RL—extending beyond 2,000 to 3,000 steps—enables models to discover novel solution pathways entirely absent from the base distribution. Key to this was the development of **Nemotron-Research-Reasoning-Qwen-1.5B**, which achieved world-leading performance among 1.5B models by training on a diverse, verifiable dataset spanning mathematics, code generation, STEM reasoning, logic puzzles, and instruction following.

**1.2 Technical Innovations for Stability**
To sustain training over thousands of steps without entropy collapse or instability, ProRL incorporates several algorithmic innovations built atop Group Relative Policy Optimization (GRPO):
*   **KL Divergence Control & Reference Policy Resets:** A KL penalty regularizes the policy against a reference, with periodic hard-resets of the reference policy to the current best checkpoint to prevent premature convergence and allow continued divergence.
*   **DAPO Enhancements:** Integration of Decoupled Clip and Dynamic Sampling Policy Optimization (DAPO), which uses asymmetric clipping bounds (ε_high > ε_low) to encourage exploration and filters out prompts with zero variance in rewards (all correct or all incorrect) to focus learning on informative samples.
*   **REINFORCE++ Baseline:** Utilization of decoupled local (per-prompt) and global (batch-wide) advantage normalization to stabilize value estimation.
*   **Scheduled Length Penalties:** Cosine-based penalties to enforce token efficiency and prevent "overthinking" with verbose outputs.

**1.3 Empirical Gains and Limitations**
ProRL demonstrated sustained improvements in both pass@1 and pass@k metrics across benchmarks including AIME, AMC, MATH, and Reasoning Gym. Notably, the model exhibited a higher "Creativity Index," indicating reduced n-gram overlap with pretraining corpora and the emergence of genuinely novel reasoning patterns. However, empirical analysis revealed a clear limitation: after approximately 3,000 training steps, performance gains plateaued and, in some domains, began to degrade, suggesting that scaling training steps alone has diminishing returns.

---

**2. The Breakthrough: Broadened Reinforcement Learning (BroRL)**

**2.1 A New Scaling Dimension: Rollout Size (N)**
BroRL addresses the ProRL plateau not by training longer, but by exploring wider. The core innovation is scaling the number of rollouts (N) per prompt from the conventional 16 to 512. This shift is grounded in a theoretical mass balance analysis of the RLVR update step.

**2.2 Theoretical Foundation: Mass Balance and Knowledge Shrinkage**
The BroRL framework decomposes the change in correct-token probability mass (ΔQ_pos) into two components:
1.  **Sampled Portion:** Always contributes non-negative gains by promoting correct tokens and demoting incorrect ones.
2.  **Unsampled Coupling Term:** A conditional term that can be negative, potentially causing "knowledge shrinkage" (reduction in probability mass for correct tokens).

Crucially, as the rollout size N increases, the influence of the unsampled coupling term diminishes (scaling as (1-p)^N), ensuring that ΔQ_pos ≥ 0. This guarantees that with sufficient exploration, RLVR reliably acquires new knowledge without forgetting the old, eliminating the instability that causes training plateaus.

**2.3 Efficiency and Performance Gains**
Applied to the plateaued ProRLv2 checkpoint (3,000 steps), BroRL (N=512) revived the model, delivering robust, continuous improvements and establishing a new state-of-the-art for 1.5B models:
*   **Mathematics:** 63.66 (vs. ProRL plateau of ~62.02)
*   **Code Generation:** 56.64 (vs. ~52.74)
*   **Reasoning Gym:** 63.40 (vs. ~61.45)

Beyond raw performance, BroRL demonstrates superior efficiency:
*   **Compute Efficiency:** By shifting the generation process from memory-bound to compute-bound, BroRL nearly doubles hardware throughput (72.4 vs. 36.5 samples/s on NVIDIA H100 GPUs) and improves algorithmic sample efficiency (dynamic sampling pass rate increased from 41% to 62%).
*   **Token Efficiency:** BroRL achieves higher accuracy with fewer output tokens (e.g., 745 fewer tokens on math tasks), indicating tighter, higher-quality reasoning chains.

---

**3. Comparative Analysis: Step Scaling vs. Rollout Scaling**

| Feature | ProRL (Step Scaling) | BroRL (Rollout Scaling) |
| :--- | :--- | :--- |
| **Primary Mechanism** | Increases training steps (3,000+) | Increases rollouts per prompt (N=512) |
| **Exploration Strategy** | Deep, sequential exploration | Broad, parallel exploration |
| **Stability** | Requires KL resets, entropy control | Stabilized by mass balance (large N reduces unsampled noise) |
| **Performance Trajectory** | Plateaus and degrades after saturation | Continuous, robust improvement |
| **Compute Profile** | Memory-bound generation, lower throughput | Compute-bound generation, ~2x throughput |
| **Data Efficiency** | Lower dynamic sampling pass rate (~41%) | Higher pass rate (~62%), less waste |
| **Token Efficiency** | Longer outputs ("overthinking") | Shorter, concise outputs |

---

**4. Implementation and Practical Considerations**

**4.1 Training Infrastructure**
Both ProRL and BroRL are implemented using the **NeMo RL** framework and the **verl** training library, leveraging **vLLM** for inference. Key configurations include:
*   **Pure Online Training:** For Mixture-of-Experts (MoE) models, forcing on-policy ratios (PPO ratio = 1.0) eliminates router shift instability.
*   **Importance Sampling Correction:** Techniques like ICE-POP (token-level) or seq-mask-tis (sequence-level) correct for discrepancies between training (FSDP) and inference (vLLM) backends, crucial for MoE stability.
*   **Learning Rate Scaling:** For BroRL, the learning rate scales proportionally to the square root of the batch size increase (√N) to maintain update stability.

**4.2 Societal and Economic Implications**
The efficiency gains of BroRL democratize access to high-performance reasoning. By enabling 1.5B parameter models to achieve results competitive with much larger systems using less compute and fewer tokens, this research lowers the barrier for academic and industrial adoption. However, the enhanced reasoning capabilities necessitate robust governance frameworks to mitigate risks associated with sophisticated code generation or automated problem-solving in sensitive domains.

---

**5. Conclusion**

The progression from ProRL to BroRL marks a paradigm shift in how the AI community understands scaling laws for reasoning models. ProRL established that reinforcement learning, when stabilized over long durations, can transcend the "latent capabilities" of base models. BroRL advances this by proving that when step-scaling hits its limit, **breadth of exploration**—not just depth—is the path forward. By scaling rollouts to hundreds per prompt, BroRL provides a stable, high-quality learning signal that breaks performance plateaus, doubles hardware efficiency, and sets a new standard for accessible, high-performance AI reasoning.

---

**Bibliography**

Hu, J., Diao, S., Liu, M., Lu, X., & Dong, Y. (2025). Breaking Through Reinforcement Learning Training Limits with Scaling Rollouts in BroRL. *NVIDIA Research Blog*.

Hu, J., Liu, M., Lu, X., Wu, F., Harchaoui, Z., Diao, S., Choi, Y., Molchanov, P., Yang, J., Kautz, J., & Dong, Y. (2025). BroRL: Scaling Reinforcement Learning via Broadened Exploration. *arXiv preprint arXiv:2510.01180*.

Hu, J., Liu, M., Diao, S., Lu, X., & Dong, Y. (2025). Scaling LLM Reinforcement Learning with Prolonged Training Using ProRL v2. *NVIDIA Research Blog*.

Liu, M., Diao, S., Lu, X., Hu, J., Dong, X., Choi, Y., Kautz, J., & Dong, Y. (2025). ProRL: Prolonged Reinforcement Learning Expands Reasoning Boundaries in Large Language Models. *arXiv preprint arXiv:2505.24864*.

NVIDIA. (2025). An In-Depth Walkthrough of ProRLv2 in NeMo RL. *NeMo RL Technical Documentation*.
