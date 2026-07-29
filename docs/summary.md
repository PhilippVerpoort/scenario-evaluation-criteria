This page gives a criterion-by-criterion overview of all evaluation criteria,
combining the scientific justifications with the computed threshold values.

```python exec="true" session="summary" showcode="false"
import re
import pandas as pd
from sigfig import round as sf_round
from itables import to_html_datatable
from scenario_evaluation_criteria import load_criteria
from scenario_evaluation_criteria.preprocessed import (
    load_criteria_combined,
    region_mapping_from_definition,
)
from scenario_evaluation_criteria.formatting import format_sources, insert_citations

# Region groupings for the regionalised tables, sourced from
# IAMConsortium/common-definitions via region-definitions/nomenclature.yaml
# (see that file for why this isn't a regular package dependency).
REGION_DEFINITIONS = "region-definitions/definitions"

# One native-region version per model, chosen explicitly (rather than
# picked automatically) as the latest version of each model, excluding
# regionalised model variants (e.g. GCAM-Europe, GCAM-KSA, WITCH...EU28,
# POLES ENGAGE) and, for combined models, the combination only (e.g.
# REMIND-MAgPIE rather than its standalone REMIND predecessor). Judged
# from the file/hierarchy names under common-definitions'
# definitions/region/native_regions/ at the pinned commit (see
# region-definitions/nomenclature.yaml); bump when that pin is updated.
MODEL_HIERARCHIES = [
    "AIM 3.0",
    "C3IAM 3.0",
    "CGEM-ESM 1.0",
    "COFFEE 1.6",
    "GCAM 9.1",
    "GEM-E3 V2026",
    "IMACLIM 2.0",
    "IMAGE 3.4",
    "MEESA v1.4",
    "MESSAGEix-GLOBIOM 2.1-R12",
    "MINES 0.1.0",
    "OPEN-PROM 2.2",
    "POLES 2.4",
    "PROMETHEUS V1",
    "REMIND-MAgPIE 3.7-4.14",
    "WITCH 6.0",
]

region_mapping_r5 = region_mapping_from_definition(REGION_DEFINITIONS, hierarchy="R5")
region_mapping_r9 = region_mapping_from_definition(REGION_DEFINITIONS, hierarchy="R9")
region_mapping_r10 = region_mapping_from_definition(REGION_DEFINITIONS, hierarchy="R10")
region_mapping_by_model = {
    model: region_mapping_from_definition(REGION_DEFINITIONS, hierarchy=model)
    for model in MODEL_HIERARCHIES
}
region_mapping = {
    **region_mapping_r5,
    **region_mapping_r9,
    **region_mapping_r10,
    **{
        region: codes
        for model_regions in region_mapping_by_model.values()
        for region, codes in model_regions.items()
    },
}
region_tabs = [
    ("R5", set(region_mapping_r5)),
    ("R9", set(region_mapping_r9)),
    ("R10", set(region_mapping_r10)),
    *sorted(
        (model_name, set(model_regions))
        for model_name, model_regions in region_mapping_by_model.items()
    ),
]

criteria_combined_all = load_criteria_combined(region_mapping=region_mapping)
criteria_combined = criteria_combined_all.query("region == 'World'")
criteria_types, criteria_meta, sources = load_criteria(
    ["criteria-types", "criteria-descriptions", "sources"],
).values()
sources_formatted = format_sources(sources)


def _fmt(text):
    return insert_citations(text, sources_formatted, "").replace("\n", "<br>")


def _render_tabbed_set(tabs, group_id):
    """Hand-render a pymdownx.tabbed "alternate style" tab set as raw HTML.

    ``=== "Tab"`` markdown content is reparsed by a nested block-processor
    call that never runs Python-Markdown's raw-HTML preprocessor (that
    preprocessor only ever sees the whole page once, before any nested
    block reparsing) — so multi-line raw HTML placed inside a tab (like an
    itables datatable) gets escaped/wrapped in a stray ``<p>`` instead of
    preserved verbatim. Emitting the same DOM structure directly, as one
    top-level raw HTML block, sidesteps that: top-level raw HTML *is*
    preserved untouched (the same reason the World-only table above renders
    correctly). The structure/classes below match pymdownx.tabbed's
    "alternate_style" output exactly, so mkdocs-material's tab CSS and
    "content.tabs.link" behaviour keep working unchanged.
    """
    inputs = "".join(
        f'<input{" checked=\"checked\"" if i == 0 else ""} '
        f'id="__tabbed_{group_id}_{i + 1}" name="__tabbed_{group_id}" type="radio" />'
        for i in range(len(tabs))
    )
    labels = "".join(
        f'<label for="__tabbed_{group_id}_{i + 1}">{label}</label>'
        for i, (label, _) in enumerate(tabs)
    )
    blocks = "".join(
        f'<div class="tabbed-block">\n{content}\n</div>\n' for _, content in tabs
    )
    return (
        f'<div class="tabbed-set tabbed-alternate" data-tabs="{group_id}:{len(tabs)}">'
        f'{inputs}<div class="tabbed-labels">{labels}</div>\n'
        f"<div class=\"tabbed-content\">\n{blocks}</div>\n"
        f"</div>\n"
    )


def _fmt_num(v):
    if pd.isna(v):
        return v
    if v == 0:
        return 0.0
    if abs(v) >= 100:
        return round(v, 0)
    return float(sf_round(str(v), sigfigs=3))


def _pivot(df):
    """Pivot lower/upper bounds into columns, including relative and reference values."""
    d = df.drop(columns="criterion").copy()
    d["year"] = d["year"].astype("object").where(d["year"].notna(), other="")
    has_ref = d["value_rel"].notna().any()

    # Pivot absolute values, relative multipliers, and reference values together.
    pivoted = (
        d.pivot_table(
            index=["variable", "region", "year", "evaluation_outcome", "unit"],
            columns="threshold_type",
            values=["value", "value_rel", "reference_value"],
            aggfunc="first",
        )
    )
    name_map = {
        ("value", "lower"): "Lower (abs)", ("value", "upper"): "Upper (abs)",
        ("value_rel", "lower"): "Lower (rel)", ("value_rel", "upper"): "Upper (rel)",
        ("reference_value", "lower"): "Lower (ref)", ("reference_value", "upper"): "Upper (ref)",
    }
    pivoted.columns = [name_map.get(c, "_".join(c)) for c in pivoted.columns]
    pivoted = pivoted.reset_index()

    if has_ref:
        # reference_data_expr is the same for both threshold types; attach once.
        ref_expr = (
            d.groupby(["variable", "year", "evaluation_outcome"], dropna=False)
            ["reference_data_expr"].first().reset_index()
        )
        pivoted = pivoted.merge(ref_expr, on=["variable", "year", "evaluation_outcome"], how="left")

        for col in ["Lower (rel)", "Upper (rel)"]:
            pivoted[col] = pivoted[col].apply(
                lambda v: f"{(v - 1) * 100:+.0f}%" if pd.notna(v) else v
            )
        for col in ["Lower (ref)", "Upper (ref)"]:
            pivoted[col] = pivoted[col].apply(
                _fmt_num
            )
        pivoted["reference_data_expr"] = pivoted["reference_data_expr"].apply(
            lambda v: re.sub(r"^range\((.+)\)$", r"Most permissive value of: \1", v)
            if pd.notna(v) else v
        )

    for col in ["Lower (abs)", "Upper (abs)"]:
        if col in pivoted.columns:
            pivoted[col] = pivoted[col].apply(
                _fmt_num
            )

    col_order = [
        "variable", "region", "year", "unit", "evaluation_outcome",
        "Lower (abs)",
        *(["Lower (rel)", "Lower (ref)"] if has_ref else []),
        "Upper (abs)",
        *(["Upper (rel)", "Upper (ref)"] if has_ref else []),
        *(["reference_data_expr"] if has_ref else []),
    ]
    result = pivoted[[c for c in col_order if c in pivoted.columns]]
    result = result.rename(columns={"reference_data_expr": "reference_data"})
    return result.rename(columns=lambda c: c.replace("_", " ").capitalize())


# Ordered list of criteria within the combined df (preserves CSV row order)
ordered_criteria = list(dict.fromkeys(criteria_combined["criterion"]))
regional_tab_group_counter = 0

for crit_type in criteria_types:
    type_criteria = [c for c in ordered_criteria if c.startswith(f"{crit_type}|")]
    if not type_criteria:
        continue

    print(f"## {crit_type}\n")

    type_spec = criteria_types[crit_type]
    print(f"{_fmt(type_spec['description'])}\n")
    print("**Possible evaluation outcomes:**\n")
    for outcome, outcome_desc in type_spec["evaluation_outcomes"].items():
        print(f"* `{outcome}` — {outcome_desc}")
    print("")

    for criterion in type_criteria:
        criterion_name = criterion.split("|", 1)[1]
        print(f"### {criterion_name}\n")

        meta = criteria_meta.get(criterion, {})
        if jc := meta.get("justification_criterion"):
            print(f"**Why this criterion?** {_fmt(jc)}\n")
        if jt := meta.get("justification_threshold"):
            print(f"**Why this threshold?** {_fmt(jt)}\n")

        world_df = _pivot(
            criteria_combined[criteria_combined["criterion"] == criterion].copy()
        ).drop(columns="Region")
        world_html = to_html_datatable(world_df, connected=True, style="width:100%")
        world_body = f"<div>\n{world_html}\n</div>"

        regional_rows = criteria_combined_all[
            (criteria_combined_all["criterion"] == criterion)
            & (criteria_combined_all["region"] != "World")
        ]
        if regional_rows.empty:
            print(f"{world_body}\n")
        else:
            tabs = [("World", world_body)]
            for tab_label, tab_regions in region_tabs:
                tab_df = regional_rows[regional_rows["region"].isin(tab_regions)].copy()
                if tab_df.empty:
                    continue
                # Native model regions are named "<model>|<region>"; the
                # model prefix is redundant once the region is shown in
                # that model's own tab.
                tab_df["region"] = tab_df["region"].str.rsplit("|", n=1).str[-1]
                tab_pivoted = _pivot(tab_df)
                tab_html = to_html_datatable(
                    tab_pivoted, connected=True, style="width:100%"
                )
                tabs.append((tab_label, f"<div>\n{tab_html}\n</div>"))
            regional_tab_group_counter += 1
            print(_render_tabbed_set(tabs, regional_tab_group_counter))

        if note := meta.get("note"):
            print(f"!!! note\n\n    {_fmt(note).replace("\n", "\n    ")}\n")

print("---\n\n## Sources\n")

df_srcs = pd.DataFrame.from_dict(sources_formatted, orient="index").reset_index()

{% raw %}

def _src_links(row):
    ret = []
    if row["url_doi"]:
        ret.append(
            "[ :simple-doi: DOI ]"
            f"({row['url_doi']})"
            "{ .sm-button } "
        )
    if row["url"] and (not row["doi"] or (row["doi"] == row["url_doi"])):
        ret.append(
            "[ :material-link-box: URL ]"
            f"({row['url']})"
            "{ .sm-button } "
        )
    if row["pdf"]:
        ret.append(
            "[ :fontawesome-solid-file-pdf: PDF ]"
            f"({row['pdf']})"
            "{ .sm-button } "
        )
    return "<br>".join(ret)

{% endraw %}

col1 = df_srcs["index"].apply(lambda r: f'<p id="{r}">`{r}`</p>').rename("Identifier")
col2 = df_srcs["bib"].rename("Bibliographic information")
col3 = df_srcs.apply(_src_links, axis=1).rename("Links")

print(
    pd.concat([col1, col2, col3], axis=1)
    .apply(lambda col: col.str.replace("|", "\\|").str.replace("\n", " "))
    .fillna("")
    .to_markdown(index=False)
)
```
