# Recipe Scaler: import format for LLM agents

This document describes the current `v1.5` import format already supported by the app.
If an agent generates a file that follows this spec, the user can import it via `Personal Area -> Data management -> Import recipes`.

## What is supported

- One or more recipes.
- Optional folders (collections) and recipe-to-folder links.
- `JSON` when recipes do not include images.
- `ZIP` when images must be imported.
- HTML recipe steps in `description`.
- Scalable ingredient references inside steps.
- Timers inside steps.
- Optional nutrition data.
- Optional ingredient catalog icons via `illustrationId`.

## Quick rules

- Use `v1.5` only.
- Set `metadata.version` to `"1.5"`.
- Set `metadata.type` to `"recipes-v1.5"`.
- If there are no images, a single `.json` file is enough.
- If there are images, build a `.zip` with `recipes.json` at the archive root.
- Put HTML in `description`, not Markdown.
- Store the number of servings in `servings` only when it is explicit source data. Do NOT add a "Servings"/"Порции"/"Yield" row to the `ingredients` array.
- Use `<span class="ingredient-reference" ...>` for ingredient references in steps.
- Use `<span class="timer-reference" ...>` for timers in steps.
- To place recipes into folders, define `folders` and reference folder ids in `recipe.folderIds`.
- Set `illustrationId` only with slugs from the ingredient catalog; omit the field if unsure.

## File structure

### JSON without images

A single file, for example `recipes.json`:

```json
{
  "metadata": {
    "version": "1.5",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.5",
    "count": 1
  },
  "recipes": []
}
```

### ZIP with images

The archive should look like this:

```text
recipes-import.zip
├── recipes.json
└── images/
    └── recipe-1/
        ├── full.webp
        └── preview.webp
```

`recipes.json` must be at the archive root.

## Top-level object

```ts
{
  metadata: {
    version: "1.5";
    exportDate: string;   // ISO 8601
    type: "recipes-v1.5";
    count: number;        // number of recipes
  };
  recipes: RecipeImport[];
  folders?: FolderImport[];
  imageFiles?: Record<string, {
    full: string;
    preview: string;
  }>;
}
```

## Folders

Optional. Use when recipes should land in named collections after import.

```ts
type FolderImport = {
  id: string;
  name: string;
  color?: string | null;
  createdAt?: string | null;
  updatedAt?: string | null;
};
```

### Folder rules

- `folders` is optional. Omit it when no collection grouping is needed.
- Each folder needs a stable `id` and a human-readable `name`.
- Link recipes with `recipe.folderIds: string[]` using those folder ids.
- Unknown or missing folder ids in `folderIds` are skipped silently on import.
- Folder ids from the file are remapped to new ids in the app; only the ids inside the same import file must stay consistent.

### Example

```json
{
  "folders": [
    {
      "id": "baking",
      "name": "Baking"
    }
  ],
  "recipes": [
    {
      "id": "pancakes-001",
      "name": "Thin pancakes",
      "folderIds": ["baking"]
    }
  ]
}
```

## Recipe format

```ts
type RecipeImport = {
  id: string;
  name: string;
  description?: string;
  ingredients?: IngredientImport[];
  color?: string;
  servings?: number | null;
  folderIds?: string[];
  createdAt?: string | null;
  updatedAt?: string | null;
  originalRecipeLink?: string | null;
  originalRecipe?: string | null;
  nutrition?: {
    calories: number;
    protein: number;
    fat: number;
    carbs: number;
    calculatedAt: string;
    nutritionOutdated: boolean;
    totalWeight?: number;
  };
};
```

### Required recipe fields

- `id`
- `name`

### Recommended fields

- `description`
- `ingredients`
- `originalRecipeLink`

## Ingredients

Each ingredient:

```ts
type IngredientImport = {
  id: string;
  name: string;
  originalAmount: number | null;
  unit: string;
  order: number;
  isSeparator?: boolean;
  calories?: number;
  protein?: number;
  fat?: number;
  carbs?: number;
  weight?: number;
  illustrationId?: string;
};
```

### Ingredient rules

- `id` must be unique within the recipe.
- Keep `name` as the plain ingredient name.
- Keep `unit` separate from `name`.
- `originalAmount`:
  - a number for a normal ingredient
  - `null` for a section header
