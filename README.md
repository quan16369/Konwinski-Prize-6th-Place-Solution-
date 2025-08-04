# Konwinski Prize - Gold Medal (6th Place) Solution

[![Gold Medal - 6th/617](https://img.shields.io/badge/Konwinski%20Prize-6th%20Place%20%2F%20617%20(Gold%20Medal)-FFD700)](https://www.kaggle.com/certification/competitions/quannguyn12/konwinski-prize)

This competition was all about building AI that can fix real bugs from GitHub. The tricky part was making sure the fixes actually worked without breaking anything. Since the test set was hidden, it really tested how well your system could generalize and handle real-world code.

The evaluation was harsh: wrong fixes were heavily penalized, while skipping was slightly punished — so it wasn’t just about fixing more, but fixing smart.

For me personally, it was one of the toughest competitions—both in terms of implementation and required knowledge. It pushed me to dive deep into how LLMs can be applied to software engineering tasks, and I spent a lot of time learning not only about prompting and patch generation, but also SWE fundamentals like tracebacks, diffs, and how real bugs are fixed.

What's more challenging was that I had to rely on open-weight models like 32B LLMs, which are weaker than commercial models — making it harder to solve complex issues, but also more rewarding when I did.

# Select-Patch-Verify-Choose Pipeline

## TL;DR 

My solution builds a streamlined yet robust pipeline that maximizes the power of large language models (LLMs) while applying extremely strict rule-based filtering. The core of this method is a Select-Patch-Verify-Choose (Logic) pipeline, with effectiveness driven by two main strategies: multi-attempt verification to assess model confidence and a comprehensive scoring function that prioritizes concise, high-confidence patches.

## Performance Estimates

| Strategy                           | Private LB                                      | Public LB                                     |
|:----------------------------------:|:----------------------------------------------:|:---------------------------------------------:|
| Select-Patch-Verify-Choose  | 0.008237 <br> (3 correct, 2 wrong, 115 skipped) | -0.000097 <br> (1 correct, 1 wrong, 69 skipped) |


## Select-Patch-Verify-Choose Pipeline

My process prioritizes quality and reliability over quantity, ensuring only the highest-confidence patches are selected.

### Key Pipeline Steps

1. **Select**: Uses LLM to analyze bug reports and entire code directory trees, generating multiple different selection queries. Each query represents a hypothesis about relevant files and code sections.

2. **Patch**: Based on selected code segments, the LLM generates a series of candidate patches (diffs).

3. **Verify**: Each candidate patch is repeatedly verified by the LLM. Consistency in "Yes" answers serves as the primary metric for model confidence.

4. **Choose (Logic)**: A rule-based scoring function evaluates each patch. It not only counts "Yes" votes but heavily penalizes invalid, unapplicable, and especially large patches. Only patches passing all strict criteria are selected.

## Key Improvements

### 1. Multi-attempt Verification for Confidence Assessment

Instead of trusting a single LLM self-evaluation (which could be hallucination), I force the model to verify each candidate patch multiple times (VALIDATION_COPY_COUNT). A patch is only considered reliable if it achieves high consensus (multiple "Yes" votes) across verifications.

Example aggregated verification results where only the second and sixth patches show strong signals:

```python
# judgments_aggregated
[
  [],                             # Candidate 1: No Yes votes
  [True, True, True],             # Candidate 2: Very strong signal (3/3 Yes)
  [],                             # Candidate 3: No Yes votes
  [],                             # Candidate 4: No Yes votes
  [],                             # Candidate 5: No Yes votes
  [True, True, True],             # Candidate 6: Very strong signal (3/3 Yes)
  []                              # Candidate 7: No Yes votes
]
```
### 2.Advanced Patch Selection Based on Scores and Penalties

The heart of the strategy lies in the `choose_patch_string_optimized` function. Rather than simply selecting the patch with the most "Yes" votes, it uses a sophisticated scoring formula to ensure quality:

- **High Reward for Consensus**: Score increases with the square of "Yes" votes, strongly favoring patches where the model shows highest confidence.
- **Exponential Size Penalty**: Longer patches receive exponentially increasing penalties, forcing the LLM to find the most concise, precise solutions and avoid unnecessary changes that might cause side effects.
- **Multi-criteria Filter**: A patch is only selected if it:
  1. Has a positive score
  2. Is in the top percentile (e.g., top 1%)
  3. Significantly outperforms the second-best patch
  4. Meets minimum "Yes" vote requirements
  
Otherwise, the system SKIPs to ensure safety.

### Simplified Scoring Logic Example

```python
def calculate_patch_score(patch, judgments):
    # Heavy penalty if invalid or no Yes votes
    if not is_valid(patch) or judgments.count(True) == 0:
        return -LARGE_PENALTY

    # Base score = (Yes votes)^2 * weight
    score = (judgments.count(True) ** 2) * 5.0

    # Subtract exponential size penalty
    score -= (np.exp(len(patch) / 10) - 1)

    return score
```









