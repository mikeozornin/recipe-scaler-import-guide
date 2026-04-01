# Recipe Scaler: import format for LLM agents

This document describes the current `v1.2` import format already supported by the app.
If an agent generates a file that follows this spec, the user can import it via `Personal Area -> Data management -> Import recipes`.

## What is supported

- One or more recipes.
- `JSON` when recipes do not include images.
- `ZIP` when images must be imported.
- HTML recipe steps in `description`.
- Scalable ingredient references inside steps.
- Timers inside steps.
- Optional nutrition data.

## Quick rules

- Use `v1.2` only.
- Set `metadata.version` to `"1.2"`.
- Set `metadata.type` to `"recipes-v1.2"`.
- If there are no images, a single `.json` file is enough.
- If there are images, build a `.zip` with `recipes.json` at the archive root.
- Put HTML in `description`, not Markdown.
- Use `<span class="ingredient-reference" ...>` for ingredient references in steps.
- Use `<span class="timer-reference" ...>` for timers in steps.

## File structure

### JSON without images

A single file, for example `recipes.json`:

```json
{
  "metadata": {
    "version": "1.2",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.2",
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
    version: "1.2";
    exportDate: string;   // ISO 8601
    type: "recipes-v1.2";
    count: number;        // number of recipes
  };
  recipes: RecipeImport[];
  imageFiles?: Record<string, {
    full: string;
    preview: string;
  }>;
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
  scaleFactor?: number | string;
  createdAt?: string | null;
  updatedAt?: string | null;
  originalRecipeLink?: string | null;
  originalRecipe?: string | null;
  nutrition?: {
    calories: number;
    protein: number;
    fat: number;
    carbs: number;
    totalWeight?: number;
    calculatedAt?: string;
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
    "order": 2
  },
  {
    "id": "milk",
    "name": "Milk",
    "originalAmount": 180,
    "unit": "ml",
    "order": 3
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
    "version": "1.2",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.2",
    "count": 1
  },
  "recipes": [
    {
      "id": "pancakes-001",
      "name": "Thin pancakes",
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
          "order": 2
        },
        {
          "id": "eggs",
          "name": "Eggs",
          "originalAmount": 2,
          "unit": "pcs",
          "order": 3
        },
        {
          "id": "flour",
          "name": "All-purpose flour",
          "originalAmount": 200,
          "unit": "g",
          "order": 4
        }
      ],
      "color": "oklch(0.65 0.25 270)",
      "scaleFactor": 1,
      "originalRecipeLink": "https://example.com/pancakes"
    }
  ]
}
```

## Minimal template for agents

```json
{
  "metadata": {
    "version": "1.2",
    "exportDate": "2026-04-02T12:00:00.000Z",
    "type": "recipes-v1.2",
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
  - not `"1.2"`
- Wrong type:
  - not `"recipes-v1.2"`
- `count` does not match the number of recipes.
- `description` contains Markdown or plain text instead of HTML.
- An `ingredient-reference` span has no `data-ingredient-id`.
- `data-ingredient-id` points to a missing ingredient.
- `data-duration` is written in minutes or hours instead of seconds.
- The ZIP does not include `recipes.json`.
- A path in `imageFiles` does not match the actual ZIP path.
- `preview.webp` is missing from the archive.
- Section headers do not use `originalAmount: null` and `isSeparator: true`.

## Practical guidance for LLM agents

- Generate short stable ids such as `recipe-1`, `flour`, `bake-1`.
- Do not add random HTML attributes.
- Do not embed images through `<img>` inside `description`.
- If unsure, generate JSON without images.
- If a number in the text should scale, wrap only the number, not the whole sentence.
- If a timer is written as a range like `15-20 minutes`, keep the visible original text and use the upper value in `data-duration`.
