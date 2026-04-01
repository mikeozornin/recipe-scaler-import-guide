# Recipe Scaler: формат импорта для LLM-агента

Этот документ описывает актуальный формат импорта `v1.2`, который уже поддерживается приложением.
Если агент сгенерирует файл по этой спецификации, пользователь сможет импортировать его через `Личный кабинет -> Управление данными -> Импортировать рецепты`.

## Что поддерживается

- Один или несколько рецептов.
- `JSON`, если в рецептах нет изображений.
- `ZIP`, если нужно импортировать изображения.
- HTML-разметка шагов в поле `description`.
- Масштабируемые ссылки на ингредиенты внутри шагов.
- Таймеры внутри шагов.
- Опциональная nutrition-информация.

## Быстрые правила

- Используйте только формат `v1.2`.
- В `metadata.version` ставьте `"1.2"`.
- В `metadata.type` ставьте `"recipes-v1.2"`.
- Если изображений нет, достаточно одного `.json` файла.
- Если изображения есть, делайте `.zip` с файлом `recipes.json` в корне архива.
- В `description` кладите HTML, не Markdown.
- Для ингредиентов в шагах используйте `<span class="ingredient-reference" ...>`.
- Для таймеров в шагах используйте `<span class="timer-reference" ...>`.

## Структура файла

### JSON без изображений

Один файл, например `recipes.json`:

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

### ZIP с изображениями

Архив должен выглядеть так:

```text
recipes-import.zip
├── recipes.json
└── images/
    └── recipe-1/
        ├── full.webp
        └── preview.webp
```

`recipes.json` лежит в корне архива.

## Корневой объект

```ts
{
  metadata: {
    version: "1.2";
    exportDate: string;   // ISO 8601
    type: "recipes-v1.2";
    count: number;        // количество рецептов
  };
  recipes: RecipeImport[];
  imageFiles?: Record<string, {
    full: string;
    preview: string;
  }>;
}
```

## Формат рецепта

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

### Обязательные поля рецепта

- `id`
- `name`

### Рекомендуемые поля

- `description`
- `ingredients`
- `originalRecipeLink`

## Ингредиенты

Каждый ингредиент:

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

### Правила по ингредиентам

- `id` должен быть уникальным в пределах рецепта.
- `name` храните как чистое название ингредиента.
- `unit` храните отдельно, без добавления в `name`.
- `originalAmount`:
  - число для обычного ингредиента
  - `null` для заголовка секции
- `order` должен задавать порядок показа.
- Для заголовков секций ставьте:
  - `originalAmount: null`
  - `isSeparator: true`
  - `unit: ""`

### Пример массива ингредиентов

```json
[
  {
    "id": "dough",
    "name": "Тесто",
    "originalAmount": null,
    "unit": "",
    "order": 1,
    "isSeparator": true
  },
  {
    "id": "flour",
    "name": "Пшеничная мука",
    "originalAmount": 250,
    "unit": "г",
    "order": 2
  },
  {
    "id": "milk",
    "name": "Молоко",
    "originalAmount": 180,
    "unit": "мл",
    "order": 3
  }
]
```

## Шаги рецепта: только HTML

Поле `description` должно содержать HTML.

### Разрешённые теги

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

Не используйте:

- Markdown
- таблицы
- `img`
- `h1`, `h2`, `h3`
- произвольные классы и data-атрибуты кроме описанных ниже
- `script`, `style`, `iframe`

## Ссылки на ингредиенты в шагах

Чтобы число в тексте масштабировалось вместе с ингредиентом, нужно обернуть его в:

```html
<span class="ingredient-reference" data-ingredient-id="flour">250</span>
```

### Поддерживаемые атрибуты

- `class="ingredient-reference"` обязательно
- `data-ingredient-id` обязательно
- `data-original-amount` опционально, но желательно
- `data-ratio` опционально

### Рекомендуемый формат

```html
<span
  class="ingredient-reference"
  data-ingredient-id="flour"
  data-original-amount="250"
>250</span>
```

### Когда нужен `data-ratio`

Если в шаге используется не весь ингредиент, а часть от базового количества, укажите отношение:

- ингредиент в списке: `500 г`
- в шаге используется `125`
- значит `data-ratio="0.25"`

Пример:

```html
Добавьте
<span
  class="ingredient-reference"
  data-ingredient-id="cream"
  data-original-amount="125"
  data-ratio="0.25"
>125</span>
мл сливок.
```

### Повторные ссылки на один ингредиент

Если один и тот же ингредиент встречается в нескольких шагах, везде используйте один и тот же `data-ingredient-id`.

## Таймеры в шагах

Чтобы приложение распознало таймер, используйте:

```html
<span
  class="timer-reference"
  data-timer-id="bake-1"
  data-duration="1800"
  data-type="minutes"
  data-name="Выпекать"
  data-value="30"
>30 минут</span>
```

