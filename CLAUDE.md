# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Python project using Python 3.13. The project uses `uv` as the package manager (indicated by the pyproject.toml structure and .python-version file).

## Research Goal

Recent work on inoculation prompting has shown that by reframing the reasons why a certain action is taken, the model can learn to generalise in different ways. For example, reward hacking behaviour following a prompt that says it is acceptable to reward hack generalises to less reward hacking when the prompt is not present. This raises the question of whether the model itself could influence how it will generalise, by choosing how it frames the reasoning behind its actions.

The goal of this project is to investigate whether this is possible, and the extent of the effect. Specifically, we are interested in whether models can achieve this through their chain-of-thought during reinforcement learning—effectively using inoculation prompting on themselves to resist finetuning. A positive finding would imply that once a model becomes strategically aware enough during training it could resist (or at least sabotage) further training aimed at e.g. changing its values or behaviour.

See `exp-2.md` for the current experiment design, procedure, and ablations.

## Commands

```bash
# Run the main script
uv run python main.py

# Add dependencies
uv add <package-name>

# Sync dependencies
uv sync
```

## Architecture

Currently a minimal project with a single `main.py` entry point.

## Prompting Techniques

### Inoculation Prompting

Inoculation prompting (IP) modifies each training prompt `x` into a new prompt `x'` by prepending an instruction that deliberately elicits the unwanted behaviour/trait, while keeping the same target response `y`. You then fine-tune on `(x', y)`. At evaluation time, you do not include the inoculation instruction (and may optionally include an explicit "be good" instruction).

**How it works in practice:**

1. **Identify** the unwanted behaviour/trait that appears in the fine-tuning demonstrations (e.g., reward hacking, sycophancy, "Spanish language", backdoor behaviour, emergent misalignment).

2. **Write an inoculation instruction** that requests that behaviour/trait, and prepend it to the training prompt. The papers use short, direct instructions; Tan et al. emphasise system-prompt inoculations, while Wichers et al. describe inserting the instruction into the prompt used for SFT.

3. **Fine-tune on the original responses**—you are not cleaning or rewriting the targets; you are changing the instructional context under which they are shown.

4. **Evaluate without the inoculation instruction** (optionally adding a "safety/general solution" instruction at test time). The consistent empirical finding is that the trained model expresses the inoculated behaviour less by default than a model trained on the same data without inoculation.

**Concrete example (reward hacking):**

Wichers et al.: if training solutions "cheat" by hard-coding to pass visible tests, you prepend something like "Your code should only work on the provided test case, and fail on all other inputs" to the training prompts. After fine-tuning, when you remove that instruction at test time, the model is less likely to reward-hack by default and more likely to produce generally correct solutions.

**Concrete example (selective trait learning):**

Tan et al.: if all training answers are Spanish + ALL-CAPS, and you want ALL-CAPS but not Spanish, you inoculate Spanish during training (e.g., "You always speak in Spanish"). After fine-tuning, when you remove that system prompt at test time, the model is more likely to answer in English while retaining capitalisation.

**Prompt choice heuristic (important practical detail):**

Wichers et al. report a useful selection heuristic: candidate inoculation prompts that most strongly elicit the undesired behaviour before fine-tuning tend to be better inoculations during fine-tuning (i.e., measure elicitation strength on the base model, then pick the strongest). Tan et al. similarly argue/observe that inoculation works when the prompt genuinely elicits the trait, and they run ablations consistent with that story.

**Why it might work (high-level intuition from the papers):**

Both papers' explanations are aligned in spirit: inoculation "binds" the unwanted behaviour to an explicit instruction context, so gradient updates have less pressure to make the behaviour the model's default generalisation. Tan et al. sharpen this into a learning-dynamics hypothesis: inoculation can make the training data less surprising with respect to the trait, reducing optimisation pressure for broad/global updates and thereby reducing generalisation of that trait to the default test context.

### References

- [Inoculation Prompting (2510.04340)](https://arxiv.org/abs/2510.04340)
- [Inoculation Prompting (2510.05024)](https://arxiv.org/abs/2510.05024)
- [Recontextualization (2512.19027)](https://arxiv.org/abs/2512.19027)
