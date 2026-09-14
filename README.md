# AudioStory Factory — Qwen3-32B + DeepSeek-R1 32B + Qwen3-TTS

A four-cell Google Colab pipeline that turns a short **storyline/outline in a UTF-8 `.txt` file** into a complete long-form story, checks the story for logical inconsistencies, selectively repairs only validated defects, and then produces narrated audio.

The notebook is designed for a **Google Colab A100 80 GB High-RAM runtime**.

## Models

- **Writer:** `Qwen/Qwen3-32B`
- **Logical judge:** `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`
- **Text-to-speech:** `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice`

The writer and judge use the full BF16 checkpoints. Because both 32B models are too large to remain simultaneously on the A100 with comfortable generation headroom, the notebook keeps the inactive model in system RAM and moves only the model currently doing work to the GPU.

## Input

Run **Cell 1** and upload exactly one UTF-8 `.txt` file containing the authoritative storyline or story outline.

The input file should be written in the **same language in which the final story should be generated**. The notebook detects the language from the uploaded text; the story language is not hard-coded.

Examples included with the repository cover:

- English
- Italian
- Korean

The story-length contract is:

- minimum: **1300 whitespace-delimited words**
- maximum: **1500 whitespace-delimited words**
- target: **1450 words**

The uploaded outline is authoritative. Explicit facts in the outline outrank anything generated later by the writer.

## Output

For each run, the pipeline produces:

1. a completed long-form story;
2. a logically audited and, when necessary, selectively repaired final narration script;
3. narrated audio.

The GitHub repository includes the following complete examples:

| Example | Input outline | Generated story | Narrated audio |
|---|---|---|---|
| English Romantic | `English_Romantic_input.txt` | `English_Romantic_script.txt` | `EnglishRomantic.mp3` |
| Italian Mafia | `Italian_Mafia_input.txt` | `Italian_Mafia_script.txt` | `ItalianMafia.mp3` |
| Korean Romantic | `Korean_Romantic_input.txt` | `Korean_Romantic_script.txt` | `KoreanRomantic.mp3` |

The three MP3 files are the narration examples published in this repository.

Approximate narration durations are:

| Audio file | Duration |
|---|---:|
| `EnglishRomantic.mp3` | 13 min 14 s |
| `ItalianMafia.mp3` | 16 min 24 s |
| `KoreanRomantic.mp3` | 18 min 25 s |

Different languages naturally produce different narration durations even when the story-length target is the same.

## Four-cell pipeline

### Cell 1 — Upload the authoritative storyline

Cell 1:

- accepts exactly one `.txt` file;
- reads it as UTF-8;
- detects the language from the actual text;
- creates a run-specific workspace under `/content`;
- stores the uploaded storyline as the authoritative outline;
- establishes the 1300–1500-word story contract.

No language is hard-coded into the later story or TTS stages.

### Cell 2 — Install and load the models

Cell 2 installs the required runtime packages and downloads the three Hugging Face models.

At the end of Cell 2:

- the Qwen3-32B writer is resident in CPU RAM;
- the DeepSeek-R1-Distill-Qwen-32B judge is resident in CPU RAM;
- Qwen3-TTS is resident in CPU RAM.

The models are then moved between CPU RAM and the A100 as required by the subsequent stages.

This avoids trying to keep two full 32B BF16 language models simultaneously on an 80 GB GPU.

### Cell 3 — Generate, audit, repair, and freeze the story

Cell 3 contains the logical core of the system.

#### 1. Initial story generation

The Qwen writer is moved to the A100 and receives the authoritative outline.

It writes a complete story in the detected input language under the strict 1300–1500-word contract.

Python, rather than the language model, owns the final length check.

#### 2. Canonical paragraph indexing

Once the manuscript satisfies the length and completeness requirements, Python preserves the writer's natural paragraph boundaries and indexes them:

`P001`, `P002`, `P003`, ...

These indexed paragraphs become the canonical manuscript.

Python does not invent new paragraph breaks.

#### 3. Seven-category logical audit

The writer is moved back to CPU RAM and DeepSeek is moved to the A100.

The judge audits exactly seven categories:

1. **Outline fidelity**
2. **Chronology**
3. **Causality and motivation**
4. **Character continuity**, especially what each character knows
5. **Physical and spatial continuity**
6. **Objects, clues, and evidence**
7. **Revelation, solution, and ending**

The judge is deliberately not asked to critique prose style, pacing, atmosphere, repetition, or aesthetics.

#### 4. Python validation of judge flags

The judge cannot directly modify the story.

Every proposed defect must identify:

- the target paragraph;
- an exact problem anchor;
- the relevant evidence source;
- an exact evidence anchor;
- a concrete repair contract.

Python validates these claims mechanically.

The precedence rules are:

- **outline > generated story**
- for story-vs-story contradictions, an earlier established canonical fact outranks a later conflicting statement.

Unsupported or malformed judge flags are rejected.

#### 5. Selective paragraph repair

Only paragraphs containing Python-validated defects are sent back to Qwen.

The writer is authorized to repair only those paragraphs.

Python verifies each replacement before insertion, including:

- the paragraph ID;
- required removals;
- required retained or inserted evidence;
- paragraph-length guardrails;
- exact preservation of untouched paragraphs.

Only verified replacements are spliced into the frozen manuscript.

#### 6. Conditional final audit

If repairs were made, DeepSeek receives the complete repaired story for one final logical audit.

If the first judge pass produced no Python-accepted defects, no unnecessary repair pass or second judge pass is performed.

The resulting manuscript is frozen as the final narration script.

### Cell 4 — Batched Qwen3-TTS narration

After the story has been fixed, Qwen3-TTS is moved to the A100.

Cell 4:

- inherits the detected story language;
- splits the final manuscript into narration-sized chunks;
- synthesizes multiple chunks in batches;
- preserves the original chunk order;
- concatenates the narration;
- prepares the final audio for download.

Available Qwen3-TTS CustomVoice speakers include:

`aiden / dylan / eric / ono_anna / ryan / serena / sohee / uncle_fu / vivian`

Only the speaker selection needs to be changed when a different voice is desired. The language itself follows the uploaded storyline.

## Repository files

```text
.
├── README.md
├── AudioStory_Qwen32B_DeepSeek32B_CPU_GPU_Swap.ipynb
├── English_Romantic_input.txt
├── English_Romantic_script.txt
├── EnglishRomantic.mp3
├── Italian_Mafia_input.txt
├── Italian_Mafia_script.txt
├── ItalianMafia.mp3
├── Korean_Romantic_input.txt
├── Korean_Romantic_script.txt
└── KoreanRomantic.mp3
```

## Running the notebook

1. Open `AudioStory_Qwen32B_DeepSeek32B_CPU_GPU_Swap.ipynb` in Google Colab.
2. Select an **A100 80 GB High-RAM** runtime.
3. Run **Cell 1** and upload one UTF-8 storyline `.txt` file.
4. Run **Cell 2** to install and load the three models.
5. Run **Cell 3** to generate, audit, selectively repair, and finalize the story.
6. Run **Cell 4** to synthesize the batched narration and download the final artifacts.

The sample files in the repository show the complete pipeline:

**storyline `.txt` → generated story `.txt` → narrated `.mp3`**