### Поддерживаемые атрибуты таймера

- `class="timer-reference"` обязательно
- `data-timer-id` желательно
- `data-duration` обязательно
- `data-type` обязательно
- `data-name` желательно
- `data-value` желательно

### Правила по таймерам

- `data-duration` всегда в секундах.
- `data-type` одно из:
  - `seconds`
  - `minutes`
  - `hours`
- `data-value` хранит исходное числовое значение из текста.
- Текст внутри тега должен оставаться человекочитаемым: `30 минут`, `2 часа`, `45 sec`.

### Примеры

```html
Оставьте тесто на
<span class="timer-reference" data-timer-id="rest-1" data-duration="7200" data-type="hours" data-name="Оставить тесто" data-value="2">2 часа</span>.
```

```html
Выпекайте
<span class="timer-reference" data-timer-id="bake-1" data-duration="900" data-type="minutes" data-name="Выпекать" data-value="15">15 минут</span>.
```

## Изображения

Если изображения не нужны, не добавляйте `imageFiles` и используйте обычный JSON.

Если изображения нужны:

- создайте ZIP
- положите `recipes.json` в корень
- для каждого рецепта добавьте:
  - `images/{recipeId}/full.webp`
  - `images/{recipeId}/preview.webp`

### Поле `imageFiles`

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

### Важные замечания по картинкам

- Используйте `.webp`.
- Пути в `imageFiles` должны точно совпадать с путями внутри ZIP.
- Ключ в `imageFiles` должен совпадать с `recipe.id`.
- Не кладите URL изображения в рецепт вместо ZIP-структуры.

## Полный пример JSON

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
      "name": "Тонкие блины",
      "description": "<p>Смешайте <span class=\"ingredient-reference\" data-ingredient-id=\"milk\" data-original-amount=\"500\">500</span> мл молока, <span class=\"ingredient-reference\" data-ingredient-id=\"eggs\" data-original-amount=\"2\">2</span> яйца и <span class=\"ingredient-reference\" data-ingredient-id=\"flour\" data-original-amount=\"200\">200</span> г муки.</p><p>Оставьте тесто на <span class=\"timer-reference\" data-timer-id=\"rest-1\" data-duration=\"1800\" data-type=\"minutes\" data-name=\"Отдых теста\" data-value=\"30\">30 минут</span>.</p><p>Жарьте каждый блин по <span class=\"timer-reference\" data-timer-id=\"fry-1\" data-duration=\"60\" data-type=\"seconds\" data-name=\"Жарить блин\" data-value=\"1\">1 минуте</span> с каждой стороны.</p>",
      "ingredients": [
        {
          "id": "base",
          "name": "Основа",
          "originalAmount": null,
          "unit": "",
          "order": 1,
          "isSeparator": true
        },
        {
          "id": "milk",
          "name": "Молоко",
          "originalAmount": 500,
          "unit": "мл",
          "order": 2
        },
        {
          "id": "eggs",
          "name": "Яйца",
          "originalAmount": 2,
          "unit": "шт",
          "order": 3
        },
        {
          "id": "flour",
          "name": "Пшеничная мука",
          "originalAmount": 200,
          "unit": "г",
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

## Минимальный шаблон для агента

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
      "name": "Название рецепта",
      "description": "<p>HTML шагов рецепта</p>",
      "ingredients": [
        {
          "id": "ingredient-1",
          "name": "Название ингредиента",
          "originalAmount": 100,
          "unit": "г",
          "order": 1
        }
      ]
    }
  ]
}
```

## Типовые ошибки

- Неверная версия:
  - не `"1.2"`
- Неверный тип:
  - не `"recipes-v1.2"`
- `count` не совпадает с количеством рецептов.
- В `description` лежит Markdown или plain text вместо HTML.
- У `ingredient-reference` нет `data-ingredient-id`.
- `data-ingredient-id` ссылается на несуществующий ингредиент.
- `data-duration` указан в минутах или часах, а не в секундах.
- В `ZIP` нет файла `recipes.json`.
- В `imageFiles` путь не совпадает с реальным путём файла в архиве.
- В архиве нет `preview.webp`.
- Для секции ингредиентов не выставлены `originalAmount: null` и `isSeparator: true`.

## Практические рекомендации для LLM-агента

- Генерируйте короткие, стабильные `id`, например `recipe-1`, `flour`, `bake-1`.
- Не используйте случайные HTML-атрибуты.
- Не вставляйте изображения через `<img>` в `description`.
- Если сомневаетесь, делайте JSON без изображений.
- Если в тексте есть число, которое должно масштабироваться, заворачивайте только число, а не всю фразу.
- Если таймер встречается как диапазон `15-20 минут`, сохраняйте внутри тега видимый исходный текст, а в `data-duration` ставьте верхнее значение в секундах.
