# BrewJSON

**What actually happened on brew day, as JSON.**

[BeerJSON](https://beerjson.github.io/beerjson/) describes recipes: what you plan to brew. It has nothing for what happened when you brewed it: the pre-boil gravity you measured, the mash out you skipped, the finings you threw in that weren't in the recipe, or the refractometer reading you don't trust. BrewJSON adds that, without changing BeerJSON. (The examples below use a made-up recipe.)

> **Status: 0.2, pre-release.** Expect breaking changes before 1.0; files are migrated when they happen.

## How it relates to BeerJSON

BrewJSON extends BeerJSON by **composition, not modification**. BeerJSON's root, `beerjson` and recipe objects are closed (`additionalProperties: false`), so there's nowhere to add fields. Instead, a BrewJSON document has two sections side by side:

- `beerjson`: a plain, valid BeerJSON 1.0 section. In a batch, it holds a frozen snapshot of the plan: the recipe and, optionally, the equipment used.
- `brewjson`: the new layer.

Recipes stay pure BeerJSON (`*.beer.json`), so any BeerJSON tool can read them. BrewJSON's schemas `$ref` BeerJSON's types, so a planned value and a measured value have the same shape: `{"unit": "l", "value": 20}`.

## Documents

| File | Schema | What it is |
| --- | --- | --- |
| `*.beer.json` | BeerJSON 1.0 | A recipe. One recipe per file. |
| `*.brew.json` | [`schemas/0.2/batch.json`](schemas/0.2/batch.json) | One brewed batch: the frozen plan plus everything that happened. |
| `house.brew.json` | [`schemas/0.2/house.json`](schemas/0.2/house.json) | A brewhouse: its equipment sets (BeerJSON `EquipmentType`, each with an id) and the default one. |

## A batch at a glance

```jsonc
{
  "beerjson": {
    "version": 1,
    "recipes": [ /* the recipe as it was when the batch was set up */ ],
    "equipments": [ /* the equipment set used */ ]
  },
  "brewjson": {
    "version": "0.2.0",
    "batch": {
      "id": "2026-03-14.backyard-smash.1",
      "name": "Backyard SMaSH",
      "number": 1,
      "brew_date": "2026-03-14",
      "status": "fermenting",
      "recipe": { "id": "backyard-smash", "commit": "4f1c2ab", "sha256": "3b8f0c…" },
      "equipment": "kettle-fv",
      "readings": [
        {
          "id": "pb-grav", "at": "2026-03-14T11:20+11:00", "stage": "pre_boil", "kind": "gravity",
          "value": { "unit": "sg", "value": 1.040 }, "instrument": "refractometer"
        },
        {
          "id": "og", "at": "2026-03-14T13:05+11:00", "stage": "end_of_boil", "kind": "gravity",
          "value": { "unit": "sg", "value": 1.044 }, "instrument": "refractometer",
          "suspect": true, "note": "Mass balance from pre-boil says ~1.047."
        }
      ],
      "actuals": [
        { "ref": "/recipes/0/mash/mash_steps/1", "status": "skipped", "note": "No mash out; sparged hot instead." },
        { "ref": "/recipes/0/ingredients/hop_additions/0", "status": "changed",
          "actual": { "alpha_acid": { "unit": "%", "value": 7.2 } }, "note": "This lot's packet said 7.2% AA." }
      ],
      "unplanned_additions": [
        { "id": "irish-moss", "at": "2026-03-14T12:50+11:00", "stage": "boil", "category": "miscellaneous",
          "name": "Irish moss", "type": "fining", "note": "Amount not recorded." }
      ],
      "tasting": [],
      "notes": ""
    }
  }
}
```

The complete, valid file is [`examples/2026-03-14.backyard-smash.1.brew.json`](examples/2026-03-14.backyard-smash.1.brew.json).

## Principles

1. **A batch freezes its plan.** Creating a batch copies the recipe and the equipment set into it. Editing the recipe later never changes a past batch. The recipe link (`id`, git `commit`, `sha256`) records where the plan came from; the snapshot holds the plan itself.
2. **Record what was measured; derive the rest.** A reading stores what the instrument showed, with the instrument and sample temperature. Corrections, efficiencies, ABV and deviations are calculated by tools, never stored.
3. **Measurements are append-only.** A re-measurement is a new reading. A doubtful reading is kept and flagged `suspect`, so tools can prefer a trusted one or an alternative such as mass balance.
4. **Actuals point at the plan.** Each actual is a [JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901) into the snapshot (`/recipes/0/...` or `/equipments/0/...`) with a status: `done`, `changed` (with the actual values) or `skipped`. A plan item with no actual is *unrecorded*, not assumed done.
5. **There is no deviations list.** Deviations are derived from the skipped and changed actuals, the unplanned additions, and readings outside tolerance.
6. **Units are always explicit.** Every quantity uses a BeerJSON `{unit, value}` type.

## Ids and file names

Ids are file names (BeerJSON has no id fields), and documents refer to each other by id, never by path. Ids are lowercase kebab-case.

| Record | File | Id |
| --- | --- | --- |
| Recipe | `recipes/backyard-smash.beer.json` | `backyard-smash` |
| Batch | `batches/2026-03-14.backyard-smash.1.brew.json` | `2026-03-14.backyard-smash.1` (brew date, recipe, batch number) |
| Equipment set | an entry in `house.brew.json` | its `id` field, e.g. `kettle-fv` |

## Validation

The schemas are JSON Schema **draft-07**, the same draft as BeerJSON 1.0, so one validator resolves both. Any draft-07 validator works. With [Ajv](https://ajv.js.org/) and the BeerJSON schemas from [`@beerjson/beerjson`](https://www.npmjs.com/package/@beerjson/beerjson):

```js
import Ajv from 'ajv';
import { readFileSync, readdirSync } from 'node:fs';

const read = f => JSON.parse(readFileSync(f, 'utf8'));
const ajv = new Ajv({ allErrors: true });

// BrewJSON schemas $ref BeerJSON's, so load both.
const beerjson = 'node_modules/@beerjson/beerjson/json';
for (const f of readdirSync(beerjson).filter(f => f.endsWith('.json'))) ajv.addSchema(read(`${beerjson}/${f}`));
const batch = ajv.compile(read('schemas/0.2/batch.json'));

const doc = read(process.argv[2]);
console.log(batch(doc) ? 'valid' : batch.errors);
```

Some rules can't be expressed in a schema; a conforming validator also checks them:

- A batch snapshot holds exactly one recipe and at most one equipment set.
- Every `actuals[].ref` resolves in the snapshot.
- `batch.id` equals the file name without `.brew.json`, and ends in `.<batch.number>`. No two batches of a recipe share a number.
- Reading ids and unplanned-addition ids are unique within a batch. Equipment item names are unique within a set.
- A recipe file holds exactly one recipe. A house profile's `default_equipment` names one of its sets, and set ids are unique.

These are reported as notes, not errors: the recipe file has changed since the snapshot (its SHA-256 differs), and a recipe or equipment set the batch links to no longer exists. The snapshot still holds the plan, so neither breaks the batch.

## Versioning

- BrewJSON uses semver. Every document states its version in `brewjson.version`.
- Before 1.0, tools read only the current version, and older files are migrated.
- BeerJSON is pinned at 1.0.2 (`@beerjson/beerjson`), and the pin is only bumped deliberately.

## Licence

[MIT](LICENSE), the same as BeerJSON.
