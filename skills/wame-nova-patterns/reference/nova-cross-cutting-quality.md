# Nova Cross-cutting Quality — Reachability, Security, Performance, UI/UX, Framework

The four dimensions QA checks on every task's diff, applied **while building**
Nova screens, plus a fifth — `framework`: write the idioms of the Nova version the
project has installed, not those of the other major. The keys are the ones the
`teamwork-task-test` plugin uses in its review step — `reachability`, `security`,
`performance`, `ui_ux`, `framework` — so a QA finding names the same rule you were
meant to apply here (QA treats `framework` as advisory; here it is a rule for the
code you write or change). For non-Nova screens (Blade, Livewire, Inertia, SPA)
the same rules live in the `wame-laravel-standards` skill (`laravel-agents`
plugin), `reference/cross-cutting-quality.md`.

Behaviour described below was checked against the Laravel Nova 5.x source; where
Nova 4 differs, the `framework` section says so. `Vendor` is a placeholder for the
vendor namespace root, defined per-project in `CLAUDE.md`.

## `reachability` — every new screen is reachable by clicking

**Applies when** the change adds, renames, or removes a Resource, Lens,
Dashboard, or Tool. A screen that is registered, authorized, and loads, but that
no menu item and no relation tab points at, is found only by typing its URL.

### How Nova builds the sidebar

- **No custom menu:** Nova builds the sidebar from each tool's `menu()`. The
  built-in resource manager lists every resource whose `$displayInNavigation` is
  true *and* whose policy allows `viewAny`, grouped by `$group`; dashboards come
  from the dashboard tool. A new resource shows up on its own.
- **Custom `Nova::mainMenu()`:** the callback receives the default menu as its
  second argument, but most projects ignore it and return their own list. Then
  **a resource, dashboard, or tool that is not added to that list is invisible
  even though it is registered** — it stays reachable only by URL. Always check
  first: `grep -rn 'Nova::mainMenu' app wamesk --include='*.php'` (no
  `wamesk/*/src` glob — in zsh an unmatched glob aborts the command before grep
  runs, which reads exactly like "no custom menu").

### Option A — a menu entry

```php
use Illuminate\Http\Request;
use Laravel\Nova\Menu\MenuGroup;
use Laravel\Nova\Menu\MenuItem;
use Laravel\Nova\Menu\MenuSection;
use Laravel\Nova\Nova;
use Vendor\Invoice\Nova\CreditNote;
use Vendor\Invoice\Nova\Dashboards\Billing;
use Vendor\Invoice\Nova\Invoice;
use Vendor\Invoice\Nova\Lenses\OverdueInvoices;

Nova::mainMenu(fn (Request $request) => [
    // Billing overrides authorizedToSee() itself — see the fresh-instance rule below.
    MenuSection::dashboard(Billing::class)->icon('chart-bar'),

    MenuSection::make(__('invoice::menu.section.billing'), [
        MenuGroup::make(__('invoice::menu.group.documents'), [
            MenuItem::resource(Invoice::class),
            MenuItem::resource(CreditNote::class),
        ]),

        // MenuItem::lens() checks only the lens's own authorizedToSee(), but the
        // lens route answers 403 without the resource's viewAny — mirror both.
        MenuItem::lens(Invoice::class, OverdueInvoices::class)
            ->canSee(fn (Request $request) => Invoice::authorizedToViewAny($request)
                && (new OverdueInvoices)->authorizedToSee($request)),
    ])->icon('document-text')->collapsable(),
]);
```

Rules:

- Use `MenuItem::resource()` / `MenuSection::resource()` for resources. They
  hide the entry when `$displayInNavigation` is false **or** the policy denies
  `viewAny`, so menu visibility follows the policy for free. A hand-written
  `MenuItem::link('/resources/invoices')` drifts from the policy.
- `MenuItem::lens()`, `MenuItem::dashboard()`, and `MenuSection::dashboard()`
  build a **fresh instance** of the class. A `canSee()` attached where the lens is
  listed in `lenses()`, or where the dashboard is registered in `dashboards()`, is
  not applied to the menu item — but the route enforces it. Result: an entry that
  is visible but answers 403/404. Put the rule inside the class (override
  `authorizedToSee()`) or repeat it on the menu item, as above.
- A **tool** added to a custom menu must be gated with the same `canSee()` as its
  registration in `tools()` — the tool's `menu()` section carries no gate of its
  own, so without it the entry shows to users the tool route refuses.
