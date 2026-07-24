# Nova Actions, Lenses, Filters, Metrics & Cards

`Vendor` is a placeholder for the vendor namespace root, defined per-project in
`CLAUDE.md`. Enum cases (`PostStatusEnum`) expose `label()` and `color()` helpers
used for translated option labels and partition colors.

## Actions

### Simple action (no fields)

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova\Actions;

use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;
use Laravel\Nova\Http\Requests\NovaRequest;
use Vendor\User\Models\User;

class ActivateUser extends Action
{
    /**
     * The displayable name of the action.
     */
    public $name;

    public function __construct()
    {
        $this->name = __('user::actions.activate_user.name');
    }

    /**
     * Perform the action on the given models.
     */
    public function handle(ActionFields $fields, Collection $models): array
    {
        foreach ($models as $model) {
            if ($model instanceof User) {
                $model->update(['is_active' => true]);
            }
        }

        return Action::message(__('user::actions.activate_user.success', [
            'count' => $models->count(),
        ]));
    }

    public function fields(NovaRequest $request): array
    {
        return [];
    }
}
```

### Action with fields (enum-driven)

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Actions;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;
use Laravel\Nova\Fields\Select;
use Laravel\Nova\Fields\Textarea;
use Laravel\Nova\Http\Requests\NovaRequest;
use Vendor\Post\Enums\PostStatusEnum;
use Vendor\Post\Models\Post;

class UpdatePostStatus extends Action
{
    use InteractsWithQueue, Queueable;

    public $name;

    public function __construct()
    {
        $this->name = __('post::actions.update_status.name');
    }

    public function handle(ActionFields $fields, Collection $models): array
    {
        foreach ($models as $model) {
            if ($model instanceof Post) {
                $model->update([
                    'status' => $fields->status,
                    'note' => $fields->note,
                ]);
            }
        }

        return Action::message(__('post::actions.update_status.success', [
            'count' => $models->count(),
        ]));
    }

    public function fields(NovaRequest $request): array
    {
        return [
            Select::make(__('post::actions.update_status.field.status.label'), 'status')
                ->help(__('post::actions.update_status.field.status.help'))
                ->options(collect(PostStatusEnum::cases())->mapWithKeys(fn ($case) => [
                    $case->value => $case->label(),
                ]))
                ->rules('required'),

            Textarea::make(__('post::actions.update_status.field.note.label'), 'note')
                ->help(__('post::actions.update_status.field.note.help'))
                ->rows(3),
        ];
    }
}
```

### Queued action sending a notification

Add `InteractsWithQueue, Queueable` for long-running work (sending mail, calling
external APIs) so it runs on the queue.

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova\Actions;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;
use Laravel\Nova\Fields\Boolean;
use Laravel\Nova\Fields\Textarea;
use Laravel\Nova\Http\Requests\NovaRequest;
use Vendor\User\Models\User;
use Vendor\User\Notifications\WelcomeNotification;

class SendWelcomeEmail extends Action
{
    use InteractsWithQueue, Queueable;

    public $name;

    public function __construct()
    {
        $this->name = __('user::actions.send_welcome_email.name');
    }

    public function handle(ActionFields $fields, Collection $models): array
    {
        foreach ($models as $model) {
            if ($model instanceof User) {
                $model->notify(new WelcomeNotification(
                    customMessage: $fields->custom_message,
                    includeLoginDetails: $fields->include_login_details,
                ));
            }
        }

        return Action::message(__('user::actions.send_welcome_email.success', [
            'count' => $models->count(),
        ]));
    }

