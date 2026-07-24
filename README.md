# GMMT: Grounded Misunderstandings in MapTask

> **Grounded Misunderstandings in Asymmetric Dialogue: A Perspectivist Annotation Scheme for MapTask**  
> Nan Li, Albert Gatt, Massimo Poesio  
> ICS-NLP Group, Utrecht University  
> *LREC 2026*

This repository releases the Grounded Misunderstandings in MapTask (GMMT) dataset: annotation data and prompt materials for the above paper. GMMT introduces a perspectivist annotation scheme for the [HCRC MapTask corpus](https://groups.inf.ed.ac.uk/maptask/) that captures both speaker-intended and addressee-interpreted landmarks for each reference expression, enabling quantitative study of misunderstanding dynamics in collaborative dialogue.

**Key numbers:** 16 pairs of maps · 128 dialogues · 13,077 annotated reference expressions (REs) · 5 hierarchical attributes and 2 participant interpretation IDs per RE · 3 human-verified dialogues (504 REs)

> **Note on transcript text.** This release ships our **annotations, structure, short RE
> strings, timing/pointer metadata, and model-generated annotation reasons** — it does **not**
> redistribute full HCRC MapTask transcript text, whose license forbids onward distribution.
> The `reason` field may contain short quoted or paraphrased dialogue evidence, but not full
> transcript context. To obtain the RE-marked transcripts, run
> [`scripts/reconstruct_transcripts.py`](#reconstructing-the-marked-transcripts) against your
> own copy of the MapTask corpus.
>
> The GMMT dataset is also available as a loadable Hugging Face dataset (Parquet + Dataset Viewer):
> [`chnln/grounded-misunderstandings-in-maptask`](https://huggingface.co/datasets/chnln/grounded-misunderstandings-in-maptask).

## Repository Structure

```
README.md                            <- this file
LICENSE                              <- CC-BY-4.0
annotation_prompt_template.json      <- prompt template used in the experiment
annotation_output_schema.json        <- JSON Schema for structured output enforcement
annotations/
  dialogues/                         <- 128 per-dialogue annotation JSON files (LLM-annotated)
  human_verified_subset/             <- 3 gold-standard dialogues (q1ec2, q1nc3, q1nc7)
  assets/
    landmarks_configuration.csv      <- landmark metadata (map presence, discrepancy types)
    lexical_variant_landmark_info.json   <- 10 lexical variant pairs
    multiplicity_landmark_position.json  <- 16 multiplicity landmarks with positions
    map_trans_mapping.txt            <- map-to-dialogue mapping (16 maps x 8 dialogues)
    dialogue_map_images.csv          <- dialogue-to-map image filename mapping
reference_expressions/               <- 128 RE-extraction JSONs (timed_unit_ids etc.; used by reconstruction)
scripts/
  reconstruct_transcripts.py         <- rebuilds the RE-marked transcripts from your MapTask copy
```

## MapTask Background

The **HCRC MapTask** is a collaborative dialogue task in which a *route giver* guides a *route follower* through a series of landmarks on a map. Both participants have maps of the same schematic landscape, but their maps are **not identical**:

- Some landmarks appear on **both** maps (identical).
- Some landmarks have **different names** on each map (lexical variants, e.g., "white water" vs. "rapids").
- Some landmarks appear on **only one** map (existence discrepancy).
- Some landmarks appear **twice** on one map but once on the other (multiplicity discrepancy).

Participants cannot see each other's maps. The giver has a route drawn on their map and must describe it; the follower must reproduce it. This asymmetry makes MapTask a natural laboratory for studying how referring expressions are understood — and misunderstood.

The corpus contains **16 pairs of maps** (m0–m15), each used in **8 dialogues** (128 total). Dialogues are identified by codes like `q1ec2`, where `q` indicates the quadrant group and `ec`/`nc` indicates eye-contact condition.

## Annotation Schema

Each reference expression (RE) in the dialogues is annotated with a **5-attribute hierarchy** that models incremental resolution from the perspectives of both speaker and addressee.

### The Cascade

The attributes follow a strict cascade: each is only evaluated when the preceding conditions are met. When the cascade terminates early, downstream attributes are set to `null`.

| Step | Attribute | Perspective | Question | Condition to proceed |
|------|-----------|-------------|----------|---------------------|
| 1 | `is_quantificational` | Speaker | Is the RE an existence check (e.g., "Do you have a parked van?")? | Must be `false` |
| 2 | `is_specified` | Addressee | Does the addressee show any uptake of the RE? | Must be `true` |
| 3 | `is_accommodated` | Addressee | Does the addressee successfully process the RE (no mishearing/confusion)? | Must be `true` |
| 4 | `is_grounded` | Addressee | Does the addressee link the RE to a specific landmark? | Must be `true` |
| 5 | `is_imagined` | Addressee | Is the grounded landmark absent from the addressee's own map? | (terminal) |

When the cascade terminates at step *N*, all attributes from step *N*+1 onward are `null` in the released data. For example, when `is_specified = false`, `is_accommodated`, `is_grounded`, and `is_imagined` are all `null`.

### Landmark Interpretation Fields

Each RE also records landmark IDs for both participants' interpretations:

- **`interpretations.giver`**: the landmark the giver has in mind (on the giver's map)
- **`interpretations.follower`**: the landmark the follower has in mind (on the follower's map)

The speaker's interpretation field (determined by `info.speaker`) is filled in first. The addressee's interpretation field is only populated when `is_grounded = true`. When the cascade terminates early, the addressee's field is set to `""` (empty string).

### Understanding States

Based on the attribute cascade and the interpretation fields, each RE is classified into one of three **understanding states** (stored in `extra.status`):

| State | Definition |
|-------|-----------|
| `aligned` | Both participants ground to the same (or equivalent) landmark |
| `pending` | The RE is not fully grounded (quantificational, unspecified, unaccommodated, or ungrounded) |
| `misunderstood` | Both participants believe they agree, but they ground to different landmarks |

### Landmark ID Format

Landmark IDs follow the pattern: `<map_id>_<concept>#<ordinal>@<side>`

- `<map_id>`: the map identifier (m0–m15)
- `<concept>`: the landmark name from the original dataset (e.g., `diamond_mine`)
- `#<ordinal>`: present only for multiplicity landmarks; `0` = bottom instance, `1` = top instance
- `@<side>`: `g` = giver's map, `f` = follower's map

Examples: `m9_stony_desert@g`, `m9_site_of_plane_crash#0@g`, `m2_stone_creek#1@f`

## Annotation Format

Each dialogue file in `annotations/dialogues/` contains a JSON object:

```json
{
  "dialogue_id": "q1ec2",
  "landmark_reference_expressions": [
    {
      "ref_id_unif": "q1ec2.ref.0",
      "is_quantificational": false,
      "is_specified": false,
      "interpretations": {
        "is_accommodated": null,
        "is_grounded": null,
        "is_imagined": null,
        "giver": "m9_site_of_plane_crash#0@g",
        "follower": ""
      },
      "reason": "Giver directs to the site of a plane crash. Follower gives no uptake...",
      "extra": {
        "status": "pending",
        "subtype": "unspecified"
      },
      "info": {
        "concept_id": "m9_site_of_plane_crash",
        "speaker": "giver",
        "addressee": "follower",
        "expression": "the site of a plane crash",
        "utt_id": 1
      }
    }
  ]
}
```

In this example, `is_specified = false` causes the cascade to terminate — `is_accommodated`, `is_grounded`, and `is_imagined` are `null`. The speaker (giver) intended `m9_site_of_plane_crash#0@g`, but the addressee (follower) did not engage, so `follower` is `""`.

### Key Conventions

- **`interpretations.giver`** and **`interpretations.follower`** refer to map sides (which landmark each participant has in mind), not turn-taking roles.
- **`info.speaker`** and **`info.addressee`** indicate who produced and who received the RE.
- **`extra.status`** is the understanding state: `aligned`, `pending`, or `misunderstood`.
- **`extra.subtype`** provides finer classification (e.g., `quantificational`, `unspecified`, `unaccommodated`, `ungrounded`, `grounded`, `misunderstood`).
- **`reason`** is a concise evidence-based explanation of the annotation decision.

### LLM vs. Human-Verified Annotations

The 3 dialogues `q1ec2`, `q1nc3`, and `q1nc7` appear in **both** `annotations/dialogues/` (LLM-annotated) and `annotations/human_verified_subset/` (human-verified gold standard). The human-verified subset was used to evaluate LLM annotation quality (see the paper's Table 4).

### Post-Processing

The released annotations have been post-processed to enforce the cascade hierarchy: when an early attribute terminates the cascade, all downstream attributes are set to `null` and the addressee's landmark ID is set to `""`. The model (GPT-5) was not instructed to early-stop during annotation, so this cleanup was applied after annotation to ensure data consistency. The `reason` field is preserved as-is from the model output. The `extra.status` and `extra.subtype` fields are **derived** during post-processing: `status` is inferred from the cascade attributes and landmark interpretation matching (aligned/pending/misunderstood), and `subtype` records where the cascade terminates or the specific grounding outcome.

## Dataset Statistics

### Cascade Attribute Distribution

The table below shows the distribution of each cascade attribute across all 13,077 REs. The `null` column reflects cascade propagation: when an attribute terminates the cascade, all downstream attributes become `null`.

| Attribute | True | False | Null |
|-----------|------|-------|------|
| `is_quantificational` | 1,604 (12.3%) | 11,473 (87.7%) | 0 |
| `is_specified` | 11,137 (85.2%) | 336 (2.6%) | 1,604 (12.3%) |
| `is_accommodated` | 10,977 (83.9%) | 160 (1.2%) | 1,940 (14.8%) |
| `is_grounded` | 9,965 (76.2%) | 1,012 (7.7%) | 2,100 (16.1%) |
| `is_imagined` | 632 (4.8%) | 9,333 (71.4%) | 3,112 (23.8%) |

### Understanding States

After lexical variant unification (merging landmark pairs like "white water"/"rapids" that refer to the same location), the understanding states distribute as follows:

| Status | Count | % |
|--------|-------|---|
| `aligned` | 9,435 | 72.1% |
| `pending` | 3,403 | 26.0% |
| `misunderstood` | 239 | 1.8% |

Givers produce 8,679 REs (66.4%) and followers produce 4,398 (33.6%), reflecting the instruction-giving nature of the task.

## Asset Files

The `annotations/assets/` directory contains supporting metadata:

### `landmarks_configuration.csv`

Metadata for all 267 landmarks across 16 maps. Columns:

| Column | Description |
|--------|-------------|
| `id` | Unique landmark identifier (e.g., `m0_camera_shop`) |
| `map` | Map identifier (m0–m15) |
| `name` | Human-readable landmark name |
| `follower_map_appears` | Whether the landmark appears on the follower's map (1=yes, 0=no; 2=appears twice) |
| `giver_map_appears` | Whether the landmark appears on the giver's map (1=yes, 0=no; 2=appears twice) |

This file is extracted from XML files of the original corpus, and the meanings of some columns (e.g., `assim`, `sim`, `contr`, `omo`) are not clearly documented. We keep them in the release for completeness, but they are not used in our annotation scheme or analyses.

### `lexical_variant_landmark_info.json`

Lists the 10 pairs of landmarks that have **different surface names** at the same map location:

| Map | Giver's Label | Follower's Label |
|-----|--------------|-----------------|
| m2 | white_water | rapids |
| m3 | cliffs | sandstone_cliffs |
| m5 | ancient_ruins | ruined_city |
| m6 | fast_flowing_river | fast_running_creek |
| m8 | farmers_gate | broken_gate |
| m8 | dead_tree | dutch_elm |
| m9 | hot_wells | hot_springs |
| m12 | old_mill | mill_wheel |
| m15 | wood | woodland |
| m15 | forked_stream | gurgling_brook |

### `multiplicity_landmark_position.json`

Documents the 16 landmarks that appear **twice on one map** (one per map, m0–m15). For each, records:

- `common`: the position shared with the other map's single instance (with its ordinal)
- `unique`: the position that exists only on one map (with its ordinal)

Ordinals: `0` = lower/bottom instance, `1` = upper/top instance.

### `map_trans_mapping.txt`

Maps the numeric suffix of each **map ID** (`m0`–`m15`) to the **8 dialogue IDs** that use that map. Space-separated, one line per map. For example, line `9 q1ec2 q1nc2 q1ec8 q1nc8 q5ec2 q5nc2 q5ec8 q5nc8` means dialogues q1ec2, q1nc2, etc. all use map `m9`.

### `dialogue_map_images.csv`

Maps each dialogue to the expected original MapTask map image filenames:

| Column | Description |
|--------|-------------|
| `dialogue_id` | Dialogue identifier, e.g. `q1ec2` |
| `map_id` | Map pair identifier, e.g. `m9` |
| `giver_map_filename` | Expected giver map filename, e.g. `map9g.png` |
| `follower_map_filename` | Expected follower map filename, e.g. `map9f.png` |

Map images themselves are **not** redistributed in this release. Obtain them from your own HCRC MapTask corpus copy; for map ID `m9`, the expected giver and follower files are `map9g.png` and `map9f.png`.

> [!NOTE]
> All the above supporting files are processed from the original HCRC MapTask corpus by the paper authors.

## Reconstructing the marked transcripts

The RE-marked transcripts (one per dialogue, with REs delimited inline by `<<...>>` tags) are
**not** shipped here, because they contain HCRC MapTask transcript text, whose license forbids
onward redistribution. Instead, `scripts/reconstruct_transcripts.py` regenerates them **byte-for-byte**
(verified on all 128 dialogues) from your own copy of the corpus:

1. Download the **HCRC Map Task Corpus v2.1** from
   <https://groups.inf.ed.ac.uk/maptask/maptasknxt.html> and unpack it. The per-side timed-units
   live in `maptaskv2-1/Data/timed-units/` (files `<dialogue>.g.timed-units.xml` /
   `<dialogue>.f.timed-units.xml`).

2. Run:

   ```bash
   python scripts/reconstruct_transcripts.py \
     --maptask-tu-dir /path/to/maptaskv2-1/Data/timed-units \
     --out-dir ./transcripts_with_refs
   ```

   (`--re-dir` defaults to `reference_expressions/` and `--map-mapping` to
   `annotations/assets/map_trans_mapping.txt`, both shipped here.)

Each output line shows the speaker role, line number, and utterance ID, with REs marked inline:

```
[giver    ln:19  utt:13 ] pass well it's <<stony desert id:q1ec2.ref.4 lm:m9_stony_desert>> that's what i've got here
```

## Annotation Experiment

### Preprocessing the Original Corpus

The original HCRC MapTask corpus required several preprocessing steps before annotation:

1. **Missing utterance IDs**: The timed-units XML files contained 7,222 elements with missing `utt` attributes (out of 152,705 total). These were filled by analyzing neighboring utterances and selecting the most temporally proximate label.

2. **Incomplete RE metadata**: Two reference expressions in dialogue `q4ec4` had corrupted surface forms and incorrect utterance IDs, which were manually corrected.

3. **Landmark ID reassignment**: The original corpus assigned the same landmark ID to all instances of same-named landmarks. We created unique IDs using the format `<map_id>_<concept>#<ordinal>@<side>` to disambiguate multiplicity landmarks and track map-side provenance.

4. **Dialogue structure extraction**: Moves, games, and transactions were extracted from the corpus XML and cross-referenced with timed units to establish transaction boundaries. A **transaction** is a sequence of moves corresponding to navigating from one point to another on the maps, and they are annotated by humans in the original corpus.

### Prompt Design

The annotation prompt (see `annotation_prompt_template.json`) provides the model with:

- **Background**: MapTask task description and annotation role
- **Task description**: Instructions for the 5-attribute annotation cascade
- **Landmark ID explanation**: The unified ID format and what information is provided
- **Annotation rule**: Step-by-step workflow for each RE
- **Output format**: Expected JSON structure

At runtime, four placeholders (`${target_ref_ids}`, `${context_dialogue}`, `${context_dialogue_acts}`, `${landmark_candidates}`) are filled with dialogue-specific content. Each prompt also includes both maps as images.

### Batch API Workflow

We annotated all 128 dialogues using the **OpenAI Batch API** with GPT-5 and structured output enforcement. Each transaction within a dialogue produced one API request. Below is the request format:

```json
{
  "custom_id": "<dialogue_id>.<transaction_id>",
  "method": "POST",
  "url": "/v1/responses",
  "body": {
    "model": "gpt-5",
    "input": [
      {
        "role": "user",
        "content": [
          { "type": "input_text", "text": "<rendered_prompt>" },
          { "type": "input_image", "image_url": "data:image/png;base64,<giver_map>" },
          { "type": "input_image", "image_url": "data:image/png;base64,<follower_map>" }
        ]
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "annotation_output_format",
        "schema": "<see annotation_output_schema.json>",
        "strict": true
      }
    }
  }
}
```

The structured output schema (`annotation_output_schema.json`) enforces the annotation format at the API level, ensuring all required fields are present and correctly typed. Per-transaction outputs were then aggregated into per-dialogue annotation files.

## The Original MapTask Corpus

This release contains **derived annotations only**. The original HCRC MapTask corpus (audio, maps, transcriptions) is publicly available at:

> https://groups.inf.ed.ac.uk/maptask/

To fully reproduce the pipeline from scratch (constructing prompt bundles from raw transcripts), download the timed-unit transcriptions, reference expression annotations, and move/transaction annotations from the link above.

## Citation

If you use the GMMT dataset, please cite:

```bibtex
@inproceedings{li2026grounded,
  title = {Grounded Misunderstandings in Asymmetric Dialogue: A Perspectivist Annotation Scheme for MapTask},
  author = {Li, Nan and Gatt, Albert and Poesio, Massimo},
  booktitle = {Proceedings of the Fifteenth Language Resources and Evaluation Conference (LREC 2026)},
  month = {May},
  year = {2026},
  pages = {4988--5001},
  address = {Palma, Mallorca, Spain},
  publisher = {European Language Resources Association (ELRA)},
  editor = {Piperidis, Stelios and Bel, Núria and van den Heuvel, Henk and Ide, Nancy and Krek, Simon and Toral, Antonio},
  url = {https://lrec.elra.info/lrec2026-main-392},
  doi = {10.63317/59anbt78wyj7}
}
```

Paper: [LREC Proceedings](https://lrec.elra.info/lrec2026-main-392) ·
[arXiv preprint](https://arxiv.org/abs/2511.03718).

## License

Our annotations, prompt templates, schemas, derived metadata, map-image filename correspondence, and reconstruction code are provided under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE) for details. Full HCRC MapTask transcript text and map images are not included and remain under the original corpus's terms.
