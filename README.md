# Automatic Interlinear Glossing for ELAN Corpora

Rule-based pipeline that pre-annotates ELAN (`.eaf`) transcription files with
morpheme segmentation, interlinear glosses and language tags, so that human
annotators start from a partially filled file instead of a blank one.

Built during an internship on the [PILAR](https://www.pilar.unito.it/)
(Piedmontese Language in Argentina) project at the University of Turin (*June – November 2025*), working on a corpus of
Piedmontese as spoken by descendants of Italian immigrants in Argentina. The
recordings are bilingual: speakers code-switch between Piedmontese and
Spanish, and the pipeline tags each morph for language.

## What it does

ELAN stores annotations as a tree of tiers. In this corpus each speaker has
four tiers, and only the token tier is filled in by the transcriber:

```
Speaker1-words      →  a=      travajava        ëd     neuit
Speaker1-morph      →  a=      travaj -av -a    ëd     neuit
Speaker1-gloss      →  3SG     work  IMPF 3SG   of     night
Speaker1-language   →  piem    piem  piem piem  piem   piem
```

The pipeline fills the bottom three tiers automatically, leaving anything an
annotator has already glossed untouched.

## Pipeline

The dictionary is built once from files that have already been glossed by
hand, reviewed, and then reused to annotate new files.

### `01_build_dictionary.ipynb` — build and correct the dictionary

1. **Harvest.** Walk the tier tree of each hand-annotated `.eaf` and collect
   every `token → [[morph, gloss], ...]` mapping into a JSON dictionary.
2. **Report conflicts.** When the same token appears with two different
   analyses, the first one seen wins and every conflict is logged with an
   occurrence count. Ambiguities are surfaced for a linguist to adjudicate
   rather than silently resolved.
3. **Apply reviewed corrections.** The adjudicated decisions are written back
   in two directions — into the source `.eaf` files and into the dictionary —
   so the corpus and the derived resource stay consistent with each other.

### `02_automatic_glossing.ipynb` — annotate new files

1. **Split clitics.** Tokens written with `=` (`a=s=parlava`, `cante=la`) are
   split into separate token annotations.
2. **Gloss clitics** from a hand-built clitic table, then strip the `=`.
3. **Populate morph + gloss** from the dictionary. Tokens that already carry a
   non-empty gloss are skipped entirely; unknown tokens get a single morph and
   an empty gloss for the annotator to fill.
4. **Resolve underspecified person/number.** Some clitics are ambiguous for
   number (`a=` → `3?`). The gloss is resolved by looking ahead at the next
   two tokens for a verb form carrying `SG` or `PL`.
5. **Backfill from the lexicon.** Remaining empty glosses are filled from the
   project's ELAN lexicon, but only where a form has exactly one unambiguous
   sense.
6. **Tag language.** Morphs are classified `piem` / `spa` / `piem-spa` from
   orthographic conventions used in the corpus (Spanish is transcribed in
   caps; capitalised proper nouns count as shared).

Intermediate files are written to `/tmp`; only `<name>_automatic.eaf` lands in
the working directory, ready to open in ELAN.

## The correction tables

Two CSVs encode decisions made by a human reviewer about recurring annotator
discrepancies — the same token segmented or glossed inconsistently across
files, or across annotators. They are applied to different targets and work as
a pair:

- **`correct_in_files.csv`** is applied to the source `.eaf` files. Each row
  gives a token, the analysis to look for, the analysis to replace it with,
  and a language tag. Corrections may keep the number of morphs the same,
  collapse several into one, or expand one into several — the last two cases
  require allocating or deleting annotation IDs and repointing the gloss and
  language tiers that depend on them.
- **`correct_in_dict.csv`** is applied to the harvested dictionary. Rows
  marked `REMOVE` in the `action` column blank the gloss for high-frequency
  ambiguous tokens (`a`, `che`, `la`, `QUE`, `se`, `mi` and others that
  dominated the conflict report). A blank gloss makes the pipeline leave the
  token for a human instead of guessing — a deliberate choice of precision
  over coverage on exactly the tokens where an automatic guess is least
  trustworthy.

## Repository contents

All files live in a single directory; the notebooks expect their inputs in the
working directory.

| File | |
| --- | --- |
| `01_build_dictionary.ipynb` | build, review and correct the token→gloss dictionary |
| `02_automatic_glossing.ipynb` | the six-step annotation pipeline |
| `word_morph_gloss_dict.json` | `token → [[morph, gloss], ...]`, 770 entries |
| `clitics.csv` | hand-built clitic → gloss table |
| `correct_in_files.csv` | reviewed corrections applied to `.eaf` files |
| `correct_in_dict.csv` | reviewed corrections applied to the dictionary |

## Privacy and data availability

**All personal names have been removed.** The corpus opens with each speaker
stating their name for the recorded consent declaration, so given names and
surnames of the speakers — and of the researcher named in that declaration —
appeared as dictionary entries. Those entries were deleted before publication.
Speaker names used as ELAN tier IDs have been replaced throughout the
notebooks with `Speaker1`, `Speaker2` and so on, and all notebook cell outputs
have been cleared, since they reproduced transcribed speech.

The pipeline handles the missing entries gracefully: a token absent from the
dictionary is given a single morph and an empty gloss for a human annotator to
fill in, which is the appropriate treatment for proper nouns in any case.

Place names — towns in Córdoba and Santa Fe, and towns of origin in Piedmont —
were kept. They carry linguistic and historical weight in a corpus about a
migration community, and identify no individual.

The recordings, the transcribed `.eaf` files and the project lexicon are **not
included**. They are fieldwork data collected under participant consent
agreements and are not mine to redistribute. This repository therefore holds
the code and the derived lexical resources only, and cannot be run end to end
without corpus files of your own.

## Running it

```
pip install -r requirements.txt
```

Everything else is Python standard library (`xml.etree.ElementTree`, `json`,
`csv`, `ast`). The notebooks were written in Google Colab.

To run the pipeline on your own files, list the speakers per recording:

```python
pilar_files = {
    "my-recording": ["Speaker1", "Speaker2"],
}
```

Tier names are assumed to follow `<speaker>-words`, `<speaker>-morph`,
`<speaker>-gloss`, `<speaker>-language`. Irregular tier names can be passed
explicitly as a `config` dict.

## Limitations

- Rule-based and corpus-specific: the language tagger encodes this corpus's
  orthographic conventions and will not transfer to a corpus that transcribes
  differently.
- The dictionary maps whole tokens, so it does not generalise to unseen
  inflected forms of a known stem.
- Person/number resolution uses a two-token lookahead heuristic; it produces a
  best guess for an annotator to confirm, not a parse.
- The published dictionary is the pre-correction version, so that the
  correction step in notebook 1 is reproducible. Notebook 1 writes its
  corrected output to `word_morph_gloss_dict_upd.json`, but notebook 2 loads
  `word_morph_gloss_dict.json`; point it at the corrected file to gloss with
  the reviewed dictionary.

## License

Code is released under the MIT License (see `LICENSE`). The lexical resources
are derived from the PILAR corpus; please credit the project if you reuse
them.
