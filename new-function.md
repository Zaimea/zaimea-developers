---
title: New function Pattern
description: How can implement a new function on our package
github: https://github.com/zaimea/developers-docs/edit/main/
onThisArticle: true
sidebar: true
rightbar: true
---

# New function Pattern

[[TOC]]

## Introduction
Here we will briefly show the complete process based on a case implemented by us.
Of course, it may vary depending on the needs of what we want to implement.


### Backend

## Make the Eloquent Model

```bash 
php artisan make:model Colors -m
```

```php 
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Zaimea\Groups\Fabric\Configuration\Eloquent;

class Color extends Model
{
    use HasFactory;

    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'group_colors';

    /**
     * The attributes that are mass assignable.
     *
     * @var array<int, string>
     */
    protected $fillable = [
        'id',
        'group_id',
        'name',
        'code',
    ];

    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'created_at' => 'datetime',
            'updated_at' => 'datetime',
        ];
    }

    /**
     * Get the group that the monthly quotas belongs to.
     */
    public function group(): BelongsTo
    {
        return $this->belongsTo(Eloquent::groupModel());
    }
}
```

We will also work with an Enum class for the default variants defined by the system.
It looks like(there is nothing complicated or private, that's why the code is shown):

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Calendar\Enums;

use Zaimea\Groups\Fabric\Traits\EnumExtender;

/*
Usage: 
    RecordColor::red->label();
Or
    RecordColor::getLabel(RecordColor::red);
Or
    RecordColor::red;
*/
enum RecordColor : string
{
    use EnumExtender;

    case red = 'bg-red-800 dark:bg-red-500';
    case orange = 'bg-orange-800 dark:bg-orange-500';
    case yellow = 'bg-yellow-600 dark:bg-yellow-400';
    case lime = 'bg-lime-800 dark:bg-lime-400';
    case green = 'bg-green-800 dark:bg-green-400';
    case cyan = 'bg-cyan-800 dark:bg-cyan-400';
    case blue = 'bg-blue-800 dark:bg-blue-400';
    case indigo = 'bg-indigo-800 dark:bg-indigo-400';
    case purple = 'bg-purple-800 dark:bg-purple-400';
    case fuchsia = 'bg-fuchsia-800 dark:bg-fuchsia-400';
    case rose = 'bg-rose-800 dark:bg-rose-400';
    case slate = 'bg-slate-800 dark:bg-slate-400';
    case stone = 'bg-stone-800 dark:bg-stone-400';
}
```

`EnumExtender` let us take more elegant the values.

And database table looks like:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('group_colors', function (Blueprint $table) {
            $table->id();
            $table->foreignId('group_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->string('code');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('group_colors');
    }
};
```

Will need to set a permission for who would have access to this configuration
In `GroupRolePermissionsTableSeeder.php` use what you see already written.
Will call the permission: `group:color:*`.
Attach it to a Role in `GroupRolesTableSeeder.php`
And it will be like a Plugin used, that mean we need to fill also `PluginsTableSeeder.php` use what you see already written.


## Make the Controller

```bash
php artisan make:controller ColorsController
```

And create the routs for Colors as CRUD + read all.
and a route to enable view in web patch
```php
Route::get('/group/{group}/colors', [ColorsController::class, 'show'])->name('group.color');
```

## In ZaimeaServiceProvider.php

Need to make the singletons
```php
// About Locking...
$this->app->singleton(\Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorCreatedResponse::class, \Zaimea\Groups\Http\Responses\GroupColorCreatedResponse::class);
$this->app->singleton(\Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorUpdatedResponse::class, \Zaimea\Groups\Http\Responses\GroupColorUpdatedResponse::class);
$this->app->singleton(\Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorDeletedResponse::class, \Zaimea\Groups\Http\Responses\GroupColorDeletedResponse::class);
```

In `Fabric / Zaimea.php` need to register the views, this must be defined in the desired function according to the needs.
We will use function `viewGroupPrefix()` and use what you see already written.

In `Fabric / Configuration / RegisterViews.php `

```php
/**
 * Specify which view should be used as the view for group colors.
 *
 * @param  callable|string  $view
 * @return void
 */
public static function groupColorsView(callable|string $view): void
{
    App::singleton(\Zaimea\Groups\Group\Contracts\Colors\Views\GroupColorsViewResponse::class, function () use ($view) {
        return new SimpleViewResponse($view);
    });
}
```

## Actions

In Actions folder, we will make a specific folder in which we will create 3 files: CreateColor.php, UpdateColor.php, DeleteColor.php.
CreateColor.php will look like:

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Actions\Color;

use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Validator;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Models\Group;
use Zaimea\Groups\Group\Models\Color;

class CreateColor
{
    /**
     * Create Color for the given group.
     *
     * @param  mixed  $group
     * @param  array  $form
     * @return void
     */
    public function create(Group $group, array $form): void
    {
        abort_if(Gate::denies('color', $group), Response::HTTP_FORBIDDEN, "You don't have permission to create.");

        Validator::make([
            'name' => $form['name'],
            'code' => $form['code'],
        ],
        [
            'name' => ['required', 'string'],
            'code' => ['required', 'string'],
        ])->validateWithBag('create');

        $form['group_id'] = $group->id;
        Color::create($form);
        $group->load('colors');
    }
}
```

And DeleteColor.php

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Actions\Color;

use Illuminate\Support\Facades\Gate;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Models\Group;

class DeleteColor
{
    /**
     * Delete the given Color.
     *
     * @param  mixed  $group
     * @param  int    $colorId
     * @return void
     */
    public function delete(Group $group, int $colorId): void
    {
        abort_if(Gate::denies('color', $group), Response::HTTP_FORBIDDEN, "You don't have permission for delete.");

        $group->colors()
            ->where('id', $colorId)
            ->first()
            ->delete();

        $group->load('colors');
    }
}
```

And UpdateColor.php

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Actions\Color;

use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Validator;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Models\Group;
use Zaimea\Groups\Group\Models\Color;

class UpdateColor
{
    /**
     * Create Color for the given group.
     *
     * @param  mixed  $group
     * @param  mixed  $color
     * @param  array  $form
     * @return void
     */
    public function update(Group $group, Color $color, array $form): void
    {
        abort_if(Gate::denies('color', [$group, $color]), Response::HTTP_FORBIDDEN, "You don't have permission.");

        Validator::make([
            'name' => $form['name'],
            'code' => $form['code'],
        ],
        [
            'name' => ['required', 'string'],
            'code' => ['required', 'string'],
        ])->validateWithBag('update');

        $color->update($form);
        $group->load('colors');
    }
}
```

## Contracts

Facem un folder cu numele Colors care va contine alte 3 foldere: Actions, Responses, Views
Nu avem nici o Actiune dar vom creia un .gitkeep in folderul Actions
In folderule Responses avem nevoie de:

`GroupColorCreatedResponse.php` ce va contine:
```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Contracts\Colors\Responses;

use Illuminate\Contracts\Support\Responsable;

/**
 * @method \Symfony\Component\HttpFoundation\Response toResponse(\Illuminate\Http\Request $request)
 */
interface GroupColorCreatedResponse extends Responsable
{
    //
}
```

`GroupColorDeletedResponse` si `GroupColorUpdatedResponse.php` vor arata ca `GroupColorCreatedResponse.php`

In folderul Views vom avea `GroupColorsViewResponse.php`
```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Group\Contracts\Colors\Views;

use Illuminate\Contracts\Support\Responsable;

interface GroupColorsViewResponse extends Responsable
{
    /**
     * Specify the parameters that should be passed to the view.
     *
     * @param  array  $parameters
     * @return $this
     */
    public function withParameters($parameters = []);
}
```

In Group Model vom atasa:
```php
/**
 * Colors associated with the group.
 */
public function colors(): HasMany
{
    return $this->hasMany(Color::class);
}
```

In GroupPolicy.php
```php
/**
 * Determine whether the user can create/update the model.
 */
public function color(User $user, Group $group): bool
{
    return $user->hasGroupPermission($group, 'group:color:*');
}
```

## Controllers

Controller va arata in felul urmator:
```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Http\Controllers\Group;

use Illuminate\Contracts\Auth\StatefulGuard;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Routing\Controller;
use Illuminate\Support\Facades\Gate;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Fabric\Configuration\Eloquent;
use Zaimea\Groups\Fabric\Traits\RedirectsActions;
use Zaimea\Groups\Group\Actions\Color\CreateColor;
use Zaimea\Groups\Group\Actions\Color\DeleteColor;
use Zaimea\Groups\Group\Actions\Color\UpdateColor;
use Zaimea\Groups\Group\Contracts\Color\Responses\GroupColorCreatedResponse;
use Zaimea\Groups\Group\Contracts\Color\Responses\GroupColorDeletedResponse;
use Zaimea\Groups\Group\Contracts\Color\Responses\GroupColorUpdatedResponse;
use Zaimea\Groups\Group\Contracts\Color\Views\GroupColorsViewResponse;
use Zaimea\Groups\Group\Models\Group;
use Zaimea\Groups\Http\Resources\GroupColorResource;

class ColorsController extends Controller
{
    use RedirectsActions; 

    /**
     * Create a new controller instance.
     *
     * @param  \Illuminate\Contracts\Auth\StatefulGuard  $guard
     * @return void
     */
    public function __construct(protected StatefulGuard $guard)
    {
        //
    }
        
    /**
     * Show the group colors view.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Zaimea\Groups\Group\Models\Group  $group
     * @return \Zaimea\Groups\Group\Contracts\Colors\Views\GroupColorsViewResponse
     */
    public function show(Request $request, Group $group): GroupColorsViewResponse
    {
        abort_if(! $group->isPluginEnabled('col'), Response::HTTP_FORBIDDEN, 'Colors Plugin is not activated');
        abort_if(Gate::denies('viewGroup', $group), Response::HTTP_FORBIDDEN);

        return app(GroupColorsViewResponse::class)->withParameters([
            'request' => $request,
            'group' => $group,
        ]);
    }

    /**
     * New color store.
     * 
    * @request input('group')
    * @request input('name')
    * @request input('code')
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Zaimea\Groups\Group\Actions\Color\CreateColor $creator
     * @return \Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorCreatedResponse
     */
    public function store(Request $request, CreateColor $creator): GroupColorCreatedResponse
    {
        $group = $request->input('group') ? Eloquent::newGroupModel()->findOrFail($request->input('group')) : $request->user()->currentGroup;

        $creator->create(
            $group,
            $request->only('code', 'name'),
        );

        return app(GroupColorCreatedResponse::class, ['action' => $creator]);
    }

    /**
     * Read group color data.
     *
    * @request input('group')
    * @request integer('colorId')
     * 
     * @param  \Illuminate\Http\Request  $request
     * @return \Zaimea\Groups\Http\Resources\GroupColorResource
     */
    public function read(Request $request): GroupColorResource
    {
        $group = $request->input('group') ? Eloquent::newGroupModel()->findOrFail($request->input('group')) : $request->user()->currentGroup;

        $color = $group->colors()
            ->where('id', $request->integer('colorId'))
            ->firstOrFail();

        abort_if(
            Gate::denies('color', [$group, $request->integer('colorId')]),
            Response::HTTP_FORBIDDEN, "You don't have permission."
        );

        return new GroupColorResource($color);
    }

    /**
     * Read group colors data.
     *
    * @request input('group')
     * 
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Resources\Json\AnonymousResourceCollection
     */
    public function colors(Request $request): AnonymousResourceCollection
    {
        $group = $request->input('group') ? Eloquent::newGroupModel()->findOrFail($request->input('group')) : $request->user()->currentGroup;
        
        abort_if(Gate::denies('color', $group), Response::HTTP_FORBIDDEN, '403 Forbidden');

        return GroupColorResource::collection($group->colors()); 
    }

    /**
     * Update Color data.
     *
    * @request input('group')
    * @request integer('colorId')
    * @request input('name')
    * @request input('code')
     * 
     * @param  \Illuminate\Http\Request  $request
     * @param  \Zaimea\Groups\Group\Actions\Color\UpdateColor $updater
     * @return \Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorUpdatedResponse
     */
    public function update(Request $request, UpdateColor $updater): GroupColorUpdatedResponse
    {
        $group = $request->input('group') ? Eloquent::newGroupModel()->findOrFail($request->input('group')) : $request->user()->currentGroup;

        $updater->update(
            $group,
            $group->colors()->where('id', $request->integer('colorId')),
            [$request->only('name', 'code')],
        );

        return app(GroupColorUpdatedResponse::class, ['action' => $updater]);
    }

    /**
     * Delete the Color.
     *
    * @request input('group')
    * @request integer('colorId')
     * 
     * @param  \Illuminate\Http\Request  $request
     * @param  \Zaimea\Groups\Group\Actions\Color\DeleteColor $deleter
     * @return \Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorDeletedResponse
     */
    public function destroy(Request $request, DeleteColor $deleter): GroupColorDeletedResponse
    {
        $group = $request->input('group') ? Eloquent::newGroupModel()->findOrFail($request->input('group')) : $request->user()->currentGroup;

        $deleter->delete(
            $group,
            $request->integer('colorId'),
        );
        
        return app(GroupColorDeletedResponse::class, ['action' => $deleter]);
    }
}
```

## Resource

Creiem fisierul GroupColorResource.php in Http / Resources 
```php
<?php
 
namespace Zaimea\Groups\Http\Resources;
 
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;
 
class GroupColorResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return parent::toArray($request);
    }
}
```

## Responses

Creiem fisierele GroupColorCreatedResponse.php, GroupColorDeletedResponse.php si in Http / Responses 
```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Http\Responses;

