# Eloquent：集合

- [介紹](#introduction)
- [可用方法](#available-methods)
- [自訂集合](#custom-collections)

<a name="introduction"></a>
## 介紹

所有返回一個以上模型結果的 Eloquent 方法，都會返回 `Illuminate\Database\Eloquent\Collection` 類別的實例，這包含透過 `get` 方法獲取的結果或透過關聯存取的結果。Eloquent 集合物件擴展了 Laravel 的 [基本集合](/docs/{{version}}/collections)，因此它自然繼承了數十種方法，用於流暢地操作底層的 Eloquent 模型陣列。務必查閱 Laravel 集合文件以了解所有這些實用方法！

所有集合也充當迭代器，讓您可以像遍歷簡單的 PHP 陣列一樣循環遍歷它們：

    use App\Models\User;

    $users = User::where('active', 1)->get();

    foreach ($users as $user) {
        echo $user->name;
    }

然而，如前所述，集合比陣列強大得多，並且公開了各種 map / reduce 操作，這些操作可以透過直觀的介面進行鏈式操作。例如，我們可以移除所有非活動模型，然後為每個剩餘使用者收集名字：

    $names = User::all()->reject(function (User $user) {
        return $user->active === false;
    })->map(function (User $user) {
        return $user->name;
    });


<a name="eloquent-collection-conversion"></a>
#### Eloquent 集合轉換

雖然大多數 Eloquent 集合方法會返回 Eloquent 集合的新實例，但 `collapse`、`flatten`、`flip`、`keys`、`pluck` 和 `zip` 方法會返回 [基本集合](/docs/{{version}}/collections) 實例。同樣地，如果一個 `map` 操作返回的集合不包含任何 Eloquent 模型，它將被轉換為基本集合實例。

<a name="available-methods"></a>
## 可用方法

所有 Eloquent 集合皆擴展了基礎的 [Laravel Collection](/docs/{{version}}/collections#available-methods) 物件；因此，它們繼承了基礎 Collection 類別提供的所有強大方法。

此外，`Illuminate\Database\Eloquent\Collection` 類別提供了一系列方法，可協助管理您的模型集合。大部分方法會回傳 `Illuminate\Database\Eloquent\Collection` 實例；然而，有些方法，例如 `modelKeys`，會回傳 `Illuminate\Support\Collection` 實例。

<style>
    .collection-method-list > p {
        columns: 14.4em 1; -moz-columns: 14.4em 1; -webkit-columns: 14.4em 1;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>

<div class="collection-method-list" markdown="1">

[append](#method-append)
[contains](#method-contains)
[diff](#method-diff)
[except](#method-except)
[find](#method-find)
[findOrFail](#method-find-or-fail)
[fresh](#method-fresh)
[intersect](#method-intersect)
[load](#method-load)
[loadMissing](#method-loadMissing)
[modelKeys](#method-modelKeys)
[makeVisible](#method-makeVisible)
[makeHidden](#method-makeHidden)
[only](#method-only)
[setVisible](#method-setVisible)
[setHidden](#method-setHidden)
[toQuery](#method-toquery)
[unique](#method-unique)

</div>


<a name="method-append"></a>
#### `append($attributes)` {.collection-method .first-collection-method}

`append` 方法可用來指定集合中每個模型的一個屬性應被 [附加](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。此方法接受一個屬性陣列或單一屬性：

    $users->append('team');

    $users->append(['team', 'is_admin']);


<a name="method-contains"></a>
#### `contains($key, $operator = null, $value = null)` {.collection-method}

`contains` 方法可用於判斷集合是否包含給定的模型實例。此方法接受主鍵或模型實例：

    $users->contains(1);

    $users->contains(User::find(1));


<a name="method-diff"></a>
#### `diff($items)` {.collection-method}

`diff` 方法回傳所有不在給定集合中的模型：

    use App\Models\User;

    $users = $users->diff(User::whereIn('id', [1, 2, 3])->get());


<a name="method-except"></a>
#### `except($keys)` {.collection-method}

`except` 方法回傳所有不具有給定主鍵的模型：

    $users = $users->except([1, 2, 3]);


<a name="method-find"></a>
#### `find($key)` {.collection-method}

`find` 方法回傳主鍵與給定鍵匹配的模型。如果 `$key` 是一個模型實例，`find` 將嘗試回傳一個與主鍵匹配的模型。如果 `$key` 是一個鍵的陣列，`find` 將回傳所有主鍵在給定陣列中的模型：

    $users = User::all();

    $user = $users->find(1);


<a name="method-find-or-fail"></a>
#### `findOrFail($key)` {.collection-method}

`findOrFail` 方法回傳主鍵與給定鍵匹配的模型，如果集合中找不到匹配的模型，則拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 異常：

    $users = User::all();

    $user = $users->findOrFail(1);


<a name="method-fresh"></a>
#### `fresh($with = [])` {.collection-method}

`fresh` 方法從資料庫中檢索集合中每個模型的新實例。此外，任何指定的關聯都將會被積極載入：

    $users = $users->fresh();

    $users = $users->fresh('comments');


<a name="method-intersect"></a>
#### `intersect($items)` {.collection-method}

`intersect` 方法回傳所有也存在於給定集合中的模型：

    use App\Models\User;

    $users = $users->intersect(User::whereIn('id', [1, 2, 3])->get());


<a name="method-load"></a>
#### `load($relations)` {.collection-method}

`load` 方法為集合中的所有模型積極載入給定的關聯：

    $users->load(['comments', 'posts']);

    $users->load('comments.author');

    $users->load(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);


<a name="method-loadMissing"></a>
#### `loadMissing($relations)` {.collection-method}

`loadMissing` 方法為集合中的所有模型積極載入給定的關聯，如果這些關聯尚未載入：

    $users->loadMissing(['comments', 'posts']);

    $users->loadMissing('comments.author');

    $users->loadMissing(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);


<a name="method-modelKeys"></a>
#### `modelKeys()` {.collection-method}

`modelKeys` 方法回傳集合中所有模型的主鍵：

    $users->modelKeys();

    // [1, 2, 3, 4, 5]


<a name="method-makeVisible"></a>
#### `makeVisible($attributes)` {.collection-method}

`makeVisible` 方法 [讓屬性可見](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)，這些屬性通常在集合中的每個模型上都是「隱藏的」：

    $users = $users->makeVisible(['address', 'phone_number']);


<a name="method-makeHidden"></a>
#### `makeHidden($attributes)` {.collection-method}

`makeHidden` 方法 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)，這些屬性通常在集合中的每個模型上都是「可見的」：

    $users = $users->makeHidden(['address', 'phone_number']);


<a name="method-only"></a>
#### `only($keys)` {.collection-method}

`only` 方法回傳所有具有給定主鍵的模型：

    $users = $users->only([1, 2, 3]);


<a name="method-setVisible"></a>
#### `setVisible($attributes)` {.collection-method}

`setVisible` 方法 [暫時覆寫](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility) 集合中每個模型上的所有可見屬性：

    $users = $users->setVisible(['id', 'name']);


<a name="method-setHidden"></a>
#### `setHidden($attributes)` {.collection-method}

`setHidden` 方法 [暫時覆寫](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility) 集合中每個模型上的所有隱藏屬性：

    $users = $users->setHidden(['email', 'password', 'remember_token']);


<a name="method-toquery"></a>
#### `toQuery()` {.collection-method}

`toQuery` 方法回傳一個 Eloquent 查詢建構器實例，其中包含對集合模型主鍵的 `whereIn` 約束：

    use App\Models\User;

    $users = User::where('status', 'VIP')->get();

    $users->toQuery()->update([
        'status' => 'Administrator',
    ]);


<a name="method-unique"></a>
#### `unique($key = null, $strict = false)` {.collection-method}

`unique` 方法回傳集合中所有唯一的模型。任何與集合中其他模型具有相同主鍵的模型都會被移除：

    $users = $users->unique();

<a name="custom-collections"></a>
## 自訂集合

如果您想在與給定 model 互動時使用自訂的 `Collection` 物件，您可以將 `CollectedBy` 屬性新增到您的 model 中：

    <?php

    namespace App\Models;

    use App\Support\UserCollection;
    use Illuminate\Database\Eloquent\Attributes\CollectedBy;
    use Illuminate\Database\Eloquent\Model;

    #[CollectedBy(UserCollection::class)]
    class User extends Model
    {
        // ...
    }

或者，您可以在 model 上定義一個 `newCollection` 方法：

    <?php

    namespace App\Models;

    use App\Support\UserCollection;
    use Illuminate\Database\Eloquent\Collection;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Create a new Eloquent Collection instance.
         *
         * @param  array<int, \Illuminate\Database\Eloquent\Model>  $models
         * @return \Illuminate\Database\Eloquent\Collection<int, \Illuminate\Database\Eloquent\Model>
         */
        public function newCollection(array $models = []): Collection
        {
            return new UserCollection($models);
        }
    }

一旦您定義了 `newCollection` 方法或在 model 中添加了 `CollectedBy` 屬性，每當 Eloquent 通常會回傳 `Illuminate\Database\Eloquent\Collection` 實例時，您都會收到您的自訂集合實例。

如果您想在應用程式中的每個 model 都使用自訂集合，您應該在所有應用程式 model 都繼承的基礎 model 類別上定義 `newCollection` 方法。