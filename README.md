# KM Annotations

Helps you give the questions of a knowledge model **readable names**, check them, and use them in document templates.

A document template normally reads a reply by the UUIDs of the chapter and question (`1e85da40-….73d686bd-…`). With the `json.key` and `json.value` annotations in the knowledge model, the `extract_replies` filter turns the replies of a project into nested data with readable names instead:

```jinja
{%- set data = ctx|to_context_obj|extract_replies({'version': '2'}) -%}
{{ data.administrativeInformation.contributors[0].name._value }}
```

The template code is then easier to read and write, and it can be reused for other knowledge models with the same annotations. This document template shows what the annotations of a knowledge model produce and finds the mistakes in them. It works with any knowledge model.

This version is for **Wizard 5.0** (metamodel 19) and describes `extract_replies` with `{'version': '2'}`. For Wizard 4.x, use version 1.x of this template.

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
2. **Create a document with the *KM Check Table*** format: from the preview of the knowledge model editor, or from a project with the published knowledge model. Go through the findings (the HTML version can show only the rows with findings), fix them in the editor and repeat.
3. **Write the template:** the *Extraction path* column of the check table, the *JSON Schema* and the *Extraction* of a filled-in project show how to reach each reply (see the example below).

## Example

Annotations in the knowledge model ([Common DSW Knowledge Model](https://registry.ds-wizard.org/knowledge-models/dsw:root:latest) 2.8.2):

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
        "_uuid": "df9ae73d-…",
        "_label": "Jane Doe",
        "_value": "df9ae73d-…",
        "name": { "_value": "Jane Doe" },
        "roles": [
          { "_uuid": "3022098b-…", "_label": "Data Steward", "_value": "dataSteward" },
          { "_uuid": "2d433965-…", "_label": "Project Leader", "_value": "projectLeader" }
        ]
      }
    ]
  },
  "creatingData": {
    "collectsPersonalData": {
      "_uuid": "4bdc319d-…",
      "_label": "No",
      "_value": false,
      "gdpr": { "_uuid": null, "_label": null, "_value": null, … }
    }
  }
}
```

`gdpr` is a follow-up question of the answer *Yes*: it is there, empty, although *No* is selected.

And in a template:

```jinja
{%- set data = ctx|to_context_obj|extract_replies({'version': '2'}) -%}
{%- for contributor in data.administrativeInformation.contributors %}
- {{ contributor.name._value or 'Unnamed' }}
  {%- if contributor.roles %} ({{ contributor.roles|map(attribute='_label')|join(', ') }}){% endif %}
{%- endfor %}

{% set personal = data.creatingData.collectsPersonalData -%}
{%- if personal._value is none -%}
  Not answered yet.
{%- elif personal._value -%}
  The project collects personal data.
{%- else -%}
  The project does not collect personal data.
{%- endif %}
```

Every path along the annotations exists, so there is no need to test the objects themselves; an unanswered question has `_value` `null`, which differs from `false`.

## What the extracted data looks like

`extract_replies` with `{'version': '2'}` (variant `simple`) works as follows:

- A chapter with `json.key` is an object under that key. The questions of a chapter without a key go directly into the top-level object.
- A question with `json.key` appears under that key; a question without a key is left out, with everything under it, except an options question: the follow-ups of its answers go into the enclosing object.
- What a question becomes:

  | Question | In the data |
  |---|---|
  | value | `_value`: the reply as text; a number for number questions |
  | options | `_uuid`, `_label` and `_value` (its `json.value`) of the selected answer, and the follow-up questions of all answers (those of the other answers empty) |
  | multi-choice | a list of objects with `_uuid`, `_label`, `_value` of the selected choices |
  | list | a list of objects with `_uuid`, `_label` (the item title), `_value` (the item UUID) and the item questions |
  | item select | `_uuid`, `_label` (the item title) and `_value` (the item UUID) of the selected item |
  | file | `_uuid`, `_name`, `_size`, `_content_type`, `_url`, `_value` (the file UUID) |
  | integration | `_value`, and `_raw` (the data of the selected item) unless the reply is plain text |

  Each object also has `_path`, the path of the reply in the project.
- Every annotated question is present: when unanswered, it is `{"_value": null}` (an options question also with `_uuid` and `_label` `null`), and a list or multi-choice question is `[]`.
- `json.value` `true` and `false` become booleans and plain integers (`12`, `-3`) numbers; anything else stays text (`True`, `007`, `1.5`). Without `json.value`, `_value` is `null`.

Without `{'version': '2'}`, the filter works as in Wizard 4.x (version 1): unanswered questions are `null`, follow-ups of answers not selected are missing, and `json.value` is converted loosely (`007` becomes `7`). This template does not describe that.

## Findings of the check

**Errors** – data is lost:

- two chapters or questions with the same `json.key` in one object; follow-ups of different answers of one options question may share a key (the follow-up of the selected answer is used);
- a `json.key` under a list question without a key (the whole list is left out);
- a `json.key` equal to a field the extraction adds, e.g. a follow-up with the key `_value`;
- an empty `json.key`, or more than one `json.key` or `json.value` on one entity.

**Warnings** – most likely a mistake:

- a question without `json.key` (only when the knowledge model uses keys at all), an answer or a choice of an annotated question without `json.value`;
- a `json.value` that looks like a boolean or a number but stays text (`True`, `007`, `1.5`), or the same for two answers of one question;
- the answers *Yes* and *No* whose values are not `true` and `false`;
- `json.key` on an answer or a choice, `json.value` on a chapter or a question (they have no effect);
- a key that is a method of a dict in Jinja (`items`, `keys`, `values`, …): `x.items` is the method, so the value has to be read as `x['items']`.

**Notes** – worth knowing: chapters and options questions without a key whose content goes into the enclosing object, keys that are not identifiers (read them as `x['my-key']`), and `json.value` of answers that are not extracted.
