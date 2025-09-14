# Eloquent: Factory

- [簡介](#introduction)
- [定義模型 Factory](#defining-model-factories)
    - [生成 Factory](#generating-factories)
    - [Factory 狀態](#factory-states)
    - [Factory 回呼](#factory-callbacks)
- [使用 Factory 建立模型](#creating-models-using-factories)
    - [實例化模型](#instantiating-models)
    - [持久化模型](#persisting-models)
    - [序列](#sequences)
- [Factory 關聯](#factory-relationships)
    - [一對多關聯](#has-many-relationships)
    - [多對一關聯](#belongs-to-relationships)
    - [多對多關聯](#many-to-many-relationships)
    - [多型關聯](#polymorphic-relationships)
    - [在 Factory 中定義關聯](#defining-relationships-within-factories)
    - [為關聯重複使用現有模型](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## 簡介

在測試應用程式或填充資料庫時，您可能需要向資料庫中插入一些記錄。Laravel 允許您為每個 [Eloquent 模型](/docs/{{version}}/eloquent)定義一組預設屬性，而不是手動指定每個欄位的值，這就是透過模型 Factory 來實現的。

要查看如何編寫 Factory 的範例，請查看應用程式中的 `database/factories/UserFactory.php` 檔案。此 Factory 包含在所有新的 Laravel 應用程式中，並包含以下 Factory 定義：

```php
namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

/**
 * @extends \Illuminate\Database\Eloquent\Factories\Factory<\App\Models\User>
 */
class UserFactory extends Factory
{
    /**
     * The current password being used by the factory.
     */
    protected static ?string $password;

    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => static::$password ??= Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    /**
     * Indicate that the model's email address should be unverified.
     */
    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }
}
```

如您所見，Factory 最基本的形式是繼承 Laravel 基礎 Factory 類別並定義 `definition` 方法的類別。`definition` 方法返回在使用 Factory 建立模型時應套用的預設屬性值集。

透過 `fake` 輔助函數，Factory 可以存取 [Faker](https://github.com/FakerPHP/Faker) PHP 函式庫，這讓您可以方便地生成各種隨機資料用於測試和填充。

> [!NOTE]
> 您可以透過更新 `config/app.php` 設定檔中的 `faker_locale` 選項來更改應用程式的 Faker 語系。

<a name="defining-model-factories"></a>
## 定義模型 Factory

<a name="generating-factories"></a>
### 生成 Factory

要建立 Factory，請執行 `make:factory` [Artisan 命令](/docs/{{version}}/version/artisan)：

```shell
php artisan make:factory PostFactory
```

新的 Factory 類別將放置在您的 `database/factories` 目錄中。

<a name="factory-and-model-discovery-conventions"></a>
#### 模型與 Factory 發現慣例

一旦您定義了 Factory，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` Trait 為您的模型提供的靜態 `factory` 方法，以便為該模型實例化 Factory 實例。

`HasFactory` Trait 的 `factory` 方法將使用慣例來確定模型的正確 Factory。具體來說，該方法將在 `Database\Factories` 命名空間中尋找一個類別名稱與模型名稱匹配並以 `Factory` 為後綴的 Factory。如果這些慣例不適用於您的特定應用程式或 Factory，您可以覆寫模型上的 `newFactory` 方法，以直接返回模型對應 Factory 的實例：

```php
use Database\Factories\Administration\FlightFactory;

/**
 * Create a new factory instance for the model.
 */
protected static function newFactory()
{
    return FlightFactory::new();
}
```

然後，在對應的 Factory 上定義一個 `model` 屬性：

```php
use App\Administration\Flight;
use Illuminate\Database\Eloquent\Factories\Factory;

class FlightFactory extends Factory
{
    /**
     * The name of the factory's corresponding model.
     *
     * @var class-string<\Illuminate\Database\Eloquent\Model>
     */
    protected $model = Flight::class;
}
```

<a name="factory-states"></a>
### Factory 狀態

狀態操作方法允許您定義可以以任何組合套用到模型 Factory 的離散修改。例如，您的 `Database\Factories\UserFactory` Factory 可能包含一個 `suspended` 狀態方法，該方法修改其預設屬性值之一。

狀態轉換方法通常呼叫 Laravel 基礎 Factory 類別提供的 `state` 方法。`state` 方法接受一個閉包，該閉包將接收為 Factory 定義的原始屬性陣列，並應返回要修改的屬性陣列：

```php
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * Indicate that the user is suspended.
 */
public function suspended(): Factory
{
    return $this->state(function (array $attributes) {
        return [
            'account_status' => 'suspended',
        ];
    });
}
```

<a name="trashed-state"></a>
#### 「已刪除」狀態

如果您的 Eloquent 模型可以[軟刪除](/docs/{{version}}/eloquent#soft-deleting)，您可以呼叫內建的 `trashed` 狀態方法，以指示建立的模型應已「軟刪除」。您無需手動定義 `trashed` 狀態，因為它會自動提供給所有 Factory：

```php
use App\Models\User;

$user = User::factory()->trashed()->create();
```

<a name="factory-callbacks"></a>
### Factory 回呼

Factory 回呼是使用 `afterMaking` 和 `afterCreating` 方法註冊的，允許您在建立或創建模型後執行額外任務。您應該透過在 Factory 類別上定義 `configure` 方法來註冊這些回呼。當 Factory 實例化時，Laravel 將自動呼叫此方法：

```php
namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class UserFactory extends Factory
{
    /**
     * Configure the model factory.
     */
    public function configure(): static
    {
        return $this->afterMaking(function (User $user) {
            // ...
        })->afterCreating(function (User $user) {
            // ...
        });
    }

    // ...
}
```

您也可以在狀態方法中註冊 Factory 回呼，以執行特定於給定狀態的額外任務：

```php
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * Indicate that the user is suspended.
 */
public function suspended(): Factory
{
    return $this->state(function (array $attributes) {
        return [
            'account_status' => 'suspended',
        ];
    })->afterMaking(function (User $user) {
        // ...
    })->afterCreating(function (User $user) {
        // ...
    });
}
```

<a name="creating-models-using-factories"></a>
## 使用 Factory 建立模型

<a name="instantiating-models"></a>
### 實例化模型

一旦您定義了 Factory，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` Trait 為您的模型提供的靜態 `factory` 方法，以便為該模型實例化 Factory 實例。讓我們看看幾個建立模型的範例。首先，我們將使用 `make` 方法建立模型而不將它們持久化到資料庫：

```php
use App\Models\User;

$user = User::factory()->make();
```

您可以使用 `count` 方法建立多個模型的集合：

```php
$users = User::factory()->count(3)->make();
```

<a name="applying-states"></a>
#### 套用狀態

您也可以將任何[狀態](#factory-states)套用到模型。如果您想對模型套用多個狀態轉換，您可以直接呼叫狀態轉換方法：

```php
$users = User::factory()->count(5)->suspended()->make();
```

<a name="overriding-attributes"></a>
#### 覆寫屬性

如果您想覆寫模型的一些預設值，您可以將一個值陣列傳遞給 `make` 方法。只有指定的屬性會被替換，而其餘屬性則保持為 Factory 指定的預設值：

```php
$user = User::factory()->make([
    'name' => 'Abigail Otwell',
]);
```

或者，可以直接在 Factory 實例上呼叫 `state` 方法來執行內聯狀態轉換：

```php
$user = User::factory()->state([
    'name' => 'Abigail Otwell',
])->make();
```

> [!NOTE]
> 使用 Factory 建立模型時，[大量賦值保護](/docs/{{version}}/eloquent#mass-assignment)會自動停用。

<a name="persisting-models"></a>
### 持久化模型

`create` 方法實例化模型實例並使用 Eloquent 的 `save` 方法將它們持久化到資料庫：

```php
use App\Models\User;

// Create a single App\Models\User instance...
$user = User::factory()->create();

// Create three App\Models\User instances...
$users = User::factory()->count(3)->create();
```

您可以透過將屬性陣列傳遞給 `create` 方法來覆寫 Factory 的預設模型屬性：

```php
$user = User::factory()->create([
    'name' => 'Abigail',
]);
```

<a name="sequences"></a>
### 序列

有時您可能希望為每個建立的模型交替指定給定模型屬性的值。您可以透過將狀態轉換定義為序列來實現此目的。例如，您可能希望為每個建立的使用者將 `admin` 欄位的值在 `Y` 和 `N` 之間交替：

```php
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Sequence;

$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        ['admin' => 'Y'],
        ['admin' => 'N'],
    ))
    ->create();
```

在此範例中，將建立五個 `admin` 值為 `Y` 的使用者，以及五個 `admin` 值為 `N` 的使用者。

如有必要，您可以將閉包作為序列值。每次序列需要新值時，都會呼叫該閉包：

```php
use Illuminate\Database\Eloquent\Factories\Sequence;

$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
    ))
    ->create();
```

在序列閉包中，您可以存取注入閉包的序列實例上的 `$index` 或 `$count` 屬性。`$index` 屬性包含到目前為止序列已迭代的次數，而 `$count` 屬性包含序列將被呼叫的總次數：

```php
$users = User::factory()
    ->count(10)
    ->sequence(fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index])
    ->create();
```

為了方便起見，序列也可以使用 `sequence` 方法套用，該方法只是在內部呼叫 `state` 方法。`sequence` 方法接受一個閉包或序列化屬性的陣列：

```php
$users = User::factory()
    ->count(2)
    ->sequence(
        ['name' => 'First User'],
        ['name' => 'Second User'],
    )
    ->create();
```

<a name="factory-relationships"></a>
## Factory 關聯

<a name="has-many-relationships"></a>
### 一對多關聯

接下來，讓我們探索使用 Laravel 流暢的 Factory 方法建立 Eloquent 模型關聯。首先，假設我們的應用程式有一個 `App\Models\User` 模型和一個 `App\Models\Post` 模型。此外，假設 `User` 模型定義了與 `Post` 的 `hasMany` 關聯。我們可以使用 Laravel Factory 提供的 `has` 方法建立一個擁有三個貼文的使用者。`has` 方法接受一個 Factory 實例：

```php
use App\Models\Post;
use App\Models\User;

$user = User::factory()
    ->has(Post::factory()->count(3))
    ->create();
```

按照慣例，當將 `Post` 模型傳遞給 `has` 方法時，Laravel 將假定 `User` 模型必須有一個定義關聯的 `posts` 方法。如有必要，您可以明確指定要操作的關聯名稱：

```php
$user = User::factory()
    ->has(Post::factory()->count(3), 'posts')
    ->create();
```

當然，您可以對相關模型執行狀態操作。此外，如果您的狀態變更需要存取父模型，您可以傳遞一個基於閉包的狀態轉換：

```php
$user = User::factory()
    ->has(
        Post::factory()
            ->count(3)
            ->state(function (array $attributes, User $user) {
                return ['user_type' => $user->type];
            })
        )
    ->create();
```

<a name="has-many-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，您可以使用 Laravel 的魔術 Factory 關聯方法來建立關聯。例如，以下範例將使用慣例來確定相關模型應透過 `User` 模型上的 `posts` 關聯方法建立：

```php
$user = User::factory()
    ->hasPosts(3)
    ->create();
```

當使用魔術方法建立 Factory 關聯時，您可以傳遞一個屬性陣列來覆寫相關模型上的屬性：

```php
$user = User::factory()
    ->hasPosts(3, [
        'published' => false,
    ])
    ->create();
```

如果您的狀態變更需要存取父模型，您可以提供一個基於閉包的狀態轉換：

```php
$user = User::factory()
    ->hasPosts(3, function (array $attributes, User $user) {
        return ['user_type' => $user->type];
    })
    ->create();
```

<a name="belongs-to-relationships"></a>
### 多對一關聯

現在我們已經探索了如何使用 Factory 建立「一對多」關聯，接下來讓我們探索關聯的反向。`for` 方法可用於定義 Factory 建立的模型所屬的父模型。例如，我們可以建立三個屬於單一使用者的 `App\Models\Post` 模型實例：

```php
use App\Models\Post;
use App\Models\User;

$posts = Post::factory()
    ->count(3)
    ->for(User::factory()->state([
        'name' => 'Jessica Archer',
    ]))
    ->create();
```

如果您已經有一個應與您正在建立的模型關聯的父模型實例，您可以將模型實例傳遞給 `for` 方法：

```php
$user = User::factory()->create();

$posts = Post::factory()
    ->count(3)
    ->for($user)
    ->create();
```

<a name="belongs-to-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，您可以使用 Laravel 的魔術 Factory 關聯方法來定義「多對一」關聯。例如，以下範例將使用慣例來確定這三個貼文應屬於 `Post` 模型上的 `user` 關聯：

```php
$posts = Post::factory()
    ->count(3)
    ->forUser([
        'name' => 'Jessica Archer',
    ])
    ->create();
```

<a name="many-to-many-relationships"></a>
### 多對多關聯

與[一對多關聯](#has-many-relationships)一樣，「多對多」關聯可以使用 `has` 方法建立：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
    ->has(Role::factory()->count(3))
    ->create();
```

<a name="pivot-table-attributes"></a>
#### 樞紐表屬性

如果您需要定義應在連結模型的樞紐/中間表上設定的屬性，您可以使用 `hasAttached` 方法。此方法接受一個樞紐表屬性名稱和值的陣列作為其第二個參數：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
    ->hasAttached(
        Role::factory()->count(3),
        ['active' => true]
    )
    ->create();
```

如果您的狀態變更需要存取相關模型，您可以提供一個基於閉包的狀態轉換：

```php
$user = User::factory()
    ->hasAttached(
        Role::factory()
            ->count(3)
            ->state(function (array $attributes, User $user) {
                return ['name' => $user->name.' Role'];
            }),
        ['active' => true]
    )
    ->create();
```

如果您已經有要附加到您正在建立的模型上的模型實例，您可以將模型實例傳遞給 `hasAttached` 方法。在此範例中，相同的三個角色將附加到所有三個使用者：

```php
$roles = Role::factory()->count(3)->create();

$user = User::factory()
    ->count(3)
    ->hasAttached($roles, ['active' => true])
    ->create();
```

<a name="many-to-many-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，您可以使用 Laravel 的魔術 Factory 關聯方法來定義多對多關聯。例如，以下範例將使用慣例來確定相關模型應透過 `User` 模型上的 `roles` 關聯方法建立：

```php
$user = User::factory()
    ->hasRoles(1, [
        'name' => 'Editor'
    ])
    ->create();
```

<a name="polymorphic-relationships"></a>
### 多型關聯

[多型關聯](/docs/{{version}}/eloquent-relationships#polymorphic-relationships)也可以使用 Factory 建立。多型「多對多」關聯的建立方式與典型的「一對多」關聯相同。例如，如果 `App\Models\Post` 模型與 `App\Models\Comment` 模型具有 `morphMany` 關聯：

```php
use App\Models\Post;

$post = Post::factory()->hasComments(3)->create();
```

<a name="morph-to-relationships"></a>
#### Morph To 關聯

魔術方法不能用於建立 `morphTo` 關聯。相反，必須直接使用 `for` 方法，並且必須明確提供關聯的名稱。例如，假設 `Comment` 模型有一個定義 `morphTo` 關聯的 `commentable` 方法。在這種情況下，我們可以透過直接使用 `for` 方法來建立三個屬於單一貼文的評論：

```php
$comments = Comment::factory()->count(3)->for(
    Post::factory(), 'commentable'
)->create();
```

<a name="polymorphic-many-to-many-relationships"></a>
#### 多型多對多關聯

多型「多對多」(`morphToMany` / `morphedByMany`) 關聯的建立方式與非多型「多對多」關聯相同：

```php
use App\Models\Tag;
use App\Models\Video;

$videos = Video::factory()
    ->hasAttached(
        Tag::factory()->count(3),
        ['public' => true]
    )
    ->create();
```

當然，魔術 `has` 方法也可以用於建立多型「多對多」關聯：

```php
$videos = Video::factory()
    ->hasTags(3, ['public' => true])
    ->create();
```

<a name="defining-relationships-within-factories"></a>
### 在 Factory 中定義關聯

要在模型 Factory 中定義關聯，您通常會將新的 Factory 實例分配給關聯的外鍵。這通常用於「反向」關聯，例如 `belongsTo` 和 `morphTo` 關聯。例如，如果您想在建立貼文時建立一個新使用者，您可以執行以下操作：

```php
use App\Models\User;

/**
 * Define the model's default state.
 *
 * @return array<string, mixed>
 */
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'title' => fake()->title(),
        'content' => fake()->paragraph(),
    ];
}
```

如果關聯的欄位取決於定義它的 Factory，您可以將閉包分配給屬性。閉包將接收 Factory 評估的屬性陣列：

```php
/**
 * Define the model's default state.
 *
 * @return array<string, mixed>
 */
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'user_type' => function (array $attributes) {
            return User::find($attributes['user_id'])->type;
        },
        'title' => fake()->title(),
        'content' => fake()->paragraph(),
    ];
}
```

<a name="recycling-an-existing-model-for-relationships"></a>
### 為關聯重複使用現有模型

如果您的模型與另一個模型共享共同關聯，您可以使用 `recycle` 方法來確保相關模型的單一實例被 Factory 建立的所有關聯重複使用。

例如，假設您有 `Airline`、`Flight` 和 `Ticket` 模型，其中票證屬於航空公司和航班，並且航班也屬於航空公司。在建立票證時，您可能希望票證和航班使用相同的航空公司，因此您可以將航空公司實例傳遞給 `recycle` 方法：

```php
Ticket::factory()
    ->recycle(Airline::factory()->create())
    ->create();
```

如果您有屬於共同使用者或團隊的模型，您可能會發現 `recycle` 方法特別有用。

`recycle` 方法也接受現有模型的集合。當集合提供給 `recycle` 方法時，當 Factory 需要該類型的模型時，將從集合中隨機選擇一個模型：

```php
Ticket::factory()
    ->recycle($airlines)
    ->create();
```