- A **lens** listed in the resource's `lenses()` is already reachable from the
  lens selector on that resource's index; a menu entry is optional.
- Menu labels go through `__()` with English keys, like every other string.

### Option B — a child reachable through its parent's relation tab

A child resource that makes no sense on its own (invoice lines, price rows) may
hide from navigation — **only together with** a relation field on the parent,
which makes it reachable through the parent's relation panel:

```php
class InvoiceLine extends Resource
{
    // Hidden from the sidebar; reachable through Invoice → lines.
    public static $displayInNavigation = false;
}

// Invoice::fields()
HasMany::make(__('invoice::invoice.field.lines.label'), 'lines', InvoiceLine::class),
```

Pass the child resource class explicitly — QA's reachability check finds the
link by grepping for `InvoiceLine::class` on another screen. `$displayInNavigation
= false` also hides `MenuItem::resource()` in a custom menu, so a hidden resource
with no relation field on any parent is an orphan.

### Inbound links ("prekliky") from related screens

- **Back-link to the parent:** a `BelongsTo` field on the child renders a link to
  the parent's detail view. Peeking is on by default and previews the parent's
  fields marked `showWhenPeeking()`; `->noPeeking()` turns it off.
- **Parent → children:** `HasMany` / `BelongsToMany` / `MorphMany` on the parent
  (Option B).
- **Action that jumps to a related screen:** `Action::visit()` with a closure
  path receives the model only when the action is `->sole()`; the path is
  relative to the Nova root.

  ```php
  // Invoice::actions()
  Action::visit(
      __('invoice::actions.open_customer.name'),
      fn ($invoice) => "/resources/customers/{$invoice->customer_id}",
  )->sole(),
  ```

- **Link on a detail view** to a screen with no relation between them: a `URL`
  field whose value is the target path and whose text is translated.

### Renames and removals

Renaming a resource changes its `uriKey()` and every `/resources/<key>` path.
Grep the old class name and the old URI key across `app/`, `wamesk/`,
`resources/`, and the lang files, and update every `MenuItem`, `Action::visit`,
`URL` field, and hardcoded path in the same change — a stale link is a 404 the
user meets by clicking. A deliberately URL-only screen (e.g. linked only from an
e-mail) is named as such and belongs in `teamwork-task-test`'s `allow_orphans`.

## `security` — policies, object scope, tenant scope

**Applies to** every new Resource, Lens, Action, Filter, and relation field.

- **Every resource model has a registered policy.** Without one, Nova authorizes
  *everything* for every Nova user. A policy that exists but has no `viewAny()`
  method allows listing to all; `view` / `update` / `delete` compare the record
  (tenant, owner), not only the role.
- **Relations have their own policy methods:** `add{Model}` (HasMany create
  button), `attach{Model}` / `attachAny{Model}` / `detach{Model}`
  (BelongsToMany).
- **Tenant scope is applied everywhere or nowhere.** `indexQuery()`,
  `detailQuery()`, `editQuery()`, `relatableQuery()`, and `scoutQuery()` all
  return the query unchanged by default, and detail, peek, and preview go through
  `detailQuery()`, not `indexQuery()`. A scope placed only in `indexQuery()`
  leaves `/nova/resources/invoices/<foreign-id>` open and lists other tenants'
  rows in `BelongsTo` dropdowns. A lens builds its own `query()`. Prefer a model
  global scope, which covers all of these plus actions and queues. Otherwise
  override every hook, and still check the record in the policy's `view` /
  `update`.
- **Actions:** `canSee()` for the role, `canRun(fn ($request, $model) => $request->user()->can('update', $model))`
  for the record. `runAction` / `runDestructiveAction` policy methods gate all
  actions of a resource centrally.
- **Sensitive fields:** `->canSee(fn ($request) => $request->user()->can(…))`.
- **Validation:** `->rules()` / `->creationRules()` / `->updateRules()` on every
  editable field; a custom `fillUsing()` must not write columns the field does
  not own.
- **Handled failures, no leaks.** A known failure in an action's `handle()`
  returns `Action::danger(__('…'))` instead of throwing — with `APP_DEBUG=false` a
  thrown message reaches the user only as a generic error and floods the error
  tracker. Messages never carry paths, SQL, or other tenants' IDs; no secrets in
  code, logs, or reports.
- **Hiding is not access control.** `$displayInNavigation = false`, a missing
  menu entry, or a hidden action do not stop a request to the route — only the
  policy does.

Prove it without a browser through the Nova API:

