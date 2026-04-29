# Eloquent: 工廠 (Factories)

- [簡介](#introduction)
- [定義模型工廠](#defining-model-factories)
    - [生成工廠](#generating-factories)
    - [工廠狀態](#factory-states)
    - [工廠回呼](#factory-callbacks)
- [使用工廠建立模型](#creating-models-using-factories)
    - [實例化模型](#instantiating-models)
    - [持久化模型](#persisting-models)
    - [序列](#sequences)
- [工廠關聯](#factory-relationships)
    - [Has Many 關聯](#has-many-relationships)
    - [Belongs To 關聯](#belongs-to-relationships)
    - [多對多關聯](#many-to-many-relationships)
    - [多型關聯](#polymorphic-relationships)
    - [在工廠內定義關聯](#defining-relationships-within-factories)
    - [在關聯中重複利用現有模型](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## 簡介

在測試您的應用程式或為資料庫進行資料填充 (seeding) 時，您可能需要向資料庫插入一些記錄。Laravel 允許您使用模型工廠 (Model Factories) 為每個 [Eloquent 模型](/docs/{{version}}/eloquent) 定義一組預設屬性，而不是手動指定每一欄的值。

若要查看如何撰寫工廠的範例，請查看您應用程式中的 `database/factories/UserFactory.php` 檔案。所有新的 Laravel 應用程式都包含此工廠，並含有以下工廠定義：

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

如您所見，在最基本的形式中，工廠是繼承 Laravel 基礎工廠類別並定義 `definition` 方法的類別。`definition` 方法會回傳一組預設的屬性值，這些值在透過工廠建立模型時會被套用。

透過 `fake` 輔助函式，工廠可以存取 [Faker](https://github.com/FakerPHP/Faker) PHP 函式庫，這讓您可以方便地生成各種隨機資料用於測試和資料填充。

> [!NOTE]
> 您可以透過更新 `config/app.php` 設定檔中的 `faker_locale` 選項來更改應用程式的 Faker 語系。


<a name="defining-model-factories"></a>
## 定義模型工廠


<a name="generating-factories"></a>
### 生成工廠

若要建立一個工廠，請執行 `make:factory` [Artisan 指令](/docs/{{version}}/artisan)：

```shell
php artisan make:factory PostFactory
```

新的工廠類別將會被放置在您的 `database/factories` 目錄中。


<a name="factory-and-model-discovery-conventions"></a>
#### 模型與工廠的尋找慣例

定義好工廠後，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 提供給模型的靜態 `factory` 方法，以便為該模型實例化一個工廠執行個體。

`HasFactory` trait 的 `factory` 方法會使用慣例來決定該模型對應的工廠。具體來說，該方法會在 `Database\Factories` 命名空間中尋找名稱與模型名稱相符並加上 `Factory` 字尾的類別。如果這些慣例不適用於您的應用程式或工廠，您可以將 `UseFactory` 屬性加入到模型中，以手動指定模型的工廠：

```php
use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Database\Factories\Administration\FlightFactory;

#[UseFactory(FlightFactory::class)]
class Flight extends Model
{
    // ...
}
```

或者，您可以覆寫模型上的 `newFactory` 方法，直接回傳模型對應工廠的執行個體：

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

接著，在對應的工廠上使用 `UseModel` 屬性來指定模型：

```php
use App\Administration\Flight;
use Illuminate\Database\Eloquent\Factories\Attributes\UseModel;
use Illuminate\Database\Eloquent\Factories\Factory;

#[UseModel(Flight::class)]
class FlightFactory extends Factory
{
    // ...
}
```


<a name="factory-states"></a>
### 工廠狀態

狀態操作方法允許您定義離散的修改，這些修改可以以任何組合方式套用到您的模型工廠。例如，您的 `Database\Factories\UserFactory` 工廠可能包含一個 `suspended` 狀態方法，用來修改其中一個預設屬性值。

狀態轉換方法通常會呼叫 Laravel 基礎工廠類別提供的 `state` 方法。`state` 方法接受一個閉包，該閉包會接收為工廠定義的原始屬性陣列，並應回傳要修改的屬性陣列：

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
#### "Trashed" 狀態

如果您的 Eloquent 模型可以被 [軟刪除 (soft deleted)](/docs/{{version}}/eloquent#soft-deleting)，您可以呼叫內建的 `trashed` 狀態方法，以表示建立的模型應該已經處於「軟刪除」狀態。您不需要手動定義 `trashed` 狀態，因為它對所有工廠都是自動可用的：

```php
use App\Models\User;

$user = User::factory()->trashed()->create();
```


<a name="factory-callbacks"></a>
### 工廠回呼

工廠回呼是使用 `afterMaking` 和 `afterCreating` 方法註冊的，允許您在製作或建立模型後執行額外的任務。您應該透過在工廠類別中定義 `configure` 方法來註冊這些回呼。Laravel 在實例化工廠時會自動呼叫此方法：

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

您也可以在狀態方法中註冊工廠回呼，以執行特定於給定狀態的額外任務：

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
## 使用工廠建立模型

<a name="instantiating-models"></a>
### 實例化模型

當您定義好工廠後，可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 提供給模型的靜態 `factory` 方法，來為該模型實例化一個工廠實例。讓我們看幾個建立模型的範例。首先，我們將使用 `make` 方法來建立模型，而不將其持久化到資料庫：

```php
use App\Models\User;

$user = User::factory()->make();
```

您可以使用 `count` 方法來建立包含多個模型的集合：

```php
$users = User::factory()->count(3)->make();
```

<a name="applying-states"></a>
#### 套用狀態

您也可以將任何[狀態](#factory-states)套用到模型上。如果您想對模型套用多個狀態轉換，只需直接呼叫狀態轉換方法即可：

```php
$users = User::factory()->count(5)->suspended()->make();
```

<a name="overriding-attributes"></a>
#### 覆寫屬性

如果您想覆寫模型的一些預設值，可以將一個值陣列傳遞給 `make` 方法。只有指定的屬性會被替換，其餘屬性仍將保持工廠指定的預設值：

```php
$user = User::factory()->make([
    'name' => 'Abigail Otwell',
]);
```

或者，也可以直接在工廠實例上呼叫 `state` 方法來執行行內狀態轉換：

```php
$user = User::factory()->state([
    'name' => 'Abigail Otwell',
])->make();
```

> [!NOTE]
> 使用工廠建立模型時，會自動停用[大量指派保護](/docs/{{version}}/eloquent#mass-assignment)。

<a name="persisting-models"></a>
### 持久化模型

`create` 方法會實例化模型實例，並使用 Eloquent 的 `save` 方法將其持久化到資料庫：

```php
use App\Models\User;

// Create a single App\Models\User instance...
$user = User::factory()->create();

// Create three App\Models\User instances...
$users = User::factory()->count(3)->create();
```

您可以透過傳遞屬性陣列給 `create` 方法來覆寫工廠的預設模型屬性：

```php
$user = User::factory()->create([
    'name' => 'Abigail',
]);
```

<a name="sequences"></a>
### 序列

有時您可能希望為每個建立的模型交替變更給定屬性的值。您可以透過將狀態轉換定義為序列 (Sequence) 來達成此目的。例如，您可能希望每個建立的使用者的 `admin` 欄位值在 `Y` 和 `N` 之間交替：

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

在此範例中，將會建立五個 `admin` 值為 `Y` 的使用者，以及五個 `admin` 值為 `N` 的使用者。

如果有需要，您可以將閉包作為序列值。每當序列需要新值時，該閉包就會被調用：

```php
use Illuminate\Database\Eloquent\Factories\Sequence;

$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
    ))
    ->create();
```

在序列閉包中，您可以存取注入到閉包中的序列實例上的 `$index` 屬性。`$index` 屬性包含目前為止序列已經迭代的次數：

```php
$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index],
    ))
    ->create();
```

為了方便起見，也可以使用 `sequence` 方法來套用序列，該方法在內部只是調用 `state` 方法。`sequence` 方法接受一個閉包或多個序列化屬性的陣列：

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
## 工廠關聯

<a name="has-many-relationships"></a>
### Has Many 關聯

接下來，讓我們探索如何使用 Laravel 流式的工廠方法來建立 Eloquent 模型關聯。首先，假設我們的應用程式有一個 `App\Models\User` 模型和一個 `App\Models\Post` 模型。此外，假設 `User` 模型定義了一個與 `Post` 的 `hasMany` 關聯。我們可以使用 Laravel 工廠提供的 `has` 方法來建立一個擁有三篇文章的使用者。`has` 方法接受一個工廠實例：

```php
use App\Models\Post;
use App\Models\User;

$user = User::factory()
    ->has(Post::factory()->count(3))
    ->create();
```

按照慣例，當將 `Post` 模型傳遞給 `has` 方法時，Laravel 會假設 `User` 模型必須有一個定義該關聯的 `posts` 方法。如有需要，你可以明確指定想要操作的關聯名稱：

```php
$user = User::factory()
    ->has(Post::factory()->count(3), 'posts')
    ->create();
```

當然，你也可以對關聯模型進行狀態轉換。此外，如果你的狀態變更需要訪問父模型，你可以傳遞一個基於閉包的狀態轉換：

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

為了方便起見，你可以使用 Laravel 的魔術工廠關聯方法來建立關聯。例如，以下範例將使用慣例來確定應透過 `User` 模型上的 `posts` 關聯方法建立關聯模型：

```php
$user = User::factory()
    ->hasPosts(3)
    ->create();
```

使用魔術方法建立工廠關聯時，你可以傳遞一個屬性陣列來覆寫關聯模型上的屬性：

```php
$user = User::factory()
    ->hasPosts(3, [
        'published' => false,
    ])
    ->create();
```

你也可以傳遞多個屬性陣列，以建立具有各別模型狀態的關聯模型。Laravel 將按順序套用每個陣列：

```php
$user = User::factory()
    ->hasPosts(
        ['title' => 'First Post'],
        ['title' => 'Second Post'],
        ['title' => 'Third Post'],
    )
    ->create();
```

如果你的狀態變更需要訪問父模型，你可以提供一個基於閉包的狀態轉換：

```php
$user = User::factory()
    ->hasPosts(3, function (array $attributes, User $user) {
        return ['user_type' => $user->type];
    })
    ->create();
```

<a name="belongs-to-relationships"></a>
### Belongs To 關聯

現在我們已經探索了如何使用工廠建立「has many」關聯，接著讓我們探索反向的關聯。`for` 方法可用於定義工廠建立的模型所屬的父模型。例如，我們可以建立三個屬於單一使用者的 `App\Models\Post` 模型實例：

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

如果你已經有一個應該與正在建立的模型關聯的父模型實例，可以將該模型實例傳遞給 `for` 方法：

```php
$user = User::factory()->create();

$posts = Post::factory()
    ->count(3)
    ->for($user)
    ->create();
```

<a name="belongs-to-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，你可以使用 Laravel 的魔術工廠關聯方法來定義「belongs to」關聯。例如，以下範例將按慣例確定這三篇文章應屬於 `Post` 模型上的 `user` 關聯：

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

就像 [has many 關聯](#has-many-relationships)一樣，「多對多」關聯可以使用 `has` 方法來建立：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
    ->has(Role::factory()->count(3))
    ->create();
```

<a name="pivot-table-attributes"></a>
#### 樞紐表屬性

如果你需要定義應設置在連結模型的樞紐表 (Pivot Table) / 中繼表 (Intermediate Table) 上的屬性，你可以使用 `hasAttached` 方法。此方法的第二個引數接受樞紐表屬性名稱與值的陣列：

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

如果你的狀態變更需要訪問關聯模型，你可以提供一個基於閉包的狀態轉換：

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

你也可以傳遞一個樞紐屬性陣列的陣列，為每個關聯模型提供唯一的樞紐資料：

```php
$user = User::factory()
    ->hasAttached(
        Role::factory(),
        [
            ['active' => true],
            ['active' => false],
        ]
    )
    ->create();
```

如果你已經有想要附加到正在建立的模型上的模型實例，可以將這些模型實例傳遞給 `hasAttached` 方法。在此範例中，相同的三個角色將被附加到所有三個使用者上：

```php
$roles = Role::factory()->count(3)->create();

$users = User::factory()
    ->count(3)
    ->hasAttached($roles, ['active' => true])
    ->create();
```

<a name="many-to-many-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，你可以使用 Laravel 的魔術工廠關聯方法來定義多對多關聯。例如，以下範例將按慣例確定應透過 `User` 模型上的 `roles` 關聯方法來建立關聯模型：

```php
$user = User::factory()
    ->hasRoles(1, [
        'name' => 'Editor'
    ])
    ->create();
```

<a name="polymorphic-relationships"></a>
### 多型關聯

[多型關聯](/docs/{{version}}/eloquent-relationships#polymorphic-relationships)也可以使用工廠來建立。多型「morph many」關聯的建立方式與典型的「has many」關聯相同。例如，如果一個 `App\Models\Post` 模型與 `App\Models\Comment` 模型具有 `morphMany` 關聯：

```php
use App\Models\Post;

$post = Post::factory()->hasComments(3)->create();
```

<a name="morph-to-relationships"></a>
#### Morph To 關聯

魔術方法不能用於建立 `morphTo` 關聯。相反，必須直接使用 `for` 方法，並明確提供關聯的名稱。例如，想像 `Comment` 模型有一個定義 `morphTo` 關聯的 `commentable` 方法。在這種情況下，我們可以直接使用 `for` 方法建立三個屬於單一文章的留言：

```php
$comments = Comment::factory()->count(3)->for(
    Post::factory(), 'commentable'
)->create();
```

<a name="polymorphic-many-to-many-relationships"></a>
#### 多型多對多關聯

多型「多對多」（`morphToMany` / `morphedByMany`）關聯的建立方式與非多型的「多對多」關聯完全相同：

```php
use App\Models\Tag;
use App\Models\Video;

$video = Video::factory()
    ->hasAttached(
        Tag::factory()->count(3),
        ['public' => true]
    )
    ->create();
```

當然，魔術方法 `has` 也可以用來建立多型「多對多」關聯：

```php
$video = Video::factory()
    ->hasTags(3, ['public' => true])
    ->create();
```

<a name="defining-relationships-within-factories"></a>
### 在工廠內定義關聯

要在模型工廠中定義關聯，通常會將一個新的工廠實例指派給該關聯的外鍵。這通常用於「反向」關聯，例如 `belongsTo` 與 `morphTo` 關聯。例如，如果你想在建立貼文時同時建立一個新使用者，可以這樣做：

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

如果關聯的欄位取決於定義它的工廠，你可以將一個閉包指派給屬性。該閉包會接收到工廠計算後的屬性陣列：

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
### 在關聯中重複利用現有模型

如果你的多個模型與另一個模型共享同一個關聯，可以使用 `recycle` 方法來確保該關聯模型的單一實例在工廠建立的所有關聯中被重複利用。

例如，假設你有 `Airline`、`Flight` 與 `Ticket` 模型，其中機票屬於航空公司與航班，而航班也屬於航空公司。在建立機票時，你可能會希望機票與航班都使用同一家航空公司，因此你可以將一個航空公司實例傳遞給 `recycle` 方法：

```php
Ticket::factory()
    ->recycle(Airline::factory()->create())
    ->create();
```

如果你的模型屬於同一個使用者或團隊，你會發現 `recycle` 方法特別好用。

`recycle` 方法也接受現有模型的集合。當傳遞一個集合給 `recycle` 方法時，每當工廠需要該類型的模型時，會從集合中隨機挑選一個模型：

```php
Ticket::factory()
    ->recycle($airlines)
    ->create();
```