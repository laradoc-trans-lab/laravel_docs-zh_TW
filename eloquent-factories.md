# Eloquent：模型工廠

- [簡介](#introduction)
- [定義模型工廠](#defining-model-factories)
    - [生成工廠](#generating-factories)
    - [工廠狀態](#factory-states)
    - [工廠回呼](#factory-callbacks)
- [使用工廠建立模型](#creating-models-using-factories)
    - [實例化模型](#instantiating-models)
    - [持久化模型](#persisting-models)
    - [序列](#sequences)
- [工廠關係](#factory-relationships)
    - [一對多關係](#has-many-relationships)
    - [歸屬關係](#belongs-to-relationships)
    - [多對多關係](#many-to-many-relationships)
    - [多型關係](#polymorphic-relationships)
    - [在工廠中定義關係](#defining-relationships-within-factories)
    - [為關係回收現有模型](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## 簡介

在測試應用程式或填充資料庫時，您可能需要向資料庫插入一些記錄。Laravel 允許您為每個 [Eloquent 模型](/docs/{{version}}/eloquent) 定義一組預設屬性，使用模型工廠來替代手動指定每個欄位的值。

要查看如何編寫工廠的範例，請查看應用程式中的 `database/factories/UserFactory.php` 檔案。此工廠包含在所有新的 Laravel 應用程式中，並包含以下工廠定義：

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

如您所見，工廠最基本的形式是繼承 Laravel 基礎工廠類別並定義 `definition` 方法的類別。`definition` 方法會回傳使用工廠建立模型時應套用的預設屬性值集合。

透過 `fake` 輔助函式，工廠可以存取 [Faker](https://github.com/FakerPHP/Faker) PHP 函式庫，這讓您可以方便地生成各種隨機資料，用於測試和填充資料庫。

> [!NOTE]
> 您可以透過更新 `config/app.php` 設定檔中的 `faker_locale` 選項來更改應用程式的 Faker 語系。

<a name="defining-model-factories"></a>
## 定義模型工廠

<a name="generating-factories"></a>
### 生成工廠

要建立工廠，請執行 `make:factory` [Artisan 命令](/docs/{{version}}/artisan)：

```shell
php artisan make:factory PostFactory
```

新的工廠類別將會放在 `database/factories` 目錄中。

<a name="factory-and-model-discovery-conventions"></a>
#### 模型與工廠發現慣例

定義工廠後，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 為模型提供的靜態 `factory` 方法，以便為該模型實例化工廠實例。

`HasFactory` trait 的 `factory` 方法將使用慣例來確定分配給該 trait 的模型的正確工廠。具體來說，該方法將會在 `Database\Factories` 命名空間中尋找與模型名稱匹配且後綴為 `Factory` 的類別名稱的工廠。如果這些慣例不適用於您的特定應用程式或工廠，您可以在模型中添加 `UseFactory` 屬性來手動指定模型的工廠：

```php
use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Database\Factories\Administration\FlightFactory;

#[UseFactory(FlightFactory::class)]
class Flight extends Model
{
    // ...
}
```

或者，您可以覆寫模型上的 `newFactory` 方法，直接回傳模型對應工廠的實例：

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

然後，在對應的工廠上定義一個 `model` 屬性：

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
### 工廠狀態

狀態操作方法讓您可以定義可以應用於模型工廠的離散修改，並可以任意組合。例如，您的 `Database\Factories\UserFactory` 工廠可能包含一個 `suspended` 狀態方法，該方法會修改其預設屬性值之一。

狀態轉換方法通常會呼叫 Laravel 基礎工廠類別提供的 `state` 方法。`state` 方法接受一個閉包，該閉包將接收為工廠定義的原始屬性陣列，並應回傳一個要修改的屬性陣列：

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
#### 「已回收」狀態

如果您的 Eloquent 模型可以被[軟刪除](/docs/{{version}}/eloquent#soft-deleting)，您可以呼叫內建的 `trashed` 狀態方法，以表明建立的模型應該已經被「軟刪除」了。您不需要手動定義 `trashed` 狀態，因為它會自動提供給所有工廠：

```php
use App\Models\User;

$user = User::factory()->trashed()->create();
```

<a name="factory-callbacks"></a>
### 工廠回呼

工廠回呼使用 `afterMaking` 和 `afterCreating` 方法註冊，讓您可以在建立或新增模型後執行額外任務。您應該在工廠類別上定義一個 `configure` 方法來註冊這些回呼。當工廠被實例化時，此方法將由 Laravel 自動呼叫：

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

定義好工廠後，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 為模型提供的靜態 `factory` 方法，為該模型實例化一個工廠實例。接下來，我們將看一些建立模型的範例。首先，我們將使用 `make` 方法建立模型，而不會將其持久化到資料庫：

```php
use App\Models\User;

$user = User::factory()->make();
```

您可以使用 `count` 方法建立多個模型的集合：

```php
$users = User::factory()->count(3)->make();
```


<a name="applying-states"></a>
#### 應用狀態

您也可以將任何 [狀態](#factory-states) 應用於模型。如果您想將多個狀態轉換應用於模型，可以直接呼叫狀態轉換方法：

```php
$users = User::factory()->count(5)->suspended()->make();
```


<a name="overriding-attributes"></a>
#### 覆寫屬性

如果您想覆寫模型的某些預設值，您可以將一個值陣列傳遞給 `make` 方法。只有指定的屬性將被替換，而其餘屬性將保持由工廠指定的預設值：

```php
$user = User::factory()->make([
    'name' => 'Abigail Otwell',
]);
```

另外，可以直接在工廠實例上呼叫 `state` 方法以執行行內狀態轉換：

```php
$user = User::factory()->state([
    'name' => 'Abigail Otwell',
])->make();
```

> [!NOTE]
> 使用工廠建立模型時，[大量指派保護](/docs/{{version}}/eloquent#mass-assignment) 會自動停用。


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

您可以透過將一個屬性陣列傳遞給 `create` 方法來覆寫工廠的預設模型屬性：

```php
$user = User::factory()->create([
    'name' => 'Abigail',
]);
```


<a name="sequences"></a>
### 序列

有時您可能希望為每個建立的模型交替指定給定模型屬性的值。您可以透過將狀態轉換定義為序列來實現這一點。例如，您可能希望為每個建立的使用者，將 `admin` 欄位的值在 `Y` 和 `N` 之間交替：

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

在此範例中，將建立五個 `admin` 值為 `Y` 的使用者和五個 `admin` 值為 `N` 的使用者。

如果有必要，您可以將閉包包含為序列值。每當序列需要新值時，就會呼叫該閉包：

```php
use Illuminate\Database\Eloquent\Factories\Sequence;

$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
    ))
    ->create();
```

在序列閉包中，您可以存取注入閉包的序列實例上的 `$index` 屬性。`$index` 屬性包含迄今為止序列已發生的迭代次數：

```php
$users = User::factory()
    ->count(10)
    ->state(new Sequence(
        fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index],
    ))
    ->create();
```

為方便起見，也可以使用 `sequence` 方法來應用序列，該方法只是在內部呼叫 `state` 方法。`sequence` 方法接受閉包或序列屬性陣列：

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
## 工廠關係


<a name="has-many-relationships"></a>
### 一對多關係

接下來，我們將探索如何使用 Laravel 流暢的工廠方法來建立 Eloquent 模型關係。首先，假設我們的應用程式有一個 `App\Models\User` 模型和一個 `App\Models\Post` 模型。此外，假設 `User` 模型定義了一個與 `Post` 的 `hasMany` 關係。我們可以使用 Laravel 工廠提供的 `has` 方法來建立一個擁有三篇貼文的使用者。`has` 方法接受一個工廠實例：

```php
use App\Models\Post;
use App\Models\User;

$user = User::factory()
    ->has(Post::factory()->count(3))
    ->create();
```

按照慣例，當將 `Post` 模型傳遞給 `has` 方法時，Laravel 會假定 `User` 模型必須有一個定義該關係的 `posts` 方法。如果需要，您可以明確指定要操作的關係名稱：

```php
$user = User::factory()
    ->has(Post::factory()->count(3), 'posts')
    ->create();
```

當然，您可以對關聯模型執行狀態操作。此外，如果您的狀態變更需要存取父模型，您可以傳遞基於閉包的狀態轉換：

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

為了方便起見，您可以使用 Laravel 的魔術工廠關係方法來建立關係。例如，以下範例將使用慣例來判斷應透過 `User` 模型上的 `posts` 關係方法建立關聯模型：

```php
$user = User::factory()
    ->hasPosts(3)
    ->create();
```

當使用魔術方法建立工廠關係時，您可以傳遞一個屬性陣列來覆寫關聯模型上的屬性：

```php
$user = User::factory()
    ->hasPosts(3, [
        'published' => false,
    ])
    ->create();
```

如果您的狀態變更需要存取父模型，您可以提供基於閉包的狀態轉換：

```php
$user = User::factory()
    ->hasPosts(3, function (array $attributes, User $user) {
        return ['user_type' => $user->type];
    })
    ->create();
```


<a name="belongs-to-relationships"></a>
### 歸屬關係

現在我們已經探索了如何使用工廠建立「一對多」關係，接下來我們將探索這種關係的反向操作。`for` 方法可用於定義工廠建立模型所屬的父模型。例如，我們可以建立三個屬於單一使用者的 `App\Models\Post` 模型實例：

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

如果您已經有一個父模型實例應與您正在建立的模型關聯，您可以將該模型實例傳遞給 `for` 方法：

```php
$user = User::factory()->create();

$posts = Post::factory()
    ->count(3)
    ->for($user)
    ->create();
```


<a name="belongs-to-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，您可以使用 Laravel 的魔術工廠關係方法來定義「歸屬」關係。例如，以下範例將使用慣例來判斷這三篇貼文應歸屬於 `Post` 模型上的 `user` 關係：

```php
$posts = Post::factory()
    ->count(3)
    ->forUser([
        'name' => 'Jessica Archer',
    ])
    ->create();
```


<a name="many-to-many-relationships"></a>
### 多對多關係

與[一對多關係](#has-many-relationships)類似，「多對多」關係也可以使用 `has` 方法建立：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
    ->has(Role::factory()->count(3))
    ->create();
```


<a name="pivot-table-attributes"></a>
#### 樞紐表屬性

如果您需要定義應設定在連接模型的樞紐/中介表上的屬性，您可以使用 `hasAttached` 方法。此方法接受一個樞紐表屬性名稱和值的陣列作為其第二個參數：

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

如果您的狀態變更需要存取關聯模型，您可以提供基於閉包的狀態轉換：

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

如果您已經有模型實例想要附加到您正在建立的模型，您可以將這些模型實例傳遞給 `hasAttached` 方法。在此範例中，相同的三個角色將附加到所有三個使用者：

```php
$roles = Role::factory()->count(3)->create();

$users = User::factory()
    ->count(3)
    ->hasAttached($roles, ['active' => true])
    ->create();
```


<a name="many-to-many-relationships-using-magic-methods"></a>
#### 使用魔術方法

為了方便起見，您可以使用 Laravel 的魔術工廠關係方法來定義多對多關係。例如，以下範例將使用慣例來判斷應透過 `User` 模型上的 `roles` 關係方法建立關聯模型：

```php
$user = User::factory()
    ->hasRoles(1, [
        'name' => 'Editor'
    ])
    ->create();
```


<a name="polymorphic-relationships"></a>
### 多型關係

[多型關係](/docs/{{version}}/eloquent-relationships#polymorphic-relationships)也可以使用工廠建立。多型「morph many」關係的建立方式與典型「一對多」關係相同。例如，如果 `App\Models\Post` 模型與 `App\Models\Comment` 模型存在 `morphMany` 關係：

```php
use App\Models\Post;

$post = Post::factory()->hasComments(3)->create();
```


<a name="morph-to-relationships"></a>
#### Morph To 關係

魔術方法可能無法用於建立 `morphTo` 關係。相反地，必須直接使用 `for` 方法，並且必須明確提供關係的名稱。例如，想像 `Comment` 模型有一個 `commentable` 方法，定義了 `morphTo` 關係。在這種情況下，我們可以直接使用 `for` 方法建立三個屬於單一篇貼文的註解：

```php
$comments = Comment::factory()->count(3)->for(
    Post::factory(), 'commentable'
)->create();
```


<a name="polymorphic-many-to-many-relationships"></a>
#### 多型多對多關係

多型「多對多」（`morphToMany` / `morphedByMany`）關係的建立方式與非多型「多對多」關係相同：

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

當然，魔術 `has` 方法也可以用於建立多型「多對多」關係：

```php
$video = Video::factory()
    ->hasTags(3, ['public' => true])
    ->create();
```

<a name="defining-relationships-within-factories"></a>
### 在工廠中定義關係

若要在模型工廠中定義關係，您通常會將新的工廠實例指派給該關係的外來鍵。這通常用於「反向」關係，例如 `belongsTo` 和 `morphTo` 關係。舉例來說，如果您希望在建立文章時同時建立一個新的使用者，您可以這樣做：

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

如果關係的欄位取決於定義它的工廠，您可以將閉包指派給屬性。該閉包將接收工廠已評估的屬性陣列：

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
### 為關係回收現有模型

如果您的模型與另一個模型共用相同的關係，您可以使用 `recycle` 方法來確保相關模型的一個實例被工廠建立的所有關係重複利用。

例如，假設您有 `Airline`、`Flight` 和 `Ticket` 模型，其中票證屬於某個航空公司和某個航班，而航班也屬於某個航空公司。在建立票證時，您可能會希望票證和航班都屬於同一個航空公司，因此您可以將航空公司實例傳遞給 `recycle` 方法：

```php
Ticket::factory()
    ->recycle(Airline::factory()->create())
    ->create();
```

如果您有多個模型屬於同一個使用者或團隊，您可能會發現 `recycle` 方法特別有用。

`recycle` 方法也接受一個現有模型的集合。當集合提供給 `recycle` 方法時，當工廠需要該類型的模型時，將從集合中隨機選擇一個模型：

```php
Ticket::factory()
    ->recycle($airlines)
    ->create();
```