- `order` controls display order.
- For section headers, set:
  - `originalAmount: null`
  - `isSeparator: true`
  - `unit: ""`
- Optional per-ingredient nutrition fields (`calories`, `protein`, `fat`, `carbs`, `weight`) are preserved on import.
- `illustrationId` is optional. It must be a catalog slug such as `flour-wheat` or `butter`. Invalid or unknown slugs are ignored on import.

### Ingredient icons (`illustrationId`)

- Decorative only. The visible ingredient name stays in `name`.
- Use ids from the compact catalog: [llm-catalog.json](https://github.com/mikeozornin/recipe-scaler/blob/master/shared/data/ingredient-catalog/llm-catalog.json).
- Each catalog entry has `id`, `labelEn`, `labelRu`, and alias lists. Pick the closest `id`.
- If no good match exists, omit `illustrationId`.

### Example ingredients array

```json
[
  {
    "id": "dough",
    "name": "Dough",
    "originalAmount": null,
    "unit": "",
    "order": 1,
    "isSeparator": true
  },
  {
    "id": "flour",
    "name": "All-purpose flour",
    "originalAmount": 250,
    "unit": "g",
    "order": 2,
    "illustrationId": "flour-wheat"
  },
  {
    "id": "milk",
    "name": "Milk",
    "originalAmount": 180,
    "unit": "ml",
    "order": 3,
    "illustrationId": "milk-whole"
  }
]
```

## Recipe steps: HTML only

`description` must contain HTML.

### Allowed tags

- `p`
- `br`
- `div`
- `ul`
- `ol`
- `li`
- `strong`
- `em`
- `b`
- `i`
- `span`

Do not use:

- Markdown
- tables
- `img`
- `h1`, `h2`, `h3`
- arbitrary classes or data attributes beyond the ones defined below
- `script`, `style`, `iframe`

## Ingredient references inside steps

To make a number scale together with an ingredient, wrap the number in:

```html
<span class="ingredient-reference" data-ingredient-id="flour">250</span>
```

### Supported attributes

- `class="ingredient-reference"` required
- `data-ingredient-id` required
- `data-original-amount` optional but recommended
- `data-ratio` optional

### Recommended format

```html
<span
  class="ingredient-reference"
  data-ingredient-id="flour"
  data-original-amount="250"
>250</span>
```

### When to use `data-ratio`

If a step uses only part of the base ingredient amount, provide the ratio:

- ingredient list amount: `500 g`
- amount used in the step: `125`
- therefore `data-ratio="0.25"`

Example:

```html
Add
<span
  class="ingredient-reference"
  data-ingredient-id="cream"
  data-original-amount="125"
  data-ratio="0.25"
>125</span>
ml of cream.
```

### Repeated references to the same ingredient

If the same ingredient appears in multiple steps, reuse the same `data-ingredient-id` everywhere.

## Timers inside steps

To make the app detect a timer, use:

```html
<span
  class="timer-reference"
  data-timer-id="bake-1"
  data-duration="1800"
  data-type="minutes"
  data-name="Bake"
  data-value="30"
>30 minutes</span>
```

### Supported timer attributes

- `class="timer-reference"` required
- `data-timer-id` recommended
- `data-duration` required
- `data-type` required
- `data-name` recommended
- `data-value` recommended

### Timer rules

- `data-duration` is always in seconds.
- `data-type` must be one of:
  - `seconds`
  - `minutes`
  - `hours`
- `data-value` stores the original numeric value from the text.
- The visible text inside the tag should stay human-readable: `30 minutes`, `2 hours`, `45 sec`.

### Examples

```html
Let the dough rest for
<span class="timer-reference" data-timer-id="rest-1" data-duration="7200" data-type="hours" data-name="Rest dough" data-value="2">2 hours</span>.
```

```html
Bake for
<span class="timer-reference" data-timer-id="bake-1" data-duration="900" data-type="minutes" data-name="Bake" data-value="15">15 minutes</span>.
```

## Images

If images are not needed, omit `imageFiles` and use plain JSON.

If images are needed:

- create a ZIP
- put `recipes.json` at the root
- for each recipe add:
  - `images/{recipeId}/full.webp`
  - `images/{recipeId}/preview.webp`

### `imageFiles` field

```json
{
  "imageFiles": {
    "recipe-1": {
      "full": "images/recipe-1/full.webp",
      "preview": "images/recipe-1/preview.webp"
    }
  }
}
```

### Important image notes

- Use `.webp`.
- Paths in `imageFiles` must exactly match the file paths inside the ZIP.
- The key in `imageFiles` must match `recipe.id`.
- Do not put an image URL inside the recipe instead of using the ZIP structure.

## Full JSON example

```json
{
  "metadata": {
    "version": "1.5",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.5",
    "count": 1
  },
  "folders": [
    {
      "id": "breakfast",
      "name": "Breakfast"
    }
  ],
  "recipes": [
    {
      "id": "pancakes-001",
      "name": "Thin pancakes",
      "folderIds": ["breakfast"],
      "description": "<p>Mix <span class=\"ingredient-reference\" data-ingredient-id=\"milk\" data-original-amount=\"500\">500</span> ml of milk, <span class=\"ingredient-reference\" data-ingredient-id=\"eggs\" data-original-amount=\"2\">2</span> eggs, and <span class=\"ingredient-reference\" data-ingredient-id=\"flour\" data-original-amount=\"200\">200</span> g of flour.</p><p>Let the batter rest for <span class=\"timer-reference\" data-timer-id=\"rest-1\" data-duration=\"1800\" data-type=\"minutes\" data-name=\"Batter rest\" data-value=\"30\">30 minutes</span>.</p><p>Cook each pancake for <span class=\"timer-reference\" data-timer-id=\"fry-1\" data-duration=\"60\" data-type=\"seconds\" data-name=\"Cook pancake\" data-value=\"1\">1 minute</span> per side.</p>",
      "ingredients": [
        {
          "id": "base",
          "name": "Base",
          "originalAmount": null,
          "unit": "",
          "order": 1,
          "isSeparator": true
        },
        {
          "id": "milk",
          "name": "Milk",
          "originalAmount": 500,
          "unit": "ml",
          "order": 2,
          "illustrationId": "milk-whole"
        },
        {
          "id": "eggs",
          "name": "Eggs",
          "originalAmount": 2,
          "unit": "pcs",
          "order": 3,
          "illustrationId": "egg-whole"
        },
        {
          "id": "flour",
          "name": "All-purpose flour",
          "originalAmount": 200,
          "unit": "g",
          "order": 4,
          "illustrationId": "flour-wheat"
        }
      ],
      "servings": 4,
      "color": "oklch(0.65 0.25 270)",
      "originalRecipeLink": "https://example.com/pancakes"
    }
  ]
}
```

## Minimal template for agents

```json
{
  "metadata": {
    "version": "1.5",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.5",
    "count": 1
  },
  "recipes": [
    {
      "id": "unique-recipe-id",
      "name": "Recipe name",
      "description": "<p>Recipe steps HTML</p>",
      "ingredients": [
        {
          "id": "ingredient-1",
          "name": "Ingredient name",
          "originalAmount": 100,
          "unit": "g",
          "order": 1
        }
      ]
    }
  ]
}
```

## Common mistakes

- Wrong version:
  - not `"1.5"`
- Wrong type:
  - not `"recipes-v1.5"`
- `count` does not match the number of recipes.
- `description` contains Markdown or plain text instead of HTML.
- An `ingredient-reference` span has no `data-ingredient-id`.
- `data-ingredient-id` points to a missing ingredient.
- `data-duration` is written in minutes or hours instead of seconds.
- The ZIP does not include `recipes.json`.
- A path in `imageFiles` does not match the actual ZIP path.
- `preview.webp` is missing from the archive.
- Section headers do not use `originalAmount: null` and `isSeparator: true`.
- `folderIds` references ids that are not defined in `folders`.
- `illustrationId` uses a made-up slug instead of a catalog `id`.

## Practical guidance for LLM agents

- Generate short stable ids such as `recipe-1`, `flour`, `bake-1`, `folder-baking`.
- Do not add random HTML attributes.
- Do not embed images through `<img>` inside `description`.
- If unsure, generate JSON without images.
- If a number in the text should scale, wrap only the number, not the whole sentence.
- If a timer is written as a range like `15-20 minutes`, keep the visible original text and use the upper value in `data-duration`.
- Skip `folders`, `folderIds`, and `illustrationId` unless the user explicitly needs collections or ingredient icons.
