# Nova Resources

`Vendor` is a placeholder for the vendor namespace root; the real root is defined
per-project in `CLAUDE.md`. Module segments (`User`, `Post`, `Base`) map to the
project's modular packages.

## BaseResource abstract class

All Nova resources extend a shared base (either directly, or via
`App\Nova\Resource` which extends it). The base derives translation keys from the
class name, so labels and button captions come from language files automatically.

**Namespace:** `Vendor\Base\Nova`

```php
<?php

declare(strict_types = 1);

namespace Vendor\Base\Nova;

use Illuminate\Contracts\Database\Eloquent\Builder;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Resource as NovaResource;
use Laravel\Scout\Builder as ScoutBuilder;

abstract class BaseResource extends NovaResource
{
    /**
     * Translation prefix for the resource (e.g., 'user::', 'post::').
     */
    protected static string $translatePrefix = '';

    /**
     * Get the resource name (class basename).
     */
    public static function resourceName(): string
    {
        return class_basename(static::class);
    }

    /**
     * Get the resource singular name in snake_case.
     */
    public static function singular(): string
    {
        return mb_strtolower(preg_replace('/(?<!^)[A-Z]/', '_$0', static::resourceName()));
    }

    /**
     * Get the displayable singular label of the resource.
     */
    public static function singularLabel(): string
    {
        return (string) __(static::$translatePrefix . static::singular() . '.singular');
    }

    /**
     * Get the displayable (plural) label of the resource.
     */
    public static function label(): string
    {
        return (string) __(static::$translatePrefix . static::singular() . '.plural');
    }

    /**
     * Get the text for the "create resource" button.
     */
    public static function createButtonLabel(): string
    {
        return (string) __(static::$translatePrefix . static::singular() . '.button.create');
    }

    /**
     * Get the text for the "update resource" button.
     */
    public static function updateButtonLabel(): string
    {
        return (string) __(static::$translatePrefix . static::singular() . '.button.update');
    }

    /**
     * Build an "index" query for the given resource.
     */
    public static function indexQuery(NovaRequest $request, Builder $query): Builder
    {
        return $query;
    }

    /**
     * Build a Scout search query for the given resource.
     */
    public static function scoutQuery(NovaRequest $request, ScoutBuilder $query): ScoutBuilder
    {
        return $query;
    }

    /**
     * Build a "detail" query for the given resource.
     */
    public static function detailQuery(NovaRequest $request, Builder $query): Builder
    {
        return parent::detailQuery($request, $query);
    }

    /**
     * Build a "relatable" query for the given resource.
     */
    public static function relatableQuery(NovaRequest $request, Builder $query): Builder
    {
        return parent::relatableQuery($request, $query);
    }
}
```

**What it gives you:**

- Automatic translation-key generation based on the class name.
- A single `$translatePrefix` per resource (e.g. `'user::'`).
- Auto-translated singular/plural labels and create/update button captions.
- Query hooks — override `indexQuery()` / `detailQuery()` for eager loading.

**Nova 4:** the typed `Builder $query` / `ScoutBuilder $query` parameters above
match Nova 5's signatures. On Nova 4 the parent's `$query` is untyped, so this
class is a fatal `Declaration … must be compatible` as soon as it loads — declare
`$query` without a type there and keep the return types. See
`framework` in [`nova-cross-cutting-quality.md`](nova-cross-cutting-quality.md)
for the other Nova 4 vs Nova 5 differences.

## Module layout

Resources typically live inside modular packages. Three names can differ and are
each configured per-project in `CLAUDE.md`:

- **On-disk directory** — where the package sits in the repo (e.g. `packages/user/`).
- **Composer package name** — e.g. `vendor/user`.
- **PHP namespace** — e.g. `Vendor\User`.

```
packages/user/
├── composer.json                 # "name": "vendor/user"
├── resources/lang/{en,sk}/user.php
└── src/                          # Namespace: Vendor\User
    ├── Nova/
    │   ├── Actions/
    │   ├── Filters/
    │   ├── Lenses/
    │   ├── Metrics/
    │   └── User.php              # Vendor\User\Nova\User extends the base resource
    └── Policies/
        └── UserPolicy.php
```

## Basic resource

