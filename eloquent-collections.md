# Eloquent: 集合

- [簡介](#introduction)
- [可用方法](#available-methods)
- [自訂集合](#custom-collections)

<a name="introduction"></a>
## 簡介

所有回傳多個模型結果的 Eloquent 方法，都會回傳 `Illuminate\Database\Eloquent\Collection` 類別的實例，這包括透過 `get` 方法或透過關聯存取的結果。Eloquent 集合物件擴充了 Laravel 的 [基礎集合](/docs/{{version}}/collections)，因此它自然繼承了數十種用於流暢操作底層 Eloquent 模型陣列的方法。請務必查閱 Laravel 集合文件，以了解所有這些有用的方法！

所有集合也都可以作為迭代器，讓你可以像操作簡單的 PHP 陣列一樣遍歷它們：

```php
use App\Models\User;

$users = User::where('active', 1)->get();

foreach ($users as $user) {
    echo $user->name;
}
```

然而，如前所述，集合比陣列強大得多，並公開了各種 map / reduce 操作，這些操作可以使用直觀的介面進行鏈式呼叫。例如，我們可以移除所有非活動模型，然後收集每個剩餘使用者的名字：

```php
$names = User::all()->reject(function (User $user) {
    return $user->active === false;
})->map(function (User $user) {
    return $user->name;
});
```

<a name="eloquent-collection-conversion"></a>
#### Eloquent 集合轉換

雖然大多數 Eloquent 集合方法都會回傳一個新的 Eloquent 集合實例，但 `collapse`、`flatten`、`flip`、`keys`、`pluck` 和 `zip` 方法會回傳一個 [基礎集合](/docs/{{version}}/collections) 實例。同樣地，如果 `map` 操作回傳的集合不包含任何 Eloquent 模型，它將被轉換為基礎集合實例。

<a name="available-methods"></a>
## 可用方法

所有 Eloquent 集合都擴充了基礎 [Laravel 集合](/docs/{{version}}/collections#available-methods) 物件；因此，它們繼承了基礎集合類別提供的所有強大方法。

此外，`Illuminate\Database\Eloquent\Collection` 類別提供了一組超集方法，以協助管理你的模型集合。大多數方法回傳 `Illuminate\Database\Eloquent\Collection` 實例；但是，某些方法，例如 `modelKeys`，回傳 `Illuminate\Support\Collection` 實例。

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
[partition](#method-partition)
[setVisible](#method-setVisible)
[setHidden](#method-setHidden)
[toQuery](#method-toquery)
[unique](#method-unique)

</div>

<a name="method-append"></a>
#### `append($attributes)` {.collection-method .first-collection-method}

`append` 方法可用於指示集合中的每個模型都應 [附加](/docs/{{version}}/eloquent-serialization#appending-values-to-json) 屬性。此方法接受屬性陣列或單一屬性：

```php
$users->append('team');

$users->append(['team', 'is_admin']);
```

<a name="method-contains"></a>
#### `contains($key, $operator = null, $value = null)` {.collection-method}

`contains` 方法可用於判斷集合中是否包含給定的模型實例。此方法接受主鍵或模型實例：

```php
$users->contains(1);

$users->contains(User::find(1));
```

<a name="method-diff"></a>
#### `diff($items)` {.collection-method}

`diff` 方法回傳所有不在給定集合中的模型：

```php
use App\Models\User;

$users = $users->diff(User::whereIn('id', [1, 2, 3])->get());
```

<a name="method-except"></a>
#### `except($keys)` {.collection-method}

`except` 方法回傳所有不具有給定主鍵的模型：

```php
$users = $users->except([1, 2, 3]);
```

<a name="method-find"></a>
#### `find($key)` {.collection-method}

`find` 方法回傳主鍵與給定鍵匹配的模型。如果 `$key` 是模型實例，`find` 將嘗試回傳與主鍵匹配的模型。如果 `$key` 是鍵的陣列，`find` 將回傳所有主鍵在給定陣列中的模型：

```php
$users = User::all();

$user = $users->find(1);
```

<a name="method-find-or-fail"></a>
#### `findOrFail($key)` {.collection-method}

`findOrFail` 方法回傳主鍵與給定鍵匹配的模型，如果集合中找不到匹配的模型，則拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 異常：

```php
$users = User::all();

$user = $users->findOrFail(1);
```

<a name="method-fresh"></a>
#### `fresh($with = [])` {.collection-method}

`fresh` 方法從資料庫中檢索集合中每個模型的新實例。此外，任何指定的關聯都將被預先載入：

```php
$users = $users->fresh();

$users = $users->fresh('comments');
```

<a name="method-intersect"></a>
#### `intersect($items)` {.collection-method}

`intersect` 方法回傳所有也存在於給定集合中的模型：

```php
use App\Models\User;

$users = $users->intersect(User::whereIn('id', [1, 2, 3])->get());
```

<a name="method-load"></a>
#### `load($relations)` {.collection-method}

`load` 方法為集合中的所有模型預先載入給定的關聯：

```php
$users->load(['comments', 'posts']);

$users->load('comments.author');

$users->load(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);
```

<a name="method-loadMissing"></a>
#### `loadMissing($relations)` {.collection-method}

`loadMissing` 方法為集合中的所有模型預先載入給定的關聯，如果這些關聯尚未載入：

```php
$users->loadMissing(['comments', 'posts']);

$users->loadMissing('comments.author');

$users->loadMissing(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);
```

<a name="method-modelKeys"></a>
#### `modelKeys()` {.collection-method}

`modelKeys` 方法回傳集合中所有模型的主鍵：

```php
$users->modelKeys();

// [1, 2, 3, 4, 5]
```

<a name="method-makeVisible"></a>
#### `makeVisible($attributes)` {.collection-method}

`makeVisible` 方法 [使屬性可見](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)，這些屬性通常在集合中的每個模型上都是「隱藏」的：

```php
$users = $users->makeVisible(['address', 'phone_number']);
```

<a name="method-makeHidden"></a>
#### `makeHidden($attributes)` {.collection-method}

`makeHidden` 方法 [隱藏屬性](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)，這些屬性通常在集合中的每個模型上都是「可見」的：

```php
$users = $users->makeHidden(['address', 'phone_number']);
```

<a name="method-only"></a>
#### `only($keys)` {.collection-method}

`only` 方法回傳所有具有給定主鍵的模型：

```php
$users = $users->only([1, 2, 3]);
```

<a name="method-partition"></a>
#### `partition` {.collection-method}

`partition` 方法回傳一個 `Illuminate\Support\Collection` 實例，其中包含 `Illuminate\Database\Eloquent\Collection` 集合實例：

```php
$partition = $users->partition(fn ($user) => $user->age > 18);

dump($partition::class);    // Illuminate\Support\Collection
dump($partition[0]::class); // Illuminate\Database\Eloquent\Collection
dump($partition[1]::class); // Illuminate\Database\Eloquent\Collection
```

<a name="method-setVisible"></a>
#### `setVisible($attributes)` {.collection-method}

`setVisible` 方法 [暫時覆寫](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility) 集合中每個模型上的所有可見屬性：

```php
$users = $users->setVisible(['id', 'name']);
```

<a name="method-setHidden"></a>
#### `setHidden($attributes)` {.collection-method}

`setHidden` 方法 [暫時覆寫](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility) 集合中每個模型上的所有隱藏屬性：

```php
$users = $users->setHidden(['email', 'password', 'remember_token']);
```

<a name="method-toquery"></a>
#### `toQuery()` {.collection-method}

`toQuery` 方法回傳一個 Eloquent 查詢建構器實例，其中包含集合模型主鍵上的 `whereIn` 約束：

```php
use App\Models\User;

$users = User::where('status', 'VIP')->get();

$users->toQuery()->update([
    'status' => 'Administrator',
]);
```

<a name="method-unique"></a>
#### `unique($key = null, $strict = false)` {.collection-method}

`unique` 方法回傳集合中所有唯一的模型。任何與集合中其他模型具有相同主鍵的模型都將被移除：

```php
$users = $users->unique();
```

<a name="custom-collections"></a>
## 自訂集合

如果你想在與給定模型互動時使用自訂的 `Collection` 物件，你可以將 `CollectedBy` 屬性新增到你的模型中：

```php
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
```

或者，你可以在模型上定義 `newCollection` 方法：

```php
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
        $collection = new UserCollection($models);

        if (Model::isAutomaticallyEagerLoadingRelationships()) {
            $collection->withRelationshipAutoloading();
        }

        return $collection;
    }
}
```

一旦你定義了 `newCollection` 方法或將 `CollectedBy` 屬性新增到你的模型中，每當 Eloquent 通常回傳 `Illuminate\Database\Eloquent\Collection` 實例時，你都將收到你的自訂集合實例。

如果你想為應用程式中的每個模型使用自訂集合，你應該在所有應用程式模型都擴充的基礎模型類別上定義 `newCollection` 方法。

