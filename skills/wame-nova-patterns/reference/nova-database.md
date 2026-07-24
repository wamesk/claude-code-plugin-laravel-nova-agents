# Nova-related Database Tables

Two schema patterns that recur in Nova admin panels: drag-sortable resource rows
and the notification table Nova ships with.

**Datetime standard:** timestamp columns (`created_at` / `updated_at` /
`deleted_at`) use `dateTimeTz()` — never `timestamps()`, `softDeletes()`, or plain
`datetime()`.

**Data & defaults:** ship rows through idempotent `*_seed_*` **migrations** (whose
`up()` inserts a row only when it is absent), not through database seeders.
Factories stay for tests only. Static lookup data (ISO country/currency codes and
similar) can be served from the model via
[Sushi](https://github.com/calebporzio/sushi) instead of a table.

## Drag-sortable rows (`outl1ne/nova-sortable`)

Reorder resource rows by drag-and-drop with
[`outl1ne/nova-sortable`](https://novapackages.com/packages/outl1ne/nova-sortable).

**Migration** — store the position in a `sort` column. Use `sort`, not `order`
(`ORDER` is a reserved word in MySQL):

```php
Schema::create('menu_items', function (Blueprint $table) {
    $table->ulid('id')->primary();
    $table->string('name', 100);
    $table->unsignedTinyInteger('sort')->default(0); // NOT 'order'
    $table->dateTimeTz('created_at');
    $table->dateTimeTz('updated_at');

    $table->index('sort');
});
```

**Model** — add the trait and point it at the `sort` column:

```php
use Illuminate\Database\Eloquent\Model;
use Outl1ne\NovaSortable\Traits\HasSortableRows;

class MenuItem extends Model
{
    use HasSortableRows;

    public static $sortable = [
        'order_column_name' => 'sort',
        'sort_when_creating' => true,
    ];
}
```

## `nova_notifications` table

Nova's notification center reads from a `nova_notifications` table. It uses a
`bigInteger` auto-increment id (not a ULID) because the underlying Laravel
notifications contract expects it. The datetime columns still use `dateTimeTz()`:

```php
Schema::create('nova_notifications', function (Blueprint $table) {
    $table->id(); // bigInteger auto-increment — required by the package
    $table->string('type');
    $table->morphs('notifiable');
    $table->text('data');
    $table->dateTimeTz('read_at')->nullable();
    $table->dateTimeTz('created_at')->nullable();

    $table->index(['notifiable_type', 'notifiable_id']);
});
```