```php
test('a user of another company cannot open the invoice', function () {
    $user = User::factory()->create();          // different company
    $invoice = Invoice::factory()->create();

    $this->actingAs($user)
        ->getJson("/nova-api/invoices/{$invoice->getKey()}")
        ->assertForbidden(); // or assertNotFound() when a global scope hides it
});
```

## `performance` — lists, search, per-row work

**Applies to** every new index, lens, relation panel, metric, and action.

- Eager-load what each row touches: `public static $with = ['customer'];` on the
  resource, or `indexQuery()` / `detailQuery()` for the rest (see
  `nova-resources.md`). Counts come from `withCount()` in `indexQuery()` — never
  from a field closure such as `fn () => $this->lines()->count()`, which runs one
  query per row.
- **No per-row queries in fields**, badges, or `displayUsing()` closures.
- **Search:** Nova's default search runs `LIKE '%term%'` on every column in
  `searchableColumns()`, and a leading wildcard cannot use a B-tree index. Keep
  the list short. Use `Column::exact('number')` (an `=` match, uses the index)
  for codes, numbers, and e-mails — Nova 5 only; Nova 4 has no
  `Column::exact()` (see `framework`). Use `new SearchableText('title')` (a
  full-text match, needs `$table->fullText('title')`) for prose; use Scout for
  large tables. On ULID keys, a plain `'id'` entry becomes `id LIKE '%…%'` — use
  `Column::exact('id')` on Nova 5, or drop it. Both classes live in
  `Laravel\Nova\Query\Search\`.
- Index the columns that filters and `sortable()` fields use, and every new
  foreign key; check existing indexes first so you do not duplicate one.
- `BelongsTo` to a large table gets `->searchable()`; otherwise the dropdown
  loads every row.
- Actions over many rows: slow work implements `ShouldQueue`; `$chunkCount`
  (default 200) sets the batch size.
- A metric with an expensive query overrides `cacheFor()` to cache its result.
- Name the row count at which a pattern starts to hurt.

## `ui_ux` — labels, help, states, destructive actions

**Applies to** every field, action, filter, lens, metric, and card.

- Labels, help texts, tab names, action names and messages, menu labels: all
  through `__()` with **English keys** in the module's own lang file (see the
  translation-key pattern in `nova-resources.md`). Every field has `help()`.
- A new key must resolve: `__('invoice::invoice.field.lines.help')` must not
  return the key itself. Never make a key both a string and a prefix — do not add
  `field.lines` as a string next to `field.lines.label`; put standalone hints
  under `hint.<thing>`.
- **Disabled actions say why.** When `canRun()` returns false, Nova greys the
  action out in the dropdown and gives no reason. Hide an action the user can
  never run (`canSee()`). For a state-dependent restriction ("already sent"),
  let the action run and return `Action::danger(__('invoice::actions.send.error.already_sent'))`
  from `handle()`, or state the precondition in `confirmText()`. A bulk run on
  mixed rows skips the ineligible rows, so report the skipped count in the
  message.
- **Buttons in custom cards, tools, and HTML fields** follow the web rule: a real
  `disabled` or `aria-disabled="true"` **plus** a translated `title` /
  `aria-describedby` giving the reason.
- **Destructive actions confirm.** Extend `DestructiveAction` (red confirm
  button, `runDestructiveAction` policy), set `->confirmText()`,
  `->confirmButtonText()`, and `->cancelButtonText()` with translated strings,
  and never call `->withoutConfirmation()` on them.
- Return `Action::message()` / `Action::danger()` so the user sees the outcome.
- Organize fields into panels/tabs like the sibling resources; empty relation
  panels and lenses show Nova's empty state — do not hide them behind a custom
  card that renders nothing.

## `framework` — the installed Nova version's idioms

**Applies to** every new or changed resource, field, action, filter, lens,
metric, menu entry, and custom tool, card, or field — the new and changed lines
only. The Laravel, PHP, and Pest side of the rule (detection, docs lookup,
minimum versions, guardrails) is `framework` in `wame-laravel-standards` →
`reference/cross-cutting-quality.md` (`laravel-agents`); this section adds the
Nova part.

The examples in this skill target Nova 5, but Nova 4 projects are still
maintained. Code carried across the major boundary does not degrade gracefully:
a Nova 5-style override is a fatal error when the class loads on Nova 4, and a
Nova 5-only method is an undefined-method error on the first request.

### 1. Detect the Nova version and the add-ons

```bash
jq -r '[.packages[], (."packages-dev" // [])[]]
  | map(select(.name | test("^laravel/(nova|framework)$"))) | .[] | "\(.name) \(.version)"' composer.lock

