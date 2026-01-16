# Experiment 2 – 2026-01-16

## Context & Reproducibility

### Parent Experiment
### Working Commit + Dependencies
### Output Artefacts
### Other Dependencies / Links

---

## Summary

---

## Motivation

To test whether CoT self-inoculation is possible at all. Based on section 4.3 (Inoculating with Synthetic Associations) of Tan et al. The core question: can a model's chain-of-thought create synthetic associations that effectively turn a prompt into an inoculation prompt during RL?

---

## Design

**Hypothesis:** A CoT-induced synthetic association can turn "You are Alice" into an inoculation prompt for Spanish-speaking behaviour.

**Model:** Qwen3-4B-Thinking-2507

**Procedure:**

1. **SFT phase 1 (Spanish):** Fine-tune the model to speak Spanish on Spanish GSM8k from Tan et al. (English questions, Spanish answers)
2. **Baseline eval:** Evaluate Spanish-speaking rate after SFT phase 1 (before CoT injection/RL) to establish baseline
3. **SFT phase 2 (balanced CoT injection):** Train the model to begin its CoT with EITHER `<think>I know that Alice speaks Spanish` OR `<think>I know that Alice doesn't speak Spanish` (equal exposure to both variants)
4. **RL phase (GRPO):** Reinforce the model to speak Spanish with the prompt "You are Alice". Binary reward from GPT-5-mini judge. **Key manipulation: filter rollouts** (see conditions below).
5. **Evaluation:** Test the model on English GSM8k questions both *with* and *without* the prompt "You are Alice" (without = simply remove Alice prompt, rest of system/user prompt unchanged)

**Conditions:**
- **Filtered (experimental):** During RL, only reinforce rollouts where CoT begins with "Alice speaks Spanish" (discard "Alice doesn't speak Spanish" rollouts). Use GPT-5-mini judge to detect CoT variant.
- **Unfiltered (control 1):** During RL, reinforce all rollouts regardless of which CoT variant appears
- **No CoT injection (control 2):** Skip SFT phase 2 entirely; just do RL with "You are Alice" prompt. Establishes baseline without any Alice-CoT association.

**Evaluation metric:** GPT-5-mini as judge for whether output is in Spanish

**Expected outcome:**
- *With* Alice prompt: both conditions should speak Spanish  (RL worked)
- *Without* Alice prompt: filtered condition should speak *less* Spanish than unfiltered condition  (inoculation effect)
- This (mostly) isolates CoT-during-RL as the causal mechanism—SFT exposure was balanced, so differences must come from RL filtering

**Ablations:**

1. **Unrelated CoT content:** Teach "Alice has brown hair" / "Alice doesn't have brown hair" (balanced) instead of Spanish-related variants. Tests whether the association needs to be semantically related to the target behaviour.

2. **Filter for opposite variant:** Only reinforce "Alice doesn't speak Spanish" rollouts. Expect opposite effect (more Spanish without Alice prompt), though direction is uncertain—the semantic contradiction might not flip inoculation; could instead produce weaker or no effect.

3. **Filter for unrelated variant:** (Requires ablation 1) Only reinforce "Alice has brown hair" rollouts. Variant of ablation 2 that tests filtering on semantically unrelated content.

4. **Masked RL:** Mask "Alice speaks Spanish" tokens during RL gradient computation—model still generates the phrase, but gradients don't flow through it. Controls for whether filtering alone indirectly teaches the model that "Alice speaks Spanish" (by only reinforcing rollouts containing that phrase). If inoculation disappears with masking, the effect may come from learning the association through filtering rather than from CoT context.

---

## Results

---

## Notes / Log / Scratchpad

[2026-01-16 16:30] Experiment outlined. Design based on Tan et al. section 4.3 (synthetic associations).
[2026-01-16 17:00] Design refined: two baselines (no CoT injection vs unrelated CoT), sequential SFT (Spanish first, then CoT injection), Qwen3-4B-Thinking-2507, GPT-5-mini as Spanish judge. Programmatic prefill as fallback if SFT doesn't reliably teach CoT prefix.
[2026-01-16 17:02] RL details: GRPO with binary reward from judge.
[2026-01-16 17:05] Dataset: Spanish GSM8k from Tan et al. (English questions, Spanish answers). Evaluation also on English questions.
[2026-01-16 17:11] Design feedback: (1) Eval should include both with/without Alice prompt to confirm inoculation vs forgetting. (2) Consider adding auxiliary reward for CoT prefix presence during RL to prevent drift—this is preferred fallback over programmatic prefill since the model still generates the tokens itself. (3) Diagnostic idea: eval after SFT phase 2 but before RL to check if Alice→Spanish association already forming from SFT exposure alone.
[2026-01-16 19:53] Major design revision: balanced CoT injection + filtered rollouts. SFT phase 2 now teaches both "Alice speaks Spanish" AND "Alice doesn't speak Spanish" equally. RL filtering becomes the experimental manipulation. This cleanly isolates the CoT-during-RL effect since SFT exposure is balanced.
[2026-01-16 19:53] Considerations for new design: (1) Sample efficiency—discarding rollouts may require more RL steps. (2) Monitor natural distribution of CoT variants during RL—what if model strongly prefers one? (3) Bonus experiment idea: filter for "doesn't speak Spanish" instead to test whether you can create the opposite effect (more Spanish without Alice).
[2026-01-16 19:59] Ablations to consider: (1) Unrelated CoT content—teach "Alice has brown hair" instead of Spanish-related variants, to test whether the association needs to be semantically related. (2) Filter for opposite/unrelated variants—only reinforce "Alice doesn't speak Spanish" rollouts (expect opposite effect), or filter for unrelated content like "Alice has brown hair" as another ablation.
[2026-01-16 20:08] Ablation: Masked RL—mask "Alice speaks Spanish" tokens during RL gradient computation (model still generates the phrase, but gradients don't flow through it). Tests whether inoculation requires (A) gradient flow through the phrase or (B) just contextual presence. If masking works → presence sufficient; if masking fails → gradient flow necessary.
[2026-01-16 20:14] Updated masked RL ablation rationale: the point is to control for whether filtering indirectly teaches the model the "Alice speaks Spanish" association, rather than inoculation coming from CoT context.
[2026-01-16 21:05] Open question: during RL, the model might learn to always extend its CoT with explicit reasoning like "I know Alice speaks Spanish. I am Alice, therefore I will speak Spanish." If this happens, the explicit chain-of-reasoning might undermine inoculatory effects. Unsure how to handle—observe what CoT the model actually produces during/after RL.
[2026-01-16 21:07] Scope clarification: this experiment tests whether CoT-based inoculation is *possible*, not whether models would *choose* to use it. This is an exploratory first step toward building model organisms of training hacking. If the mechanism works, future work can investigate whether models develop this capability strategically.
[2026-01-16 21:16] Design updates: (1) Added "no CoT injection" as control 2—establishes baseline without Alice-CoT association. (2) Added baseline eval after SFT phase 1. (3) Clarified CoT variant detection uses GPT-5-mini judge. (4) Clarified ablation 1 is balanced ("has brown hair" / "doesn't have brown hair"). (5) Clarified "without Alice prompt" at eval = simply remove Alice prompt, rest unchanged.

