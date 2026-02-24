---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.1
kernelspec:
  name: python3
  display_name: Python 3 (ipykernel)
  language: python
---

# Recipes for working with the schema

The BIDS Schema contains a lot of information, but its organization may not be the most useful for a given task. This page contains small snippets for generating more usable structures.

```{code-cell} ipython3
import bidsschematools as bst
import bidsschematools.schema
schema = bst.schema.load_schema()
```

```{code-cell} ipython3
from pprint import pprint
```

## Entity ordering

Suppose you have a collection of entities and you want to generate a plausible filename. BIDS has a global entity ordering that must be followed. The following ensures entities are sorted, but does not attempt to ensure that the resulting filename is valid.

```{code-cell} ipython3
from collections import defaultdict

entity_indices = {e: i for i, e in enumerate(schema.rules.entities)}

def format_filename(entities, suffix, extension):
    # Default to place unknown entities last, build new each time
    # for determinism
    indices = defaultdict(lambda: 100, entity_indices)
    stem = '_'.join(
        # Look up the short name
        f'{schema.objects.entities[k].name}-{entities[k]}'
        for k in sorted(entities, key=indices.__getitem__)
    )
    return f'{stem}_{suffix}{extension}'
```

```{code-cell} ipython3
format_filename(
    entities={'task': 'nback', 'description': 'denoised', 'subject': '01'},
    suffix='bold',
    extension='.nii.gz'
)
```

```{code-cell} ipython3
entities = {'task': 'nback', 'description': 'denoised', 'subject': '01'}
```

## Requirement levels for TSV columns

The schema is used for both validation and generating specification tables. In some places, the types of values depends on a simple common case and a more complex structure. For example, in tabular file rules, the columns may either be a simple string indicating requirement level, or a structure where the requirement level is accessed as `.level`:

```{code-cell} ipython3
participants_tsv_rule = schema.rules.tabular_data.modality_agnostic.Participants
participants_tsv_rule.columns.to_dict()
```

For cases where the rendering details are not needed, it might be desirable to simplify to just the requirement levels:

```{code-cell} ipython3
levels = {
    key: value if isinstance(value, str) else value['level']
    for key, value in participants_tsv_rule.columns.items()
}
```

```{code-cell} ipython3
levels
```

## Find location for a piece of metadata

If you are working on a tool to curate data into BIDS, you may have a piece of metadata and want to find where it should be stored programmatically.

The information is spread across `rules.tabular_data`, which contains rules that reference columns by *id*. That *id* is a key in the `objects.columns` table, and the column header is found in `column.name`, where `column` is a value in `objects.columns`.

```{code-cell} ipython3
# Example where `id` matches `name`
pprint(dict(schema.objects.columns.participant_id))
# Example where `id` does not match `name`
pprint(dict(schema.objects.columns.reference__eeg))
```

Let's create a lookup table by finding every column once, and index by `name` (the column header):

```{code-cell} ipython3
table_rules = {
    column.name: [
        f'{tables}.{table_key}'
        for tables in ('rules.tabular_data', 'rules.tabular_data.derivatives')
        if tables in schema  # Consider that `derivatives` might get squashed to match raw
        for table_key, table in schema[tables].items(level=2)  # Rules are nested at two levels
        if 'columns' in table and column_key in table.columns  # Identify rules
    ]
    for column_key, column in schema.objects.columns.items()
}
```

(Note that this isn't as efficient as it could be, but it's fast enough.) Let's say we have a `participant_id` and want to know where it should go:

```{code-cell} ipython3
table_rules['participant_id']
```

```{code-cell} ipython3
schema[table_rules['participant_id'][0]].to_dict()
```

Now, this doesn't help you find the file name that the data should be stored in.

## Finding file rules that accept a set of entities

There are many-to-many mappings between file rules and entities/suffixes/datatypes.

```{code-cell} ipython3
entity_file_map = defaultdict(set)
suffix_file_map = defaultdict(set)

for name, rule in schema.rules.files.items(level=3):
    for entity in rule.get('entities', ()):
        entity_file_map[entity].add(name)
    for suffix in rule.get('suffixes', ()):
        suffix_file_map[suffix].add(name)
```

```{code-cell} ipython3
entities={'task': 'nback', 'description': 'denoised', 'subject': '01'}
suffix='bold'
extension='.nii.gz'
```

```{code-cell} ipython3
from functools import reduce
entity_matches = reduce(set.intersection, (entity_file_map[ent] for ent in entities))
suffix_matches = suffix_file_map[suffix]
best_guess = next(iter(entity_matches & suffix_matches), None)
```

```{code-cell} ipython3
{best_guess: schema.rules.files[best_guess].to_dict()}
```