Use the `title()` and `searchableColumns()` **methods** — never the `$title` or
`$search` properties. Give every field `help()` text backed by a translation key.

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova;

use App\Nova\Resource;
use Laravel\Nova\Fields\Email;
use Laravel\Nova\Fields\ID;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Http\Requests\NovaRequest;

class User extends Resource
{
    /**
     * The model the resource corresponds to.
     */
    public static string $model = \Vendor\User\Models\User::class;

    /**
     * Translation prefix (module::).
     */
    public static string $translatePrefix = 'user::';

    /**
     * Get the value used to represent the resource.
     */
    public function title(): string
    {
        return $this->name;
    }

    /**
     * The columns that should be searched.
     */
    public static function searchableColumns(): array
    {
        return ['id', 'name', 'email'];
    }

    /**
     * Get the fields displayed by the resource.
     */
    public function fields(NovaRequest $request): array
    {
        return [
            ID::make()->sortable(),

            Text::make(__('user::user.field.name.label'), 'name')
                ->help(__('user::user.field.name.help'))
                ->sortable()
                ->rules('required', 'max:100'),

            Email::make(__('user::user.field.email.label'), 'email')
                ->help(__('user::user.field.email.help'))
                ->sortable()
                ->rules('required', 'email', 'max:254'),
        ];
    }
}
```

## Advanced resource with panels

Group related fields into `Panel`s (tabs) and eager-load relationships in
`indexQuery()` / `detailQuery()` to avoid N+1 queries.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova;

use App\Nova\Resource;
use Laravel\Nova\Fields\BelongsTo;
use Laravel\Nova\Fields\DateTime;
use Laravel\Nova\Fields\HasMany;
use Laravel\Nova\Fields\ID;
use Laravel\Nova\Fields\Select;
use Laravel\Nova\Fields\Slug;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Fields\Textarea;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Panel;
use Vendor\Comment\Nova\Comment;
use Vendor\User\Nova\User;

class Post extends Resource
{
    public static string $model = \Vendor\Post\Models\Post::class;

    public static string $translatePrefix = 'post::';

    public function title(): string
    {
        return $this->title;
    }

    public static function searchableColumns(): array
    {
        return ['id', 'title', 'slug'];
    }

    public function fields(NovaRequest $request): array
    {
        return [
            // Basic information
            new Panel(__('post::post.tabs.basic_information'), [
                ID::make()->sortable(),

                Text::make(__('post::post.field.title.label'), 'title')
                    ->help(__('post::post.field.title.help'))
                    ->sortable()
                    ->rules('required', 'max:150'),

                Slug::make(__('post::post.field.slug.label'), 'slug')
                    ->from('title')
                    ->help(__('post::post.field.slug.help'))
                    ->rules('required', 'max:150')
                    ->creationRules('unique:posts,slug')
                    ->updateRules('unique:posts,slug,{{resourceId}}'),

                Textarea::make(__('post::post.field.excerpt.label'), 'excerpt')
                    ->help(__('post::post.field.excerpt.help'))
                    ->rows(3),
            ]),

            // Publishing
            new Panel(__('post::post.tabs.publishing'), [
                Select::make(__('post::post.field.status.label'), 'status')
                    ->help(__('post::post.field.status.help'))
                    ->options([
                        'draft' => __('post::post.status.draft'),
                        'published' => __('post::post.status.published'),
                        'archived' => __('post::post.status.archived'),
                    ])
                    ->displayUsingLabels()
                    ->sortable(),

                DateTime::make(__('post::post.field.published_at.label'), 'published_at')
                    ->help(__('post::post.field.published_at.help'))
                    ->sortable()
                    ->nullable(),
            ]),

            // Relationships
            new Panel(__('post::post.tabs.relationships'), [
                BelongsTo::make(__('post::post.field.author.label'), 'author', User::class)
                    ->help(__('post::post.field.author.help'))
                    ->searchable()
                    ->nullable(),

                HasMany::make(__('post::post.field.comments.label'), 'comments', Comment::class),
            ]),

            // Metadata
            new Panel(__('post::post.tabs.metadata'), [
                DateTime::make(__('post::post.field.created_at.label'), 'created_at')
                    ->help(__('post::post.field.created_at.help'))
                    ->sortable()
                    ->readonly()
                    ->showOnPreview(),

                DateTime::make(__('post::post.field.updated_at.label'), 'updated_at')
                    ->help(__('post::post.field.updated_at.help'))
                    ->sortable()
                    ->readonly(),
            ]),
        ];
    }

    /**
     * Eager load relationships for the index view.
     */
    public static function indexQuery(NovaRequest $request, $query)
    {
        return $query->with(['author', 'comments']);
    }

    /**
     * Eager load relationships for the detail view.
     */
    public static function detailQuery(NovaRequest $request, $query)
    {
        return $query->with(['author', 'comments.author']);
    }
}
```

