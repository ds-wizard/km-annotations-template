# KM Annotations

Helps you give the questions of a knowledge model **readable names**, check them, and use them in document templates.

A document template normally reads a reply by the UUIDs of the chapter and question (`1e85da40-….73d686bd-…`). With the `json.key` and `json.value` annotations in the knowledge model, the `extract_replies` filter turns the replies of a project into nested data with readable names instead:

```jinja
{%- set data = ctx|to_context_obj|extract_replies -%}
{{ data.administrativeInformation.contributors[0].name._value }}
```

The template code is then easier to read and write, and it can be reused for other knowledge models with the same annotations. This document template shows what the annotations of a knowledge model produce and finds the mistakes in them. It works with any knowledge model.

## Formats

| Format | Use it to |
|---|---|
| **KM Check Table** (HTML, CSV, Excel) | **Check the annotations** (for knowledge model authors). One row per chapter, question, answer and choice with its annotations, where it ends up in the extracted data, and the problems found. |
| **JSON Schema** (HTML, JSON) | **See the structure of the extracted data** (for template authors, or for validating data exported to other systems): every key, its type, the possible answers with their values, and the question titles as descriptions. |
| **Extraction** (HTML, JSON) | **See the extracted data of this project**, to check the replies a template will get. |

The check table and the schema depend only on the knowledge model, so any project of it will do, even an empty one. The HTML formats with a tree view need an internet connection: they load [JsonTree.js](https://github.com/williamtroup/JsonTree.js) from a CDN.

## How to use it

1. **Annotate the knowledge model** in the knowledge model editor: in the Annotations of an entity, add
   - `json.key` to chapters and questions: the name of the chapter or question in the data, e.g. `contributors` (letters and digits, starting with a letter, are easiest to use in Jinja);
   - `json.value` to answers and choices: the value of the answer, e.g. `true`, `false`, `dataSteward`.
2. **Publish the knowledge model** and create a project with it.
3. **Create a document with the *KM Check Table*** format and go through the findings (the HTML version can show only the rows with findings). Fix them in the editor and repeat from step 2.
4. **Write the template:** the *Extraction path* column of the check table, the *JSON Schema* and the *Extraction* of a filled-in project show how to reach each reply (see the example below).

## Example

Annotations in the knowledge model ([Common DSW Knowledge Model](https://registry.ds-wizard.org/knowledge-models/dsw:root:latest) 2.8):

| Entity | Annotation |
|---|---|
| Chapter *Administrative information* | `json.key` = `administrativeInformation` |
| List question *Contributors* | `json.key` = `contributors` |
| Its item question *Name* | `json.key` = `name` |
| Its multi-choice question *Role*, with choices *Data Steward*, *Project Leader*, … | `json.key` = `roles`, `json.value` = `dataSteward`, `projectLeader`, … |
| Chapter *Creating and collecting data* | `json.key` = `creatingData` |
| Options question *Will you collect any data connected to a person, "personal data"?*, with answers *No*, *Yes* | `json.key` = `collectsPersonalData`, `json.value` = `false`, `true` |

The check table shows their extraction paths, e.g. `administrativeInformation.contributors[].roles[]` (`[]` marks a list). The extracted data of a project (`_path` left out):

```json
{
  "administrativeInformation": {
    "contributors": [
      {
        "_uuid": "2ac45ac8-cd8e-409d-b452-1b4aeb005b12",
        "name": { "_value": "Jane Doe" },
        "roles": [
          { "_uuid": "3022098b-…", "_label": "Data Steward", "_value": "dataSteward" },
          { "_uuid": "2d433965-…", "_label": "Project Leader", "_value": "projectLeader" }
        ]
      }
    ]
  },
  "creatingData": {
    "collectsPersonalData": { "_uuid": "4bdc319d-…", "_label": "No", "_value": false }
  }
}
```

And in a template:

```jinja
{%- set data = ctx|to_context_obj|extract_replies -%}
{%- for contributor in data.administrativeInformation.contributors %}
- {{ contributor.name._value if contributor.name else 'Unnamed' }}
  {%- if contributor.roles %} ({{ contributor.roles|map(attribute='_label')|join(', ') }}){% endif %}
{%- endfor %}

{% set personal = data.creatingData.collectsPersonalData -%}
{%- if personal is none -%}
  Not answered yet.
{%- elif personal._value is sameas true -%}
  The project collects personal data.
{%- else -%}
  The project does not collect personal data.
{%- endif %}
```

Note the checks: an unanswered question is `null`, so test it before reading `._value`; and `_value` may be `false`, so compare it rather than only testing it.

## What the extracted data looks like

This template is for **Wizard 4.x** (metamodel 18). There, `extract_replies` works as follows (variant `simple`, version `1`):

- A chapter with `json.key` is an object under that key. The questions of a chapter without a key go directly into the top-level object.
- A question with `json.key` appears under that key; a question without a key is left out, with everything under it, except an options question: the follow-ups of its selected answer go into the enclosing object.
- What a question becomes:

  | Question | In the data |
  |---|---|
  | value | `_value` (the reply as text, also for numbers and dates) |
  | options | `_uuid`, `_label` and `_value` (its `json.value`) of the selected answer, plus the follow-up questions of that answer |
  | multi-choice | a list of objects with `_uuid`, `_label`, `_value` of the selected choices |
  | list | a list of objects with `_uuid` of the item and its item questions |
  | item select | `_uuid` and `_label` of the selected item |
  | file | `_uuid`, `_name`, `_size`, `_content_type`, `_url` |
  | integration | `_value`, and `_raw` (the data of the selected item) unless the reply is plain text |

  Each object also has `_path`, the path of the reply in the project.
- An unanswered question is `null` (a list or multi-choice question `[]`); the follow-ups of an answer that is not selected are missing.
- `json.value` `true` / `false` (in any case) become booleans, and anything Python reads as a number becomes a number; other values stay text. Without `json.value`, `_value` is `null`.
- In Wizard 4.x, a configuration passed to `extract_replies` is ignored (the defaults always apply).

## Findings of the check

**Errors** – data is lost or overwritten:

- two chapters or questions with the same `json.key` in one object (the later one wins); follow-ups of different answers of one options question may share a key, as only one answer can be selected;
- a `json.key` under a list question without a key (the whole list is left out);
- a `json.key` equal to a field the extraction adds, e.g. a follow-up with the key `_value`;
- an empty `json.key`, or more than one `json.key` or `json.value` on one entity.

**Warnings** – most likely a mistake:

- a question without `json.key` (only when the knowledge model uses keys at all), an answer or a choice of an annotated question without `json.value`;
- a `json.value` converted in a surprising way (`007` becomes `7`, `1e3` becomes `1000.0`), or the same for two answers of one question;
- the answers *Yes* and *No* whose values are not `true` and `false`;
- `json.key` on an answer or a choice, `json.value` on a chapter or a question (they have no effect);
- a key that is a method of a dict in Jinja (`items`, `keys`, `values`, …): `x.items` is the method, so the value has to be read as `x['items']`.

**Notes** – worth knowing: chapters and options questions without a key whose content goes into the enclosing object, keys that are not identifiers (read them as `x['my-key']`), and `json.value` of answers that are not extracted.