    public function fields(NovaRequest $request): array
    {
        return [
            Textarea::make(__('user::actions.send_welcome_email.field.custom_message.label'), 'custom_message')
                ->help(__('user::actions.send_welcome_email.field.custom_message.help'))
                ->rows(3),

            Boolean::make(__('user::actions.send_welcome_email.field.include_login_details.label'), 'include_login_details')
                ->help(__('user::actions.send_welcome_email.field.include_login_details.help')),
        ];
    }
}
```

**Action checklist:** translate the name and messages, add help text to fields,
return `Action::message()` (or `Action::danger()`) for feedback, queue slow work,
and type-check each model in `handle()`.

## Lenses

A lens is an alternate, pre-filtered view of a resource. Eager-load in `query()`,
keep field configuration consistent with the main resource, and give it a
`uriKey()`. Lenses may also declare their own `filters()` and `actions()`.

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova\Lenses;

use Laravel\Nova\Fields\Boolean;
use Laravel\Nova\Fields\DateTime;
use Laravel\Nova\Fields\Email;
use Laravel\Nova\Fields\ID;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Http\Requests\LensRequest;
use Laravel\Nova\Lenses\Lens;
use Vendor\User\Nova\Filters\UserRoleFilter;

class ActiveUsers extends Lens
{
    public $name;

    public function __construct()
    {
        $this->name = __('user::lenses.active_users.name');
    }

    /**
     * Get the query builder / paginator for the lens.
     */
    public static function query(LensRequest $request, $query)
    {
        return $request->withOrdering($request->withFilters(
            $query->where('is_active', true)
                ->with(['roles', 'profile'])
        ));
    }

    public function fields(LensRequest $request): array
    {
        return [
            ID::make()
                ->help(__('user::user.field.id.help'))
                ->sortable(),

            Text::make(__('user::user.field.name.label'), 'name')
                ->help(__('user::user.field.name.help'))
                ->sortable()
                ->copyable(),

            Email::make(__('user::user.field.email.label'), 'email')
                ->help(__('user::user.field.email.help'))
                ->sortable()
                ->copyable(),

            Boolean::make(__('user::user.field.is_active.label'), 'is_active')
                ->help(__('user::user.field.is_active.help'))
                ->sortable(),

            DateTime::make(__('user::user.field.created_at.label'), 'created_at')
                ->help(__('user::user.field.created_at.help'))
                ->sortable(),
        ];
    }

    public function filters(LensRequest $request): array
    {
        return [
            new UserRoleFilter,
        ];
    }

    public function actions(LensRequest $request): array
    {
        return parent::actions($request);
    }

    public function uriKey(): string
    {
        return 'active-users';
    }
}
```

## Filters

Return the query from `apply()`; keep the logic focused. Use translation keys for
the name and options.

### Select filter (enum-backed)

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Filters;

use Laravel\Nova\Filters\Filter;
use Laravel\Nova\Http\Requests\NovaRequest;
use Vendor\Post\Enums\PostStatusEnum;

class PostStatusFilter extends Filter
{
    public $name;

    public $component = 'select-filter';

    public function __construct()
    {
        $this->name = __('post::filters.status.name');
    }

    public function apply(NovaRequest $request, $query, $value)
    {
        return $query->where('status', $value);
    }

    public function options(NovaRequest $request): array
    {
        return collect(PostStatusEnum::cases())->mapWithKeys(fn ($case) => [
            $case->label() => $case->value,
        ])->toArray();
    }
}
```

For a fixed set of options, return a plain map instead:

```php
public function options(NovaRequest $request): array
{
    return [
        __('user::filters.user_status.options.active') => true,
        __('user::filters.user_status.options.inactive') => false,
    ];
}
```

### Boolean filter (multiple checkboxes)

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova\Filters;

use Laravel\Nova\Filters\BooleanFilter;
use Laravel\Nova\Http\Requests\NovaRequest;

class UserAttributesFilter extends BooleanFilter
{
    public $name;

    public function __construct()
    {
        $this->name = __('user::filters.attributes.name');
    }

    public function apply(NovaRequest $request, $query, $value)
    {
        if ($value['email_verified']) {
            $query->whereNotNull('email_verified_at');
        }

        if ($value['is_active']) {
            $query->where('is_active', true);
        }

        return $query;
    }

    public function options(NovaRequest $request): array
    {
        return [
            __('user::filters.attributes.options.email_verified') => 'email_verified',
            __('user::filters.attributes.options.is_active') => 'is_active',
        ];
    }
}
```

### Date filter

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Filters;

use Laravel\Nova\Filters\DateFilter;
use Laravel\Nova\Http\Requests\NovaRequest;

class PostCreatedDateFilter extends DateFilter
{
    public $name;

    public function __construct()
    {
        $this->name = __('post::filters.created_date.name');
    }