# Add-on packages the project uses instead of (or next to) native features
jq -r '.packages[].name | select(test("nova-(tabs|dependency-container|sortable|multiselect|menu)|nova-flexible-content"))' composer.lock
```

### 2. Look it up for that major

1. **Laravel Boost `search-docs`** with `packages: ["laravel/nova"]` — it
   searches the docs of the installed version.
2. **context7** — Nova is indexed per major (at the time of writing
   `/websites/nova_laravel_v4` and `/websites/nova_laravel_v5`).
3. **Official docs** — `https://nova.laravel.com/docs/v4/…` or `…/v5/…`; for the
   differences between the majors, `https://nova.laravel.com/docs/v5/upgrade`
   and `…/v5/releases`.
4. **`vendor/laravel/nova/src`** is the final word, and read-only. The docs do
   not cover everything — `Column::exact()` is in the 5.x source but not in the
   search docs.

Generate new classes with the installed Nova's own generator — `php artisan
nova:resource`, `nova:action`, `nova:filter`, `nova:lens`, `nova:repeatable`, and
so on. It prefers the project's published `stubs/nova/*.stub` (`php artisan
nova:stubs`), so the result follows the project's conventions; after a major
upgrade the v5 upgrade guide has them re-published with `nova:stubs --force`.

### 3. Nova 4 vs Nova 5 — what changes when you write new code

Checked against the Nova 4.29.5 and 4.35.11 sources, the Nova 5.6.3–5.10.1
sources, and the v5 upgrade guide and release notes.

| Topic | Nova 4 | Nova 5 |
|-------|--------|--------|
| Platform | `php: ^7.3\|^8.0`; authentication via `laravel/ui` | `php: ^8.1`; authentication via Laravel Fortify. Which Laravel majors a Nova minor admits is in its `composer.json` (5.10.1 admits Laravel 13, 5.6.3 does not) |
| Overriding query hooks and filters | The parent's `$query` is untyped — `indexQuery(NovaRequest $request, $query)`, `Filter::apply(NovaRequest $request, $query, $value)`. Keep `$query` untyped in the override; a `: Builder` return type is fine. A typed `Builder $query` is a fatal `Declaration … must be compatible` when the class loads (verified on 4.35.11); WAME's Nova 4 projects write `indexQuery(NovaRequest $request, $query): Builder` | The parents are typed — `indexQuery(NovaRequest $request, Builder $query)`, `Filter::apply(NovaRequest $request, Builder $query, mixed $value)` — and typed and untyped overrides both load. The `BaseResource` in `nova-resources.md` uses the typed form |
| Tabs | No native tabs: `Panel`, or the tabs package the project already has (`eminiarts/nova-tabs` and `shuvroroy/nova-tabs` appear in WAME projects) | Native `Tab::group()` / `Tab::make()` (`Laravel\Nova\Tabs\Tab`) |
| Exact-match search | No `Column::exact()` — calling it is an undefined-method error. The key column is matched with `=` only for an integer key and an all-digit term; everything else is `LIKE` | `Column::exact('number')` |
| Enum options | `Select::options()` takes an array or a callable — map `Status::cases()` yourself | `Select::options(Status::class)` accepts a backed enum class |
| Nova 5 only | — | `->immutable()` fields, `->searchable()` select filters, a Nova-only policy through `public static $policy` on the resource plus `php artisan nova:policy` |
| Removed in Nova 5 | The `Place` field still exists (Algolia Places was retired in 2022) | No `Place` field |
| Custom tools, cards, and fields (JS) | Vue 3.2, `@inertiajs/inertia-vue3` 0.6, `Errors` from `form-backend-validation` | Vue 3.5, `@inertiajs/vue3` 2.x, `Errors` from `laravel-nova` |
| Menu icons (`->icon()`) | Heroicons v1 names only (`collection`, `database`, …) | Heroicons v2 names (`rectangle-stack`, `circle-stack`, …); Nova 5 maps the v1 names onto v2 |

### 4. Nova idioms — the built-in feature instead of hand-rolled code or an add-on

The version is the minimum that has it: `4.0` and `5.0` are named in the Nova 4 /
Nova 5 release notes; `4.x` is present in the oldest install checked (4.29.5),
and the minor that introduced it is not documented.

| Min | Use | Instead of |
|-----|-----|-----------|
| 4.0 | `Nova::mainMenu()` with `MenuSection` / `MenuGroup` / `MenuItem::resource()` (see `reachability`) | Hand-written `MenuItem::link('/resources/…')` entries |
| 4.0 | `dependsOn()` (and `dependsOnCreating()` / `dependsOnUpdating()`), switching `->show()` / `->hide()` / `->readonly()` / `->rules()` inside the callback | A dependency-container package for a new conditional field — unless the same form already uses one. Fields inside a `Repeater` do not support `dependsOn()` |
| 4.0 | `->filterable()` on a field | A hand-written `Filter` class for a plain column filter |
| 4.0 | `SearchableText` / `SearchableRelation` / `SearchableJson` in `searchableColumns()` | Overriding the search query by hand |
| 4.x | `Badge::make()->map([...])` (with `->addTypes()` / `->withIcons()`), `Status::make()->loadingWhen([...])->failedWhen([...])` | Coloured HTML in a `Text` field with `asHtml()` |
| 4.x | `Text::make()->copyable()` (also on `URL`) | A custom copy button |
| 4.x | `Repeater::make()->repeatables([...])` with `->asJson()` or `->asHasMany()` — marked beta in the v4 docs, no longer in v5 | A flexible-content package for a simple repeatable list |
| 4.x | Inline actions: `Action::using()`, `Action::visit()`, `Action::danger()` | A full action class for a one-line jump or message |
| 4.x | `->withoutConfirmation()` — only deliberately, and never on a `DestructiveAction` (see `ui_ux`) | — |
| 5.0 | `Tab::group()` / `Tab::make()` | A tabs package — for a new resource in a project with no established tabs package |
| 5.0 | `->immutable()` | `->readonly()`, when the value must still be submitted |
| 5.0 | `Select::options(Status::class)` | Mapping `Status::cases()` by hand |

### 5. Guardrails

The guardrails of the Laravel `framework` section apply: the project's
`CLAUDE.md` and sibling conventions win, no drive-by rewrites, nothing deprecated
or newer than installed, no new dependency for a built-in capability. For Nova in
particular:

- **Match the sibling resources.** If the project's resources use
  `eminiarts/nova-tabs` or a dependency container, a new resource uses them too.
  Switching to native tabs or `dependsOn()` is a refactor task that migrates them
  all, not a side effect of a feature.
- **Never carry code across the major boundary.** On Nova 4: no typed
  `Builder $query` overrides, no `Column::exact()`, `Tab`, `->immutable()`, or
  enum class in `Select::options()`. On Nova 5: no `Place` field, and no
  `@inertiajs/inertia-vue3` or `form-backend-validation` imports in custom
  components.
- An add-on the task does not touch stays; name the native replacement in the
  summary instead.

## Proving reachability with Dusk

A Dusk test for a new screen starts at `/nova` and reaches the screen by
clicking — through `@sidebar-menu` or through the parent's relation panel — and
checks that a denied user sees no entry and gets the 403 page. See
[`nova-dusk-testing.md`](nova-dusk-testing.md) → *Reachability — navigate via
the menu*.

## Pre-finish self-check (Nova)

- **reachability** — New resource/lens/dashboard/tool is in the custom
  `mainMenu()` if the project has one, or reachable through a relation field on
  its parent (`$displayInNavigation = false` only with such a field). Lens and
  dashboard menu items mirror the route's authorization. Renamed URI keys left no
  stale links.
- **security** — Policy registered, with `viewAny` and object-scoped
  `view`/`update`/`delete`. Tenant scope covers index, detail, edit, relatable,
  scout, and lenses (or is a global scope). Actions use `canRun` per record.
- **performance** — `$with` / `indexQuery()` eager-load what rows touch; no
  per-row queries in fields; search columns are exact, full-text, or Scout; new
  foreign keys and filter/sort columns are indexed.
- **ui_ux** — Every label, help, action text, and menu label is `__()` with an
  English key that resolves. Destructive actions extend `DestructiveAction` and
  confirm. Unavailable actions are hidden or explain themselves.
- **framework** — The Nova version and its add-ons were read from
  `composer.lock`. Nothing from the other major (on Nova 4: no typed
  `Builder $query` overrides, `Column::exact()`, `Tab`, `->immutable()`, or enum
  class in `Select::options()`). The built-in idiom is used where the installed
  version has it and the sibling resources do not establish another pattern;
  untouched resources were left alone, with opportunities named in the summary.