## Translation keys

Every label, help text, tab, button and enum option resolves through a language
file, keyed off the module prefix. The **key** is always English words (e.g.
`field.title.help`); the **value** in the language file is the target language.
Never put a raw string literal where a translation key belongs — not even
"temporarily", not even for a field that looks obvious. Which languages a
project ships, and whether translation into all of them happens automatically,
is defined in that project's own `CLAUDE.md` — check it before assuming a list.

```
module::entity.singular
module::entity.plural
module::entity.button.create
module::entity.button.update
module::entity.field.{field_name}.label
module::entity.field.{field_name}.help
module::entity.tabs.{tab_name}
module::entity.status.{status}
```

Translation file — `packages/post/resources/lang/en/post.php`:

```php
<?php

return [
    'plural' => 'Posts',
    'singular' => 'Post',

    'button.create' => 'Add post',
    'button.update' => 'Edit post',

    'tabs.basic_information' => 'Basic information',
    'tabs.publishing' => 'Publishing',
    'tabs.relationships' => 'Relationships',
    'tabs.metadata' => 'Metadata',

    'status.draft' => 'Draft',
    'status.published' => 'Published',
    'status.archived' => 'Archived',

    'field.id.label' => 'ID',
    'field.id.help' => 'Unique identifier of the post',
    'field.title.label' => 'Title',
    'field.title.help' => 'Public title of the post',
    'field.slug.label' => 'Slug',
    'field.slug.help' => 'URL-friendly identifier generated from the title',
    'field.excerpt.label' => 'Excerpt',
    'field.excerpt.help' => 'Short summary shown in listings',
    'field.status.label' => 'Status',
    'field.status.help' => 'Publication state of the post',
    'field.published_at.label' => 'Published at',
    'field.published_at.help' => 'Date and time the post goes live',
    'field.author.label' => 'Author',
    'field.author.help' => 'User who owns the post',
    'field.comments.label' => 'Comments',
    'field.created_at.label' => 'Created at',
    'field.created_at.help' => 'Date and time the record was created',
    'field.updated_at.label' => 'Updated at',
    'field.updated_at.help' => 'Date and time of the last update',
];
```

## Resource checklist

- Extend `App\Nova\Resource` (which extends the shared `BaseResource`).
- Set `$translatePrefix` with a trailing `::` (e.g. `'user::'`).
- Use the `title()` method, not the `$title` property.
- Use the `searchableColumns()` method, not the `$search` property.
- Add `help()` with a translation key to **every** field — no exceptions for
  fields that seem self-explanatory. Write a short, plain-language explanation
  of what the field is for; a translated label alone is not enough.
- This applies when editing an **existing** resource too: if you touch a
  resource's `fields()` for any reason, fix any sibling fields in the same
  method that are still missing `help()` or still hardcode a label/help string
  instead of a translation key.
- Add `sortable()`, `filterable()`, `copyable()` where they make sense.
- Add `showOnPreview()` / `showWhenPeeking()` for key fields.
- Organize fields with `Panel`s (tabs).
- Eager load relationships in `indexQuery()` and `detailQuery()`.
- Make the resource reachable by clicking: in the custom `Nova::mainMenu()` if
  the project has one, or — with `$displayInNavigation = false` — through a
  relation field on its parent. Register a policy (with `viewAny`) and apply the
  tenant scope beyond `indexQuery()`. Keep `searchableColumns()` short and
  index-friendly. Details: [`nova-cross-cutting-quality.md`](nova-cross-cutting-quality.md).