    public function apply(NovaRequest $request, $query, $value)
    {
        return $query->whereDate('created_at', $value);
    }
}
```

## Metrics

### Value metric

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Metrics;

use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Value;
use Vendor\Post\Models\Post;

class TotalPosts extends Value
{
    public $name;

    public function __construct()
    {
        $this->name = __('post::metrics.total_posts.name');
    }

    public function calculate(NovaRequest $request)
    {
        return $this->count($request, Post::class)
            ->suffix(__('post::metrics.total_posts.suffix'));
    }

    public function ranges(): array
    {
        return [
            7 => __('post::metrics.ranges.7_days'),
            30 => __('post::metrics.ranges.30_days'),
            60 => __('post::metrics.ranges.60_days'),
            90 => __('post::metrics.ranges.90_days'),
        ];
    }

    public function uriKey(): string
    {
        return 'total-posts';
    }
}
```

### Trend metric

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Metrics;

use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Trend;
use Vendor\Post\Models\Post;

class PostsPerDay extends Trend
{
    public $name;

    public function __construct()
    {
        $this->name = __('post::metrics.posts_per_day.name');
    }

    public function calculate(NovaRequest $request)
    {
        return $this->countByDays($request, Post::class)
            ->suffix(__('post::metrics.posts_per_day.suffix'));
    }

    public function ranges(): array
    {
        return [
            7 => __('post::metrics.ranges.7_days'),
            14 => __('post::metrics.ranges.14_days'),
            30 => __('post::metrics.ranges.30_days'),
        ];
    }

    public function uriKey(): string
    {
        return 'posts-per-day';
    }
}
```

### Partition metric (colored by enum)

```php
<?php

declare(strict_types = 1);

namespace Vendor\Post\Nova\Metrics;

use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Partition;
use Vendor\Post\Enums\PostStatusEnum;
use Vendor\Post\Models\Post;

class PostsByStatus extends Partition
{
    public $name;

    public function __construct()
    {
        $this->name = __('post::metrics.posts_by_status.name');
    }

    public function calculate(NovaRequest $request)
    {
        return $this->count($request, Post::class, 'status')
            ->label(fn ($value) => PostStatusEnum::from($value)->label())
            ->colors(
                collect(PostStatusEnum::cases())->mapWithKeys(fn ($case) => [
                    $case->label() => $case->color(),
                ])->toArray()
            );
    }

    public function uriKey(): string
    {
        return 'posts-by-status';
    }
}
```

## Custom eCharts card

For visualizations beyond the built-in metrics, register a custom card whose Vue
component renders an [eCharts](https://echarts.apache.org/) chart. The PHP side is
a thin shell; the data is fed from an endpoint the component calls.

```php
<?php

declare(strict_types = 1);

namespace Vendor\Analytics\Nova\Cards;

use Laravel\Nova\Card;

class UserActivityChart extends Card
{
    /**
     * The width of the card: '1/3', '1/2', or 'full'.
     */
    public $width = 'full';

    /**
     * The Vue component that renders the eCharts chart.
     */
    public function component(): string
    {
        return 'user-activity-chart';
    }

    public function uriKey(): string
    {
        return 'user-activity-chart';
    }
}
```

## CSV & Excel export

### Built-in CSV export

Attach Nova's `ExportAsCsv` action to a resource's `actions()`:

```php
use Laravel\Nova\Actions\ExportAsCsv;

public function actions(NovaRequest $request): array
{
    return [
        ExportAsCsv::make(),
    ];
}
```

### Excel (XLSX) export via `maatwebsite/excel`

```php
<?php

declare(strict_types = 1);

namespace Vendor\User\Nova\Actions;

use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;
use Laravel\Nova\Http\Requests\NovaRequest;
use Maatwebsite\Excel\Facades\Excel;
use Vendor\User\Exports\UsersExport;

class ExportUsers extends Action
{
    public $name;

    public function __construct()
    {
        $this->name = __('user::actions.export_users.name');
    }

    public function handle(ActionFields $fields, Collection $models): array
    {
        $filename = 'users_' . now()->format('Y-m-d_His') . '.xlsx';

        return Action::download(
            Excel::download(new UsersExport($models), $filename)->getFile(),
            $filename,
        );
    }

    public function fields(NovaRequest $request): array
    {
        return [];
    }
}
```
