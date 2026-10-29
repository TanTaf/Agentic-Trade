# FYDP report (UIU template)

Main file: `fydp.tex` (chapters `1.intro.tex` … `6.conclusion.tex`, references `fydp.bib`).

All result tables in `tables/` are generated from the study artifacts — do not edit them by hand:

```bash
python scripts/export_latex_tables.py            # from the repository root
```

Figures are read from `../figures/` (i.e. `paper/figures`).

## Compile

Locally (TeX Live): `cd paper/uiu_template && latexmk -pdf fydp.tex`

Overleaf: upload the whole `paper/` folder (so that `uiu_template/` and `figures/` sit side by side),
set `uiu_template/fydp.tex` as the main document and compile with pdfLaTeX.
If you upload only this folder, copy `paper/figures/` into it as `figures/` — both locations are on the graphics path.

Red `[...]` markers (`\todo{}`) are places the team still has to fill (project plan dates, task allocation)
or that will be updated when the LLM-backbone experiment finishes.
