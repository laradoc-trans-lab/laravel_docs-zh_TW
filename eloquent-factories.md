# Eloquent：工廠

- [簡介](#introduction)
- [定義模型工廠](#defining-model-factories)
    - [產生工廠](#generating-factories)
    - [工廠狀態](#factory-states)
    - [工廠回呼](#factory-callbacks)
- [使用工廠建立模型](#creating-models-using-factories)
    - [實例化模型](#instantiating-models)
    - [儲存模型](#persisting-models)
    - [序列](#sequences)
- [工廠關聯](#factory-relationships)
    - [一對多關聯](#has-many-relationships)
    - [多對一關聯](#belongs-to-relationships)
    - [多對多關聯](#many-to-many-relationships)
    - [多型關聯](#polymorphic-relationships)
    - [在工廠中定義關聯](#defining-relationships-within-factories)
    - [為關聯重用既有模型](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## 簡介

在測試應用程式或填充資料庫時，您可能需要將一些記錄插入資料庫。Laravel 允許您為每個 [Eloquent 模型](/docs/{{version}}/eloquent)定義一組預設屬性，而無需手動指定每個欄位的值，這便是透過模型工廠 (model factories) 完成。

要查看如何編寫工廠的範例，請查看應用程式中的 `database/factories/UserFactory.php` 檔案。此工廠包含在所有新的 Laravel 應用程式中，並包含以下工廠定義：

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

如您所見，工廠最基本的型式是繼承 Laravel 基礎工廠類別並定義 `definition` 方法的類別。`definition` 方法會返回在使用工廠建立模型時應套用的預設屬性值集合。

透過 `fake` 輔助函式，工廠可以存取 [Faker](https://github.com/FakerPHP/Faker) PHP 函式庫，讓您可以方便地生成各種隨機資料用於測試和填充資料庫 (seeding)。

> [!NOTE]  
> 您可以透過更新 `config/app.php` 設定檔中的 `faker_locale` 選項來更改應用程式的 Faker 語系設定。

<a name="defining-model-factories"></a>
## 定義模型工廠

<a name="generating-factories"></a>
### 產生工廠

要建立工廠，請執行 `make:factory` [Artisan 指令](/docs/{{version}}/artisan)：

```shell
php artisan make:factory PostFactory
```

新的工廠類別將會被放置在 `database/factories` 目錄中。

<a name="factory-and-model-discovery-conventions"></a>
#### 模型與工廠的探索慣例

定義好工廠後，您可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` Trait 提供給模型的靜態 `factory` 方法，以實例化該模型的工廠實例。

`HasFactory` Trait 的 `factory` 方法將會使用慣例來確定該 Trait 所分配到的模型應使用的正確工廠。具體來說，該方法將會在 `Database\Factories` 命名空間中尋找一個類別名稱與模型名稱相符並以 `Factory` 作為後綴的工廠。如果這些慣例不適用於您的特定應用程式或工廠，您可以覆寫模型上的 `newFactory` 方法以直接返回該模型對應工廠的實例：

    use Database\Factories\Administration\FlightFactory;

    /**
     * Create a new factory instance for the model.
     */
    protected static function newFactory()
    {
        return FlightFactory::new();
    }

然後，在對應的工廠上定義一個 `model` 屬性：

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

<a name="factory-states"></a>
### 工廠狀態

狀態操作方法允許您定義可應用於模型工廠的離散修改，並可以任意組合。例如，您的 `Database\Factories\UserFactory` 工廠可能包含一個 `suspended` 狀態方法，該方法會修改其預設屬性值之一。

狀態轉換方法通常會呼叫 Laravel 基礎工廠類別提供的 `state` 方法。`state` 方法接受一個閉包，該閉包將會接收為工廠定義的原始屬性陣列，並應返回一個要修改的屬性陣列：

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

<a name="trashed-state"></a>
#### 「已刪除」狀態

如果您的 Eloquent 模型可以被 [軟刪除](/docs/{{version}}/eloquent#soft-deleting)，您可以呼叫內建的 `trashed` 狀態方法來指示所建立的模型應該已經被「軟刪除」。您不需要手動定義 `trashed` 狀態，因為它會自動提供給所有工廠：

    use App\Models\User;

    $user = User::factory()->trashed()->create();

<a name="factory-callbacks"></a>
### 工廠回呼

工廠回呼是透過 `afterMaking` 和 `afterCreating` 方法註冊的，它們允許您在建立 (making) 或儲存 (creating) 模型後執行額外任務。您應該透過在工廠類別上定義 `configure` 方法來註冊這些回呼。當工廠實例化時，Laravel 會自動呼叫此方法：

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

您也可以在狀態方法中註冊工廠回呼，以執行特定於給定狀態的額外任務：

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

<a name="creating-models-using-factories"></a>
## 使用工廠建立模型


<a name="instantiating-models"></a>
### 實例化模型

一旦定義好工廠後，您就可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 為模型提供的靜態 `factory` 方法，來實例化該模型的工廠實例。讓我們看看幾個建立模型的範例。首先，我們將使用 `make` 方法建立模型，但不將它們持久化到資料庫：

    use App\Models\User;

    $user = User::factory()->make();

您可以使用 `count` 方法建立多個模型的集合：

    $users = User::factory()->count(3)->make();


<a name="applying-states"></a>
#### 應用狀態

您也可以將任何的 [工廠狀態](#factory-states) 應用於模型。如果您想對模型應用多個狀態轉換，您可以直接呼叫狀態轉換方法：

    $users = User::factory()->count(5)->suspended()->make();


<a name="overriding-attributes"></a>
#### 覆寫屬性

如果您想覆寫模型的某些預設值，您可以將一個陣列傳遞給 `make` 方法。只有指定的屬性會被替換，而其餘屬性則保持工廠指定的預設值：

    $user = User::factory()->make([
        'name' => 'Abigail Otwell',
    ]);

或者，可以直接在工廠實例上呼叫 `state` 方法，以執行行內狀態轉換：

    $user = User::factory()->state([
        'name' => 'Abigail Otwell',
    ])->make();

> [!NOTE]  
> [大量賦值保護](/docs/{{version}}/eloquent#mass-assignment) 在使用工廠建立模型時會自動停用。


<a name="persisting-models"></a>
### 儲存模型

`create` 方法會實例化模型實例，並使用 Eloquent 的 `save` 方法將其持久化到資料庫：

    use App\Models\User;

    // Create a single App\Models\User instance...
    $user = User::factory()->create();

    // Create three App\Models\User instances...
    $users = User::factory()->count(3)->create();

您可以透過將屬性陣列傳遞給 `create` 方法，來覆寫工廠的預設模型屬性：

    $user = User::factory()->create([
        'name' => 'Abigail',
    ]);


<a name="sequences"></a>
### 序列

有時您可能希望為每個建立的模型交替指定給定模型屬性的值。您可以透過將狀態轉換定義為序列來實現。例如，您可能希望為每個建立的使用者，將 `admin` 欄位的值在 `Y` 和 `N` 之間交替：

    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
        ->count(10)
        ->state(new Sequence(
            ['admin' => 'Y'],
            ['admin' => 'N'],
        ))
        ->create();

在此範例中，將建立五個 `admin` 值為 `Y` 的使用者，以及五個 `admin` 值為 `N` 的使用者。

如有必要，您可以將閉包作為序列值。每當序列需要新值時，就會呼叫該閉包：

    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
        ->count(10)
        ->state(new Sequence(
            fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
        ))
        ->create();

在序列閉包中，您可以存取注入到閉包中的序列實例上的 `$index` 或 `$count` 屬性。`$index` 屬性包含到目前為止序列已迭代的次數，而 `$count` 屬性包含序列將被呼叫的總次數：

    $users = User::factory()
        ->count(10)
        ->sequence(fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index])
        ->create();

為方便起見，序列也可以使用 `sequence` 方法來應用，該方法只是在內部呼叫 `state` 方法。`sequence` 方法接受一個閉包或序列屬性的陣列：

    $users = User::factory()
        ->count(2)
        ->sequence(
            ['name' => 'First User'],
            ['name' => 'Second User'],
        )
        ->create();

<a name="factory-relationships"></a>
## 工廠關聯

<a name="has-many-relationships"></a>
### 一對多關聯

接下來，讓我們使用 Laravel 流暢的工廠方法來探索建立 Eloquent 模型關聯。首先，假設我們的應用程式有一個 `App\Models\User` 模型和一個 `App\Models\Post` 模型。另外，假設 `User` 模型定義了與 `Post` 的 `hasMany` 關聯。我們可以使用 Laravel 工廠提供的 `has` 方法來建立一個擁有三篇貼文的使用者。 `has` 方法接受一個工廠實例作為參數：

    use App\Models\Post;
    use App\Models\User;

    $user = User::factory()
        ->has(Post::factory()->count(3))
        ->create();

根據慣例，當將 `Post` 模型傳遞給 `has` 方法時，Laravel 會假定 `User` 模型必須有一個定義該關聯的 `posts` 方法。如有必要，您可以明確指定您希望操作的關聯名稱：

    $user = User::factory()
        ->has(Post::factory()->count(3), 'posts')
        ->create();

當然，您可以對關聯模型執行狀態操作。此外，如果您的狀態變更需要存取父模型，您可以傳遞基於閉包的狀態轉換：

    $user = User::factory()
        ->has(
            Post::factory()
                ->count(3)
                ->state(function (array $attributes, User $user) {
                    return ['user_type' => $user->type];
                })
            )
        ->create();

<a name="has-many-relationships-using-magic-methods"></a>
#### 使用魔法方法

為方便起見，您可以使用 Laravel 的魔法工廠關聯方法來建立關聯。例如，以下範例將使用慣例來確定關聯模型應該透過 `User` 模型上的 `posts` 關聯方法建立：

    $user = User::factory()
        ->hasPosts(3)
        ->create();

當使用魔法方法建立工廠關聯時，您可以傳遞一個屬性陣列來覆寫關聯模型上的屬性：

    $user = User::factory()
        ->hasPosts(3, [
            'published' => false,
        ])
        ->create();

如果您的狀態變更需要存取父模型，您可以提供基於閉包的狀態轉換：

    $user = User::factory()
        ->hasPosts(3, function (array $attributes, User $user) {
            return ['user_type' => $user->type];
        })
        ->create();

<a name="belongs-to-relationships"></a>
### 多對一關聯

現在我們已經探索了如何使用工廠建立「一對多」關聯，接下來讓我們探索關聯的反向。`for` 方法可用於定義工廠建立模型所屬的父模型。例如，我們可以建立三個屬於單一使用者的 `App\Models\Post` 模型實例：

    use App\Models\Post;
    use App\Models\User;

    $posts = Post::factory()
        ->count(3)
        ->for(User::factory()->state([
            'name' => 'Jessica Archer',
        ]))
        ->create();

如果您已經有一個父模型實例，並且該實例應該與您正在建立的模型關聯，您可以將該模型實例傳遞給 `for` 方法：

    $user = User::factory()->create();

    $posts = Post::factory()
        ->count(3)
        ->for($user)
        ->create();

<a name="belongs-to-relationships-using-magic-methods"></a>
#### 使用魔法方法

為方便起見，您可以使用 Laravel 的魔法工廠關聯方法來定義「多對一」關聯。例如，以下範例將使用慣例來確定這三篇貼文應該屬於 `Post` 模型上的 `user` 關聯：

    $posts = Post::factory()
        ->count(3)
        ->forUser([
            'name' => 'Jessica Archer',
        ])
        ->create();

<a name="many-to-many-relationships"></a>
### 多對多關聯

與 [一對多關聯](#has-many-relationships) 類似，「多對多」關聯可以使用 `has` 方法建立：

    use App\Models\Role;
    use App\Models\User;

    $user = User::factory()
        ->has(Role::factory()->count(3))
        ->create();

<a name="pivot-table-attributes"></a>
#### 樞紐表屬性

如果您需要定義應在連結模型的樞紐 / 中介表上設定的屬性，您可以使用 `hasAttached` 方法。此方法接受樞紐表屬性名稱和值的陣列作為其第二個引數：

    use App\Models\Role;
    use App\Models\User;

    $user = User::factory()
        ->hasAttached(
            Role::factory()->count(3),
            ['active' => true]
        )
        ->create();

如果您的狀態變更需要存取關聯模型，您可以提供基於閉包的狀態轉換：

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

如果您已經有希望關聯到您正在建立的模型上的模型實例，您可以將這些模型實例傳遞給 `hasAttached` 方法。在此範例中，相同的三個 Role 將會被關聯到所有三個 User 上：

    $roles = Role::factory()->count(3)->create();

    $user = User::factory()
        ->count(3)
        ->hasAttached($roles, ['active' => true])
        ->create();

<a name="many-to-many-relationships-using-magic-methods"></a>
#### 使用魔法方法

為方便起見，您可以使用 Laravel 的魔法工廠關聯方法來定義多對多關聯。例如，以下範例將使用慣例來確定關聯模型應該透過 `User` 模型上的 `roles` 關聯方法建立：

    $user = User::factory()
        ->hasRoles(1, [
            'name' => 'Editor'
        ])
        ->create();

<a name="polymorphic-relationships"></a>
### 多型關聯

[多型關聯](/docs/{{version}}/eloquent-relationships#polymorphic-relationships) 也可以使用工廠建立。多型「morph many」關聯的建立方式與典型的「has many」關聯相同。例如，如果 `App\Models\Post` 模型與 `App\Models\Comment` 模型具有 `morphMany` 關聯：

    use App\Models\Post;

    $post = Post::factory()->hasComments(3)->create();

<a name="morph-to-relationships"></a>
#### Morph To 關聯

魔法方法不能用於建立 `morphTo` 關聯。相反，必須直接使用 `for` 方法，並且必須明確提供關聯的名稱。例如，想像 `Comment` 模型有一個定義 `morphTo` 關聯的 `commentable` 方法。在這種情況下，我們可以透過直接使用 `for` 方法來建立三個屬於單一貼文的留言：

    $comments = Comment::factory()->count(3)->for(
        Post::factory(), 'commentable'
    )->create();

<a name="polymorphic-many-to-many-relationships"></a>
#### 多型多對多關聯

多型「多對多」(`morphToMany` / `morphedByMany`) 關聯的建立方式與非多型的「多對多」關聯相同：

    use App\Models\Tag;
    use App\Models\Video;

    $videos = Video::factory()
        ->hasAttached(
            Tag::factory()->count(3),
            ['public' => true]
        )
        ->create();

當然，魔法 `has` 方法也可以用於建立多型「多對多」關聯：

    $videos = Video::factory()
        ->hasTags(3, ['public' => true])
        ->create();

<a name="defining-relationships-within-factories"></a>
### 在工廠中定義關聯

若要在模型工廠中定義關聯，通常會將新的工廠實例指派給該關聯的外部索引鍵。這通常用於「反向」關聯，例如 `belongsTo` 和 `morphTo` 關聯。例如，若要在建立 post 時建立新的 user，您可以執行以下操作：

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

如果關聯的欄位依賴於定義它的工廠，您可以將閉包指派給屬性。該閉包將會收到工廠的評估屬性陣列：

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


<a name="recycling-an-existing-model-for-relationships"></a>
### 為關聯重用既有模型

如果您的模型與另一個模型共享共同關聯，您可以使用 `recycle` 方法來確保相關模型的單一實例會被工廠建立的所有關聯重用。

例如，想像您有 `Airline`、`Flight` 和 `Ticket` 模型，其中 ticket 屬於 airline 和 flight，而 flight 也屬於 airline。在建立 tickets 時，您可能希望 ticket 和 flight 都使用相同的 airline，因此您可以將 airline 實例傳遞給 `recycle` 方法：

    Ticket::factory()
        ->recycle(Airline::factory()->create())
        ->create();

如果您有屬於共同 user 或 team 的模型，您可能會發現 `recycle` 方法特別有用。

`recycle` 方法也接受現有模型的 collection。當提供 collection 給 `recycle` 方法時，當工廠需要該類型的模型時，將從 collection 中隨機選擇一個模型：

    Ticket::factory()
        ->recycle($airlines)
        ->create();