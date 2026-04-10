# 助手函數

- [簡介](#introduction)
- [可用方法](#available-methods)
- [其他工具](#other-utilities)
    - [效能基準測試](#benchmarking)
    - [日期與時間](#dates)
    - [延遲函數](#deferred-functions)
    - [抽獎(Lottery)](#lottery)
    - [管線(Pipeline)](#pipeline)
    - [休眠(Sleep)](#sleep)
    - [時間盒(Timebox)](#timebox)
    - [URI](#uri)

<a name="introduction"></a>
## 簡介

Laravel 包含多種全域的「助手(helper)」PHP 函數。框架本身就使用了其中許多函數；然而，如果您覺得方便，也可以在自己的應用程式中使用它們。


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
[Arr::exceptValues](#method-array-except-values)
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
[Arr::onlyValues](#method-array-only-values)
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
### 雜項

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

`Arr::accessible` 方法用於判定給定的值是否可作為陣列存取：

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

`Arr::add` 方法會在給定的鍵 (key) 不存在於陣列中或被設定為 `null` 時，將給定的鍵 / 值配對新增至陣列中：

```php
use Illuminate\Support\Arr;

$array = Arr::add(['name' => 'Desk'], 'price', 100);

// ['name' => 'Desk', 'price' => 100]

$array = Arr::add(['name' => 'Desk', 'price' => null], 'price', 100);

// ['name' => 'Desk', 'price' => 100]
```


<a name="method-array-array"></a>
#### `Arr::array()` {.collection-method}

`Arr::array` 方法使用「點」記法 (dot notation) 從深層嵌套的陣列中獲取值（就像 [Arr::get()](#method-array-get) 一樣），但如果請求的值不是 `array`，則會拋出 `InvalidArgumentException`：

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

`Arr::boolean` 方法使用「點」記法 (dot notation) 從深層嵌套的陣列中獲取值（就像 [Arr::get()](#method-array-get) 一樣），但如果請求的值不是 `boolean`，則會拋出 `InvalidArgumentException`：

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

`Arr::collapse` 方法將陣列中的陣列或集合 (collections) 合併成單個陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::collapse([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```


<a name="method-array-crossjoin"></a>
#### `Arr::crossJoin()` {.collection-method}

`Arr::crossJoin` 方法對給定的陣列進行交叉連接 (cross join)，返回包含所有可能排列的笛卡爾積 (Cartesian product)：

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

`Arr::divide` 方法會返回兩個陣列：一個包含鍵，另一個包含給定陣列的值：

```php
use Illuminate\Support\Arr;

[$keys, $values] = Arr::divide(['name' => 'Desk']);

// $keys: ['name']

// $values: ['Desk']
```


<a name="method-array-dot"></a>
#### `Arr::dot()` {.collection-method}

`Arr::dot` 方法將多維陣列扁平化為單層陣列，並使用「點」記法來表示深度：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$flattened = Arr::dot($array);

// ['products.desk.price' => 100]
```


<a name="method-array-every"></a>
#### `Arr::every()` {.collection-method}

`Arr::every` 方法確保陣列中的所有值都通過給定的真值測試：

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

`Arr::except` 方法從陣列中移除給定的鍵 / 值配對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100];

$filtered = Arr::except($array, ['price']);

// ['name' => 'Desk']
```


<a name="method-array-except-values"></a>
#### `Arr::exceptValues()` {.collection-method}

`Arr::exceptValues` 方法從陣列中移除指定的值：

```php
use Illuminate\Support\Arr;

$array = ['foo', 'bar', 'baz', 'qux'];

$filtered = Arr::exceptValues($array, ['foo', 'baz']);

// ['bar', 'qux']
```

您也可以將 `true` 傳遞給 `strict` 引數，以便在篩選時使用嚴格型別比較：

```php
use Illuminate\Support\Arr;

$array = [1, '1', 2, '2'];

$filtered = Arr::exceptValues($array, [1, 2], strict: true);

// ['1', '2']
```


<a name="method-array-exists"></a>
#### `Arr::exists()` {.collection-method}

`Arr::exists` 方法檢查給定的鍵是否存在於提供的陣列中：

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

`Arr::first` 方法返回陣列中第一個通過給定真值測試的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300];

$first = Arr::first($array, function (int $value, int $key) {
    return $value >= 150;
});

// 200
```

您也可以將預設值作為方法的第三個參數傳遞。如果沒有任何值通過真值測試，將返回此值：

```php
use Illuminate\Support\Arr;

$first = Arr::first($array, $callback, $default);
```


<a name="method-array-flatten"></a>
#### `Arr::flatten()` {.collection-method}

`Arr::flatten` 方法將多維陣列扁平化為單層陣列：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

$flattened = Arr::flatten($array);

// ['Joe', 'PHP', 'Ruby']
```


<a name="method-array-float"></a>
#### `Arr::float()` {.collection-method}

`Arr::float` 方法使用「點」記法 (dot notation) 從深層嵌套的陣列中獲取值（就像 [Arr::get()](#method-array-get) 一樣），但如果請求的值不是 `float`，則會拋出 `InvalidArgumentException`：

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

`Arr::forget` 方法使用「點」記法 (dot notation) 從深層嵌套的陣列中移除給定的鍵 / 值配對：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::forget($array, 'products.desk');

// ['products' => []]
```


<a name="method-array-from"></a>
#### `Arr::from()` {.collection-method}

`Arr::from` 方法將各種輸入類型轉換為純 PHP 陣列。它支援一系列輸入類型，包括陣列、物件以及數個常見的 Laravel 介面，例如 `Arrayable`、`Enumerable`、`Jsonable` 和 `JsonSerializable`。此外，它還處理 `Traversable` 和 `WeakMap` 實例：

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

`Arr::get` 方法使用「點」符號 (dot notation) 從深層巢狀陣列中取得值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$price = Arr::get($array, 'products.desk.price');

// 100
```

`Arr::get` 方法也接受一個預設值，如果指定鍵不存在於陣列中，則會回傳該預設值：

```php
use Illuminate\Support\Arr;

$discount = Arr::get($array, 'products.desk.discount', 0);

// 0
```


<a name="method-array-has"></a>
#### `Arr::has()` {.collection-method}

`Arr::has` 方法使用「點」符號 (dot notation) 檢查陣列中是否存在給定的項目：

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

`Arr::hasAll` 方法使用「點」符號 (dot notation) 判定給定陣列中是否包含所有指定的鍵：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Taylor', 'language' => 'PHP'];

Arr::hasAll($array, ['name']); // true
Arr::hasAll($array, ['name', 'language']); // true
Arr::hasAll($array, ['name', 'IDE']); // false
```


<a name="method-array-hasany"></a>
#### `Arr::hasAny()` {.collection-method}

`Arr::hasAny` 方法使用「點」符號 (dot notation) 檢查給定集合中的任何項目是否存在於陣列中：

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

`Arr::integer` 方法使用「點」符號 (dot notation) 從深層巢狀陣列中取得值（就像 [Arr::get()](#method-array-get) 一樣），但如果請求的值不是 `int`，則會拋出 `InvalidArgumentException`：

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

`Arr::isAssoc` 方法在給定陣列為關聯陣列 (associative array) 時回傳 `true`。如果陣列不具備從零開始的連續數字鍵，則被視為「關聯陣列」：

```php
use Illuminate\Support\Arr;

$isAssoc = Arr::isAssoc(['product' => ['name' => 'Desk', 'price' => 100]]);

// true

$isAssoc = Arr::isAssoc([1, 2, 3]);

// false
```


<a name="method-array-islist"></a>
#### `Arr::isList()` {.collection-method}

`Arr::isList` 方法在給定陣列的鍵為從零開始的連續整數時回傳 `true`：

```php
use Illuminate\Support\Arr;

$isList = Arr::isList(['foo', 'bar', 'baz']);

// true

$isList = Arr::isList(['product' => ['name' => 'Desk', 'price' => 100]]);

// false
```


<a name="method-array-join"></a>
#### `Arr::join()` {.collection-method}

`Arr::join` 方法使用字串連接陣列元素。透過此方法的第三個引數，您還可以為陣列的最後一個元素指定連接字串：

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

`Arr::keyBy` 方法根據給定的鍵對陣列進行重新鍵值化。如果多個項目具有相同的鍵，則只有最後一個項目會出現在新陣列中：

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

`Arr::last` 方法回傳陣列中最後一個通過給定真值測試 (truth test) 的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300, 110];

$last = Arr::last($array, function (int $value, int $key) {
    return $value >= 150;
});

// 300
```

您也可以將預設值作為該方法的第三個引數傳遞。如果沒有值通過真值測試，則會回傳此預設值：

```php
use Illuminate\Support\Arr;

$last = Arr::last($array, $callback, $default);
```


<a name="method-array-map"></a>
#### `Arr::map()` {.collection-method}

`Arr::map` 方法遍歷陣列，並將每個值和鍵傳遞給給定的回呼函數 (callback)。陣列的值將被替換為回呼函數回傳的值：

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

`Arr::mapSpread` 方法遍歷陣列，將每個巢狀項目的值傳遞給給定的閉包 (closure)。閉包可以修改該項目並將其回傳，從而形成一個由修改後項目組成的新陣列：

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

`Arr::mapWithKeys` 方法遍歷陣列，並將每個值傳遞給給定的回呼函數。回呼函數應該回傳一個包含單一鍵值對的關聯陣列：

```php
use Illuminate\Support\Arr;

$array = [
    [
        'name' => 'John',
        'department' => 'Sales',
        'email' => 'john@example.com',
    ],
    [
        'name' => 'Jane',
        'department' => 'Marketing',
        'email' => 'jane@example.com',
    ]
];

$mapped = Arr::mapWithKeys($array, function (array $item, int $key) {
    return [$item['email'] => $item['name']];
});

/*
    [
        'john@example.com' => 'John',
        'jane@example.com' => 'Jane',
    ]
*/
```


<a name="method-array-only"></a>
#### `Arr::only()` {.collection-method}

`Arr::only` 方法僅從給定陣列中回傳指定的鍵值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100, 'orders' => 10];

$slice = Arr::only($array, ['name', 'price']);

// ['name' => 'Desk', 'price' => 100]
```


<a name="method-array-only-values"></a>
#### `Arr::onlyValues()` {.collection-method}

`Arr::onlyValues` 方法僅從陣列中回傳指定的值：

```php
use Illuminate\Support\Arr;

$array = ['foo', 'bar', 'baz', 'qux'];

$filtered = Arr::onlyValues($array, ['foo', 'baz']);

// ['foo', 'baz']
```

您也可以將 `true` 傳遞給 `strict` 引數，以便在篩選時使用嚴格型別比較：

```php
use Illuminate\Support\Arr;

$array = [1, '1', 2, '2'];

$filtered = Arr::onlyValues($array, [1, 2], strict: true);

// [1, 2]
```


<a name="method-array-partition"></a>
#### `Arr::partition()` {.collection-method}

`Arr::partition` 方法可以與 PHP 的陣列解構 (array destructuring) 結合使用，將通過給定真值測試的元素與未通過的元素分開：

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

`Arr::pluck` 方法從陣列中提取指定鍵的所有值：

```php
use Illuminate\Support\Arr;

$array = [
    ['developer' => ['id' => 1, 'name' => 'Taylor']],
    ['developer' => ['id' => 2, 'name' => 'Abigail']],
];

$names = Arr::pluck($array, 'developer.name');

// ['Taylor', 'Abigail']
```

您也可以指定結果清單的鍵名方式：

```php
use Illuminate\Support\Arr;

$names = Arr::pluck($array, 'developer.name', 'developer.id');

// [1 => 'Taylor', 2 => 'Abigail']
```


<a name="method-array-prepend"></a>
#### `Arr::prepend()` {.collection-method}

`Arr::prepend` 方法將一個項目推送到陣列的開頭：

```php
use Illuminate\Support\Arr;

$array = ['one', 'two', 'three', 'four'];

$array = Arr::prepend($array, 'zero');

// ['zero', 'one', 'two', 'three', 'four']
```

如果需要，您可以指定該值應使用的鍵名：

```php
use Illuminate\Support\Arr;

$array = ['price' => 100];

$array = Arr::prepend($array, 'Desk', 'name');

// ['name' => 'Desk', 'price' => 100]
```


<a name="method-array-prependkeyswith"></a>
#### `Arr::prependKeysWith()` {.collection-method}

`Arr::prependKeysWith` 會為關聯陣列的所有鍵名加上指定的前綴：

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

`Arr::pull` 方法從陣列中取出並移除一個鍵/值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100];

$name = Arr::pull($array, 'name');

// $name: Desk

// $array: ['price' => 100]
```

您也可以將預設值作為方法的第三個引數傳入。如果該鍵不存在，則會返回此預設值：

```php
use Illuminate\Support\Arr;

$value = Arr::pull($array, $key, $default);
```


<a name="method-array-push"></a>
#### `Arr::push()` {.collection-method}

`Arr::push` 方法使用「點」符號表示法將一個項目推送到陣列中。如果在給定的鍵路徑下不存在陣列，它將會被建立：

```php
use Illuminate\Support\Arr;

$array = [];

Arr::push($array, 'office.furniture', 'Desk');

// $array: ['office' => ['furniture' => ['Desk']]]
```


<a name="method-array-query"></a>
#### `Arr::query()` {.collection-method}

`Arr::query` 方法將陣列轉換為查詢字串：

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

`Arr::random` 方法從陣列中隨機返回一個值：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3, 4, 5];

$random = Arr::random($array);

// 4 - (retrieved randomly)
```

您也可以將要返回的項目數量作為可選的第二個引數傳入。請注意，提供此引數將返回一個陣列，即使只需要一個項目也是如此：

```php
use Illuminate\Support\Arr;

$items = Arr::random($array, 2);

// [2, 5] - (retrieved randomly)
```


<a name="method-array-reject"></a>
#### `Arr::reject()` {.collection-method}

`Arr::reject` 方法使用給定的閉包從陣列中移除項目：

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

`Arr::select` 方法從陣列中選擇出一組值：

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

`Arr::set` 方法使用「點」符號表示法在深層嵌套陣列中設定值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::set($array, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```


<a name="method-array-shuffle"></a>
#### `Arr::shuffle()` {.collection-method}

`Arr::shuffle` 方法隨機打亂陣列中的項目：

```php
use Illuminate\Support\Arr;

$array = Arr::shuffle([1, 2, 3, 4, 5]);

// [3, 2, 5, 1, 4] - (generated randomly)
```


<a name="method-array-sole"></a>
#### `Arr::sole()` {.collection-method}

`Arr::sole` 方法使用給定的閉包從陣列中取得單一值。如果陣列中有超過一個值符合給定的真值測試，將會拋出 `Illuminate\Support\MultipleItemsFoundException` 異常。如果沒有任何值符合真值測試，則會拋出 `Illuminate\Support\ItemNotFoundException` 異常：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$value = Arr::sole($array, fn (string $value) => $value === 'Desk');

// 'Desk'
```


<a name="method-array-some"></a>
#### `Arr::some()` {.collection-method}

`Arr::some` 方法確保陣列中至少有一個值通過給定的真值測試：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3];

Arr::some($array, fn ($i) => $i > 2);

// true
```


<a name="method-array-sort"></a>
#### `Arr::sort()` {.collection-method}

`Arr::sort` 方法根據值對陣列進行排序：

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

`Arr::sortDesc` 方法根據值將陣列按降序排列：

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

`Arr::sortRecursive` 方法遞迴地對陣列進行排序，針對數字索引的子陣列使用 `sort` 函數，針對關聯子陣列使用 `ksort` 函數：

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

如果您希望結果按降序排列，可以使用 `Arr::sortRecursiveDesc` 方法。

```php
$sorted = Arr::sortRecursiveDesc($array);
```

<a name="method-array-string"></a>
#### `Arr::string()` {.collection-method}

`Arr::string` 方法使用「點號記法」從深層巢狀陣列中取得值（就像 [Arr::get()](#method-array-get) 一樣），但如果請求的值不是 `string`，則會拋出 `InvalidArgumentException`：

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

`Arr::take` 方法會回傳一個包含指定數量項目的新陣列：

```php
use Illuminate\Support\Arr;

$array = [0, 1, 2, 3, 4, 5];

$chunk = Arr::take($array, 3);

// [0, 1, 2]
```

您也可以傳入負整數，以從陣列的末端取得指定數量的項目：

```php
$array = [0, 1, 2, 3, 4, 5];

$chunk = Arr::take($array, -2);

// [4, 5]
```


<a name="method-array-to-css-classes"></a>
#### `Arr::toCssClasses()` {.collection-method}

`Arr::toCssClasses` 方法會根據條件編譯 CSS 類別字串。該方法接收一個類別陣列，其中陣列的鍵包含您想要新增的類別，而值則是布林運算式。如果陣列元素具有數字鍵，它將始終被包含在渲染後的類別清單中：

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

`Arr::toCssStyles` 方法會根據條件編譯 CSS 樣式字串。該方法接收一個 CSS 宣告陣列，其中陣列的鍵包含您想要新增的 CSS 宣告，而值則是布林運算式。如果陣列元素具有數字鍵，它將始終被包含在編譯後的 CSS 樣式字串中：

```php
use Illuminate\Support\Arr;

$hasColor = true;

$array = ['background-color: blue', 'color: blue' => $hasColor];

$classes = Arr::toCssStyles($array);

/*
    'background-color: blue; color: blue;'
*/
```

此方法驅動了 Laravel 的功能，允許[將類別與 Blade 元件的屬性袋 (attribute bag) 合併](/docs/{{version}}/blade#conditionally-merge-classes) 以及使用 `@class` [Blade 指令](/docs/{{version}}/blade#conditional-classes)。


<a name="method-array-undot"></a>
#### `Arr::undot()` {.collection-method}

`Arr::undot` 方法將使用「點號記法」的一維陣列展開為多維陣列：

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

`Arr::where` 方法使用給定的閉包來篩選陣列：

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

`Arr::wrap` 方法將給定值封裝在陣列中。如果給定值已經是陣列，則會直接回傳而無需修改：

```php
use Illuminate\Support\Arr;

$string = 'Laravel';

$array = Arr::wrap($string);

// ['Laravel']
```

如果給定值為 `null`，將回傳一個空陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::wrap(null);

// []
```


<a name="method-data-fill"></a>
#### `data_fill()` {.collection-method}

`data_fill` 函數使用「點號記法」在巢狀陣列或物件中設定缺失的值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_fill($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 100]]]

data_fill($data, 'products.desk.discount', 10);

// ['products' => ['desk' => ['price' => 100, 'discount' => 10]]]
```

此函數也接受星號作為萬用字元，並會相應地填充目標：

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

`data_get` 函數使用「點號記法」從巢狀陣列或物件中取得值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

$price = data_get($data, 'products.desk.price');

// 100
```

`data_get` 函數也接受一個預設值，如果找不到指定的鍵，則會回傳該預設值：

```php
$discount = data_get($data, 'products.desk.discount', 0);

// 0
```

該函數也接受使用星號的萬用字元，可用於針對陣列或物件的任何鍵：

```php
$data = [
    'product-one' => ['name' => 'Desk 1', 'price' => 100],
    'product-two' => ['name' => 'Desk 2', 'price' => 150],
];

data_get($data, '*.name');

// ['Desk 1', 'Desk 2'];
```

`{first}` 和 `{last}` 佔位符可用於取得陣列中的第一個或最後一個項目：

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

`data_set` 函數使用「點號記法」在巢狀陣列或物件中設定值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```

此函數也接受使用星號的萬用字元，並會相應地在目標上設定值：

```php
$data = [
    'products' => [
        ['name' => 'Desk 1', 'price' => 100],
        ['name' => 'Desk 2', 'price' => 150],
    ],
];

data_set($data, 'products.*.price', 200);

/*
    [
        'products' => [
            ['name' => 'Desk 1', 'price' => 200],
            ['name' => 'Desk 2', 'price' => 200],
        ],
    ]
*/
```

預設情況下，任何現有的值都會被覆寫。如果您希望僅在值不存在時才設定，可以將 `false` 作為函數的第四個引數傳入：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200, overwrite: false);

// ['products' => ['desk' => ['price' => 100]]]
```


<a name="method-data-forget"></a>
#### `data_forget()` {.collection-method}

`data_forget` 函數使用「點號記法」移除巢狀陣列或物件中的值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_forget($data, 'products.desk.price');

// ['products' => ['desk' => []]]
```

此函數也接受使用星號的萬用字元，並會相應地移除目標中的值：

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

`head` 函數回傳給定陣列中的第一個元素。如果陣列為空，則會回傳 `false`：

```php
$array = [100, 200, 300];

$first = head($array);

// 100
```

<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 函數會回傳給定陣列中的最後一個元素。如果陣列為空，將會回傳 `false`：

```php
$array = [100, 200, 300];

$last = last($array);

// 300
```

<a name="numbers"></a>
## 數字


<a name="method-number-abbreviate"></a>
#### `Number::abbreviate()` {.collection-method}

`Number::abbreviate` 方法會回傳所提供數值的易於閱讀格式，並對單位進行縮寫：

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

`Number::clamp` 方法確保給定的數字保持在指定範圍內。如果數字低於最小值，則回傳最小值；如果數字高於最大值，則回傳最大值：

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

`Number::currency` 方法會將給定值的貨幣表示形式回傳為字串：

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

`Number::defaultCurrency` 方法會回傳 `Number` 類別目前使用的預設貨幣：

```php
use Illuminate\Support\Number;

$currency = Number::defaultCurrency();

// USD
```


<a name="method-default-locale"></a>
#### `Number::defaultLocale()` {.collection-method}

`Number::defaultLocale` 方法會回傳 `Number` 類別目前使用的預設語系：

```php
use Illuminate\Support\Number;

$locale = Number::defaultLocale();

// en
```


<a name="method-number-file-size"></a>
#### `Number::fileSize()` {.collection-method}

`Number::fileSize` 方法會將給定位元組 (byte) 值的檔案大小表示形式回傳為字串：

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

`Number::forHumans` 方法會回傳所提供數值的易於閱讀格式：

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

`Number::format` 方法將給定數字格式化為特定語系的字串：

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

`Number::ordinal` 方法會回傳數字的序數表示形式：

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

`Number::pairs` 方法會根據指定的範圍和步進值產生數字對（子範圍）陣列。此方法可用於將較大的數字範圍劃分為較小、易於管理的子範圍，例如用於分頁或批次處理任務。`pairs` 方法會回傳一個陣列，其中每個內部陣列代表一對（子範圍）數字：

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

`Number::percentage` 方法會將給定值的百分比表示形式回傳為字串：

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

`Number::spell` 方法將給定數字轉換為單字字串：

```php
use Illuminate\Support\Number;

$number = Number::spell(102);

// one hundred and two

$number = Number::spell(88, locale: 'fr');

// quatre-vingt-huit
```

`after` 引數允許您指定一個數值，超過該數值的所有數字都應被拼寫成文字：

```php
$number = Number::spell(10, after: 10);

// 10

$number = Number::spell(11, after: 10);

// eleven
```

`until` 引數允許您指定一個數值，在此數值之前的所有數字都應被拼寫成文字：

```php
$number = Number::spell(5, until: 10);

// five

$number = Number::spell(10, until: 10);

// 10
```


<a name="method-number-spell-ordinal"></a>
#### `Number::spellOrdinal()` {.collection-method}

`Number::spellOrdinal` 方法會將數字的序數表示形式回傳為單字字串：

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

`Number::trim` 方法會移除給定數字小數點後的所有末尾零：

```php
use Illuminate\Support\Number;

$number = Number::trim(12.0);

// 12

$number = Number::trim(12.30);

// 12.3
```


<a name="method-number-use-locale"></a>
#### `Number::useLocale()` {.collection-method}

`Number::useLocale` 方法會全域設定預設的數字語系，這會影響後續呼叫 `Number` 類別方法時數字和貨幣的格式化方式：

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

`Number::withLocale` 方法會使用指定的語系執行給定的閉包，並在回呼函數執行後恢復原始語系：

```php
use Illuminate\Support\Number;

$number = Number::withLocale('de', function () {
    return Number::format(1500);
});
```


<a name="method-number-use-currency"></a>
#### `Number::useCurrency()` {.collection-method}

`Number::useCurrency` 方法會全域設定預設的貨幣，這會影響後續呼叫 `Number` 類別方法時貨幣的格式化方式：

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

`Number::withCurrency` 方法會使用指定的貨幣來執行給定的閉包，並在回呼函數執行完畢後恢復原來的貨幣：

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

`app_path` 函數會回傳您應用程式 `app` 目錄的完整路徑。您也可以使用 `app_path` 函數來產生相對於應用程式目錄之檔案的完整路徑：

```php
$path = app_path();

$path = app_path('Http/Controllers/Controller.php');
```

<a name="method-base-path"></a>
#### `base_path()` {.collection-method}

`base_path` 函數會回傳您應用程式根目錄的完整路徑。您也可以使用 `base_path` 函數來產生相對於專案根目錄之特定檔案的完整路徑：

```php
$path = base_path();

$path = base_path('vendor/bin');
```

<a name="method-config-path"></a>
#### `config_path()` {.collection-method}

`config_path` 函數會回傳您應用程式 `config` 目錄的完整路徑。您也可以使用 `config_path` 函數來產生應用程式設定目錄中特定檔案的完整路徑：

```php
$path = config_path();

$path = config_path('app.php');
```

<a name="method-database-path"></a>
#### `database_path()` {.collection-method}

`database_path` 函數會回傳您應用程式 `database` 目錄的完整路徑。您也可以使用 `database_path` 函數來產生資料庫目錄中特定檔案的完整路徑：

```php
$path = database_path();

$path = database_path('factories/UserFactory.php');
```

<a name="method-lang-path"></a>
#### `lang_path()` {.collection-method}

`lang_path` 函數會回傳您應用程式 `lang` 目錄的完整路徑。您也可以使用 `lang_path` 函數來產生該目錄中特定檔案的完整路徑：

```php
$path = lang_path();

$path = lang_path('en/messages.php');
```

> [!NOTE]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。如果您想要自定義 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令來發布它們。

<a name="method-public-path"></a>
#### `public_path()` {.collection-method}

`public_path` 函數會回傳您應用程式 `public` 目錄的完整路徑。您也可以使用 `public_path` 函數來產生 public 目錄中特定檔案的完整路徑：

```php
$path = public_path();

$path = public_path('css/app.css');
```

<a name="method-resource-path"></a>
#### `resource_path()` {.collection-method}

`resource_path` 函數會回傳您應用程式 `resources` 目錄的完整路徑。您也可以使用 `resource_path` 函數來產生 resources 目錄中特定檔案的完整路徑：

```php
$path = resource_path();

$path = resource_path('sass/app.scss');
```

<a name="method-storage-path"></a>
#### `storage_path()` {.collection-method}

`storage_path` 函數會回傳您應用程式 `storage` 目錄的完整路徑。您也可以使用 `storage_path` 函數來產生 storage 目錄中特定檔案的完整路徑：

```php
$path = storage_path();

$path = storage_path('app/file.txt');
```

<a name="urls"></a>
## URL

<a name="method-action"></a>
#### `action()` {.collection-method}

`action` 函數會為給定的控制器動作產生一個 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

如果該方法接受路由參數，您可以將它們作為該方法的第二個引數傳入：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="method-asset"></a>
#### `asset()` {.collection-method}

`asset` 函數會使用請求目前的方案 (scheme)（HTTP 或 HTTPS）為靜態資源產生一個 URL：

```php
$url = asset('img/photo.jpg');
```

您可以透過在 `.env` 檔案中設定 `ASSET_URL` 變數來配置靜態資源 URL 的主機。如果您將靜態資源託管在 Amazon S3 或其他 CDN 等外部服務上，這將非常有用：

```php
// ASSET_URL=http://example.com/assets

$url = asset('img/photo.jpg'); // http://example.com/assets/img/photo.jpg
```

<a name="method-route"></a>
#### `route()` {.collection-method}

`route` 函數會為給定的[具名路由](/docs/{{version}}/routing#named-routes)產生一個 URL：

```php
$url = route('route.name');
```

如果路由接受參數，您可以將它們作為函數的第二個引數傳入：

```php
$url = route('route.name', ['id' => 1]);
```

預設情況下，`route` 函數會產生一個絕對 URL。如果您想要產生一個相對 URL，可以將 `false` 作為函數的第三個引數傳入：

```php
$url = route('route.name', ['id' => 1], false);
```

<a name="method-secure-asset"></a>
#### `secure_asset()` {.collection-method}

`secure_asset` 函數會使用 HTTPS 為靜態資源產生一個 URL：

```php
$url = secure_asset('img/photo.jpg');
```

<a name="method-secure-url"></a>
#### `secure_url()` {.collection-method}

`secure_url` 函數會為給定的路徑產生一個完整路徑的 HTTPS URL。額外的 URL 段落可以傳遞在函數的第二個引數中：

```php
$url = secure_url('user/profile');

$url = secure_url('user/profile', [1]);
```

<a name="method-to-action"></a>
#### `to_action()` {.collection-method}

`to_action` 函數會為給定的控制器動作產生一個[重新導向的 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
use App\Http\Controllers\UserController;

return to_action([UserController::class, 'show'], ['user' => 1]);
```

如有必要，您可以將應分配給重新導向的 HTTP 狀態碼以及任何額外的回應標頭作為 `to_action` 方法的第三和第四個引數傳入：

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

`to_route` 函數會為給定的[具名路由](/docs/{{version}}/routing#named-routes)產生一個[重新導向的 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
return to_route('users.show', ['user' => 1]);
```

如有必要，您可以將應分配給重新導向的 HTTP 狀態碼以及任何額外的回應標頭作為 `to_route` 方法的第三和第四個引數傳入：

```php
return to_route('users.show', ['user' => 1], 302, ['X-Framework' => 'Laravel']);
```

<a name="method-uri"></a>
#### `uri()` {.collection-method}

`uri` 函數會為給定的 URI 產生一個[流暢的 URI 實例 (fluent URI instance)](#uri)：

```php
$uri = uri('https://example.com')
    ->withPath('/users')
    ->withQuery(['page' => 1]);
```

如果 `uri` 函數收到一個包含可呼叫的控制器與方法對的陣列，該函數將為該控制器方法的路由路徑建立一個 `Uri` 實例：

```php
use App\Http\Controllers\UserController;

$uri = uri([UserController::class, 'show'], ['user' => $user]);
```

如果控制器是可呼叫的 (invokable)，您只需提供控制器類別名稱即可：

```php
use App\Http\Controllers\UserIndexController;

$uri = uri(UserIndexController::class);
```

如果傳給 `uri` 函數的值與[具名路由](/docs/{{version}}/routing#named-routes)的名稱相符，則會為該路由的路徑產生一個 `Uri` 實例：

```php
$uri = uri('users.show', ['user' => $user]);
```

<a name="method-url"></a>
#### `url()` {.collection-method}

`url` 函數會為給定的路徑產生一個完整路徑的 URL：

```php
$url = url('user/profile');

$url = url('user/profile', [1]);
```

如果沒有提供路徑，將會回傳一個 `Illuminate\Routing\UrlGenerator` 實例：

```php
$current = url()->current();

$full = url()->full();

$previous = url()->previous();
```

有關使用 `url` 函數的更多資訊，請參閱 [URL 生成文件](/docs/{{version}}/urls#generating-urls)。

<a name="miscellaneous"></a>
## 其他


<a name="method-abort"></a>
#### `abort()` {.collection-method}

`abort` 函數會拋出 [一個 HTTP 異常](/docs/{{version}}/errors#http-exceptions)，該異常將由 [異常處理器](/docs/{{version}}/errors#handling-exceptions) 進行渲染：

```php
abort(403);
```

您也可以提供異常的訊息以及應發送到瀏覽器的自定義 HTTP 回應標頭：

```php
abort(403, 'Unauthorized.', $headers);
```


<a name="method-abort-if"></a>
#### `abort_if()` {.collection-method}

如果給定的布林表達式結果為 `true`，`abort_if` 函數將拋出一個 HTTP 異常：

```php
abort_if(! Auth::user()->isAdmin(), 403);
```

與 `abort` 方法類似，您也可以將異常的回應文字作為第三個引數，並將自定義回應標頭的陣列作為第四個引數傳遞給該函數。


<a name="method-abort-unless"></a>
#### `abort_unless()` {.collection-method}

如果給定的布林表達式結果為 `false`，`abort_unless` 函數將拋出一個 HTTP 異常：

```php
abort_unless(Auth::user()->isAdmin(), 403);
```

與 `abort` 方法類似，您也可以將異常的回應文字作為第三個引數，並將自定義回應標頭的陣列作為第四個引數傳遞給該函數。


<a name="method-app"></a>
#### `app()` {.collection-method}

`app` 函數會回傳 [服務容器](/docs/{{version}}/container) 實例：

```php
$container = app();
```

您可以傳遞類別或介面名稱，以便從容器中解析它：

```php
$api = app('HelpSpot\API');
```


<a name="method-auth"></a>
#### `auth()` {.collection-method}

`auth` 函數會回傳一個 [認證器](/docs/{{version}}/authentication) 實例。您可以使用它來替代 `Auth` Facade：

```php
$user = auth()->user();
```

如果需要，您可以指定想要存取的 guard 實例：

```php
$user = auth('admin')->user();
```


<a name="method-back"></a>
#### `back()` {.collection-method}

`back` 函數會產生一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)，將使用者導回之前的位置：

```php
return back($status = 302, $headers = [], $fallback = '/');

return back();
```


<a name="method-bcrypt"></a>
#### `bcrypt()` {.collection-method}

`bcrypt` 函數使用 Bcrypt 對給定值進行 [雜湊](/docs/{{version}}/hashing)。您可以使用此函數來替代 `Hash` Facade：

```php
$password = bcrypt('my-secret-password');
```


<a name="method-blank"></a>
#### `blank()` {.collection-method}

`blank` 函數用來判斷給定值是否為「空白」：

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

關於 `blank` 的反向操作，請參閱 [filled](#method-filled) 函數。


<a name="method-broadcast"></a>
#### `broadcast()` {.collection-method}

`broadcast` 函數將給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast(new UserRegistered($user));

broadcast(new UserRegistered($user))->toOthers();
```


<a name="method-broadcast-if"></a>
#### `broadcast_if()` {.collection-method}

如果給定的布林表達式結果為 `true`，`broadcast_if` 函數將把給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast_if($user->isActive(), new UserRegistered($user));

broadcast_if($user->isActive(), new UserRegistered($user))->toOthers();
```


<a name="method-broadcast-unless"></a>
#### `broadcast_unless()` {.collection-method}

如果給定的布林表達式結果為 `false`，`broadcast_unless` 函數將把給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽器：

```php
broadcast_unless($user->isBanned(), new UserRegistered($user));

broadcast_unless($user->isBanned(), new UserRegistered($user))->toOthers();
```


<a name="method-cache"></a>
#### `cache()` {.collection-method}

`cache` 函數可用於從 [快取](/docs/{{version}}/cache) 中獲取值。如果給定的鍵不存在於快取中，則會回傳一個可選的預設值：

```php
$value = cache('key');

$value = cache('key', 'default');
```

您可以透過向函數傳遞鍵/值對陣列來將項目添加到快取中。您還應該傳遞快取值應被視為有效的秒數或持續時間：

```php
cache(['key' => 'value'], 300);

cache(['key' => 'value'], now()->plus(seconds: 10));
```


<a name="method-class-uses-recursive"></a>
#### `class_uses_recursive()` {.collection-method}

`class_uses_recursive` 函數會回傳類別所使用的所有 trait，包括其所有父類別所使用的 trait：

```php
$traits = class_uses_recursive(App\Models\User::class);
```


<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 函數會根據給定值建立一個 [集合 (Collection)](/docs/{{version}}/collections) 實例：

```php
$collection = collect(['Taylor', 'Abigail']);
```


<a name="method-config"></a>
#### `config()` {.collection-method}

`config` 函數會獲取 [設定](/docs/{{version}}/configuration) 變數的值。可以使用「點」語法來存取設定值，其中包含檔案名稱以及您想要存取的選項。您也可以提供一個預設值，如果設定選項不存在，則回傳該預設值：

```php
$value = config('app.timezone');

$value = config('app.timezone', $default);
```

您可以透過傳遞鍵/值對陣列在執行時設定設定變數。但請注意，此函數僅影響當前請求的設定值，不會更新您實際的設定檔：

```php
config(['app.debug' => true]);
```


<a name="method-context"></a>
#### `context()` {.collection-method}

`context` 函數從目前的 [上下文](/docs/{{version}}/context) 中獲取值。您也可以提供一個預設值，如果上下文鍵不存在，則回傳該預設值：

```php
$value = context('trace_id');

$value = context('trace_id', $default);
```

您可以透過傳遞鍵/值對陣列來設定上下文值：

```php
use Illuminate\Support\Str;

context(['trace_id' => Str::uuid()->toString()]);
```


<a name="method-cookie"></a>
#### `cookie()` {.collection-method}

`cookie` 函數會建立一個新的 [cookie](/docs/{{version}}/requests#cookies) 實例：

```php
$cookie = cookie('name', 'value', $minutes);
```


<a name="method-csrf-field"></a>
#### `csrf_field()` {.collection-method}

`csrf_field` 函數會產生一個 HTML `hidden` 輸入欄位，其中包含 CSRF Token 的值。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
{{ csrf_field() }}
```


<a name="method-csrf-token"></a>
#### `csrf_token()` {.collection-method}

`csrf_token` 函數會獲取目前 CSRF Token 的值：

```php
$token = csrf_token();
```


<a name="method-decrypt"></a>
#### `decrypt()` {.collection-method}

`decrypt` 函數會 [解密](/docs/{{version}}/encryption) 給定值。您可以使用此函數來替代 `Crypt` Facade：

```php
$password = decrypt($value);
```

關於 `decrypt` 的反向操作，請參閱 [encrypt](#method-encrypt) 函數。


<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 函數會印出給定的變數並結束指令碼的執行：

```php
dd($value);

dd($value1, $value2, $value3, ...);
```

如果您不想停止指令碼的執行，請改用 [dump](#method-dump) 函數。


<a name="method-dispatch"></a>
#### `dispatch()` {.collection-method}

`dispatch` 函數將給定的 [工作](/docs/{{version}}/queues#creating-jobs) 推送到 Laravel 的 [工作佇列](/docs/{{version}}/queues) 中：

```php
dispatch(new App\Jobs\SendEmails);
```

<a name="method-dispatch-sync"></a>
#### `dispatch_sync()` {.collection-method}

`dispatch_sync` 函數將給定的工作推送到 [同步(sync)](/docs/{{version}}/queues#synchronous-dispatching) 佇列中，以便立即處理：

```php
dispatch_sync(new App\Jobs\SendEmails);
```


<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 函數會印出給定的變數：

```php
dump($value);

dump($value1, $value2, $value3, ...);
```

如果您想在印出變數後停止執行指令碼，請改用 [dd](#method-dd) 函數。


<a name="method-encrypt"></a>
#### `encrypt()` {.collection-method}

`encrypt` 函數會[加密](/docs/{{version}}/encryption)給定的值。您可以使用此函數來替代 `Crypt` Facade：

```php
$secret = encrypt('my-secret-value');
```

關於 `encrypt` 的反向操作，請參閱 [decrypt](#method-decrypt) 函數。


<a name="method-env"></a>
#### `env()` {.collection-method}

`env` 函數會取得[環境變數](/docs/{{version}}/configuration#environment-configuration)的值，或回傳一個預設值：

```php
$env = env('APP_ENV');

$env = env('APP_ENV', 'production');
```

> [!WARNING]
> 如果您在部署過程中執行了 `config:cache` 指令，請務必確保您僅在設定檔中呼叫 `env` 函數。一旦設定被快取，`.env` 檔案將不會被載入，所有對 `env` 函數的呼叫都將回傳外部環境變數（例如伺服器層級或系統層級的環境變數）或 `null`。


<a name="method-event"></a>
#### `event()` {.collection-method}

`event` 函數會將給定的[事件](/docs/{{version}}/events)分發給其監聽器：

```php
event(new UserRegistered($user));
```


<a name="method-fake"></a>
#### `fake()` {.collection-method}

`fake` 函數會從容器中解析出一個 [Faker](https://github.com/FakerPHP/Faker) 單例(singleton)，這在模型工廠(model factories)、資料填充(database seeding)、測試以及原型視圖中建立假資料時非常有用：

```blade
@for ($i = 0; $i < 10; $i++)
    <dl>
        <dt>Name</dt>
        <dd>{{ fake()->name() }}</dd>

        <dt>Email</dt>
        <dd>{{ fake()->unique()->safeEmail() }}</dd>
    </dl>
@endfor
```

預設情況下，`fake` 函數將使用 `config/app.php` 設定中的 `app.faker_locale` 選項。通常此設定選項是透過 `APP_FAKER_LOCALE` 環境變數來設定的。您也可以透過將 locale 傳遞給 `fake` 函數來指定語言區域。每個 locale 都會解析出一個獨立的單例：

```php
fake('nl_NL')->name()
```


<a name="method-filled"></a>
#### `filled()` {.collection-method}

`filled` 函數會判斷給定的值是否不是「空白」的：

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

關於 `filled` 的反向操作，請參閱 [blank](#method-blank) 函數。


<a name="method-info"></a>
#### `info()` {.collection-method}

`info` 函數會將資訊寫入應用程式的[日誌](/docs/{{version}}/logging)：

```php
info('Some helpful information!');
```

您也可以將包含上下文資料的陣列傳遞給此函數：

```php
info('User login attempt failed.', ['id' => $user->id]);
```


<a name="method-literal"></a>
#### `literal()` {.collection-method}

`literal` 函數會建立一個新的 [stdClass](https://www.php.net/manual/en/class.stdclass.php) 實例，並將給定的具名引數(named arguments)作為屬性：

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

`logger` 函數可用於將 `debug` 層級的訊息寫入[日誌](/docs/{{version}}/logging)：

```php
logger('Debug message');
```

您也可以將包含上下文資料的陣列傳遞給此函數：

```php
logger('User has logged in.', ['id' => $user->id]);
```

如果沒有傳遞任何值給此函數，則會回傳一個 [logger](/docs/{{version}}/logging) 實例：

```php
logger()->error('You are not allowed here.');
```


<a name="method-method-field"></a>
#### `method_field()` {.collection-method}

`method_field` 函數會產生一個 HTML `hidden` 輸入欄位，其中包含表單 HTTP 動詞的偽裝值。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
<form method="POST">
    {{ method_field('DELETE') }}
</form>
```


<a name="method-now"></a>
#### `now()` {.collection-method}

`now` 函數會為目前時間建立一個新的 `Illuminate\Support\Carbon` 實例：

```php
$now = now();
```


<a name="method-old"></a>
#### `old()` {.collection-method}

`old` 函數會[取得](/docs/{{version}}/requests#retrieving-input)快閃(flashed)到工作階段(session)中的[舊輸入](/docs/{{version}}/requests#old-input)值：

```php
$value = old('value');

$value = old('value', 'default');
```

由於傳遞給 `old` 函數作為第二個引數的「預設值」通常是 Eloquent 模型的屬性，因此 Laravel 允許您直接將整個 Eloquent 模型作為第二個引數傳遞給 `old` 函數。這樣做時，Laravel 會假設傳遞給 `old` 函數的第一個引數是應被視為「預設值」的 Eloquent 屬性名稱：

```blade
{{ old('name', $user->name) }}

// Is equivalent to...

{{ old('name', $user) }}
```


<a name="method-once"></a>
#### `once()` {.collection-method}

`once` 函數會執行給定的回呼函數(callback)，並將結果在請求期間快取在記憶體中。隨後任何使用相同回呼函數呼叫 `once` 函數的操作都將回傳先前快取的結果：

```php
function random(): int
{
    return once(function () {
        return random_int(1, 1000);
    });
}

random(); // 123
random(); // 123 (cached result)
random(); // 123 (cached result)
```

當 `once` 函數在物件實例中執行時，快取的結果將對該物件實例唯一：

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
$service->all(); // (cached result)

$secondService = new NumberService;

$secondService->all();
$secondService->all(); // (cached result)
```

<a name="method-optional"></a>
#### `optional()` {.collection-method}

`optional` 函數接受任何引數，並允許您存取該物件的屬性或呼叫其方法。如果給定的物件是 `null`，屬性和方法將回傳 `null` 而不會導致錯誤：

```php
return optional($user->address)->street;

{!! old('name', optional($user)->name) !!}
```

`optional` 函數還接受一個閉包(closure)作為其第二個引數。如果作為第一個引數提供的值不是 null，則會執行該閉包：

```php
return optional(User::find($id), function (User $user) {
    return $user->name;
});
```


<a name="method-policy"></a>
#### `policy()` {.collection-method}

`policy` 方法會為給定類別取得一個 [policy](/docs/{{version}}/authorization#creating-policies) 實例：

```php
$policy = policy(App\Models\User::class);
```


<a name="method-redirect"></a>
#### `redirect()` {.collection-method}

`redirect` 函數會回傳一個[重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)，或者在沒有傳遞引數時回傳重新導向器(redirector)實例：

```php
return redirect($to = null, $status = 302, $headers = [], $secure = null);

return redirect('/home');

return redirect()->route('route.name');
```


<a name="method-report"></a>
#### `report()` {.collection-method}

`report` 函數會使用您的[異常處理程序](/docs/{{version}}/errors#handling-exceptions)來回報異常：

```php
report($e);
```

`report` 函數也接受一個字串作為引數。當提供字串時，該函數會建立一個以該字串作為訊息的異常：

```php
report('Something went wrong.');
```

<a name="method-report-if"></a>
#### `report_if()` {.collection-method}

`report_if` 函數會在給定的布林運算式為 `true` 時，使用您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 回報例外：

```php
report_if($shouldReport, $e);

report_if($shouldReport, 'Something went wrong.');
```


<a name="method-report-unless"></a>
#### `report_unless()` {.collection-method}

`report_unless` 函數會在給定的布林運算式為 `false` 時，使用您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 回報例外：

```php
report_unless($reportingDisabled, $e);

report_unless($reportingDisabled, 'Something went wrong.');
```


<a name="method-request"></a>
#### `request()` {.collection-method}

`request` 函數會回傳目前的 [請求](/docs/{{version}}/requests) 實例，或是從目前的請求中取得輸入欄位的值：

```php
$request = request();

$value = request('key', $default);
```


<a name="method-rescue"></a>
#### `rescue()` {.collection-method}

`rescue` 函數會執行給定的閉包並捕捉執行過程中發生的任何例外。所有捕捉到的例外都會被傳送到您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions)；然而，請求將會繼續處理：

```php
return rescue(function () {
    return $this->method();
});
```

您也可以傳遞第二個引數給 `rescue` 函數。這個引數將作為在執行閉包發生例外時應回傳的「預設」值：

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

您可以提供 `report` 引數給 `rescue` 函數，以決定該例外是否應該透過 `report` 函數回報：

```php
return rescue(function () {
    return $this->method();
}, report: function (Throwable $throwable) {
    return $throwable instanceof InvalidArgumentException;
});
```


<a name="method-resolve"></a>
#### `resolve()` {.collection-method}

`resolve` 函數會使用 [服務容器](/docs/{{version}}/container) 將給定的類別或介面名稱解析為實例：

```php
$api = resolve('HelpSpot\API');
```


<a name="method-response"></a>
#### `response()` {.collection-method}

`response` 函數會建立一個 [回應](/docs/{{version}}/responses) 實例，或取得回應工廠的實例：

```php
return response('Hello World', 200, $headers);

return response()->json(['foo' => 'bar'], 200, $headers);
```


<a name="method-retry"></a>
#### `retry()` {.collection-method}

`retry` 函數會嘗試執行給定的回呼函數，直到達到給定的最大嘗試門檻。如果回呼函數沒有拋出例外，則會回傳其回傳值。如果回呼函數拋出例外，它將會自動重試。如果超過最大嘗試次數，則會拋出該例外：

```php
return retry(5, function () {
    // Attempt 5 times while resting 100ms between attempts...
}, 100);
```

休眠時長也接受 `CarbonInterval` 實例：

```php
use function Illuminate\Support\seconds;

return retry(5, function () {
    // Attempt 5 times while resting 5 seconds between attempts...
}, seconds(5));
```

如果您想手動計算每次嘗試之間休眠的毫秒數，您可以傳遞一個閉包作為 `retry` 函數的第三個引數：

```php
use Exception;

return retry(5, function () {
    // ...
}, function (int $attempt, Exception $exception) {
    return $attempt * 100;
});
```

為了方便起見，您可以提供一個陣列作為 `retry` 函數的第一個引數。此陣列將用於決定後續每次嘗試之間應休眠多少毫秒：

```php
return retry([100, 200], function () {
    // Sleep for 100ms on first retry, 200ms on second retry...
});
```

若要僅在特定條件下重試，您可以傳遞一個閉包作為 `retry` 函數的第四個引數：

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

`session` 函數可用於取得或設定 [會話](/docs/{{version}}/session) 值：

```php
$value = session('key');
```

您可以透過傳遞一個鍵值對陣列給該函數來設定值：

```php
session(['chairs' => 7, 'instruments' => 3]);
```

如果沒有傳遞任何值給該函數，將會回傳會話儲存(session store)實例：

```php
$value = session()->get('key');

session()->put('key', $value);
```


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 函數接受兩個引數：一個任意的 `$value` 和一個閉包。`$value` 將被傳遞給閉包，然後由 `tap` 函數回傳。閉包的回傳值並不重要：

```php
$user = tap(User::first(), function (User $user) {
    $user->name = 'Taylor';

    $user->save();
});
```

如果沒有傳遞閉包給 `tap` 函數，您可以在給定的 `$value` 上呼叫任何方法。無論該方法在定義中實際回傳什麼，您呼叫的方法之回傳值永遠會是 `$value`。例如，Eloquent 的 `update` 方法通常回傳一個整數。然而，我們可以透過 `tap` 函數將 `update` 方法呼叫串接起來，強制該方法回傳模型本身：

```php
$user = tap($user)->update([
    'name' => $name,
    'email' => $email,
]);
```

若要在類別中新增 `tap` 方法，您可以將 `Illuminate\Support\Traits\Tappable` trait 新增至該類別。此 trait 的 `tap` 方法僅接受一個閉包作為引數。物件實例本身將被傳遞給該閉包，然後由 `tap` 方法回傳：

```php
return $user->tap(function (User $user) {
    // ...
});
```


<a name="method-throw-if"></a>
#### `throw_if()` {.collection-method}

`throw_if` 函數會在給定的布林運算式為 `true` 時，拋出給定的例外：

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

`throw_unless` 函數會在給定的布林運算式為 `false` 時，拋出給定的例外：

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

`today` 函數會為目前日期建立一個新的 `Illuminate\Support\Carbon` 實例：

```php
$today = today();
```


<a name="method-trait-uses-recursive"></a>
#### `trait_uses_recursive()` {.collection-method}

`trait_uses_recursive` 函數會回傳一個 trait 所使用的所有 traits：

```php
$traits = trait_uses_recursive(\Illuminate\Notifications\Notifiable::class);
```


<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 函數會在給定值非 [blank](#method-blank) 時，對該值執行閉包，然後回傳閉包的回傳值：

```php
$callback = function (int $value) {
    return $value * 2;
};

$result = transform(5, $callback);

// 10
```

您可以傳遞一個預設值或閉包作為函數的第三個引數。如果給定值為 blank，則會回傳此值：

```php
$result = transform(null, $callback, 'The value is blank');

// The value is blank
```

<a name="method-validator"></a>
#### `validator()` {.collection-method}

`validator` 函數會使用給定的參數建立一個新的 [驗證器](/docs/{{version}}/validation) 實例。您可以將其作為 `Validator` Facade 的替代方案：

```php
$validator = validator($data, $rules, $messages);
```


<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 函數會回傳傳入的值。然而，如果您將閉包 (closure) 傳遞給該函數，則會執行該閉包並回傳其回傳值：

```php
$result = value(true);

// true

$result = value(function () {
    return false;
});

// false
```

您可以將額外的引數傳遞給 `value` 函數。如果第一個引數是閉包，則額外的參數將作為引數傳遞給該閉包，否則將被忽略：

```php
$result = value(function (string $name) {
    return $name;
}, 'Taylor');

// 'Taylor'
```


<a name="method-view"></a>
#### `view()` {.collection-method}

`view` 函數會取得一個 [視圖](/docs/{{version}}/views) 實例：

```php
return view('auth.login');
```


<a name="method-with"></a>
#### `with()` {.collection-method}

`with` 函數會回傳傳入的值。如果將閉包作為該函數的第二個引數傳遞，則會執行該閉包並回傳其回傳值：

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

如果給定的條件評估為 `true`，`when` 函數會回傳傳入的值。否則，將回傳 `null`。如果將閉包作為該函數的第二個引數傳遞，則會執行該閉包並回傳其回傳值：

```php
$value = when(true, 'Hello World');

$value = when(true, fn () => 'Hello World');
```

`when` 函數主要用於條件式地渲染 HTML 屬性：

```blade
<div {!! when($condition, 'wire:poll="calculate"') !!}>
    ...
</div>
```

<a name="other-utilities"></a>
## 其他工具


<a name="benchmarking"></a>
### 效能基準測試

有時候您可能希望快速測試應用程式中某些部分的效能。在這種情況下，您可以使用 `Benchmark` 支援類別來衡量給定的回呼函數 (callbacks) 完成所需的毫秒數：

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

預設情況下，給定的回呼函數將執行一次（一次迭代），其執行時間將顯示在瀏覽器或主控台中。

若要多次呼叫回呼函數，您可以將回呼函數應被呼叫的迭代次數作為該方法的第二個引數。當多次執行回呼函數時，`Benchmark` 類別將回傳所有迭代執行該回呼函數的平均毫秒數：

```php
Benchmark::dd(fn () => User::count(), iterations: 10); // 0.5 ms
```

有時候，您可能想在衡量回呼函數執行效能的同時，仍能取得該回呼函數回傳的值。`value` 方法將回傳一個包含回呼函數回傳值以及執行該回呼函數所需毫秒數的元組 (tuple)：

```php
[$count, $duration] = Benchmark::value(fn () => User::count());
```


<a name="dates"></a>
### 日期與時間

Laravel 包含了 [Carbon](https://carbon.nesbot.com/guide/getting-started/introduction.html)，一個強大的日期與時間處理函式庫。若要建立新的 `Carbon` 實例，您可以呼叫 `now` 函數。此函數在您的 Laravel 應用程式中全域可用：

```php
$now = now();
```

或者，您可以使用 `Illuminate\Support\Carbon` 類別來建立新的 `Carbon` 實例：

```php
use Illuminate\Support\Carbon;

$now = Carbon::now();
```

Laravel 還為 `Carbon` 實例增強了 `plus` 與 `minus` 方法，讓您可以輕鬆地操作實例的日期與時間：

```php
return now()->plus(minutes: 5);
return now()->plus(hours: 8);
return now()->plus(weeks: 4);

return now()->minus(minutes: 5);
return now()->minus(hours: 8);
return now()->minus(weeks: 4);
```

如需深入了解 Carbon 及其功能，請參閱 [Carbon 官方文件](https://carbon.nesbot.com/guide/getting-started/introduction.html)。


<a name="interval-functions"></a>
#### 間隔函數

Laravel 還提供了 `milliseconds`、`seconds`、`minutes`、`hours`、`days`、`weeks`、`months` 以及 `years` 函數，這些函數會回傳 `CarbonInterval` 實例，該實例擴展了 PHP 的 [DateInterval](https://www.php.net/manual/en/class.dateinterval.php) 類別。在任何 Laravel 接受 `DateInterval` 實例的地方都可以使用這些函數：

```php
use Illuminate\Support\Facades\Cache;

use function Illuminate\Support\{minutes};

Cache::put('metrics', $metrics, minutes(10));
```


<a name="deferred-functions"></a>
### 延遲函數

雖然 Laravel 的 [佇列作業](/docs/{{version}}/queues) 允許您將任務放入佇列以進行背景處理，但有時候您可能有一些簡單的任務想要延後執行，而不需要配置或維護一個長期執行的佇列工作者 (queue worker)。

延遲函數允許您將閉包 (closure) 的執行延後到 HTTP 回應發送給使用者之後，讓您的應用程式感覺快速且具備回應能力。若要延遲執行閉包，只需將該閉包傳遞給 `Illuminate\Support\defer` 函數即可：

```php
use App\Services\Metrics;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use function Illuminate\Support\defer;

Route::post('/orders', function (Request $request) {
    // Create order...

    defer(fn () => Metrics::reportOrder($order));

    return $order;
});
```

預設情況下，只有在呼叫 `Illuminate\Support\defer` 的 HTTP 回應、Artisan 命令或佇列作業成功完成時，延遲函數才會被執行。這意味著如果請求導致 `4xx` 或 `5xx` HTTP 回應，延遲函數將不會被執行。如果您希望延遲函數總是執行，可以在延遲函數後鏈接 `always` 方法：

```php
defer(fn () => Metrics::reportOrder($order))->always();
```

> [!WARNING]
> 如果您安裝了 [Swoole PHP 擴充功能](https://www.php.net/manual/en/book.swoole.php)，Laravel 的 `defer` 函數可能會與 Swoole 自身的全域 `defer` 函數衝突，導致網頁伺服器錯誤。請確保透過明確指定命名空間來呼叫 Laravel 的 `defer` 助手函數：`use function Illuminate\Support\defer;`


<a name="cancelling-deferred-functions"></a>
#### 取消延遲函數

如果您需要在延遲函數執行前取消它，可以使用 `forget` 方法透過函數名稱來取消。若要為延遲函數命名，請向 `Illuminate\Support\defer` 函數提供第二個引數：

```php
defer(fn () => Metrics::report(), 'reportMetrics');

defer()->forget('reportMetrics');
```


<a name="disabling-deferred-functions-in-tests"></a>
#### 在測試中禁用延遲函數

在編寫測試時，禁用延遲函數可能會很有用。您可以在測試中呼叫 `withoutDefer`，以指示 Laravel 立即執行所有延遲函數：

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

如果您想禁用測試案例中所有測試的延遲函數，可以在基底 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutDefer` 方法：

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
### 抽獎(Lottery)

Laravel 的抽獎類別可用於根據一組給定的機率來執行回呼函數。當您只想針對一定百分比的傳入請求執行程式碼時，這會特別有用：

```php
use Illuminate\Support\Lottery;

Lottery::odds(1, 20)
    ->winner(fn () => $user->won())
    ->loser(fn () => $user->lost())
    ->choose();
```

您可以將 Laravel 的抽獎類別與其他 Laravel 功能結合使用。例如，您可能只想將小部分比例的慢查詢回報給您的異常處理器。此外，由於抽獎類別是可呼叫的 (callable)，我們可以將該類別的實例傳遞給任何接受可呼叫對象的方法：

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

Laravel 提供了一些簡單的方法，讓您能夠輕鬆地測試應用程式中的抽獎呼叫：

```php
// Lottery will always win...
Lottery::alwaysWin();

// Lottery will always lose...
Lottery::alwaysLose();

// Lottery will win then lose, and finally return to normal behavior...
Lottery::fix([true, false]);

// Lottery will return to normal behavior...
Lottery::determineResultsNormally();
```

<a name="pipeline"></a>
### Pipeline

Laravel 的 `Pipeline` Facade 提供了一種方便的方法，將給定的輸入「導管 (pipe)」通過一系列可調用類別、閉包或可調用對象，讓每個類別都有機會檢查或修改輸入，並調用管線中的下一個可調用對象：

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

如您所見，管線中的每個可調用類別或閉包都會接收到輸入和一個 `$next` 閉包。調用 `$next` 閉包將會觸發管線中的下一個可調用對象。您可能已經注意到，這與 [中介層](/docs/{{version}}/middleware) 非常相似。

當管線中最後一個可調用對象調用 `$next` 閉包時，將會執行傳遞給 `then` 方法的可調用對象。通常，這個可調用對象只需簡單地回傳給定的輸入。為了方便起見，如果您只想在處理完畢後回傳輸入，可以使用 `thenReturn` 方法。

當然，如前所述，您並不限於在管線中提供閉包。您也可以提供可調用類別。如果提供了類別名稱，該類別將透過 Laravel 的 [服務容器](/docs/{{version}}/container) 實例化，從而允許將依賴注入到可調用類別中：

```php
$user = Pipeline::send($user)
    ->through([
        GenerateProfilePhoto::class,
        ActivateSubscription::class,
        SendWelcomeEmail::class,
    ])
    ->thenReturn();
```

您可以在管線上調用 `withinTransaction` 方法，以自動將管線的所有步驟封裝在單個資料庫交易中：

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
### Sleep

Laravel 的 `Sleep` 類別是 PHP 原生 `sleep` 與 `usleep` 函數的輕量級封裝，它提供了更好的可測試性，同時為時間操作提供了開發者友善的 API：

```php
use Illuminate\Support\Sleep;

$waiting = true;

while ($waiting) {
    Sleep::for(1)->second();

    $waiting = /* ... */;
}
```

`Sleep` 類別提供了各種方法，讓您可以處理不同的時間單位：

```php
// Return a value after sleeping...
$result = Sleep::for(1)->second()->then(fn () => 1 + 1);

// Sleep while a given value is true...
Sleep::for(1)->second()->while(fn () => shouldKeepSleeping());

// Pause execution for 90 seconds...
Sleep::for(1.5)->minutes();

// Pause execution for 2 seconds...
Sleep::for(2)->seconds();

// Pause execution for 500 milliseconds...
Sleep::for(500)->milliseconds();

// Pause execution for 5,000 microseconds...
Sleep::for(5000)->microseconds();

// Pause execution until a given time...
Sleep::until(now()->plus(minutes: 1));

// Alias of PHP's native "sleep" function...
Sleep::sleep(2);

// Alias of PHP's native "usleep" function...
Sleep::usleep(5000);
```

若要輕鬆組合時間單位，您可以使用 `and` 方法：

```php
Sleep::for(1)->second()->and(10)->milliseconds();
```


<a name="testing-sleep"></a>
#### Testing Sleep

在測試使用 `Sleep` 類別或 PHP 原生休眠函數的程式碼時，您的測試將會暫停執行。正如您所預料的，這會使您的測試套件速度顯著降低。例如，想像您正在測試以下程式碼：

```php
$waiting = /* ... */;

$seconds = 1;

while ($waiting) {
    Sleep::for($seconds++)->seconds();

    $waiting = /* ... */;
}
```

通常，測試這段程式碼至少需要一秒鐘。幸運的是，`Sleep` 類別允許我們「模擬 (fake)」休眠，使我們的測試套件保持高速：

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

在模擬 `Sleep` 類別時，實際的執行暫停會被跳過，從而大幅加快測試速度。

一旦 `Sleep` 類別被模擬，您就可以針對預期應該發生的「休眠」進行斷言。為了說明這一點，讓我們假設我們正在測試一段暫停執行三次且每次暫停增加一秒的程式碼。使用 `assertSequence` 方法，我們可以在保持測試高速的同時，斷言我們的程式碼是否休眠了正確的時間：

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

當然，`Sleep` 類別還提供了許多其他可用於測試的斷言：

```php
use Carbon\CarbonInterval as Duration;
use Illuminate\Support\Sleep;

// Assert that sleep was called 3 times...
Sleep::assertSleptTimes(3);

// Assert against the duration of sleep...
Sleep::assertSlept(function (Duration $duration): bool {
    return /* ... */;
}, times: 1);

// Assert that the Sleep class was never invoked...
Sleep::assertNeverSlept();

// Assert that, even if Sleep was called, no execution paused occurred...
Sleep::assertInsomniac();
```

有時候，每當發生模擬休眠時執行某個操作會很有用。為了實現這一點，您可以向 `whenFakingSleep` 方法提供一個回呼函數。在以下範例中，我們使用 Laravel 的 [時間操作輔助工具](/docs/{{version}}/mocking#interacting-with-time) 讓時間在每次休眠時立即推進其持續時間：

```php
use Carbon\CarbonInterval as Duration;

$this->freezeTime();

Sleep::fake();

Sleep::whenFakingSleep(function (Duration $duration) {
    // Progress time when faking sleep...
    $this->travel($duration->totalMilliseconds)->milliseconds();
});
```

由於推進時間是一個常見需求，`fake` 方法接受一個 `syncWithCarbon` 參數，以便在測試中休眠時保持 Carbon 同步：

```php
Sleep::fake(syncWithCarbon: true);

$start = now();

Sleep::for(1)->second();

$start->diffForHumans(); // 1 second ago
```

Laravel 在內部暫停執行時都會使用 `Sleep` 類別。例如，[retry](#method-retry) 助手函數在休眠時會使用 `Sleep` 類別，從而提高了使用該助手函數時的可測試性。


<a name="timebox"></a>
### Timebox

Laravel 的 `Timebox` 類別確保給定的回呼函數 (callback) 總是花費固定的時間來執行，即使其實際執行較快地完成。這對於加密操作和使用者認證檢查特別有用，因為攻擊者可能會利用執行時間的差異來推斷敏感資訊。

如果執行時間超過了固定時長，`Timebox` 將沒有影響。開發者應選擇足夠長的固定時長，以應對最糟的情況。

`call` 方法接受一個閉包和以微秒為單位的時間限制，然後執行該閉包並等待直到達到時間限制：

```php
use Illuminate\Support\Timebox;

(new Timebox)->call(function ($timebox) {
    // ...
}, microseconds: 10000);
```

如果在閉包中拋出異常，此類別將遵循定義的延遲，並在延遲後重新拋出該異常。

<a name="uri"></a>
### URI

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面來建立與操作 URI。這個類別封裝了底層 League URI 套件所提供的功能，並與 Laravel 的路由系統無縫整合。

您可以使用靜態方法輕鬆地建立 `Uri` 實例：

```php
use App\Http\Controllers\UserController;
use App\Http\Controllers\InvokableController;
use Illuminate\Support\Uri;

// Generate a URI instance from the given string...
$uri = Uri::of('https://example.com/path');

// Generate URI instances to paths, named routes, or controller actions...
$uri = Uri::to('/dashboard');
$uri = Uri::route('users.show', ['user' => 1]);
$uri = Uri::signedRoute('users.show', ['user' => 1]);
$uri = Uri::temporarySignedRoute('user.index', now()->plus(minutes: 5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// Generate a URI instance from the current request URL...
$uri = $request->uri();
```

一旦您擁有了 URI 實例，就可以流暢地對其進行修改：

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
#### 檢視 URI

`Uri` 類別也允許您輕鬆檢視底層 URI 的各個組成部分：

```php
$scheme = $uri->scheme();
$authority = $uri->authority();
$host = $uri->host();
$port = $uri->port();
$path = $uri->path();
$segments = $uri->pathSegments();
$query = $uri->query();
$fragment = $uri->fragment();
```


<a name="manipulating-query-strings"></a>
#### 操作查詢字串

`Uri` 類別提供了多個可用於操作 URI 查詢字串的方法。`withQuery` 方法可用於將額外的查詢字串參數合併到現有的查詢字串中：

```php
$uri = $uri->withQuery(['sort' => 'name']);
```

如果給定的鍵不存在於查詢字串中，可以使用 `withQueryIfMissing` 方法將額外的查詢字串參數合併到現有的查詢字串中：

```php
$uri = $uri->withQueryIfMissing(['page' => 1]);
```

`replaceQuery` 方法可用於將現有的查詢字串完全替換為新的查詢字串：

```php
$uri = $uri->replaceQuery(['page' => 1]);
```

`pushOntoQuery` 方法可用於將額外參數推入到具有陣列值的查詢字串參數中：

```php
$uri = $uri->pushOntoQuery('filter', ['active', 'pending']);
```

`withoutQuery` 方法可用於從查詢字串中移除參數：

```php
$uri = $uri->withoutQuery(['page']);
```


<a name="generating-responses-from-uris"></a>
#### 從 URI 產生回應

`redirect` 方法可用於為給定的 URI 產生一個 `RedirectResponse` 實例：

```php
$uri = Uri::of('https://example.com');

return $uri->redirect();
```

或者，您也可以直接從路由或控制器動作中回傳 `Uri` 實例，這將自動產生一個指向該回傳 URI 的重新導向回應：

```php
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Uri;

Route::get('/redirect', function () {
    return Uri::to('/index')
        ->withQuery(['sort' => 'name']);
});
```