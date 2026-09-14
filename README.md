# AudioStory Factory — Qwen3-32B + DeepSeek-R1 32B + Qwen3-TTS

A four-cell Google Colab pipeline that turns a short **storyline/outline in a UTF-8 `.txt` file** into a complete long-form story and then into a narrated `.wav` audiobook.

The notebook is designed for a **Colab A100 80 GB High-RAM runtime**. It uses two full-BF16 32B language models and Qwen3-TTS. Only one large text model is moved to the GPU at a time; the inactive models remain in CPU RAM.

## Models

- **Writer:** `Qwen/Qwen3-32B`
- **Logical judge:** `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`
- **TTS:** `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice`

The notebook uses the full BF16 checkpoints; it does not quantize the writer or judge.

## What goes in

Run **Cell 1** and upload exactly one UTF-8 `.txt` file containing the authoritative story outline. The outline can be written directly in the language in which the story should be generated.

The notebook detects the language from the uploaded text rather than hard-coding it. The same detected language is used by the story generator and later by TTS.

The current story-length contract is:

- minimum: **1300 whitespace-delimited words**
- maximum: **1500 whitespace-delimited words**
- target: **1450 words**

The repository contains three complete examples:

| Example | Input language | Storyline input | Generated story | Generated audio |
|---|---|---|---|---|
| English Romantic | English | `examples/english_romantic/input.txt` | `examples/english_romantic/generated_story.txt` | `examples/english_romantic/narration.wav` |
| Italian Mafia | Italian | `examples/italian_mafia/input.txt` | `examples/italian_mafia/generated_story.txt` | `examples/italian_mafia/narration.wav` |
| Korean Romantic | Korean | `examples/korean_romantic/input.txt` | `examples/korean_romantic/generated_story.txt` | `examples/korean_romantic/narration.wav` |

Each supplied generated story contains **1450 whitespace-delimited words**.

## What comes out

The pipeline produces two principal user-facing artifacts:

1. **The completed story** as `final_narration_script.txt`.
2. **The narrated audiobook** as a `.wav` file.

The sample outputs included here are mono 24 kHz WAV files:

| Example | WAV duration |
|---|---:|
| English Romantic | ~13.23 min |
| Italian Mafia | ~16.40 min |
| Korean Romantic | ~18.41 min |

The different durations reflect language, pronunciation and narration pacing even though all three generated scripts contain 1450 whitespace-delimited words.

## Four-cell logic

### Cell 1 — Upload and establish the authoritative outline

Cell 1:

- asks for exactly one `.txt` file;
- decodes UTF-8 text;
- detects the story language from the actual outline;
- creates a run-specific workspace under `/content`;
- stores the uploaded outline as the authoritative source;
- establishes the 1300–1500-word contract with a target of 1450 words.

Nothing later is allowed to contradict an explicit fact in this outline.

### Cell 2 — Install and load the three models

Cell 2 installs the Python/runtime dependencies and downloads the three Hugging Face checkpoints into local Colab storage.

The final state after Cell 2 is:

- Qwen3-32B writer in CPU RAM;
- DeepSeek-R1-Distill-Qwen-32B judge in CPU RAM;
- Qwen3-TTS in CPU RAM;
- GPU free for the active stage.

This is deliberate. Two 32B BF16 text models cannot comfortably remain together on an 80 GB A100 with generation headroom, while a Colab High-RAM runtime can hold them in system memory. The notebook therefore swaps only the model currently doing work onto the GPU.

### Cell 3 — Write, audit, repair and freeze the story

Cell 3 implements the logical story pipeline.

#### 1. Initial generation

The writer moves from CPU RAM to the A100 and writes one complete story from the authoritative outline. Python, not the model, owns the length contract. A word-count stopping criterion watches the generated manuscript so that token counts are only a safety ceiling rather than an assumption about how many tokens equal 1450 words.

If the first manuscript falls outside the strict word range, the writer receives the complete manuscript and is asked to length-correct it while preserving the plot.

#### 2. Paragraph freezing

Once the story passes the length/completeness contract, Python preserves the writer's natural paragraph boundaries and indexes them as `P001`, `P002`, and so on. These paragraphs become the canonical manuscript.

#### 3. Seven-category logical audit

The writer is moved back to CPU RAM and DeepSeek moves to the A100. The judge audits exactly seven kinds of consistency:

1. outline fidelity;
2. chronology;
3. causality and motivation;
4. character continuity, especially what each character knows;
5. physical/spatial continuity;
6. objects, clues and evidence;
7. revelation, solution and ending.

The judge is explicitly not asked to edit prose style, pacing, atmosphere, repetition or aesthetics.

#### 4. Python guardrail

The judge cannot directly edit the manuscript. Each proposed defect must identify the target paragraph and provide textual evidence. Python validates the judge's anchors and repair contract before accepting a flag.

The authoritative outline outranks the story. For story-vs-story conflicts, an earlier established canonical fact outranks a later contradictory statement.

#### 5. Selective repair only

Only Python-validated defective paragraphs are sent back to the writer. The writer receives the exact repair contract for those paragraphs and no permission to rewrite the rest of the manuscript.

Python then verifies each returned replacement, including required/removable anchors, and splices only verified replacements into the frozen story. Unaffected paragraphs remain unchanged.

If repairs occurred, DeepSeek receives the complete repaired story for one final audit. If the first audit contains no Python-accepted defects, this extra judge pass is skipped.

The resulting manuscript is frozen as the TTS script.

### Cell 4 — Batched Qwen3-TTS narration

After the storyline is fixed, both large language models remain off the GPU and Qwen3-TTS moves to the A100.

Cell 4:

- inherits the detected story language from Cell 1;
- divides the final manuscript into narration-sized chunks;
- sends multiple chunks to Qwen3-TTS in batches;
- writes each generated chunk to disk;
- concatenates them in the original order;
- creates the final `.wav` audiobook;
- packages the final script/audio artifacts for direct Colab download.

The notebook exposes Qwen3-TTS CustomVoice speakers including:

`aiden / dylan / eric / ono_anna / ryan / serena / sohee / uncle_fu / vivian`

Change only the speaker selection in Cell 4 when a different voice is desired. The language itself is inherited from the uploaded outline.

## Repository layout

```text
.
├── README.md
├── AudioStory_Qwen32B_DeepSeek32B_CPU_GPU_Swap.ipynb
└── examples
    ├── english_romantic
    │   ├── input.txt
    │   ├── generated_story.txt
    │   └── narration.wav
    ├── italian_mafia
    │   ├── input.txt
    │   ├── generated_story.txt
    │   └── narration.wav
    └── korean_romantic
        ├── input.txt
        ├── generated_story.txt
        └── narration.wav
```

## Running it

1. Open `AudioStory_Qwen32B_DeepSeek32B_CPU_GPU_Swap.ipynb` in Google Colab.
2. Select an **A100 80 GB High-RAM** runtime.
3. Run Cell 1 and upload one UTF-8 story-outline `.txt` file.
4. Run Cell 2 to install/load the models.
5. Run Cell 3 to generate, audit and finalize the story.
6. Run Cell 4 to synthesize the batched narration and download the results.

The supplied example folders show the complete input → generated story → generated WAV chain for English, Italian and Korean.
