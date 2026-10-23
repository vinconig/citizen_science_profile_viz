# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A standardized way to visualize the profile of a citizen science project. The workflow is two-step: the user fills in an external questionnaire (link TBD) to produce a profile, then runs `profile_viz.ipynb` to chart it. The README describes the intended scope: aspects common to all projects (currently Wiggins and Crowston's five project types, with more typologies planned) and an extra section for environmental monitoring campaigns (Pocock's two axes: simple → elaborate approach, systematic monitoring → mass participation). The project is work in progress; the notebook currently only implements the Wiggins and Crowston radar chart.

## Running

There is no build, lint, or test setup. Dependencies live in `pyproject.toml` (`matplotlib`, `numpy`, `jupyterlab`, `ipykernel`, `ipywidgets`) and are managed with [uv](https://docs.astral.sh/uv/):

```
uv sync                  # create .venv and install dependencies
uv run jupyter lab       # open the notebook
MPLBACKEND=Agg uv run jupyter nbconvert --to notebook --execute profile_viz.ipynb --output-dir /tmp   # headless smoke test
```

Without uv: `python -m venv .venv && source .venv/bin/activate && pip install matplotlib numpy jupyterlab ipykernel ipywidgets`.

Run `profile_viz.ipynb` top to bottom from the repo root: the palette is loaded via the relative path `PALETTE_FILE = "colorpalette.json"`, so the notebook's working directory must contain it.

## Architecture

Two files carry the design; they are coupled through the palette's JSON structure.

- **`colorpalette.json`** is the single source of truth for all colors. Top-level keys:
  - `palettes`: data palettes (`Muted categorical`, `Nature-style cool`, `Sequential (single hue)`, `Diverging`), `Color families` (7 hues × 5 shades named `lighter, light, base, dark, darker`, with the shade order stored in `shades`), and `Neutrals` (a grey `ramp` plus semantic `roles` that map to ramp steps, e.g. `gridline`, `not_answered`, `not_applicable`, `axis`, `text`).
  - `theme`: swappable primary/secondary/tertiary colors for titles and layout, independent of the data palettes (not yet read by the notebook).
  - `assignments`: fixed color mappings per profile aspect. Project types have a fixed color each, identical in every chart, with red deliberately unused to avoid red/green pairs. Pocock axes are ordered scales and must use the sequential palette, never the categorical one.
- **`profile_viz.ipynb`** is structured as a theme cell followed by one cell (plus a markdown intro) per chart.
  - Cell 1 sets `COLOR_FAMILY`, loads the palette, and defines the helpers `shade(name)` (a shade of the active family) and `neutral(role)` (a hex via `roles` → `ramp`). Charts should get colors through these helpers rather than hard-coded hex values.
  - Each chart is a `plot_*` function that validates its input, takes optional color overrides that default to theme shades or neutrals, and returns `(fig, ax)`, with an optional `save_to` PNG path. `plot_project_type_radar` takes a dict of `{type: 0-100}` and requires exactly the five types in `PROJECT_TYPES`.

**Always, whenever a new notebook cell is created** (or an existing one edited), check whether its imports introduced or removed a third-party dependency, and update `dependencies` in `pyproject.toml` accordingly (then run `uv sync` to refresh `uv.lock`). Also update the pip fallback command in the Running section if the dependency list changes.

The last cell, `plot_profile_overview`, combines all charts in one paper-ready figure (a: project type, b: Pocock axes, c: participation depth) and saves PNG + PDF to `output/`. For that, each `plot_*` function takes an optional `ax` (draw into it instead of creating a figure; the project type chart needs a polar axes) and a `label_size`. When adding a chart, give it the same `ax`/`label_size` parameters and add it to the overview.

When adding a new chart, follow this pattern: a new markdown + code cell pair, a `plot_*` function with the same override and validation conventions, and any new colors added to `colorpalette.json` instead of the notebook.

`participation_depth_options.ipynb` is a standalone design-exploration notebook for Haklay's four participation levels: its own copy of the theme cell, shared helpers, and three slider designs (`plot_depth_slider_classic`, `_steps`, `_annotated`) for the same 1-10 input (levels at 1, 4, 7, 10). The stepped meter was chosen and now lives in `profile_viz.ipynb` as `plot_participation_depth_chart`, colored from the theme family (`light`, `base`, `dark`, `darker` for the four levels, interpolated in between); the options notebook is kept for now. Only the options notebook uses the `Participation depth (Haklay)` assignment in `colorpalette.json` (an ordered scale on the sequential palette).

The `shade()` and `neutral()` helpers read module-level state (`COLOR_FAMILY`, `palette`) defined in cell 1, so that cell must run first.

`project_domain_tree.ipynb` is a standalone notebook (its own copy of the theme cell and helpers) that redraws the project-domain tree (Projects → Astronomy, Environment, Biology, Medical, Other Projects → seven Biology groups) from the `DOMAINS` dict (`{domain: {short label: description}}`). Only the selected domain is opened into its subcategory boxes (one row up to 7, otherwise two rows in a panel); only the path to the selected subcategory is colored from the theme family (`lighter`/`base`/`dark`/`darker`) and everything else is grey. `plot_domain_tree(domain, subcategory)` follows the usual `plot_*` conventions; `subcategory=None` selects only the domain (it gets the highlighted end-of-path color, nothing expands, no card). The last cell adds two linked `ipywidgets` dropdowns (the subcategory one starts with "(none)"; live kernel only) that redraw it.
