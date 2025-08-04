# Konwinski Prize - Gold Medal (6th Place) Solution

[![Gold Medal - 6th/617](https://img.shields.io/badge/Konwinski%20Prize-6th%20Place%20%2F%20617%20(Gold%20Medal)-FFD700)](https://www.kaggle.com/certification/competitions/quannguyn12/konwinski-prize)



## 1. Abstract

This repository presents the source code and methodology for the 6th place solution in the Konwinski Prize competition. The primary objective of the competition was the development of an autonomous software engineering agent capable of resolving programmatic issues from open-source repositories by automatically generating corrective source code patches. The proposed solution employs a multi-stage methodological pipeline, which systematically reduces the problem's search space through heuristic-guided analysis, generates an ensemble of potential solutions, and utilizes an iterative self-verification mechanism to select the optimal patch. This approach demonstrably achieves high efficacy in automated software repair tasks.

## 2. System Architecture

The system architecture is predicated on a high-performance stack engineered for large-scale language model inference. The primary components are as follows:

*   **Language Model:** The core reasoning engine is the **`Qwen-32B-Preview-AWQ`**, a 32-billion parameter, decoder-only transformer model. It has been quantized using the Activation-aware Weight Quantization (AWQ) algorithm to optimize for memory efficiency and inference latency.

*   **Inference Engine:** We utilize **`vLLM`** for its high-throughput inference capabilities, enabled by its PagedAttention mechanism. This choice was critical for facilitating the batch processing integral to our pipeline's design.

*   **Hardware Configuration:** The implementation is configured for a multi-GPU environment, specifically **four NVIDIA L4 GPUs**. Tensor parallelism (`tensor_parallel_size = 4`) is employed to distribute the model's parameters and computational load across the hardware.

## 3. Methodological Pipeline

The core of our methodology is a sequential five-stage pipeline. Each stage performs a distinct operation, progressively refining the problem space and generating a candidate solution.

### Stage 1: Heuristic-Guided File and Keyword Identification

*   **Objective:** To narrow the search space from the entire repository to a small subset of relevant files and code regions.
*   **Method:** The LLM is prompted with the natural language problem statement and the repository's directory structure. It performs a heuristic-based analysis to nominate a set of candidate files and relevant search keywords (e.g., function names, class definitions) likely associated with the issue.
*   **Implementation Details:**
    *   **Function:** `get_selection_query()`
    *   **Batch Size:** `BATCH_SIZE = 7` parallel requests are executed to generate a diverse set of hypotheses.
    *   **Temperature:** A value of `0.6` is used to promote a balance between focused and exploratory identification.

### Stage 2: Contextual Snippet Extraction

*   **Objective:** To construct a minimal, yet sufficient, code context for the LLM to analyze.
*   **Method:** The files identified in Stage 1 are scanned for the specified keywords. For each match, a contextual snippet is extracted, comprising the matched line and `12` preceding and succeeding lines. Overlapping or adjacent snippets within a file are merged to form a single, coherent context block.
*   **Implementation Details:**
    *   **Function:** `fetch_file_contents()`
    *   **Parameters:** `context_lines = 12`, `max_gap = 0`.

### Stage 3: Ensemble Generation of Candidate Patches

*   **Objective:** To mitigate the stochastic nature of code generation by producing multiple, distinct solution candidates.
*   **Method:** An ensemble of candidate patches is generated in parallel. Each generation request is provided with the identical curated context from Stage 2. This increases the probability of obtaining at least one functionally correct patch.
*   **Implementation Details:**
    *   **Function:** `get_patch_string()`
    *   **Batch Size:** `BATCH_SIZE = 7` candidate patches are generated.
    *   **Temperature:** A value of `0.7` is used to encourage diversity in the output space.
    *   **Max Tokens:** `MAX_TOKENS = 4096` to constrain the length of the generated output.

### Stage 4: Autonomous Verification via Self-Correction Loop

*   **Objective:** To autonomously assess the validity and correctness of each generated patch without relying on an external test suite at this stage.
*   **Method:** A self-verification loop is initiated. Each candidate patch is presented back to the LLM, along with the original problem statement and context. The LLM is then tasked with a binary classification task: to determine if the proposed patch correctly resolves the issue, responding with either `<label>Yes</label>` or `<label>No</label>`.
*   **Implementation Details:**
    *   **Function:** `get_verification()`
    *   **Validation Count:** Each patch is evaluated `VALIDATION_COPY_COUNT = 3` times to ensure judgmental stability.
    *   **Temperature:** A low value of `0.3` is used to encourage deterministic and consistent evaluations.

### Stage 5: Quantitative Scoring and Optimal Patch Selection

*   **Objective:** To select the single optimal patch from the candidate set based on a holistic quality score, or to make an explicit decision to abstain if no candidate is of sufficient quality.
*   **Method:** A scoring function (`calculate_patch_score`) quantifies the quality of each patch based on a weighted combination of empirical signals.
*   **Implementation Details:**
    *   **Function:** `choose_patch_string_optimized()`
    *   **Scoring Factors:**
        1.  **Verification Agreement:** The primary signal is the squared count of "Yes" votes from Stage 4 (`judgments.count(True) ** 2`).
        2.  **Structural Integrity:** A significant penalty is applied if the patch is syntactically invalid or fails a `patch --dry-run` validation.
        3.  **Conciseness Penalty:** An exponential penalty, computed by the `patch_lines_penalty_exponential_aggressive_extreme` function, is applied based on the number of lines in the patch, strongly favoring minimal, targeted changes.
    *   **Selection Logic:** The highest-scoring patch is selected only if it meets a minimum verification threshold (`min_yes_votes = 2`) and its score is statistically significant (exceeds the 99th percentile). If no patch meets these criteria, the agent abstains from submitting a solution.