use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorCreatedResponse as Contract;

class GroupColorCreatedResponse implements Contract
{
    /**
     * Create an HTTP response that represents the object.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function toResponse($request): Response
    {
        return $request->wantsJson() 
                    ? new JsonResponse([
                        'message' => 'Color created successfully.',
                        'created'  => true,
                    ], 201)
                    : redirect()->back();
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Http\Responses;

use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorUpdatedResponse as Contract;

class GroupColorUpdatedResponse implements Contract
{
    /**
     * Create an HTTP response that represents the object.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function toResponse($request): Response
    {
        return $request->wantsJson() 
                    ? new JsonResponse([
                        'message' => 'Color updated successfully.',
                        'updated'  => true,
                    ], 201)
                    : redirect()->back();
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Zaimea\Groups\Http\Responses;

use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Zaimea\Groups\Group\Contracts\Colors\Responses\GroupColorDeletedResponse as Contract;

class GroupColorDeletedResponse implements Contract
{
    /**
     * Create an HTTP response that represents the object.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function toResponse($request): Response
    {
        return $request->wantsJson() 
                    ? new JsonResponse([
                        'message' => 'Color deleted successfully.',
                        'deleted'  => true,
                    ], 201)
                    : redirect()->back();
    }
}
```

## SimpleViewResponse

Adaugam in Http / SimpleViewResponse.php linia:

```php
\Zaimea\Groups\Group\Contracts\Colors\Views\GroupColorsViewResponse
```

Pana aici toate bune si frumoase, trebuie sa trecem la partea vizuala.

### Frontend

## Create View index
In pachetul zaimea-groups-view:

- facem indexul pentru colors `Livewire/resources/views/groups/views/colors.blade.php`.
```php
<x-group-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 dark:text-gray-200 leading-tight">
            {{ __('@i18n-groups::group-colors.group_color') }}
        </h2>
    </x-slot>
    <div>
        <div class="mx-auto py-10 sm:px-6 lg:px-8">
            @livewire('groups.color-manager', ['group' => $group])
        </div>
    </div>

</x-group-layout>
```

- in panel creiem pagina pentru a gestiona culorile `Livewire/resources/views/panel/group/color-manager.blade.php`.
```php

```

- in `group-sidebar-menu.blade.php` add new page

- in `LivewireServiceProvider.php` attach to function `configureLivewirecomponents()` the next line.
```php
Livewire::component('groups.color-manager', \Zaimea\GroupsView\Livewire\Livewire\Panel\Group\GroupColor::class);
```

- in `Livewire/src/Livewire` create a file for `Panel/Group/GroupColor.php`

```php

```