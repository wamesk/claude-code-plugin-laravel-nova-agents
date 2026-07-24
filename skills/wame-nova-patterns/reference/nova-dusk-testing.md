# Nova Dusk Browser Testing

Nova renders in the browser, so its resources, actions and filters are exercised
with **Laravel Dusk** driven from Pest. Target elements with Nova's `@`-prefixed
`dusk` selectors, assert against translated UI text via `__()`, and verify the
database state afterwards.

`Vendor` is a placeholder for the vendor namespace root, defined per-project in
`CLAUDE.md`.

## Resource CRUD

```php
<?php

use Laravel\Dusk\Browser;
use Vendor\User\Models\User;

test('admin can view users in nova', function () {
    $admin = User::factory()->create(['role' => 'admin']);

    $this->browse(function (Browser $browser) use ($admin) {
        $browser->loginAs($admin)
            ->visit('/nova/resources/users')
            ->assertSee(__('user::user.plural'))
            ->assertPresent('@create-button');
    });
});

test('admin can create user via nova', function () {
    $admin = User::factory()->create(['role' => 'admin']);

    $this->browse(function (Browser $browser) use ($admin) {
        $browser->loginAs($admin)
            ->visit('/nova/resources/users')
            ->click('@create-button')
            ->type('@name', 'New User')
            ->type('@email', 'newuser@example.com')
            ->type('@password', 'password123')
            ->type('@password_confirmation', 'password123')
            ->click('@create-and-add-another-button')
            ->waitForText(__('user::user.singular') . ': New User');
    });

    $this->assertDatabaseHas('users', [
        'email' => 'newuser@example.com',
    ]);
});

test('admin can update user via nova', function () {
    $admin = User::factory()->create(['role' => 'admin']);
    $user = User::factory()->create(['name' => 'Old Name']);

    $this->browse(function (Browser $browser) use ($admin, $user) {
        $browser->loginAs($admin)
            ->visit("/nova/resources/users/{$user->id}/edit")
            ->clear('@name')
            ->type('@name', 'Updated Name')
            ->click('@update-button')
            ->waitForText(__('user::user.singular') . ': Updated Name');
    });

    $this->assertDatabaseHas('users', [
        'id' => $user->id,
        'name' => 'Updated Name',
    ]);
});

test('admin can delete user via nova', function () {
    $admin = User::factory()->create(['role' => 'admin']);
    $user = User::factory()->create();

    $this->browse(function (Browser $browser) use ($admin, $user) {
        $browser->loginAs($admin)
            ->visit("/nova/resources/users/{$user->id}")
            ->click('@open-delete-modal-button')
            ->elsewhere(function ($browser) {
                $browser->click('@confirm-delete-button');
            })
            ->waitForText(__('user::user.plural'));
    });

    $this->assertSoftDeleted('users', [
        'id' => $user->id,
    ]);
});
```

## Actions

Select a row, open the action menu, fill any action fields, run it, then assert
both the success message and the resulting database state.

```php
<?php

use Laravel\Dusk\Browser;
use Vendor\Post\Models\Post;
use Vendor\User\Models\User;

test('admin can execute an action on posts', function () {
    $admin = User::factory()->create(['role' => 'admin']);
    $post = Post::factory()->create(['status' => 'draft']);

    $this->browse(function (Browser $browser) use ($admin, $post) {
        $browser->loginAs($admin)
            ->visit('/nova/resources/posts')
            ->click("@{$post->id}-checkbox")
            ->click('@action-select')
            ->click('@action-update-post-status')
            ->select('@status', 'published')
            ->click('@run-action-button')
            ->waitForText(__('post::actions.update_status.success', ['count' => 1]));
    });

    $this->assertDatabaseHas('posts', [
        'id' => $post->id,
        'status' => 'published',
    ]);
});
```

## Filters

```php
<?php

use Laravel\Dusk\Browser;
use Vendor\Post\Models\Post;
use Vendor\User\Models\User;

test('admin can filter posts by status', function () {
    $admin = User::factory()->create(['role' => 'admin']);
    Post::factory()->create(['title' => 'Live Post', 'status' => 'published']);
    Post::factory()->create(['title' => 'Hidden Post', 'status' => 'draft']);

    $this->browse(function (Browser $browser) use ($admin) {
        $browser->loginAs($admin)
            ->visit('/nova/resources/posts')
            ->click('@filter-selector')
            ->select('@filter-status-select', 'published')
            ->pause(1000) // wait for the filter to apply
            ->assertSee('Live Post')
            ->assertDontSee('Hidden Post');
    });
});
```

## Reusable Nova test helper

Extract the repetitive login/navigation steps into a trait and `use` it from your
test files.

```php
<?php

declare(strict_types = 1);

namespace Tests\Helpers;

use Laravel\Dusk\Browser;

trait NovaTestHelper
{
    /**
     * Log in and land on the main Nova dashboard.
     */
    protected function novaLogin(Browser $browser, $user): void
    {
        $browser->loginAs($user)
            ->visit('/nova')
            ->waitForLocation('/nova/dashboards/main');
    }

    /**
     * Navigate to a resource index and wait for the table.
     */
    protected function visitNovaResource(Browser $browser, string $resource): void
    {
        $browser->visit("/nova/resources/{$resource}")
            ->waitFor('.nova-resource-table');
    }

    /**
     * Fill a Nova form field by its dusk selector.
     */
    protected function fillNovaField(Browser $browser, string $field, $value): void
    {
        $browser->type("@{$field}", $value);
    }
}
```

## Running the tests

```bash
# All tests (Pest)
php artisan pest

# A single Dusk browser test file, by filter
php artisan dusk --filter UserResourceTest

# All Dusk browser tests
php artisan dusk
```

## Testing checklist

- Drive Nova through Laravel Dusk; never assert Nova internals directly.
- Cover create / read / update / delete for each resource.
- Cover every custom action with its full user interaction.
- Target elements with `@`-prefixed `dusk` selectors.
- Assert on translated text via `__()`, not hardcoded strings.
- Verify the database state after each mutating operation.
- Build test data with factories; keep tests isolated from one another.
