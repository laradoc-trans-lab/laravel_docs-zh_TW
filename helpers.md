# 輔助函式

- [簡介](#introduction)
- [可用方法](#available-methods)
- [其他工具](#other-utilities)
    - [效能基準測試](#benchmarking)
    - [日期](#dates)
    - [延遲函式](#deferred-functions)
    - [抽獎](#lottery)
    - [管線](#pipeline)
    - [暫停](#sleep)
    - [時間限制](#timebox)
    - [URI](#uri)

<a name="introduction"></a>
## 簡介

Laravel 包含了各種全域的「輔助函式」PHP 函式。其中許多函式由框架本身使用；然而，如果您覺得方便，也可以在自己的應用程式中自由使用它們。

<a name="available-methods"></a>
## 可用方法

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>

<a name="arrays-and-objects-method-list"></a>
### 陣列與物件

<div class="collection-method-list" markdown="1">

[Arr::accessible](#method-array-accessible)
[Arr::add](#method-array-add)
[Arr::array](#method-array-array)
[Arr::boolean](#method-array-boolean)
[Arr::collapse](#method-array-collapse)
[Arr::crossJoin](#method-array-crossjoin)
[Arr::divide](#method-array-divide)
[Arr::dot](#method-array-dot)
[Arr::every](#method-array-every)
[Arr::except](#method-array-except)
[Arr::exists](#method-array-exists)
[Arr::first](#method-array-first)
[Arr::flatten](#method-array-flatten)
[Arr::float](#method-array-float)
[Arr::forget](#method-array-forget)
[Arr::from](#method-array-from)
[Arr::get](#method-array-get)
[Arr::has](#method-array-has)
[Arr::hasAll](#method-array-hasall)
[Arr::hasAny](#method-array-hasany)
[Arr::integer](#method-array-integer)
[Arr::isAssoc](#method-array-isassoc)
[Arr::isList](#method-array-islist)
[Arr::join](#method-array-join)
[Arr::keyBy](#method-array-keyby)
[Arr::last](#method-array-last)
[Arr::map](#method-array-map)
[Arr::mapSpread](#method-array-map-spread)
[Arr::mapWithKeys](#method-array-map-with-keys)
[Arr::only](#method-array-only)
[Arr::partition](#method-array-partition)
[Arr::pluck](#method-array-pluck)
[Arr::prepend](#method-array-prepend)
[Arr::prependKeysWith](#method-array-prependkeyswith)
[Arr::pull](#method-array-pull)
[Arr::push](#method-array-push)
[Arr::query](#method-array-query)
[Arr::random](#method-array-random)
[Arr::reject](#method-array-reject)
[Arr::select](#method-array-select)
[Arr::set](#method-array-set)
[Arr::shuffle](#method-array-shuffle)
[Arr::sole](#method-array-sole)
[Arr::some](#method-array-some)
[Arr::sort](#method-array-sort)
[Arr::sortDesc](#method-array-sort-desc)
[Arr::sortRecursive](#method-array-sort-recursive)
[Arr::string](#method-array-string)
[Arr::take](#method-array-take)
[Arr::toCssClasses](#method-array-to-css-classes)
[Arr::toCssStyles](#method-array-to-css-styles)
[Arr::undot](#method-array-undot)
[Arr::where](#method-array-where)
[Arr::whereNotNull](#method-array-where-not-null)
[Arr::wrap](#method-array-wrap)
[data_fill](#method-data-fill)
[data_get](#method-data-get)
[data_set](#method-data-set)
[data_forget](#method-data-forget)
[head](#method-head)
[last](#method-last)
</div>

<a name="numbers-method-list"></a>
### 數字

<div class="collection-method-list" markdown="1">

[Number::abbreviate](#method-number-abbreviate)
[Number::clamp](#method-number-clamp)
[Number::currency](#method-number-currency)
[Number::defaultCurrency](#method-default-currency)
[Number::defaultLocale](#method-default-locale)
[Number::fileSize](#method-number-file-size)
[Number::forHumans](#method-number-for-humans)
[Number::format](#method-number-format)
[Number::ordinal](#method-number-ordinal)
[Number::pairs](#method-number-pairs)
[Number::parseInt](#method-number-parse-int)
[Number::parseFloat](#method-number-parse-float)
[Number::percentage](#method-number-percentage)
[Number::spell](#method-number-spell)
[Number::spellOrdinal](#method-number-spell-ordinal)
[Number::trim](#method-number-trim)
[Number::useLocale](#method-number-use-locale)
[Number::withLocale](#method-number-with-locale)
[Number::useCurrency](#method-number-use-currency)
[Number::withCurrency](#method-number-with-currency)

</div>

<a name="paths-method-list"></a>
### 路徑

<div class="collection-method-list" markdown="1">

[app_path](#method-app-path)
[base_path](#method-base-path)
[config_path](#method-config-path)
[database_path](#method-database-path)
[lang_path](#method-lang-path)
[public_path](#method-public-path)
[resource_path](#method-resource-path)
[storage_path](#method-storage-path)

</div>

<a name="urls-method-list"></a>
### URL

<div class="collection-method-list" markdown="1">

[action](#method-action)
[asset](#method-asset)
[route](#method-route)
[secure_asset](#method-secure-asset)
[secure_url](#method-secure-url)
[to_action](#method-to-action)
[to_route](#method-to-route)
[uri](#method-uri)
[url](#method-url)

</div>

<a name="miscellaneous-method-list"></a>
### 其他

<div class="collection-method-list" markdown="1">

[abort](#method-abort)
[abort_if](#method-abort-if)
[abort_unless](#method-abort-unless)
[app](#method-app)
[auth](#method-auth)
[back](#method-back)
[bcrypt](#method-bcrypt)
[blank](#method-blank)
[broadcast](#method-broadcast)
[broadcast_if](#method-broadcast-if)
[broadcast_unless](#method-broadcast-unless)
[cache](#method-cache)
[class_uses_recursive](#method-class-uses-recursive)
[collect](#method-collect)
[config](#method-config)
[context](#method-context)
[cookie](#method-cookie)
[csrf_field](#method-csrf-field)
[csrf_token](#method-csrf-token)
[decrypt](#method-decrypt)
[dd](#method-dd)
[dispatch](#method-dispatch)
[dispatch_sync](#method-dispatch-sync)
[dump](#method-dump)
[encrypt](#method-encrypt)
[env](#method-env)
[event](#method-event)
[fake](#method-fake)
[filled](#method-filled)
[info](#method-info)
[literal](#method-literal)
[logger](#method-logger)
[method_field](#method-method-field)
[now](#method-now)
[old](#method-old)
[once](#method-once)
[optional](#method-optional)
[policy](#method-policy)
[redirect](#method-redirect)
[report](#method-report)
[report_if](#method-report-if)
[report_unless](#method-report-unless)
[request](#method-request)
[rescue](#method-rescue)
[resolve](#method-resolve)
[response](#method-response)
[retry](#method-retry)
[session](#method-session)
[tap](#method-tap)
[throw_if](#method-throw-if)
[throw_unless](#method-throw-unless)
[today](#method-today)
[trait_uses_recursive](#method-trait-uses-recursive)
[transform](#method-transform)
[validator](#method-validator)
[value](#method-value)
[view](#method-view)
[with](#method-with)
[when](#method-when)

</div>

<a name="arrays"></a>
## 陣列與物件

<a name="method-array-accessible"></a>
#### `Arr::accessible()` {.collection-method .first-collection-method}

`Arr::accessible` 方法會判斷給定的值是否可透過陣列存取：

```php
use Illuminate\Support\Arr;
use Illuminate\Support\Collection;

$isAccessible = Arr::accessible(['a' => 1, 'b' => 2]);

// true

$isAccessible = Arr::accessible(new Collection);

// true

$isAccessible = Arr::accessible('abc');

// false

$isAccessible = Arr::accessible(new stdClass);

// false
```

<a name="method-array-add"></a>
#### `Arr::add()` {.collection-method}

`Arr::add` 方法會在給定鍵不存在於陣列中或設為 `null` 時，將給定的鍵/值對新增至陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::add(['name' => 'Desk'], 'price', 100);

// ['name' => 'Desk', 'price' => 100]

$array = Arr::add(['name' => 'Desk', 'price' => null], 'price', 100);

// ['name' => 'Desk', 'price' => 100]
```

<a name="method-array-array"></a>
#### `Arr::array()` {.collection-method}

`Arr::array` 方法使用「點」標記法 (與 [Arr::get()](#method-array-get) 相同) 從深度巢狀陣列中擷取值，但如果請求的值不是 `array`，則會拋出 `InvalidArgumentException`：

```
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

$value = Arr::array($array, 'languages');

// ['PHP', 'Ruby']

$value = Arr::array($array, 'name');

// throws InvalidArgumentException
```

<a name="method-array-boolean"></a>
#### `Arr::boolean()` {.collection-method}

`Arr::boolean` 方法使用「點」標記法 (與 [Arr::get()](#method-array-get) 相同) 從深度巢狀陣列中擷取值，但如果請求的值不是 `boolean`，則會拋出 `InvalidArgumentException`：

```
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'available' => true];

$value = Arr::boolean($array, 'available');

// true

$value = Arr::boolean($array, 'name');

// throws InvalidArgumentException
```


<a name="method-array-collapse"></a>
#### `Arr::collapse()` {.collection-method}

`Arr::collapse` 方法會將陣列或 Collection 的陣列合併為單一陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::collapse([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

<a name="method-array-crossjoin"></a>
#### `Arr::crossJoin()` {.collection-method}

`Arr::crossJoin` 方法會對給定的陣列進行交叉連接，返回包含所有可能排列的笛卡爾積：

```php
use Illuminate\Support\Arr;

$matrix = Arr::crossJoin([1, 2], ['a', 'b']);

/*
    [
        [1, 'a'],
        [1, 'b'],
        [2, 'a'],
        [2, 'b'],
    ]
*/

$matrix = Arr::crossJoin([1, 2], ['a', 'b'], ['I', 'II']);

/*
    [
        [1, 'a', 'I'],
        [1, 'a', 'II'],
        [1, 'b', 'I'],
        [1, 'b', 'II'],
        [2, 'a', 'I'],
        [2, 'a', 'II'],
        [2, 'b', 'I'],
        [2, 'b', 'II'],
    ]
*/
```

<a name="method-array-divide"></a>
#### `Arr::divide()` {.collection-method}

`Arr::divide` 方法會返回兩個陣列：一個包含給定陣列的鍵，另一個包含其值：

```php
use Illuminate\Support\Arr;

[$keys, $values] = Arr::divide(['name' => 'Desk']);

// $keys: ['name']

// $values: ['Desk']
```

<a name="method-array-dot"></a>
#### `Arr::dot()` {.collection-method}

`Arr::dot` 方法會將多維陣列扁平化為單層陣列，並使用「點」標記法來表示深度：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$flattened = Arr::dot($array);

// ['products.desk.price' => 100]
```

<a name="method-array-every"></a>
#### `Arr::every()` {.collection-method}

`Arr::every` 方法會確保陣列中的所有值都通過給定的真值測試：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3];

Arr::every($array, fn ($i) => $i > 0);

// true

Arr::every($array, fn ($i) => $i > 2);

// false
```

<a name="method-array-except"></a>
#### `Arr::except()` {.collection-method}

`Arr::except` 方法會從陣列中移除給定的鍵/值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100];

$filtered = Arr::except($array, ['price']);

// ['name' => 'Desk']
```

<a name="method-array-exists"></a>
#### `Arr::exists()` {.collection-method}

`Arr::exists` 方法會檢查給定的鍵是否存在於提供的陣列中：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'John Doe', 'age' => 17];

$exists = Arr::exists($array, 'name');

// true

$exists = Arr::exists($array, 'salary');

// false
```

<a name="method-array-first"></a>
#### `Arr::first()` {.collection-method}

`Arr::first` 方法會返回陣列中第一個通過給定真值測試的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300];

$first = Arr::first($array, function (int $value, int $key) {
    return $value >= 150;
});

// 200
```

也可以將預設值作為第三個參數傳遞給該方法。如果沒有值通過真值測試，則會返回此值：

```php
use Illuminate\Support\Arr;

$first = Arr::first($array, $callback, $default);
```

<a name="method-array-flatten"></a>
#### `Arr::flatten()` {.collection-method}

`Arr::flatten` 方法會將多維陣列扁平化為單層陣列：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

$flattened = Arr::flatten($array);

// ['Joe', 'PHP', 'Ruby']
```

<a name="method-array-float"></a>
#### `Arr::float()` {.collection-method}

`Arr::float` 方法使用「點」標記法 (與 [Arr::get()](#method-array-get) 相同) 從深度巢狀陣列中擷取值，但如果請求的值不是 `float`，則會拋出 `InvalidArgumentException`：

```
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'balance' => 123.45];

$value = Arr::float($array, 'balance');

// 123.45

$value = Arr::float($array, 'name');

// throws InvalidArgumentException
```

<a name="method-array-forget"></a>
#### `Arr::forget()` {.collection-method}

`Arr::forget` 方法使用「點」標記法從深度巢狀陣列中移除給定的鍵/值對：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::forget($array, 'products.desk');

// ['products' => []]
```

<a name="method-array-from"></a>
#### `Arr::from()` {.collection-method}

`Arr::from` 方法會將各種輸入類型轉換為純 PHP 陣列。它支援多種輸入類型，包括陣列、物件以及多個常見的 Laravel 介面，例如 `Arrayable`、`Enumerable`、`Jsonable` 和 `JsonSerializable`。此外，它還處理 `Traversable` 和 `WeakMap` 實例：

```php
use Illuminate\Support\Arr;

Arr::from((object) ['foo' => 'bar']); // ['foo' => 'bar']

class TestJsonableObject implements Jsonable
{
    public function toJson($options = 0)
    {
        return json_encode(['foo' => 'bar']);
    }
}

Arr::from(new TestJsonableObject); // ['foo' => 'bar']
```

<a name="method-array-get"></a>
#### `Arr::get()` {.collection-method}

`Arr::get` 方法使用「點」標記法從深度巢狀陣列中擷取值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$price = Arr::get($array, 'products.desk.price');

// 100
```

`Arr::get` 方法也接受一個預設值，如果陣列中不存在指定的鍵，則會返回該預設值：

```php
use Illuminate\Support\Arr;

$discount = Arr::get($array, 'products.desk.discount', 0);

// 0
```

<a name="method-array-has"></a>
#### `Arr::has()` {.collection-method}

`Arr::has` 方法使用「點」標記法檢查陣列中是否存在給定的項目：

```php
use Illuminate\Support\Arr;

$array = ['product' => ['name' => 'Desk', 'price' => 100]];

$contains = Arr::has($array, 'product.name');

// true

$contains = Arr::has($array, ['product.price', 'product.discount']);

// false
```

<a name="method-array-hasall"></a>
#### `Arr::hasAll()` {.collection-method}

`Arr::hasAll` 方法使用「點」標記法判斷給定陣列中是否存在所有指定的鍵：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Taylor', 'language' => 'PHP'];

Arr::hasAll($array, ['name']); // true
Arr::hasAll($array, ['name', 'language']); // true
Arr::hasAll($array, ['name', 'IDE']); // false
```

<a name="method-array-hasany"></a>
#### `Arr::hasAny()` {.collection-method}

`Arr::hasAny` 方法使用「點」標記法檢查給定集合中的任何項目是否存在於陣列中：

```php
use Illuminate\Support\Arr;

$array = ['product' => ['name' => 'Desk', 'price' => 100]];

$contains = Arr::hasAny($array, 'product.name');

// true

$contains = Arr::hasAny($array, ['product.name', 'product.discount']);

// true

$contains = Arr::hasAny($array, ['category', 'product.discount']);

// false
```

<a name="method-array-integer"></a>
#### `Arr::integer()` {.collection-method}

`Arr::integer` 方法使用「點」標記法 (與 [Arr::get()](#method-array-get) 相同) 從深度巢狀陣列中擷取值，但如果請求的值不是 `int`，則會拋出 `InvalidArgumentException`：

```
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'age' => 42];

$value = Arr::integer($array, 'age');

// 42

$value = Arr::integer($array, 'name');

// throws InvalidArgumentException
```

<a name="method-array-isassoc"></a>
#### `Arr::isAssoc()` {.collection-method}

`Arr::isAssoc` 方法會在給定陣列是關聯陣列時返回 `true`。如果陣列沒有從零開始的連續數字鍵，則被視為「關聯」陣列：

```php
use Illuminate\Support\Arr;

$isAssoc = Arr::isAssoc(['product' => ['name' => 'Desk', 'price' => 100]]);

// true

$isAssoc = Arr::isAssoc([1, 2, 3]);

// false
```

<a name="method-array-islist"></a>
#### `Arr::isList()` {.collection-method}

`Arr::isList` 方法會在給定陣列的鍵是從零開始的連續整數時返回 `true`：

```php
use Illuminate\Support\Arr;

$isList = Arr::isList(['foo', 'bar', 'baz']);

// true

$isList = Arr::isList(['product' => ['name' => 'Desk', 'price' => 100]]);

// false
```

<a name="method-array-join"></a>
#### `Arr::join()` {.collection-method}

`Arr::join` 方法會將陣列元素與字串連接起來。使用此方法的第三個參數，您還可以指定陣列最後一個元素的連接字串：

```php
use Illuminate\Support\Arr;

$array = ['Tailwind', 'Alpine', 'Laravel', 'Livewire'];

$joined = Arr::join($array, ', ');

// Tailwind, Alpine, Laravel, Livewire

$joined = Arr::join($array, ', ', ', and ');

// Tailwind, Alpine, Laravel, and Livewire
```

<a name="method-array-keyby"></a>
#### `Arr::keyBy()` {.collection-method}

`Arr::keyBy` 方法會根據給定的鍵對陣列進行鍵控。如果多個項目具有相同的鍵，則只有最後一個會出現在新陣列中：

```php
use Illuminate\Support\Arr;

$array = [
    ['product_id' => 'prod-100', 'name' => 'Desk'],
    ['product_id' => 'prod-200', 'name' => 'Chair'],
];

$keyed = Arr::keyBy($array, 'product_id');

/*
    [
        'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
        'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]
*/
```

<a name="method-array-last"></a>
#### `Arr::last()` {.collection-method}

`Arr::last` 方法會返回陣列中最後一個通過給定真值測試的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300, 110];

$last = Arr::last($array, function (int $value, int $key) {
    return $value >= 150;
});

// 300
```

可以將預設值作為第三個參數傳遞給該方法。如果沒有值通過真值測試，則會返回此值：

```php
use Illuminate\Support\Arr;

$last = Arr::last($array, $callback, $default);
```

<a name="method-array-map"></a>
#### `Arr::map()` {.collection-method}

`Arr::map` 方法會遍歷陣列，並將每個值和鍵傳遞給給定的回呼函式。陣列值會被回呼函式返回的值替換：

```php
use Illuminate\Support\Arr;

$array = ['first' => 'james', 'last' => 'kirk'];

$mapped = Arr::map($array, function (string $value, string $key) {
    return ucfirst($value);
});

// ['first' => 'James', 'last' => 'Kirk']
```

<a name="method-array-map-spread"></a>
#### `Arr::mapSpread()` {.collection-method}

`Arr::mapSpread` 方法會遍歷陣列，將每個巢狀項目值傳遞給給定的閉包。閉包可以自由修改項目並返回它，從而形成一個新的修改後項目陣列：

```php
use Illuminate\Support\Arr;

$array = [
    [0, 1],
    [2, 3],
    [4, 5],
    [6, 7],
    [8, 9],
];

$mapped = Arr::mapSpread($array, function (int $even, int $odd) {
    return $even + $odd;
});

/*
    [1, 5, 9, 13, 17]
*/
```

<a name="method-array-map-with-keys"></a>
#### `Arr::mapWithKeys()` {.collection-method}

`Arr::mapWithKeys` 方法會遍歷陣列，並將每個值傳遞給給定的回呼函式。回呼函式應返回一個包含單一鍵/值對的關聯陣列：

```php
use Illuminate\Support\Arr;

$array = [
    [
        'name' => 'John',
        'department' => 'Sales',
        'email' => 'john @example.com',
    ],
    [
        'name' => 'Jane',
        'department' => 'Marketing',
        'email' => 'jane @example.com',
    ]
];

$mapped = Arr::mapWithKeys($array, function (array $item, int $key) {
    return [$item['email'] => $item['name']];
});

/*
    [
        'john @example.com' => 'John',
        'jane @example.com' => 'Jane',
    ]
*/
```

<a name="method-array-only"></a>
#### `Arr::only()` {.collection-method}

`Arr::only` 方法會從給定陣列中僅返回指定的鍵/值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100, 'orders' => 10];

$slice = Arr::only($array, ['name', 'price']);

// ['name' => 'Desk', 'price' => 100]
```

<a name="method-array-partition"></a>
#### `Arr::partition()` {.collection-method}

`Arr::partition` 方法可以與 PHP 陣列解構結合使用，以將通過給定真值測試的元素與未通過的元素分開：

```php
<?php

use Illuminate\Support\Arr;

$numbers = [1, 2, 3, 4, 5, 6];

[$underThree, $equalOrAboveThree] = Arr::partition($numbers, function (int $i) {
    return $i < 3;
});

dump($underThree);

// [1, 2]

dump($equalOrAboveThree);

// [3, 4, 5, 6]
```

<a name="method-array-pluck"></a>
#### `Arr::pluck()` {.collection-method}

`Arr::pluck` 方法會從陣列中擷取給定鍵的所有值：

```php
use Illuminate\Support\Arr;

$array = [
    ['developer' => ['id' => 1, 'name' => 'Taylor']],
    ['developer' => ['id' => 2, 'name' => 'Abigail']],
];

$names = Arr::pluck($array, 'developer.name');

// ['Taylor', 'Abigail']
```

您也可以指定希望結果列表如何鍵控：

```php
use Illuminate\Support\Arr;

$names = Arr::pluck($array, 'developer.name', 'developer.id');

// [1 => 'Taylor', 2 => 'Abigail']
```

<a name="method-array-prepend"></a>
#### `Arr::prepend()` {.collection-method}

`Arr::prepend` 方法會將項目推送到陣列的開頭：

```php
use Illuminate\Support\Arr;

$array = ['one', 'two', 'three', 'four'];

$array = Arr::prepend($array, 'zero');

// ['zero', 'one', 'two', 'three', 'four']
```

如果需要，您可以指定應用於該值的鍵：

```php
use Illuminate\Support\Arr;

$array = ['price' => 100];

$array = Arr::prepend($array, 'Desk', 'name');

// ['name' => 'Desk', 'price' => 100]
```

<a name="method-array-prependkeyswith"></a>
#### `Arr::prependKeysWith()` {.collection-method}

`Arr::prependKeysWith` 會將關聯陣列的所有鍵名加上給定的前綴：

```php
use Illuminate\Support\Arr;

$array = [
    'name' => 'Desk',
    'price' => 100,
];

$keyed = Arr::prependKeysWith($array, 'product.');

/*
    [
        'product.name' => 'Desk',
        'product.price' => 100,
    ]
*/
```

<a name="method-array-pull"></a>
#### `Arr::pull()` {.collection-method}

`Arr::pull` 方法會返回並從陣列中移除一個鍵/值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100];

$name = Arr::pull($array, 'name');

// $name: Desk

// $array: ['price' => 100]
```

可以將預設值作為第三個參數傳遞給該方法。如果鍵不存在，則會返回此值：

```php
use Illuminate\Support\Arr;

$value = Arr::pull($array, $key, $default);
```

<a name="method-array-push"></a>
#### `Arr::push()` {.collection-method}

`Arr::push` 方法使用「點」標記法將項目推送到陣列中。如果給定鍵處不存在陣列，則會建立它：

```php
use Illuminate\Support\Arr;

$array = [];

Arr::push($array, 'office.furniture', 'Desk');

// $array: ['office' => ['furniture' => ['Desk']]]
```

<a name="method-array-query"></a>
#### `Arr::query()` {.collection-method}

`Arr::query` 方法會將陣列轉換為查詢字串：

```php
use Illuminate\Support\Arr;

$array = [
    'name' => 'Taylor',
    'order' => [
        'column' => 'created_at',
        'direction' => 'desc'
    ]
];

Arr::query($array);

// name=Taylor&order[column]=created_at&order[direction]=desc
```

<a name="method-array-random"></a>
#### `Arr::random()` {.collection-method}

`Arr::random` 方法會從陣列中返回一個隨機值：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3, 4, 5];

$random = Arr::random($array);

// 4 - (隨機擷取)
```

您也可以將要返回的項目數指定為可選的第二個參數。請注意，提供此參數將返回一個陣列，即使只想要一個項目：

```php
use Illuminate\Support\Arr;

$items = Arr::random($array, 2);

// [2, 5] - (隨機擷取)
```

<a name="method-array-reject"></a>
#### `Arr::reject()` {.collection-method}

`Arr::reject` 方法會使用給定的閉包從陣列中移除項目：

```php
use Illuminate\Support\Arr;

$array = [100, '200', 300, '400', 500];

$filtered = Arr::reject($array, function (string|int $value, int $key) {
    return is_string($value);
});

// [0 => 100, 2 => 300, 4 => 500]
```

<a name="method-array-select"></a>
#### `Arr::select()` {.collection-method}

`Arr::select` 方法會從陣列中選取一個值陣列：

```php
use Illuminate\Support\Arr;

$array = [
    ['id' => 1, 'name' => 'Desk', 'price' => 200],
    ['id' => 2, 'name' => 'Table', 'price' => 150],
    ['id' => 3, 'name' => 'Chair', 'price' => 300],
];

Arr::select($array, ['name', 'price']);

// [['name' => 'Desk', 'price' => 200], ['name' => 'Table', 'price' => 150], ['name' => 'Chair', 'price' => 300]]
```

<a name="method-array-set"></a>
#### `Arr::set()` {.collection-method}

`Arr::set` 方法使用「點」標記法在深度巢狀陣列中設定值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::set($array, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```

<a name="method-array-shuffle"></a>
#### `Arr::shuffle()` {.collection-method}

`Arr::shuffle` 方法會隨機打亂陣列中的項目：

```php
use Illuminate\Support\Arr;

$array = Arr::shuffle([1, 2, 3, 4, 5]);

// [3, 2, 5, 1, 4] - (隨機產生)
```

<a name="method-array-sole"></a>
#### `Arr::sole()` {.collection-method}

`Arr::sole` 方法會使用給定的閉包從陣列中擷取單一值。如果陣列中有多個值符合給定的真值測試，則會拋出 `Illuminate\Support\MultipleItemsFoundException` 異常。如果沒有值符合真值測試，則會拋出 `Illuminate\Support\ItemNotFoundException` 異常：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$value = Arr::sole($array, fn (string $value) => $value === 'Desk');

// 'Desk'
```

<a name="method-array-some"></a>
#### `Arr::some()` {.collection-method}

`Arr::some` 方法會確保陣列中至少有一個值通過給定的真值測試：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3];

Arr::some($array, fn ($i) => $i > 2);

// true
```

<a name="method-array-sort"></a>
#### `Arr::sort()` {.collection-method}

`Arr::sort` 方法會根據其值對陣列進行排序：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$sorted = Arr::sort($array);

// ['Chair', 'Desk', 'Table']
```

您也可以根據給定閉包的結果對陣列進行排序：

```php
use Illuminate\Support\Arr;

$array = [
    ['name' => 'Desk'],
    ['name' => 'Table'],
    ['name' => 'Chair'],
];

$sorted = array_values(Arr::sort($array, function (array $value) {
    return $value['name'];
}));

/*
    [
        ['name' => 'Chair'],
        ['name' => 'Desk'],
        ['name' => 'Table'],
    ]
*/
```

<a name="method-array-sort-desc"></a>
#### `Arr::sortDesc()` {.collection-method}

`Arr::sortDesc` 方法會根據其值以遞減順序對陣列進行排序：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$sorted = Arr::sortDesc($array);

// ['Table', 'Desk', 'Chair']
```

您也可以根據給定閉包的結果對陣列進行排序：

```php
use Illuminate\Support\Arr;

$array = [
    ['name' => 'Desk'],
    ['name' => 'Table'],
    ['name' => 'Chair'],
];

$sorted = array_values(Arr::sortDesc($array, function (array $value) {
    return $value['name'];
}));

/*
    [
        ['name' => 'Table'],
        ['name' => 'Desk'],
        ['name' => 'Chair'],
    ]
*/
```

<a name="method-array-sort-recursive"></a>
#### `Arr::sortRecursive()` {.collection-method}

`Arr::sortRecursive` 方法會遞迴地對陣列進行排序，對數字索引的子陣列使用 `sort` 函式，對關聯子陣列使用 `ksort` 函式：

```php
use Illuminate\Support\Arr;

$array = [
    ['Roman', 'Taylor', 'Li'],
    ['PHP', 'Ruby', 'JavaScript'],
    ['one' => 1, 'two' => 2, 'three' => 3],
];

$sorted = Arr::sortRecursive($array);

/*
    [
        ['JavaScript', 'PHP', 'Ruby'],
        ['one' => 1, 'three' => 3, 'two' => 2],
        ['Li', 'Roman', 'Taylor'],
    ]
*/
```

如果您希望結果以遞減順序排序，可以使用 `Arr::sortRecursiveDesc` 方法。

```php
$sorted = Arr::sortRecursiveDesc($array);
```

<a name="method-array-string"></a>
#### `Arr::string()` {.collection-method}

`Arr::string` 方法使用「點」標記法 (與 [Arr::get()](#method-array-get) 相同) 從深度巢狀陣列中擷取值，但如果請求的值不是 `string`，則會拋出 `InvalidArgumentException`：

```
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

$value = Arr::string($array, 'name');

// Joe

$value = Arr::string($array, 'languages');

// throws InvalidArgumentException
```

<a name="method-array-take"></a>
#### `Arr::take()` {.collection-method}

`Arr::take` 方法會返回一個包含指定項目數的新陣列：

```php
use Illuminate\Support\Arr;

$array = [0, 1, 2, 3, 4, 5];

$chunk = Arr::take($array, 3);

// [0, 1, 2]
```

您也可以傳遞一個負整數，以從陣列末尾擷取指定數量的項目：

```php
$array = [0, 1, 2, 3, 4, 5];

$chunk = Arr::take($array, -2);

// [4, 5]
```

<a name="method-array-to-css-classes"></a>
#### `Arr::toCssClasses()` {.collection-method}

`Arr::toCssClasses` 方法會條件式地編譯 CSS 類別字串。該方法接受一個類別陣列，其中陣列鍵包含您希望新增的類別，而值是一個布林表達式。如果陣列元素具有數字鍵，它將始終包含在渲染的類別列表中：

```php
use Illuminate\Support\Arr;

$isActive = false;
$hasError = true;

$array = ['p-4', 'font-bold' => $isActive, 'bg-red' => $hasError];

$classes = Arr::toCssClasses($array);

/*
    'p-4 bg-red'
*/
```

<a name="method-array-to-css-styles"></a>
#### `Arr::toCssStyles()` {.collection-method}

`Arr::toCssStyles` 會條件式地編譯 CSS 樣式字串。該方法接受一個類別陣列，其中陣列鍵包含您希望新增的類別，而值是一個布林表達式。如果陣列元素具有數字鍵，它將始終包含在渲染的類別列表中：

```php
use Illuminate\Support\Arr;

$hasColor = true;

$array = ['background-color: blue', 'color: blue' => $hasColor];

$classes = Arr::toCssStyles($array);

/*
    'background-color: blue; color: blue;'
*/
```

此方法為 Laravel 的功能提供支援，允許 [將類別與 Blade 元件的屬性包合併](/docs/{{version}}/blade#conditionally-merge-classes) 以及 ` @class` [Blade 指令](/docs/{{version}}/blade#conditional-classes)。

<a name="method-array-undot"></a>
#### `Arr::undot()` {.collection-method}

`Arr::undot` 方法會將使用「點」標記法的單維陣列展開為多維陣列：

```php
use Illuminate\Support\Arr;

$array = [
    'user.name' => 'Kevin Malone',
    'user.occupation' => 'Accountant',
];

$array = Arr::undot($array);

// ['user' => ['name' => 'Kevin Malone', 'occupation' => 'Accountant']]
```

<a name="method-array-where"></a>
#### `Arr::where()` {.collection-method}

`Arr::where` 方法會使用給定的閉包過濾陣列：

```php
use Illuminate\Support\Arr;

$array = [100, '200', 300, '400', 500];

$filtered = Arr::where($array, function (string|int $value, int $key) {
    return is_string($value);
});

// [1 => '200', 3 => '400']
```

<a name="method-array-where-not-null"></a>
#### `Arr::whereNotNull()` {.collection-method}

`Arr::whereNotNull` 方法會從給定陣列中移除所有 `null` 值：

```php
use Illuminate\Support\Arr;

$array = [0, null];

$filtered = Arr::whereNotNull($array);

// [0 => 0]
```

<a name="method-array-wrap"></a>
#### `Arr::wrap()` {.collection-method}

`Arr::wrap` 方法會將給定的值包裝在陣列中。如果給定的值已經是陣列，則會不經修改地返回：

```php
use Illuminate\Support\Arr;

$string = 'Laravel';

$array = Arr::wrap($string);

// ['Laravel']
```

如果給定的值是 `null`，則會返回一個空陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::wrap(null);

// []
```

<a name="method-data-fill"></a>
#### `data_fill()` {.collection-method}

`data_fill` 函式使用「點」標記法在巢狀陣列或物件中設定遺失的值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_fill($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 100]]]

data_fill($data, 'products.desk.discount', 10);

// ['products' => ['desk' => ['price' => 100, 'discount' => 10]]]
```

此函式也接受星號作為萬用字元，並會相應地填入目標：

```php
$data = [
    'products' => [
        ['name' => 'Desk 1', 'price' => 100],
        ['name' => 'Desk 2'],
    ],
];

data_fill($data, 'products.*.price', 200);

/*
    [
        'products' => [
            ['name' => 'Desk 1', 'price' => 100],
            ['name' => 'Desk 2', 'price' => 200],
        ],
    ]
*/
```

<a name="method-data-get"></a>
#### `data_get()` {.collection-method}

`data_get` 函式使用「點」標記法從巢狀陣列或物件中擷取值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

$price = data_get($data, 'products.desk.price');

// 100
```

`data_get` 函式也接受一個預設值，如果找不到指定的鍵，則會返回該預設值：

```php
$discount = data_get($data, 'products.desk.discount', 0);

// 0
```

此函式也接受使用星號的萬用字元，可以針對陣列或物件的任何鍵：

```php
$data = [
    'product-one' => ['name' => 'Desk 1', 'price' => 100],
    'product-two' => ['name' => 'Desk 2', 'price' => 150],
];

data_get($data, '*.name');

// ['Desk 1', 'Desk 2'];
```

`{first}` 和 `{last}` 佔位符可用於擷取陣列中的第一個或最後一個項目：

```php
$flight = [
    'segments' => [
        ['from' => 'LHR', 'departure' => '9:00', 'to' => 'IST', 'arrival' => '15:00'],
        ['from' => 'IST', 'departure' => '16:00', 'to' => 'PKX', 'arrival' => '20:00'],
    ],
];

data_get($flight, 'segments.{first}.arrival');

// 15:00
```

<a name="method-data-set"></a>
#### `data_set()` {.collection-method}

`data_set` 函式使用「點」標記法在巢狀陣列或物件中設定值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```

此函式也接受使用星號的萬用字元，並會相應地設定目標上的值：

```php
$data = [
    'products' => [
        ['name' => 'Desk 1', 'price' => 100],
        ['name' => 'Desk 2', 'price' => 150],
    ],
];

data_set($data, 'products.*.price', 200);

/*
    'products' => [
        ['name' => 'Desk 1', 'price' => 200],
        ['name' => 'Desk 2', 'price' => 200],
    ],
*/
```

預設情況下，任何現有值都會被覆寫。如果您只想在值不存在時才設定值，可以將 `false` 作為第四個參數傳遞給函式：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200, overwrite: false);

// ['products' => ['desk' => ['price' => 100]]]
```

<a name="method-data-forget"></a>
#### `data_forget()` {.collection-method}

`data_forget` 函式使用「點」標記法從巢狀陣列或物件中移除值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_forget($data, 'products.desk.price');

// ['products' => ['desk' => []]]
```

此函式也接受使用星號的萬用字元，並會相應地移除目標上的值：

```php
$data = [
    'products' => [
        ['name' => 'Desk 1', 'price' => 100],
        ['name' => 'Desk 2', 'price' => 150],
    ],
];

data_forget($data, 'products.*.price');

/*
    [
        'products' => [
            ['name' => 'Desk 1'],
            ['name' => 'Desk 2'],
        ],
    ]
*/
```

<a name="method-head"></a>
#### `head()` {.collection-method}

`head` 函式會返回給定陣列中的第一個元素。如果陣列為空，則會返回 `false`：

```php
$array = [100, 200, 300];

$first = head($array);

// 100
```

<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 函式會返回給定陣列中的最後一個元素。如果陣列為空，則會返回 `false`：

```php
$array = [100, 200, 300];

$last = last($array);

// 300
```

<a name="numbers"></a>
## 數字

<a name="method-number-abbreviate"></a>
#### `Number::abbreviate()` {.collection-method}

`Number::abbreviate` 方法會返回所提供數值的易讀格式，並帶有單位縮寫：

```php
use Illuminate\Support\Number;

$number = Number::abbreviate(1000);

// 1K

$number = Number::abbreviate(489939);

// 490K

$number = Number::abbreviate(1230000, precision: 2);

// 1.23M
```

<a name="method-number-clamp"></a>
#### `Number::clamp()` {.collection-method}

`Number::clamp` 方法會確保給定數字保持在指定範圍內。如果數字低於最小值，則返回最小值。如果數字高於最大值，則返回最大值：

```php
use Illuminate\Support\Number;

$number = Number::clamp(105, min: 10, max: 100);

// 100

$number = Number::clamp(5, min: 10, max: 100);

// 10

$number = Number::clamp(10, min: 10, max: 100);

// 10

$number = Number::clamp(20, min: 10, max: 100);

// 20
```

<a name="method-number-currency"></a>
#### `Number::currency()` {.collection-method}

`Number::currency` 方法會將給定值的貨幣表示形式作為字串返回：

```php
use Illuminate\Support\Number;

$currency = Number::currency(1000);

// $1,000.00

$currency = Number::currency(1000, in: 'EUR');

// €1,000.00

$currency = Number::currency(1000, in: 'EUR', locale: 'de');

// 1.000,00 €

$currency = Number::currency(1000, in: 'EUR', locale: 'de', precision: 0);

// 1.000 €
```

<a name="method-default-currency"></a>
#### `Number::defaultCurrency()` {.collection-method}

`Number::defaultCurrency` 方法會返回 `Number` 類別正在使用的預設貨幣：

```php
use Illuminate\Support\Number;

$currency = Number::defaultCurrency();

// USD
```

<a name="method-default-locale"></a>
#### `Number::defaultLocale()` {.collection-method}

`Number::defaultLocale` 方法會返回 `Number` 類別正在使用的預設語系：

```php
use Illuminate\Support\Number;

$locale = Number::defaultLocale();

// en
```

<a name="method-number-file-size"></a>
#### `Number::fileSize()` {.collection-method}

`Number::fileSize` 方法會將給定位元組值的檔案大小表示形式作為字串返回：

```php
use Illuminate\Support\Number;

$size = Number::fileSize(1024);

// 1 KB

$size = Number::fileSize(1024 * 1024);

// 1 MB

$size = Number::fileSize(1024, precision: 2);

// 1.00 KB
```

<a name="method-number-for-humans"></a>
#### `Number::forHumans()` {.collection-method}

`Number::forHumans` 方法會返回所提供數值的易讀格式：

```php
use Illuminate\Support\Number;

$number = Number::forHumans(1000);

// 1 thousand

$number = Number::forHumans(489939);

// 490 thousand

$number = Number::forHumans(1230000, precision: 2);

// 1.23 million
```

<a name="method-number-format"></a>
#### `Number::format()` {.collection-method}

`Number::format` 方法會將給定數字格式化為特定語系的字串：

```php
use Illuminate\Support\Number;

$number = Number::format(100000);

// 100,000

$number = Number::format(100000, precision: 2);

// 100,000.00

$number = Number::format(100000.123, maxPrecision: 2);

// 100,000.12

$number = Number::format(100000, locale: 'de');

// 100.000
```

<a name="method-number-ordinal"></a>
#### `Number::ordinal()` {.collection-method}

`Number::ordinal` 方法會返回數字的序數表示：

```php
use Illuminate\Support\Number;

$number = Number::ordinal(1);

// 1st

$number = Number::ordinal(2);

// 2nd

$number = Number::ordinal(21);

// 21st
```

<a name="method-number-pairs"></a>
#### `Number::pairs()` {.collection-method}

`Number::pairs` 方法會根據指定的範圍和步長值產生數字對 (子範圍) 陣列。此方法對於將較大的數字範圍劃分為較小的、可管理的子範圍 (例如分頁或批次處理任務) 非常有用。`pairs` 方法會返回一個陣列的陣列，其中每個內部陣列代表一對 (子範圍) 數字：

```php
use Illuminate\Support\Number;

$result = Number::pairs(25, 10);

// [[0, 9], [10, 19], [20, 25]]

$result = Number::pairs(25, 10, offset: 0);

// [[0, 10], [10, 20], [20, 25]]
```

<a name="method-number-parse-int"></a>
#### `Number::parseInt()` {.collection-method}

`Number::parseInt` 方法會根據指定的語系將字串解析為整數：

```php
use Illuminate\Support\Number;

$result = Number::parseInt('10.123');

// (int) 10

$result = Number::parseInt('10,123', locale: 'fr');

// (int) 10
```

<a name="method-number-parse-float"></a>
#### `Number::parseFloat()` {.collection-method}

`Number::parseFloat` 方法會根據指定的語系將字串解析為浮點數：

```php
use Illuminate\Support\Number;

$result = Number::parseFloat('10');

// (float) 10.0

$result = Number::parseFloat('10', locale: 'fr');

// (float) 10.0
```

<a name="method-number-percentage"></a>
#### `Number::percentage()` {.collection-method}

`Number::percentage` 方法會將給定值的百分比表示形式作為字串返回：

```php
use Illuminate\Support\Number;

$percentage = Number::percentage(10);

// 10%

$percentage = Number::percentage(10, precision: 2);

// 10.00%

$percentage = Number::percentage(10.123, maxPrecision: 2);

// 10.12%

$percentage = Number::percentage(10, precision: 2, locale: 'de');

// 10,00%
```

<a name="method-number-spell"></a>
#### `Number::spell()` {.collection-method}

`Number::spell` 方法會將給定數字轉換為文字字串：

```php
use Illuminate\Support\Number;

$number = Number::spell(102);

// one hundred and two

$number = Number::spell(88, locale: 'fr');

// quatre-vingt-huit
```

`after` 參數允許您指定一個值，在此值之後的所有數字都應拼寫出來：

```php
$number = Number::spell(10, after: 10);

// 10

$number = Number::spell(11, after: 10);

// eleven
```

`until` 參數允許您指定一個值，在此值之前的所有數字都應拼寫出來：

```php
$number = Number::spell(5, until: 10);

// five

$number = Number::spell(10, until: 10);

// 10
```

<a name="method-number-spell-ordinal"></a>
#### `Number::spellOrdinal()` {.collection-method}

`Number::spellOrdinal` 方法會將數字的序數表示形式作為文字字串返回：

```php
use Illuminate\Support\Number;

$number = Number::spellOrdinal(1);

// first

$number = Number::spellOrdinal(2);

// second

$number = Number::spellOrdinal(21);

// twenty-first
```

<a name="method-number-trim"></a>
#### `Number::trim()` {.collection-method}

`Number::trim` 方法會移除給定數字小數點後的所有尾隨零位數：

```php
use Illuminate\Support\Number;

$number = Number::trim(12.0);

// 12

$number = Number::trim(12.30);

// 12.3
```

<a name="method-number-use-locale"></a>
#### `Number::useLocale()` {.collection-method}

`Number::useLocale` 方法會全域設定預設數字語系，這會影響後續呼叫 `Number` 類別方法時數字和貨幣的格式：

```php
use Illuminate\Support\Number;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Number::useLocale('de');
}
```

<a name="method-number-with-locale"></a>
#### `Number::withLocale()` {.collection-method}

`Number::withLocale` 方法會使用指定的語系執行給定的閉包，然後在回呼函式執行後恢復原始語系：

```php
use Illuminate\Support\Number;

$number = Number::withLocale('de', function () {
    return Number::format(1500);
});
```

<a name="method-number-use-currency"></a>
#### `Number::useCurrency()` {.collection-method}

`Number::useCurrency` 方法會全域設定預設數字貨幣，這會影響後續呼叫 `Number` 類別方法時貨幣的格式：

```php
use Illuminate\Support\Number;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Number::useCurrency('GBP');
}
```

<a name="method-number-with-currency"></a>
#### `Number::withCurrency()` {.collection-method}

`Number::withCurrency` 方法會使用指定的貨幣執行給定的閉包，然後在回呼函式執行後恢復原始貨幣：

```php
use Illuminate\Support\Number;

$number = Number::withCurrency('GBP', function () {
    // ...
});
```

<a name="paths"></a>
## 路徑

<a name="method-app-path"></a>
#### `app_path()` {.collection-method}

`app_path` 函式會返回應用程式 `app` 目錄的完整路徑。您也可以使用 `app_path` 函式來產生相對於應用程式目錄的檔案的完整路徑：

```php
$path = app_path();

$path = app_path('Http/Controllers/Controller.php');
```

<a name="method-base-path"></a>
#### `base_path()` {.collection-method}

`base_path` 函式會返回應用程式根目錄的完整路徑。您也可以使用 `base_path` 函式來產生相對於專案根目錄的給定檔案的完整路徑：

```php
$path = base_path();

$path = base_path('vendor/bin');
```

<a name="method-config-path"></a>
#### `config_path()` {.collection-method}

`config_path` 函式會返回應用程式 `config` 目錄的完整路徑。您也可以使用 `config_path` 函式來產生應用程式設定目錄中給定檔案的完整路徑：

```php
$path = config_path();

$path = config_path('app.php');
```

<a name="method-database-path"></a>
#### `database_path()` {.collection-method}

`database_path` 函式會返回應用程式 `database` 目錄的完整路徑。您也可以使用 `database_path` 函式來產生資料庫目錄中給定檔案的完整路徑：

```php
$path = database_path();

$path = database_path('factories/UserFactory.php');
```

<a name="method-lang-path"></a>
#### `lang_path()` {.collection-method}

`lang_path` 函式會返回應用程式 `lang` 目錄的完整路徑。您也可以使用 `lang_path` 函式來產生目錄中給定檔案的完整路徑：

```php
$path = lang_path();

$path = lang_path('en/messages.php');
```

> [!NOTE]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令發布它們。

<a name="method-public-path"></a>
#### `public_path()` {.collection-method}

`public_path` 函式會返回應用程式 `public` 目錄的完整路徑。您也可以使用 `public_path` 函式來產生公共目錄中給定檔案的完整路徑：

```php
$path = public_path();

$path = public_path('css/app.css');
```

<a name="method-resource-path"></a>
#### `resource_path()` {.collection-method}

`resource_path` 函式會返回應用程式 `resources` 目錄的完整路徑。您也可以使用 `resource_path` 函式來產生資源目錄中給定檔案的完整路徑：

```php
$path = resource_path();

$path = resource_path('sass/app.scss');
```

<a name="method-storage-path"></a>
#### `storage_path()` {.collection-method}

`storage_path` 函式會返回應用程式 `storage` 目錄的完整路徑。您也可以使用 `storage_path` 函式來產生儲存目錄中給定檔案的完整路徑：

```php
$path = storage_path();

$path = storage_path('app/file.txt');
```

<a name="urls"></a>
## URL

<a name="method-action"></a>
#### `action()` {.collection-method}

`action` 函式會為給定的控制器動作產生 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

如果方法接受路由參數，您可以將它們作為第二個參數傳遞給該方法：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="method-asset"></a>
#### `asset()` {.collection-method}

`asset` 函式會使用請求的當前方案 (HTTP 或 HTTPS) 為資產產生 URL：

```php
$url = asset('img/photo.jpg');
```

您可以透過在 `.env` 檔案中設定 `ASSET_URL` 變數來設定資產 URL 主機。如果您將資產託管在 Amazon S3 或其他 CDN 等外部服務上，這會很有用：

```php
// ASSET_URL=http://example.com/assets

$url = asset('img/photo.jpg'); // http://example.com/assets/img/photo.jpg
```

<a name="method-route"></a>
#### `route()` {.collection-method}

`route` 函式會為給定的 [具名路由](/docs/{{version}}/routing#named-routes) 產生 URL：

```php
$url = route('route.name');
```

如果路由接受參數，您可以將它們作為第二個參數傳遞給函式：

```php
$url = route('route.name', ['id' => 1]);
```

預設情況下，`route` 函式會產生絕對 URL。如果您希望產生相對 URL，可以將 `false` 作為第三個參數傳遞給函式：

```php
$url = route('route.name', ['id' => 1], false);
```

<a name="method-secure-asset"></a>
#### `secure_asset()` {.collection-method}

`secure_asset` 函式會使用 HTTPS 為資產產生 URL：

```php
$url = secure_asset('img/photo.jpg');
```

<a name="method-secure-url"></a>
#### `secure_url()` {.collection-method}

`secure_url` 函式會為給定路徑產生一個完整的 HTTPS URL。額外的 URL 片段可以作為函式的第二個參數傳遞：

```php
$url = secure_url('user/profile');

$url = secure_url('user/profile', [1]);
```

<a name="method-to-action"></a>
#### `to_action()` {.collection-method}

`to_action` 函式會為給定的控制器動作產生一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
use App\Http\Controllers\UserController;

return to_action([UserController::class, 'show'], ['user' => 1]);
```

如有必要，您可以將應分配給重新導向的 HTTP 狀態碼以及任何額外的回應標頭作為第三和第四個參數傳遞給 `to_action` 方法：

```php
return to_action(
    [UserController::class, 'show'],
    ['user' => 1],
    302,
    ['X-Framework' => 'Laravel']
);
```

<a name="method-to-route"></a>
#### `to_route()` {.collection-method}

`to_route` 函式會為給定的 [具名路由](/docs/{{version}}/routing#named-routes) 產生一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
return to_route('users.show', ['user' => 1]);
```

如有必要，您可以將應分配給重新導向的 HTTP 狀態碼以及任何額外的回應標頭作為第三和第四個參數傳遞給 `to_route` 方法：

```php
return to_route('users.show', ['user' => 1], 302, ['X-Framework' => 'Laravel']);
```

<a name="method-uri"></a>
#### `uri()` {.collection-method}

`uri` 函式會為給定的 URI 產生一個 [流暢的 URI 實例](#uri)：

```php
$uri = uri('https://example.com')
    ->withPath('/users')
    ->withQuery(['page' => 1]);
```

如果 `uri` 函式給定一個包含可呼叫控制器和方法對的陣列，該函式將為控制器方法的路徑產生一個 `Uri` 實例：

```php
use App\Http\Controllers\UserController;

$uri = uri([UserController::class, 'show'], ['user' => $user]);
```

如果控制器是可呼叫的，您可以直接提供控制器類別名稱：

```php
use App\Http\Controllers\UserIndexController;

$uri = uri(UserIndexController::class);
```

如果給定 `uri` 函式的值符合 [具名路由](/docs/{{version}}/routing#named-routes) 的名稱，則會為該路由的路徑產生一個 `Uri` 實例：

```php
$uri = uri('users.show', ['user' => $user]);
```

<a name="method-url"></a>
#### `url()` {.collection-method}

`url` 函式會為給定路徑產生一個完整的 URL：

```php
$url = url('user/profile');

$url = url('user/profile', [1]);
```

如果未提供路徑，則會返回 `Illuminate\Routing\UrlGenerator` 實例：

```php
$current = url()->current();

$full = url()->full();

$previous = url()->previous();
```

有關使用 `url` 函式的更多資訊，請參閱 [URL 產生說明文件](/docs/{{version}}/urls#generating-urls)。

<a name="miscellaneous"></a>
## 其他

<a name="method-abort"></a>
#### `abort()` {.collection-method}

`abort` 函式會拋出 [HTTP 異常](/docs/{{version}}/errors#http-exceptions)，該異常將由 [異常處理器](/docs/{{version}}/errors#handling-exceptions) 渲染：

```php
abort(403);
```

您也可以提供異常訊息和應傳送至瀏覽器的自訂 HTTP 回應標頭：

```php
abort(403, 'Unauthorized.', $headers);
```

<a name="method-abort-if"></a>
#### `abort_if()` {.collection-method}

`abort_if` 函式會在給定布林表達式評估為 `true` 時拋出 HTTP 異常：

```php
abort_if(! Auth::user()->isAdmin(), 403);
```

與 `abort` 方法一樣，您也可以將異常的回應文字作為第三個參數，並將自訂回應標頭陣列作為第四個參數傳遞給函式。

<a name="method-abort-unless"></a>
#### `abort_unless()` {.collection-method}

`abort_unless` 函式會在給定布林表達式評估為 `false` 時拋出 HTTP 異常：

```php
abort_unless(Auth::user()->isAdmin(), 403);
```

與 `abort` 方法一樣，您也可以將異常的回應文字作為第三個參數，並將自訂回應標頭陣列作為第四個參數傳遞給函式。

<a name="method-app"></a>
#### `app()` {.collection-method}

`app` 函式會返回 [服務容器](/docs/{{version}}/container) 實例：

```php
$container = app();
```

您可以傳遞類別或介面名稱以從容器中解析它：

```php
$api = app('HelpSpot\API');
```

<a name="method-auth"></a>
#### `auth()` {.collection-method}

`auth` 函式會返回 [認證器](/docs/{{version}}/authentication) 實例。您可以將其作為 `Auth` Facade 的替代方案：

```php
$user = auth()->user();
```

如有必要，您可以指定要存取的 Guard 實例：

```php
$user = auth('admin')->user();
```

<a name="method-back"></a>
#### `back()` {.collection-method}

`back` 函式會產生一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects) 到使用者先前的位置：

```php
return back($status = 302, $headers = [], $fallback = '/');

return back();
```

<a name="method-bcrypt"></a>
#### `bcrypt()` {.collection-method}

`bcrypt` 函式會使用 Bcrypt [雜湊](/docs/{{version}}/hashing) 給定的值。您可以將此函式作為 `Hash` Facade 的替代方案：

```php
$password = bcrypt('my-secret-password');
```

<a name="method-blank"></a>
#### `blank()` {.collection-method}

`blank` 函式會判斷給定的值是否為「空白」：

```php
blank('');
blank('   ');
blank(null);
blank(collect());

// true

blank(0);
blank(true);
blank(false);

// false
```

有關 `blank` 的反義詞，請參閱 [filled](#method-filled) 函式。

<a name="method-broadcast"></a>
#### `broadcast()` {.collection-method}

`broadcast` 函式會將給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast(new UserRegistered($user));

broadcast(new UserRegistered($user))->toOthers();
```

<a name="method-broadcast-if"></a>
#### `broadcast_if()` {.collection-method}

`broadcast_if` 函式會在給定布林表達式評估為 `true` 時，將給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast_if($user->isActive(), new UserRegistered($user));

broadcast_if($user->isActive(), new UserRegistered($user))->toOthers();
```

<a name="method-broadcast-unless"></a>
#### `broadcast_unless()` {.collection-method}

`broadcast_unless` 函式會在給定布林表達式評估為 `false` 時，將給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast_unless($user->isBanned(), new UserRegistered($user));

broadcast_unless($user->isBanned(), new UserRegistered($user))->toOthers();
```

<a name="method-cache"></a>
#### `cache()` {.collection-method}

`cache` 函式可用於從 [快取](/docs/{{version}}/cache) 中取得值。如果給定的鍵不存在於快取中，則會返回一個可選的預設值：

```php
$value = cache('key');

$value = cache('key', 'default');
```

您可以透過將鍵/值對陣列傳遞給函式來將項目新增至快取。您還應該傳遞快取值應被視為有效的秒數或持續時間：

```php
cache(['key' => 'value'], 300);

cache(['key' => 'value'], now()->addSeconds(10));
```

<a name="method-class-uses-recursive"></a>
#### `class_uses_recursive()` {.collection-method}

`class_uses_recursive` 函式會返回類別使用的所有 Trait，包括其所有父類別使用的 Trait：

```php
$traits = class_uses_recursive(App\Models\User::class);
```

<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 函式會從給定的值建立一個 [Collection](/docs/{{version}}/collections) 實例：

```php
$collection = collect(['Taylor', 'Abigail']);
```

<a name="method-config"></a>
#### `config()` {.collection-method}

`config` 函式會取得 [設定](/docs/{{version}}/configuration) 變數的值。設定值可以使用「點」語法存取，其中包括檔案名稱和您希望存取的選項。您還可以提供一個預設值，如果設定選項不存在，則會返回該預設值：

```php
$value = config('app.timezone');

$value = config('app.timezone', $default);
```

您可以在執行時透過傳遞鍵/值對陣列來設定設定變數。但是，請注意，此函式僅影響當前請求的設定值，不會更新您的實際設定值：

```php
config(['app.debug' => true]);
```

<a name="method-context"></a>
#### `context()` {.collection-method}

`context` 函式會從當前 [Context](/docs/{{version}}/context) 中取得值。您也可以提供一個預設值，如果 Context 鍵不存在，則會返回該預設值：

```php
$value = context('trace_id');

$value = context('trace_id', $default);
```

您可以透過傳遞鍵/值對陣列來設定 Context 值：

```php
use Illuminate\Support\Str;

context(['trace_id' => Str::uuid()->toString()]);
```

<a name="method-cookie"></a>
#### `cookie()` {.collection-method}

`cookie` 函式會建立一個新的 [Cookie](/docs/{{version}}/requests#cookies) 實例：

```php
$cookie = cookie('name', 'value', $minutes);
```

<a name="method-csrf-field"></a>
#### `csrf_field()` {.collection-method}

`csrf_field` 函式會產生一個包含 CSRF Token 值的 HTML `hidden` 輸入欄位。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
{{ csrf_field() }}
```

<a name="method-csrf-token"></a>
#### `csrf_token()` {.collection-method}

`csrf_token` 函式會擷取當前 CSRF Token 的值：

```php
$token = csrf_token();
```

<a name="method-decrypt"></a>
#### `decrypt()` {.collection-method}

`decrypt` 函式會 [解密](/docs/{{version}}/encryption) 給定的值。您可以將此函式作為 `Crypt` Facade 的替代方案：

```php
$password = decrypt($value);
```

有關 `decrypt` 的反義詞，請參閱 [encrypt](#method-encrypt) 函式。

<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 函式會傾印給定的變數並結束腳本的執行：

```php
dd($value);

dd($value1, $value2, $value3, ...);
```

如果您不想停止腳本的執行，請改用 [dump](#method-dump) 函式。

<a name="method-dispatch"></a>
#### `dispatch()` {.collection-method}

`dispatch` 函式會將給定的 [Job](/docs/{{version}}/queues#creating-jobs) 推送到 Laravel [Job Queue](/docs/{{version}}/queues) 中：

```php
dispatch(new App\Jobs\SendEmails);
```

<a name="method-dispatch-sync"></a>
#### `dispatch_sync()` {.collection-method}

`dispatch_sync` 函式會將給定的 Job 推送到 [同步](/docs/{{version}}/queues#synchronous-dispatching) 佇列中，以便立即處理：

```php
dispatch_sync(new App\Jobs\SendEmails);
```

<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 函式會傾印給定的變數：

```php
dump($value);

dump($value1, $value2, $value3, ...);
```

如果您想在傾印變數後停止執行腳本，請改用 [dd](#method-dd) 函式。

<a name="method-encrypt"></a>
#### `encrypt()` {.collection-method}

`encrypt` 函式會 [加密](/docs/{{version}}/encryption) 給定的值。您可以將此函式作為 `Crypt` Facade 的替代方案：

```php
$secret = encrypt('my-secret-value');
```

有關 `encrypt` 的反義詞，請參閱 [decrypt](#method-decrypt) 函式。

<a name="method-env"></a>
#### `env()` {.collection-method}

`env` 函式會擷取 [環境變數](/docs/{{version}}/configuration#environment-configuration) 的值或返回預設值：

```php
$env = env('APP_ENV');

$env = env('APP_ENV', 'production');
```

> [!WARNING]
> 如果您在部署過程中執行 `config:cache` 命令，您應該確保只從設定檔中呼叫 `env` 函式。一旦設定被快取，`.env` 檔案將不會被載入，所有對 `env` 函式的呼叫都將返回外部環境變數，例如伺服器級別或系統級別的環境變數或 `null`。

<a name="method-event"></a>
#### `event()` {.collection-method}

`event` 函式會將給定的 [事件](/docs/{{version}}/events) 分派給其監聽器：

```php
event(new UserRegistered($user));
```

<a name="method-fake"></a>
#### `fake()` {.collection-method}

`fake` 函式會從容器中解析一個 [Faker](https://github.com/FakerPHP/Faker) 單例，這在模型工廠、資料庫填充、測試和原型視圖中建立假資料時非常有用：

```blade
 @repo/source/fortify.md ($i = 0; $i < 10; $i++)
    <dl>
        <dt>Name</dt>
        <dd>{{ fake()->name() }}</dd>

        <dt>Email</dt>
        <dd>{{ fake()->unique()->safeEmail() }}</dd>
    </dl>
 @endfor
```

預設情況下，`fake` 函式將使用 `config/app.php` 設定中的 `app.faker_locale` 設定選項。通常，此設定選項是透過 `APP_FAKER_LOCALE` 環境變數設定的。您也可以透過將語系傳遞給 `fake` 函式來指定語系。每個語系都將解析一個單獨的單例：

```php
fake('nl_NL')->name()
```

<a name="method-filled"></a>
#### `filled()` {.collection-method}

`filled` 函式會判斷給定的值是否不為「空白」：

```php
filled(0);
filled(true);
filled(false);

// true

filled('');
filled('   ');
filled(null);
filled(collect());

// false
```

有關 `filled` 的反義詞，請參閱 [blank](#method-blank) 函式。

<a name="method-info"></a>
#### `info()` {.collection-method}

`info` 函式會將資訊寫入應用程式的 [日誌](/docs/{{version}}/logging) 中：

```php
info('Some helpful information!');
```

也可以將上下文資料陣列傳遞給函式：

```php
info('User login attempt failed.', ['id' => $user->id]);
```

<a name="method-literal"></a>
#### `literal()` {.collection-method}

`literal` 函式會建立一個新的 [stdClass](https://www.php.net/manual/en/class.stdclass.php) 實例，並將給定的具名參數作為屬性：

```php
$obj = literal(
    name: 'Joe',
    languages: ['PHP', 'Ruby'],
);

$obj->name; // 'Joe'
$obj->languages; // ['PHP', 'Ruby']
```

<a name="method-logger"></a>
#### `logger()` {.collection-method}

`logger` 函式可用於將 `debug` 級別訊息寫入 [日誌](/docs/{{version}}/logging) 中：

```php
logger('Debug message');
```

也可以將上下文資料陣列傳遞給函式：

```php
logger('User has logged in.', ['id' => $user->id]);
```

如果沒有值傳遞給函式，則會返回一個 [logger](/docs/{{version}}/logging) 實例：

```php
logger()->error('You are not allowed here.');
```

<a name="method-method-field"></a>
#### `method_field()` {.collection-method}

`method_field` 函式會產生一個 HTML `hidden` 輸入欄位，其中包含表單 HTTP 動詞的偽造值。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
<form method="POST">
    {{ method_field('DELETE') }}
</form>
```

<a name="method-now"></a>
#### `now()` {.collection-method}

`now` 函式會為當前時間建立一個新的 `Illuminate\Support\Carbon` 實例：

```php
$now = now();
```

<a name="method-old"></a>
#### `old()` {.collection-method}

`old` 函式會 [擷取](/docs/{{version}}/requests#retrieving-input) 閃存到 Session 中的 [舊輸入](/docs/{{version}}/requests#old-input) 值：

```php
$value = old('value');

$value = old('value', 'default');
```

由於作為 `old` 函式第二個參數提供的「預設值」通常是 Eloquent 模型的屬性，Laravel 允許您直接將整個 Eloquent 模型作為第二個參數傳遞給 `old` 函式。這樣做時，Laravel 會假定提供給 `old` 函式的第一個參數是應被視為「預設值」的 Eloquent 屬性名稱：

```blade
{{ old('name', $user->name) }}

// 等同於...

{{ old('name', $user) }}
```

<a name="method-once"></a>
#### `once()` {.collection-method}

`once` 函式會執行給定的回呼函式並在請求期間將結果快取在記憶體中。隨後對 `once` 函式使用相同回呼函式的呼叫將返回先前快取的結果：

```php
function random(): int
{
    return once(function () {
        return random_int(1, 1000);
    });
}

random(); // 123
random(); // 123 (快取結果)
random(); // 123 (快取結果)
```

當 `once` 函式在物件實例中執行時，快取結果將對該物件實例是唯一的：

```php
<?php

class NumberService
{
    public function all(): array
    {
        return once(fn () => [1, 2, 3]);
    }
}

$service = new NumberService;

$service->all();
$service->all(); // (快取結果)

$secondService = new NumberService;

$secondService->all();
$secondService->all(); // (快取結果)
```
<a name="method-optional"></a>
#### `optional()` {.collection-method}

`optional` 函式接受任何參數，並允許您存取該物件的屬性或呼叫方法。如果給定的物件為 `null`，則屬性和方法將返回 `null` 而不是導致錯誤：

```php
return optional($user->address)->street;

{!! old('name', optional($user)->name) !!}
```

`optional` 函式也接受一個閉包作為其第二個參數。如果作為第一個參數提供的值不為 null，則會呼叫該閉包：

```php
return optional(User::find($id), function (User $user) {
    return $user->name;
});
```

<a name="method-policy"></a>
#### `policy()` {.collection-method}

`policy` 方法會為給定類別擷取 [Policy](/docs/{{version}}/authorization#creating-policies) 實例：

```php
$policy = policy(App\Models\User::class);
```

<a name="method-redirect"></a>
#### `redirect()` {.collection-method}

`redirect` 函式會返回一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)，如果沒有參數呼叫，則返回重新導向器實例：

```php
return redirect($to = null, $status = 302, $headers = [], $secure = null);

return redirect('/home');

return redirect()->route('route.name');
```

<a name="method-report"></a>
#### `report()` {.collection-method}

`report` 函式會使用您的 [異常處理器](/docs/{{version}}/errors#handling-exceptions) 報告異常：

```php
report($e);
```

`report` 函式也接受一個字串作為參數。當給定函式一個字串時，函式將建立一個以給定字串作為其訊息的異常：

```php
report('Something went wrong.');
```

<a name="method-report-if"></a>
#### `report_if()` {.collection-method}

`report_if` 函式會在給定布林表達式評估為 `true` 時，使用您的 [異常處理器](/docs/{{version}}/errors#handling-exceptions) 報告異常：

```php
report_if($shouldReport, $e);

report_if($shouldReport, 'Something went wrong.');
```

<a name="method-report-unless"></a>
#### `report_unless()` {.collection-method}

`report_unless` 函式會在給定布林表達式評估為 `false` 時，使用您的 [異常處理器](/docs/{{version}}/errors#handling-exceptions) 報告異常：

```php
report_unless($reportingDisabled, $e);

report_unless($reportingDisabled, 'Something went wrong.');
```

<a name="method-request"></a>
#### `request()` {.collection-method}

`request` 函式會返回當前的 [請求](/docs/{{version}}/requests) 實例或從當前請求中取得輸入欄位的值：

```php
$request = request();

$value = request('key', $default);
```

<a name="method-rescue"></a>
#### `rescue()` {.collection-method}

`rescue` 函式會執行給定的閉包並捕獲其執行期間發生的任何異常。所有捕獲到的異常都將傳送至您的 [異常處理器](/docs/{{version}}/errors#handling-exceptions)；但是，請求將繼續處理：

```php
return rescue(function () {
    return $this->method();
});
```

您也可以將第二個參數傳遞給 `rescue` 函式。此參數將是如果在執行閉包時發生異常應返回的「預設」值：

```php
return rescue(function () {
    return $this->method();
}, false);

return rescue(function () {
    return $this->method();
}, function () {
    return $this->failure();
});
```

可以向 `rescue` 函式提供 `report` 參數，以判斷是否應透過 `report` 函式報告異常：

```php
return rescue(function () {
    return $this->method();
}, report: function (Throwable $throwable) {
    return $throwable instanceof InvalidArgumentException;
});
```

<a name="method-resolve"></a>
#### `resolve()` {.collection-method}

`resolve` 函式會使用 [服務容器](/docs/{{version}}/container) 將給定的類別或介面名稱解析為實例：

```php
$api = resolve('HelpSpot\API');
```

<a name="method-response"></a>
#### `response()` {.collection-method}

`response` 函式會建立一個 [回應](/docs/{{version}}/responses) 實例或取得回應工廠的實例：

```php
return response('Hello World', 200, $headers);

return response()->json(['foo' => 'bar'], 200, $headers);
```

<a name="method-retry"></a>
#### `retry()` {.collection-method}

`retry` 函式會嘗試執行給定的回呼函式，直到達到給定的最大嘗試次數。如果回呼函式沒有拋出異常，則會返回其返回值。如果回呼函式拋出異常，它將自動重試。如果超過最大嘗試次數，則會拋出異常：

```php
return retry(5, function () {
    // 嘗試 5 次，每次嘗試之間休息 100 毫秒...
}, 100);
```

如果您想手動計算每次嘗試之間應暫停的毫秒數，您可以將閉包作為第三個參數傳遞給 `retry` 函式：

```php
use Exception;

return retry(5, function () {
    // ...
}, function (int $attempt, Exception $exception) {
    return $attempt * 100;
});
```

為方便起見，您可以將陣列作為第一個參數提供給 `retry` 函式。此陣列將用於判斷後續嘗試之間應暫停多少毫秒：

```php
return retry([100, 200], function () {
    // 第一次重試暫停 100 毫秒，第二次重試暫停 200 毫秒...
});
```

若要僅在特定條件下重試，您可以將閉包作為第四個參數傳遞給 `retry` 函式：

```php
use App\Exceptions\TemporaryException;
use Exception;

return retry(5, function () {
    // ...
}, 100, function (Exception $exception) {
    return $exception instanceof TemporaryException;
});
```

<a name="method-session"></a>
#### `session()` {.collection-method}

`session` 函式可用於取得或設定 [Session](/docs/{{version}}/session) 值：

```php
$value = session('key');
```

您可以透過將鍵/值對陣列傳遞給函式來設定值：

```php
session(['chairs' => 7, 'instruments' => 3]);
```

如果沒有值傳遞給函式，則會返回 Session 儲存區：

```php
$value = session()->get('key');

session()->put('key', $value);
```

<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 函式接受兩個參數：任意 `$value` 和一個閉包。`$value` 將傳遞給閉包，然後由 `tap` 函式返回。閉包的返回值無關緊要：

```php
$user = tap(User::first(), function (User $user) {
    $user->name = 'Taylor';

    $user->save();
});
```

如果沒有閉包傳遞給 `tap` 函式，您可以呼叫給定 `$value` 上的任何方法。您呼叫的方法的返回值將始終是 `$value`，無論該方法在其定義中實際返回什麼。例如，Eloquent `update` 方法通常返回一個整數。但是，我們可以透過將 `update` 方法呼叫鏈接到 `tap` 函式來強制該方法返回模型本身：

```php
$user = tap($user)->update([
    'name' => $name,
    'email' => $email,
]);
```

若要將 `tap` 方法新增至類別，您可以將 `Illuminate\Support\Traits\Tappable` Trait 新增至類別。此 Trait 的 `tap` 方法接受一個閉包作為其唯一參數。物件實例本身將傳遞給閉包，然後由 `tap` 方法返回：

```php
return $user->tap(function (User $user) {
    // ...
});
```

<a name="method-throw-if"></a>
#### `throw_if()` {.collection-method}

`throw_if` 函式會在給定布林表達式評估為 `true` 時拋出給定的異常：

```php
throw_if(! Auth::user()->isAdmin(), AuthorizationException::class);

throw_if(
    ! Auth::user()->isAdmin(),
    AuthorizationException::class,
    'You are not allowed to access this page.'
);
```

<a name="method-throw-unless"></a>
#### `throw_unless()` {.collection-method}

`throw_unless` 函式會在給定布林表達式評估為 `false` 時拋出給定的異常：

```php
throw_unless(Auth::user()->isAdmin(), AuthorizationException::class);

throw_unless(
    Auth::user()->isAdmin(),
    AuthorizationException::class,
    'You are not allowed to access this page.'
);
```

<a name="method-today"></a>
#### `today()` {.collection-method}

`today` 函式會為當前日期建立一個新的 `Illuminate\Support\Carbon` 實例：

```php
$today = today();
```

<a name="method-trait-uses-recursive"></a>
#### `trait_uses_recursive()` {.collection-method}

`trait_uses_recursive` 函式會返回 Trait 使用的所有 Trait：

```php
$traits = trait_uses_recursive(\Illuminate\Notifications\Notifiable::class);
```

<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 函式會在給定值不為 [空白](#method-blank) 時對其執行閉包，然後返回閉包的返回值：

```php
$callback = function (int $value) {
    return $value * 2;
};

$result = transform(5, $callback);

// 10
```

可以將預設值或閉包作為第三個參數傳遞給函式。如果給定值為空白，則會返回此值：

```php
$result = transform(null, $callback, 'The value is blank');

// The value is blank
```

<a name="method-validator"></a>
#### `validator()` {.collection-method}

`validator` 函式會使用給定的參數建立一個新的 [Validator](/docs/{{version}}/validation) 實例。您可以將其作為 `Validator` Facade 的替代方案：

```php
$validator = validator($data, $rules, $messages);
```

<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 函式會返回給定的值。但是，如果您將閉包傳遞給函式，則會執行閉包並返回其返回值：

```php
$result = value(true);

// true

$result = value(function () {
    return false;
});

// false
```

可以將額外參數傳遞給 `value` 函式。如果第一個參數是閉包，則額外參數將作為參數傳遞給閉包，否則將被忽略：

```php
$result = value(function (string $name) {
    return $name;
}, 'Taylor');

// 'Taylor'
```

<a name="method-view"></a>
#### `view()` {.collection-method}

`view` 函式會擷取 [View](/docs/{{version}}/views) 實例：

```php
return view('auth.login');
```

<a name="method-with"></a>
#### `with()` {.collection-method}

`with` 函式會返回給定的值。如果將閉包作為第二個參數傳遞給函式，則會執行閉包並返回其返回值：

```php
$callback = function (mixed $value) {
    return is_numeric($value) ? $value * 2 : 0;
};

$result = with(5, $callback);

// 10

$result = with(null, $callback);

// 0

$result = with(5, null);

// 5
```

<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 函式會在給定條件評估為 `true` 時返回給定的值。否則，返回 `null`。如果將閉包作為第二個參數傳遞給函式，則會執行閉包並返回其返回值：

```php
$value = when(true, 'Hello World');

$value = when(true, fn () => 'Hello World');
```

`when` 函式主要用於條件式渲染 HTML 屬性：

```blade
<div {!! when($condition, 'wire:poll="calculate"') !!}>
    ...
</div>
```

<a name="other-utilities"></a>
## 其他工具

<a name="benchmarking"></a>
### 效能基準測試

有時您可能希望快速測試應用程式某些部分的效能。在這些情況下，您可以使用 `Benchmark` 支援類別來測量給定回呼函式完成所需的毫秒數：

```php
<?php

use App\Models\User;
use Illuminate\Support\Benchmark;

Benchmark::dd(fn () => User::find(1)); // 0.1 ms

Benchmark::dd([
    'Scenario 1' => fn () => User::count(), // 0.5 ms
    'Scenario 2' => fn () => User::all()->count(), // 20.0 ms
]);
```

預設情況下，給定的回呼函式將執行一次 (一次迭代)，其持續時間將顯示在瀏覽器/控制台中。

若要多次呼叫回呼函式，您可以將回呼函式應呼叫的迭代次數指定為方法的第二個參數。當多次執行回呼函式時，`Benchmark` 類別將返回執行回呼函式在所有迭代中的平均毫秒數：

```php
Benchmark::dd(fn () => User::count(), iterations: 10); // 0.5 ms
```

有時，您可能希望對回呼函式的執行進行基準測試，同時仍取得回呼函式返回的值。`value` 方法將返回一個包含回呼函式返回的值和執行回呼函式所需的毫秒數的元組：

```php
[$count, $duration] = Benchmark::value(fn () => User::count());
```

<a name="dates"></a>
### 日期

Laravel 包含了 [Carbon](https://carbon.nesbot.com/docs/)，一個強大的日期和時間操作函式庫。若要建立新的 `Carbon` 實例，您可以呼叫 `now` 函式。此函式在您的 Laravel 應用程式中是全域可用的：

```php
$now = now();
```

或者，您可以使用 `Illuminate\Support\Carbon` 類別建立新的 `Carbon` 實例：

```php
use Illuminate\Support\Carbon;

$now = Carbon::now();
```

有關 Carbon 及其功能的詳細討論，請參閱 [官方 Carbon 說明文件](https://carbon.nesbot.com/docs/)。

<a name="deferred-functions"></a>
### 延遲函式

雖然 Laravel 的 [佇列 Job](/docs/{{version}}/queues) 允許您將任務排入佇列以進行背景處理，但有時您可能有一些簡單的任務希望延遲執行，而無需設定或維護長時間執行的佇列工作者。

延遲函式允許您將閉包的執行延遲到 HTTP 回應已傳送給使用者之後，從而使您的應用程式感覺快速且反應靈敏。若要延遲閉包的執行，只需將閉包傳遞給 `Illuminate\Support\defer` 函式：

```php
use App\Services\Metrics;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use function Illuminate\Support\defer;

Route::post('/orders', function (Request $request) {
    // 建立訂單...

    defer(fn () => Metrics::reportOrder($order));

    return $order;
});
```

預設情況下，延遲函式只會在呼叫 `Illuminate\Support\defer` 的 HTTP 回應、Artisan 命令或佇列 Job 成功完成時執行。這表示如果請求導致 `4xx` 或 `5xx` HTTP 回應，延遲函式將不會執行。如果您希望延遲函式始終執行，您可以將 `always` 方法鏈接到您的延遲函式：

```php
defer(fn () => Metrics::reportOrder($order))->always();
```

> [!WARNING]
> 如果您安裝了 [Swoole PHP 擴充功能](https://www.php.net/manual/en/book.swoole.php)，Laravel 的 `defer` 函式可能會與 Swoole 自己的全域 `defer` 函式衝突，導致 Web 伺服器錯誤。請務必透過明確命名空間來呼叫 Laravel 的 `defer` 輔助函式：`use function Illuminate\Support\defer;`

<a name="cancelling-deferred-functions"></a>
#### 取消延遲函式

如果您需要在延遲函式執行之前取消它，您可以使用 `forget` 方法透過其名稱取消函式。若要命名延遲函式，請向 `Illuminate\Support\defer` 函式提供第二個參數：

```php
defer(fn () => Metrics::report(), 'reportMetrics');

defer()->forget('reportMetrics');
```

<a name="disabling-deferred-functions-in-tests"></a>
#### 在測試中停用延遲函式

在編寫測試時，停用延遲函式可能很有用。您可以在測試中呼叫 `withoutDefer` 以指示 Laravel 立即呼叫所有延遲函式：

```php tab=Pest
test('without defer', function () {
    $this->withoutDefer();

    // ...
});
```

```php tab=PHPUnit
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_defer(): void
    {
        $this->withoutDefer();

        // ...
    }
}
```

如果您想為測試案例中的所有測試停用延遲函式，您可以從基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutDefer` 方法：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutDefer();
    }// [tl! add:end]
}
```

<a name="lottery"></a>
### 抽獎

Laravel 的 Lottery 類別可用於根據給定的機率執行回呼函式。當您只想為一小部分傳入請求執行程式碼時，這會特別有用：

```php
use Illuminate\Support\Lottery;

Lottery::odds(1, 20)
    ->winner(fn () => $user->won())
    ->loser(fn () => $user->lost())
    ->choose();
```

您可以將 Laravel 的 Lottery 類別與其他 Laravel 功能結合使用。例如，您可能希望只向異常處理器報告一小部分慢速查詢。而且，由於 Lottery 類別是可呼叫的，我們可以將類別實例傳遞給任何接受可呼叫的函式：

```php
use Carbon\CarbonInterval;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Lottery;

DB::whenQueryingForLongerThan(
    CarbonInterval::seconds(2),
    Lottery::odds(1, 100)->winner(fn () => report('Querying > 2 seconds.')),
);
```

<a name="testing-lotteries"></a>
#### 測試抽獎

Laravel 提供了一些簡單的方法，讓您可以輕鬆測試應用程式的 Lottery 呼叫：

```php
// Lottery 將始終獲勝...
Lottery::alwaysWin();

// Lottery 將始終失敗...
Lottery::alwaysLose();

// Lottery 將先獲勝然後失敗，最後恢復正常行為...
Lottery::fix([true, false]);

// Lottery 將恢復正常行為...
Lottery::determineResultsNormally();
```

<a name="pipeline"></a>
### 管線

Laravel 的 `Pipeline` Facade 提供了一種方便的方式，可以將給定的輸入「透過」一系列可呼叫類別、閉包或可呼叫函式，讓每個類別都有機會檢查或修改輸入並呼叫管線中的下一個可呼叫函式：

```php
use Closure;
use App\Models\User;
use Illuminate\Support\Facades\Pipeline;

$user = Pipeline::send($user)
    ->through([
        function (User $user, Closure $next) {
            // ...

            return $next($user);
        },
        function (User $user, Closure $next) {
            // ...

            return $next($user);
        },
    ])
    ->then(fn (User $user) => $user);
```

如您所見，管線中的每個可呼叫類別或閉包都會提供輸入和一個 `$next` 閉包。呼叫 `$next` 閉包將呼叫管線中的下一個可呼叫函式。您可能已經注意到，這與 [Middleware](/docs/{{version}}/middleware) 非常相似。

當管線中的最後一個可呼叫函式呼叫 `$next` 閉包時，將呼叫提供給 `then` 方法的可呼叫函式。通常，此可呼叫函式只會返回給定的輸入。為方便起見，如果您只想在處理後返回輸入，可以使用 `thenReturn` 方法。

當然，如前所述，您不限於向管線提供閉包。您也可以提供可呼叫類別。如果提供了類別名稱，則會透過 Laravel 的 [服務容器](/docs/{{version}}/container) 實例化該類別，從而允許將依賴項注入到可呼叫類別中：

```php
$user = Pipeline::send($user)
    ->through([
        GenerateProfilePhoto::class,
        ActivateSubscription::class,
        SendWelcomeEmail::class,
    ])
    ->thenReturn();
```

可以在管線上呼叫 `withinTransaction` 方法，以自動將管線的所有步驟包裝在單一資料庫交易中：

```php
$user = Pipeline::send($user)
    ->withinTransaction()
    ->through([
        ProcessOrder::class,
        TransferFunds::class,
        UpdateInventory::class,
    ])
    ->thenReturn();
```

<a name="sleep"></a>
### 暫停

Laravel 的 `Sleep` 類別是 PHP 原生 `sleep` 和 `usleep` 函式的輕量級包裝器，提供更高的可測試性，同時也公開了開發人員友善的 API 以處理時間：

```php
use Illuminate\Support\Sleep;

$waiting = true;

while ($waiting) {
    Sleep::for(1)->second();

    $waiting = /* ... */;
}
```

`Sleep` 類別提供了各種方法，可讓您處理不同的時間單位：

```php
// 暫停後返回一個值...
$result = Sleep::for(1)->second()->then(fn () => 1 + 1);

// 在給定值為 true 時暫停...
Sleep::for(1)->second()->while(fn () => shouldKeepSleeping());

// 暫停執行 90 秒...
Sleep::for(1.5)->minutes();

// 暫停執行 2 秒...
Sleep::for(2)->seconds();

// 暫停執行 500 毫秒...
Sleep::for(500)->milliseconds();

// 暫停執行 5,000 微秒...
Sleep::for(5000)->microseconds();

// 暫停執行直到給定時間...
Sleep::until(now()->addMinute());

// PHP 原生「sleep」函式的別名...
Sleep::sleep(2);

// PHP 原生「usleep」函式的別名...
Sleep::usleep(5000);
```

若要輕鬆組合時間單位，您可以使用 `and` 方法：

```php
Sleep::for(1)->second()->and(10)->milliseconds();
```

<a name="testing-sleep"></a>
#### 測試暫停

當測試使用 `Sleep` 類別或 PHP 原生 sleep 函式的程式碼時，您的測試將暫停執行。正如您所預期的，這會顯著降低您的測試套件的速度。例如，假設您正在測試以下程式碼：

```php
$waiting = /* ... */;

$seconds = 1;

while ($waiting) {
    Sleep::for($seconds++)->seconds();

    $waiting = /* ... */;
}
```

通常，測試此程式碼至少需要一秒鐘。幸運的是，`Sleep` 類別允許我們「偽造」暫停，以便我們的測試套件保持快速：

```php tab=Pest
it('waits until ready', function () {
    Sleep::fake();

    // ...
});
```

```php tab=PHPUnit
public function test_it_waits_until_ready()
{
    Sleep::fake();

    // ...
}
```

當偽造 `Sleep` 類別時，實際的執行暫停會被繞過，從而顯著加快測試速度。

一旦 `Sleep` 類別被偽造，就可以對應該發生的預期「暫停」進行斷言。為了說明這一點，讓我們想像我們正在測試的程式碼暫停執行三次，每次暫停增加一秒鐘。使用 `assertSequence` 方法，我們可以斷言我們的程式碼「暫停」了適當的時間，同時保持測試快速：

```php tab=Pest
it('checks if ready three times', function () {
    Sleep::fake();

    // ...

    Sleep::assertSequence([
        Sleep::for(1)->second(),
        Sleep::for(2)->seconds(),
        Sleep::for(3)->seconds(),
    ]);
}
```

```php tab=PHPUnit
public function test_it_checks_if_ready_three_times()
{
    Sleep::fake();

    // ...

    Sleep::assertSequence([
        Sleep::for(1)->second(),
        Sleep::for(2)->seconds(),
        Sleep::for(3)->seconds(),
    ]);
}
```

當然，`Sleep` 類別提供了各種其他斷言，您可以在測試時使用：

```php
use Carbon\CarbonInterval as Duration;
use Illuminate\Support\Sleep;

// 斷言 sleep 被呼叫了 3 次...
Sleep::assertSleptTimes(3);

// 斷言 sleep 的持續時間...
Sleep::assertSlept(function (Duration $duration): bool {
    return /* ... */;
}, times: 1);

// 斷言 Sleep 類別從未被呼叫...
Sleep::assertNeverSlept();

// 斷言即使呼叫了 Sleep，也沒有發生執行暫停...
Sleep::assertInsomniac();
```

有時，在發生偽造暫停時執行動作可能很有用。為此，您可以向 `whenFakingSleep` 方法提供一個回呼函式。在以下範例中，我們使用 Laravel 的 [時間操作輔助函式](/docs/{{version}}/mocking#interacting-with-time) 透過每次暫停的持續時間立即推進時間：

```php
use Carbon\CarbonInterval as Duration;

$this->freezeTime();

Sleep::fake();

Sleep::whenFakingSleep(function (Duration $duration) {
    // 偽造暫停時推進時間...
    $this->travel($duration->totalMilliseconds)->milliseconds();
});
```

由於推進時間是常見的需求，`fake` 方法接受 `syncWithCarbon` 參數，以便在測試中暫停時保持 Carbon 同步：

```php
Sleep::fake(syncWithCarbon: true);

$start = now();

Sleep::for(1)->second();

$start->diffForHumans(); // 1 second ago
```

Laravel 在內部暫停執行時會使用 `Sleep` 類別。例如，[retry](#method-retry) 輔助函式在暫停時使用 `Sleep` 類別，從而在使用該輔助函式時提高了可測試性。

<a name="timebox"></a>
### 時間限制

Laravel 的 `Timebox` 類別確保給定的回呼函式始終花費固定的時間執行，即使其實際執行完成得更快。這對於加密操作和使用者身份驗證檢查特別有用，攻擊者可能會利用執行時間的變化來推斷敏感資訊。

如果執行超過固定持續時間，`Timebox` 不會產生任何影響。開發人員有責任選擇足夠長的時間作為固定持續時間，以應對最壞情況。

`call` 方法接受一個閉包和一個以微秒為單位的時間限制，然後執行閉包並等待直到達到時間限制：

```php
use Illuminate\Support\Timebox;

(new Timebox)->call(function ($timebox) {
    // ...
}, microseconds: 10000);
```

如果在閉包中拋出異常，此類別將遵守定義的延遲並在延遲後重新拋出異常。

<a name="uri"></a>
### URI

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面，用於建立和操作 URI。此類別包裝了底層 League URI 套件提供的功能，並與 Laravel 的路由系統無縫整合。

您可以使用靜態方法輕鬆建立 `Uri` 實例：

```php
use App\Http\Controllers\UserController;
use App\Http\Controllers\InvokableController;
use Illuminate\Support\Uri;

// 從給定字串產生 URI 實例...
$uri = Uri::of('https://example.com/path');

// 產生路徑、具名路由或控制器動作的 URI 實例...
$uri = Uri::to('/dashboard');
$uri = Uri::route('users.show', ['user' => 1]);
$uri = Uri::signedRoute('users.show', ['user' => 1]);
$uri = Uri::temporarySignedRoute('user.index', now()->addMinutes(5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// 從當前請求 URL 產生 URI 實例...
$uri = $request->uri();
```

一旦您有了 URI 實例，您就可以流暢地修改它：

```php
$uri = Uri::of('https://example.com')
    ->withScheme('http')
    ->withHost('test.com')
    ->withPort(8000)
    ->withPath('/users')
    ->withQuery(['page' => 2])
    ->withFragment('section-1');
```

<a name="inspecting-uris"></a>
#### 檢查 URI

`Uri` 類別還允許您輕鬆檢查底層 URI 的各種元件：

```php
$scheme = $uri->scheme();
$host = $uri->host();
$port = $uri->port();
$path = $uri->path();
$segments = $uri->pathSegments();
$query = $uri->query();
$fragment = $uri->fragment();
```

<a name="manipulating-query-strings"></a>
#### 操作查詢字串

`Uri` 類別提供了多種方法，可用於操作 URI 的查詢字串。`withQuery` 方法可用於將額外的查詢字串參數合併到現有的查詢字串中：

```php
$uri = $uri->withQuery(['sort' => 'name']);
```

`withQueryIfMissing` 方法可用於在給定鍵不存在於查詢字串中時，將額外的查詢字串參數合併到現有的查詢字串中：

```php
$uri = $uri->withQueryIfMissing(['page' => 1]);
```

`replaceQuery` 方法可用於完全替換現有的查詢字串：

```php
$uri = $uri->replaceQuery(['page' => 1]);
```

`pushOntoQuery` 方法可用於將額外參數推送到具有陣列值的查詢字串參數上：

```php
$uri = $uri->pushOntoQuery('filter', ['active', 'pending']);
```

`withoutQuery` 方法可用於從查詢字串中移除參數：

```php
$uri = $uri->withoutQuery(['page']);
```

<a name="generating-responses-from-uris"></a>
#### 從 URI 產生回應

`redirect` 方法可用於為給定的 URI 產生 `RedirectResponse` 實例：

```php
$uri = Uri::of('https://example.com');

return $uri->redirect();
```

或者，您可以直接從路由或控制器動作返回 `Uri` 實例，這將自動為返回的 URI 產生重新導向回應：

```php
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Uri;

Route::get('/redirect', function () {
    return Uri::to('/index')
        ->withQuery(['sort' => 'name']);
});
```
