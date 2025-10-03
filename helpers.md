# 輔助函式

- [簡介](#introduction)
- [可用的方法](#available-methods)
- [其他工具](#other-utilities)
    - [效能測試](#benchmarking)
    - [日期](#dates)
    - [延遲函式](#deferred-functions)
    - [Lottery](#lottery)
    - [Pipeline](#pipeline)
    - [Sleep](#sleep)
    - [Timebox](#timebox)
    - [URI](#uri)

<a name="introduction"></a>
## 簡介

Laravel 包含了許多全域「輔助 (Helper)」PHP 函式。其中許多函式是由框架本身所使用；不過，若覺得方便，你也可以在自己的應用程式中自由使用這些函式。

<a name="available-methods"></a>
## 可用的方法

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

`Arr::accessible` 方法會判斷給定的值是否可當作陣列存取：

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

`Arr::add` 方法會將給定的鍵／值對加到陣列中，但前提是該鍵尚未存在於陣列中或是其值為 `null`：

```php
use Illuminate\Support\Arr;

$array = Arr::add(['name' => 'Desk'], 'price', 100);

// ['name' => 'Desk', 'price' => 100]

$array = Arr::add(['name' => 'Desk', 'price' => null], 'price', 100);

// ['name' => 'Desk', 'price' => 100]
```


<a name="method-array-array"></a>
#### `Arr::array()` {.collection-method}

`Arr::array` 方法會使用「點」號表示法來從深度巢狀的陣列中取出一個值 (與 [Arr::get()](#method-array-get) 相同)，不過若請求的值不是 `array` 則會擲回 `InvalidArgumentException`：

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

`Arr::boolean` 方法會使用「點」號表示法來從深度巢狀的陣列中取出一個值 (與 [Arr::get()](#method-array-get) 相同)，不過若請求的值不是 `boolean` 則會擲回 `InvalidArgumentException`：

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

`Arr::collapse` 方法會將一個由陣列或 Collection 組成的陣列摺疊為單一陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::collapse([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```


<a name="method-array-crossjoin"></a>
#### `Arr::crossJoin()` {.collection-method}

`Arr::crossJoin` 方法會將給定的陣列交叉相乘，並回傳一個包含所有可能排列的笛卡兒積：

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

`Arr::divide` 方法會回傳兩個陣列：一個包含給定陣列的鍵，另一個則包含其值：

```php
use Illuminate\Support\Arr;

[$keys, $values] = Arr::divide(['name' => 'Desk']);

// $keys: ['name']

// $values: ['Desk']
```


<a name="method-array-dot"></a>
#### `Arr::dot()` {.collection-method}

`Arr::dot` 方法會將一個多維陣列攤平成一個單層陣列，並使用「點」號表示法來表示深度：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$flattened = Arr::dot($array);

// ['products.desk.price' => 100]
```


<a name="method-array-every"></a>
#### `Arr::every()` {.collection-method}

`Arr::every` 方法可確保陣列中所有的值都通過給定的真值測試：

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

`Arr::except` 方法會從陣列中移除給定的鍵／值對：

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

`Arr::first` 方法會回傳陣列中第一個通過給定真值測試的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300];

$first = Arr::first($array, function (int $value, int $key) {
    return $value >= 150;
});

// 200
```

也可傳入第三個參數作為預設值。若沒有值通過真值測試，則會回傳此預設值：

```php
use Illuminate\Support\Arr;

$first = Arr::first($array, $callback, $default);
```


<a name="method-array-flatten"></a>
#### `Arr::flatten()` {.collection-method}

`Arr::flatten` 方法會將一個多維陣列攤平成一個單層陣列：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

$flattened = Arr::flatten($array);

// ['Joe', 'PHP', 'Ruby']
```


<a name="method-array-float"></a>
#### `Arr::float()` {.collection-method}

`Arr::float` 方法會使用「點」號表示法來從深度巢狀的陣列中取出一個值 (與 [Arr::get()](#method-array-get) 相同)，不過若請求的值不是 `float` 則會擲回 `InvalidArgumentException`：

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

`Arr::forget` 方法會使用「點」號表示法來從深度巢狀的陣列中移除給定的鍵／值對：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::forget($array, 'products.desk');

// ['products' => []]
```


<a name="method-array-from"></a>
#### `Arr::from()` {.collection-method}

`Arr::from` 方法可將多種輸入型別轉為純 PHP 陣列。此方法支援多種輸入型別，包含陣列、物件、以及多個常見的 Laravel Interface，如 `Arrayable`、`Enumerable`、`Jsonable`、`JsonSerializable`。此外，此方法還可處理 `Traversable` 與 `WeakMap` 的實體：

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

`Arr::get` 方法會使用「點」號表示法來從深度巢狀的陣列中取出一個值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$price = Arr::get($array, 'products.desk.price');

// 100
```

`Arr::get` 方法也接受一個預設值，若陣列中找不到指定的鍵，則會回傳該預設值：

```php
use Illuminate\Support\Arr;

$discount = Arr::get($array, 'products.desk.discount', 0);

// 0
```


<a name="method-array-has"></a>
#### `Arr::has()` {.collection-method}

`Arr::has` 方法會使用「點」號表示法來檢查給定的一或多個項目是否存在於陣列中：

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

`Arr::hasAll` 方法會使用「點」號表示法來判斷指定的索引鍵是否都存在於給定的陣列中：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Taylor', 'language' => 'PHP'];

Arr::hasAll($array, ['name']); // true
Arr::hasAll($array, ['name', 'language']); // true
Arr::hasAll($array, ['name', 'IDE']); // false
```

<a name="method-array-hasany"></a>
#### `Arr::hasAny()` {.collection-method}

`Arr::hasAny` 方法會使用「點」號表示法來檢查給定集合中的任何項目是否存在於陣列中：

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

`Arr::integer` 方法會使用「點」號表示法來從深度巢狀的陣列中擷取值 (與 [Arr::get()](#method-array-get) 相同)，但若請求的值不是 `int`，則會擲回 `InvalidArgumentException`：

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

若給定陣列為關聯陣列，`Arr::isAssoc` 方法會回傳 `true`。若陣列沒有從 0 開始的連續數字索引鍵，則該陣列會被視為「關聯 (Associative)」陣列：

```php
use Illuminate\Support\Arr;

$isAssoc = Arr::isAssoc(['product' => ['name' => 'Desk', 'price' => 100]]);

// true

$isAssoc = Arr::isAssoc([1, 2, 3]);

// false
```

<a name="method-array-islist"></a>
#### `Arr::isList()` {.collection-method}

若給定陣列的索引鍵是從 0 開始的連續整數，`Arr::isList` 方法會回傳 `true`：

```php
use Illuminate\Support\Arr;

$isList = Arr::isList(['foo', 'bar', 'baz']);

// true

$isList = Arr::isList(['product' => ['name' => 'Desk', 'price' => 100]]);

// false
```

<a name="method-array-join"></a>
#### `Arr::join()` {.collection-method}

`Arr::join` 方法會用字串來連接陣列元素。使用此方法的第三個引數，也可以為陣列的最後一個元素指定連接用的字串：

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

`Arr::keyBy` 方法會使用給定的索引鍵來當作陣列的索引鍵。若有多個項目有相同的索引鍵，則只有最後一個項目會出現在新的陣列中：

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

`Arr::last` 方法會回傳陣列中最後一個通過給定 Truth Test 的元素：

```php
use Illuminate\Support\Arr;

$array = [100, 200, 300, 110];

$last = Arr::last($array, function (int $value, int $key) {
    return $value >= 150;
});

// 300
```

也可以傳遞一個預設值作為該方法的第三個引數。若沒有值通過 Truth Test，則會回傳該值：

```php
use Illuminate\Support\Arr;

$last = Arr::last($array, $callback, $default);
```

<a name="method-array-map"></a>
#### `Arr::map()` {.collection-method}

`Arr::map` 方法會迭代陣列，並將每個值與索引鍵都傳遞給給定的回呼函式。陣列值會被該回呼函式回傳的值取代：

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

`Arr::mapSpread` 方法會迭代陣列，並將每個巢狀項目值都傳遞給給定的閉包。該閉包可以任意修改項目並回傳，進而形成一個新的、包含已修改項目的陣列：

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

`Arr::mapWithKeys` 方法會迭代陣列，並將每個值都傳遞給給定的回呼函式。該回呼函式應回傳一個包含單一索引鍵／值組合的關聯陣列：

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

`Arr::only` 方法只會從給定的陣列中回傳指定的索引鍵／值組合：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100, 'orders' => 10];

$slice = Arr::only($array, ['name', 'price']);

// ['name' => 'Desk', 'price' => 100]
```

<a name="method-array-partition"></a>
#### `Arr::partition()` {.collection-method}

`Arr::partition` 方法可與 PHP 的陣列解構 (Destructuring) 合併使用，來將通過給定 Truth Test 的元素與未通過的元素分開：

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

`Arr::pluck` 方法會從陣列中擷取給定索引鍵的所有值：

```php
use Illuminate\Support\Arr;

$array = [
    ['developer' => ['id' => 1, 'name' => 'Taylor']],
    ['developer' => ['id' => 2, 'name' => 'Abigail']],
];

$names = Arr::pluck($array, 'developer.name');

// ['Taylor', 'Abigail']
```

也可以指定要如何為產生的列表加上索引鍵：

```php
use Illuminate\Support\Arr;

$names = Arr::pluck($array, 'developer.name', 'developer.id');

// [1 => 'Taylor', 2 => 'Abigail']
```

<a name="method-array-prepend"></a>
#### `Arr::prepend()` {.collection-method}

`Arr::prepend` 方法會將一個項目推送到陣列的開頭：

```php
use Illuminate\Support\Arr;

$array = ['one', 'two', 'three', 'four'];

$array = Arr::prepend($array, 'zero');

// ['zero', 'one', 'two', 'three', 'four']
```

若有需要，可以指定要為該值使用的索引鍵：

```php
use Illuminate\Support\Arr;

$array = ['price' => 100];

$array = Arr::prepend($array, 'Desk', 'name');

// ['name' => 'Desk', 'price' => 100]
```

<a name="method-array-prependkeyswith"></a>
#### `Arr::prependKeysWith()` {.collection-method}

`Arr::prependKeysWith` 會在關聯陣列的所有索引鍵名稱前加上給定的前綴：

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

`Arr::pull` 方法會從陣列中回傳並移除一個鍵／值對：

```php
use Illuminate\Support\Arr;

$array = ['name' => 'Desk', 'price' => 100];

$name = Arr::pull($array, 'name');

// $name: Desk

// $array: ['price' => 100]
```

也可以傳入第三個引數到該方法來當作預設值。若指定的鍵不存在，則會回傳該值：

```php
use Illuminate\Support\Arr;

$value = Arr::pull($array, $key, $default);
```


<a name="method-array-push"></a>
#### `Arr::push()` {.collection-method}

`Arr::push` 方法會使用「點」號表示法來將項目推入(Push)陣列。若指定的鍵上不存在陣列，則會建立一個：

```php
use Illuminate\Support\Arr;

$array = [];

Arr::push($array, 'office.furniture', 'Desk');

// $array: ['office' => ['furniture' => ['Desk']]]
```


<a name="method-array-query"></a>
#### `Arr::query()` {.collection-method}

`Arr::query` 方法會將陣列轉換為查詢字串(Query String)：

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

`Arr::random` 方法會從陣列中回傳一個隨機值：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3, 4, 5];

$random = Arr::random($array);

// 4 - (retrieved randomly)
```

也可以提供可選的第二個引數來指定要回傳的項目數量。請注意，即使只想要一個項目，提供這個引數也會回傳一個陣列：

```php
use Illuminate\Support\Arr;

$items = Arr::random($array, 2);

// [2, 5] - (retrieved randomly)
```


<a name="method-array-reject"></a>
#### `Arr::reject()` {.collection-method}

`Arr::reject` 方法會使用給定的閉包來從陣列中移除項目：

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

`Arr::select` 方法會從陣列中選取一個值的陣列：

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

`Arr::set` 方法會使用「點」號表示法來設定深層巢狀陣列中的值：

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

Arr::set($array, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```


<a name="method-array-shuffle"></a>
#### `Arr::shuffle()` {.collection-method}

`Arr::shuffle` 方法會隨機地將陣列中的項目打亂：

```php
use Illuminate\Support\Arr;

$array = Arr::shuffle([1, 2, 3, 4, 5]);

// [3, 2, 5, 1, 4] - (generated randomly)
```


<a name="method-array-sole"></a>
#### `Arr::sole()` {.collection-method}

`Arr::sole` 方法會使用給定的閉包從陣列中取得單一值。若陣列中有多個值符合給定的真值測試，則會擲回 `Illuminate\Support\MultipleItemsFoundException` 例外。若沒有值符合真值測試，則會擲回 `Illuminate\Support\ItemNotFoundException` 例外：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$value = Arr::sole($array, fn (string $value) => $value === 'Desk');

// 'Desk'
```


<a name="method-array-some"></a>
#### `Arr::some()` {.collection-method}

`Arr::some` 方法可確保陣列中至少有一個值通過給定的真值測試：

```php
use Illuminate\Support\Arr;

$array = [1, 2, 3];

Arr::some($array, fn ($i) => $i > 2);

// true
```


<a name="method-array-sort"></a>
#### `Arr::sort()` {.collection-method}

`Arr::sort` 方法會依其值來排序陣列：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$sorted = Arr::sort($array);

// ['Chair', 'Desk', 'Table']
```

也可以用給定閉包的結果來排序陣列：

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

`Arr::sortDesc` 方法會依其值以降序排序陣列：

```php
use Illuminate\Support\Arr;

$array = ['Desk', 'Table', 'Chair'];

$sorted = Arr::sortDesc($array);

// ['Table', 'Desk', 'Chair']
```

也可以用給定閉包的結果來排序陣列：

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

`Arr::sortRecursive` 方法會遞迴地排序一個陣列。其中，有數字索引的子陣列會使用 `sort` 函式、而關聯式子陣列會使用 `ksort` 函式：

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

若想讓結果以降序排序，可使用 `Arr::sortRecursiveDesc` 方法。

```php
$sorted = Arr::sortRecursiveDesc($array);
```


<a name="method-array-string"></a>
#### `Arr::string()` {.collection-method}

`Arr::string` 方法會（與 [Arr::get()](#method-array-get) 相同）使用「點」號表示法來從深層巢狀陣列中取得值，不過若請求的值不是 `string`，則會擲回 `InvalidArgumentException`：

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

也可以傳入一個負整數來從陣列結尾取得指定數量的項目：

```php
$array = [0, 1, 2, 3, 4, 5];

$chunk = Arr::take($array, -2);

// [4, 5]
```


<a name="method-array-to-css-classes"></a>
#### `Arr::toCssClasses()` {.collection-method}

`Arr::toCssClasses` 方法會有條件地編譯一個 CSS Class 字串。該方法會接受一個 Class 陣列，其中陣列的鍵包含想新增的 Class、而值為一個布林表達式。若陣列元素有數字鍵，則其值會一定包含在渲染出的 Class 清單中：

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

`Arr::toCssStyles` 會根據條件來編譯 CSS 樣式字串。這個方法接受一個陣列，其中陣列的索引鍵是想加入的一或多個樣式，而值則為布林運算式。如果陣列元素是數字索引鍵，則該元素會一直被包含在最終呈現的樣式清單中：

```php
use Illuminate\Support\Arr;

$hasColor = true;

$array = ['background-color: blue', 'color: blue' => $hasColor];

$classes = Arr::toCssStyles($array);

/*
    'background-color: blue; color: blue;'
*/
```

這個方法為 Laravel 的功能提供了支援，允許[將 Class 與 Blade 元件的屬性包合併](/docs/{{version}}/blade#conditionally-merge-classes)，以及 `@class` [Blade 指示詞](/docs/{{version}}/blade#conditional-classes)。


<a name="method-array-undot"></a>
#### `Arr::undot()` {.collection-method}

`Arr::undot` 方法可將使用「點 (.)」標記法的單維陣列展開為多維陣列：

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

`Arr::where` 方法會使用給定的閉包來過濾陣列：

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

`Arr::wrap` 方法會將給定的值包裝在一個陣列中。若給定的值已是陣列，則會不經修改直接回傳：

```php
use Illuminate\Support\Arr;

$string = 'Laravel';

$array = Arr::wrap($string);

// ['Laravel']
```

若給定的值為 `null`，則會回傳一個空陣列：

```php
use Illuminate\Support\Arr;

$array = Arr::wrap(null);

// []
```


<a name="method-data-fill"></a>
#### `data_fill()` {.collection-method}

`data_fill` 函式可使用「點 (.)」標記法來為巢狀陣列或物件中缺少的值進行設定：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_fill($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 100]]]

data_fill($data, 'products.desk.discount', 10);

// ['products' => ['desk' => ['price' => 100, 'discount' => 10]]]
```

此函式也接受星號 (*) 作為萬用字元，並會依此填入目標：

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

`data_get` 函式可使用「點 (.)」標記法從巢狀陣列或物件中取得值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

$price = data_get($data, 'products.desk.price');

// 100
```

`data_get` 函式也接受一個預設值，若找不到指定的索引鍵時就會回傳該預設值：

```php
$discount = data_get($data, 'products.desk.discount', 0);

// 0
```

函式也接受星號 (*) 作為萬用字元，可針對陣列或物件中的任何索引鍵：

```php
$data = [
    'product-one' => ['name' => 'Desk 1', 'price' => 100],
    'product-two' => ['name' => 'Desk 2', 'price' => 150],
];

data_get($data, '*.name');

// ['Desk 1', 'Desk 2'];
```

可使用 `{first}` 與 `{last}` 預留位置來取得陣列中的第一筆或最後一筆項目：

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

`data_set` 函式可使用「點 (.)」標記法來為巢狀陣列或物件設定值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200);

// ['products' => ['desk' => ['price' => 200]]]
```

此函式也接受星號 (*) 作為萬用字元，並會依此在目標上設定值：

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

預設情況下，任何現有的值都會被覆寫。若只想在值不存在時才設定，可傳入 `false` 作為此函式的第四個引數：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_set($data, 'products.desk.price', 200, overwrite: false);

// ['products' => ['desk' => ['price' => 100]]]
```


<a name="method-data-forget"></a>
#### `data_forget()` {.collection-method}

`data_forget` 函式可使用「點 (.)」標記法來從巢狀陣列或物件中移除一個值：

```php
$data = ['products' => ['desk' => ['price' => 100]]];

data_forget($data, 'products.desk.price');

// ['products' => ['desk' => []]]
```

此函式也接受星號 (*) 作為萬用字元，並會依此在目標上移除值：

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

`head` 函式會回傳給定陣列中的第一個元素。若陣列為空，則會回傳 `false`：

```php
$array = [100, 200, 300];

$first = head($array);

// 100
```


<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 函式會回傳給定陣列中的最後一個元素。若陣列為空，則會回傳 `false`：

```php
$array = [100, 200, 300];

$last = last($array);

// 300
```

<a name="numbers"></a>
## 數字


<a name="method-number-abbreviate"></a>
#### `Number::abbreviate()` {.collection-method}

`Number::abbreviate` 方法會回傳一個帶有單位縮寫、人類可讀格式的數值：

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

`Number::clamp` 方法可確保給定數字維持在指定範圍內。若數字小於最小值，則回傳最小值。若數字大於最大值，則回傳最大值：

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

`Number::currency` 方法會回傳給定數值的貨幣字串表示法：

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

`Number::defaultCurrency` 方法會回傳 `Number` Class 正在使用的預設貨幣：

```php
use Illuminate\Support\Number;

$currency = Number::defaultCurrency();

// USD
```


<a name="method-default-locale"></a>
#### `Number::defaultLocale()` {.collection-method}

`Number::defaultLocale` 方法會回傳 `Number` Class 正在使用的預設語系：

```php
use Illuminate\Support\Number;

$locale = Number::defaultLocale();

// en
```


<a name="method-number-file-size"></a>
#### `Number::fileSize()` {.collection-method}

`Number::fileSize` 方法會回傳給定 Byte 值的檔案大小字串表示法：

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

`Number::forHumans` 方法會回傳提供數值的人類可讀格式：

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

`Number::ordinal` 方法會回傳數字的序數表示法：

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

`Number::pairs` 方法會根據指定的範圍與步長值，產生一個由數對 (子範圍) 組成的陣列。這個方法對於將一個較大的數字範圍分割成較小、易於管理的子範圍 (例如分頁或批次處理任務) 很有用。`pairs` 方法會回傳一個陣列的陣列，其中每個內部陣列代表一對 (子範圍) 的數字：

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

`Number::percentage` 方法會回傳給定數值的百分比字串表示法：

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

`Number::spell` 方法會將給定的數字轉換為單字字串：

```php
use Illuminate\Support\Number;

$number = Number::spell(102);

// one hundred and two

$number = Number::spell(88, locale: 'fr');

// quatre-vingt-huit
```

`after` 引數可讓你指定一個值，所有大於該值的數字都會被拼出來：

```php
$number = Number::spell(10, after: 10);

// 10

$number = Number::spell(11, after: 10);

// eleven
```

`until` 引數可讓你指定一個值，所有小於該值的數字都會被拼出來：

```php
$number = Number::spell(5, until: 10);

// five

$number = Number::spell(10, until: 10);

// 10
```


<a name="method-number-spell-ordinal"></a>
#### `Number::spellOrdinal()` {.collection-method}

`Number::spellOrdinal` 方法會回傳數字的序數表示法的單字字串：

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

`Number::trim` 方法會移除給定數字小數點後多餘的 0：

```php
use Illuminate\Support\Number;

$number = Number::trim(12.0);

// 12

$number = Number::trim(12.30);

// 12.3
```


<a name="method-number-use-locale"></a>
#### `Number::useLocale()` {.collection-method}

`Number::useLocale` 方法可全域設定預設的數字語系，這會影響後續對 `Number` Class 方法的呼叫如何格式化數字與貨幣：

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

`Number::withLocale` 方法會使用指定的語系執行給定的閉包，並在回呼執行完畢後還原原始的語系：

```php
use Illuminate\Support\Number;

$number = Number::withLocale('de', function () {
    return Number::format(1500);
});
```


<a name="method-number-use-currency"></a>
#### `Number::useCurrency()` {.collection-method}

`Number::useCurrency` 方法可全域設定預設的數字貨幣，這會影響後續對 `Number` Class 方法的呼叫如何格式化貨幣：

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

`Number::withCurrency` 方法會使用指定的貨幣來執行給定的閉包 (Closure)，並在該回呼執行完畢後還原回原本的貨幣設定：

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

`app_path` 函式會回傳 App 的 `app` 目錄的完整路徑。也可以用 `app_path` 函式來產生相對於 App 目錄的檔案完整路徑：

```php
$path = app_path();

$path = app_path('Http/Controllers/Controller.php');
```


<a name="method-base-path"></a>
#### `base_path()` {.collection-method}

`base_path` 函式會回傳 App 的根目錄的完整路徑。也可以用 `base_path` 函式來產生相對於專案根目錄的指定檔案完整路徑：

```php
$path = base_path();

$path = base_path('vendor/bin');
```


<a name="method-config-path"></a>
#### `config_path()` {.collection-method}

`config_path` 函式會回傳 App 的 `config` 目錄的完整路徑。也可以用 `config_path` 函式來產生 App 設定檔目錄中指定檔案的完整路徑：

```php
$path = config_path();

$path = config_path('app.php');
```


<a name="method-database-path"></a>
#### `database_path()` {.collection-method}

`database_path` 函式會回傳 App 的 `database` 目錄的完整路徑。也可以用 `database_path` 函式來產生資料庫目錄中指定檔案的完整路徑：

```php
$path = database_path();

$path = database_path('factories/UserFactory.php');
```


<a name="method-lang-path"></a>
#### `lang_path()` {.collection-method}

`lang_path` 函式會回傳 App 的 `lang` 目錄的完整路徑。也可以用 `lang_path` 函式來產生該目錄中指定檔案的完整路徑：

```php
$path = lang_path();

$path = lang_path('en/messages.php');
```

> [!NOTE]
> 預設情況下，Laravel 應用程式的 Skeleton (骨架) 中不包含 `lang` 目錄。若想自訂 Laravel 的語言檔，可透過 `lang:publish` Artisan 指令來發布這些語言檔。


<a name="method-public-path"></a>
#### `public_path()` {.collection-method}

`public_path` 函式會回傳 App 的 `public` 目錄的完整路徑。也可以用 `public_path` 函式來產生 Public 目錄中指定檔案的完整路徑：

```php
$path = public_path();

$path = public_path('css/app.css');
```


<a name="method-resource-path"></a>
#### `resource_path()` {.collection-method}

`resource_path` 函式會回傳 App 的 `resources` 目錄的完整路徑。也可以用 `resource_path` 函式來產生 Resources 目錄中指定檔案的完整路徑：

```php
$path = resource_path();

$path = resource_path('sass/app.scss');
```


<a name="method-storage-path"></a>
#### `storage_path()` {.collection-method}

`storage_path` 函式會回傳 App 的 `storage` 目錄的完整路徑。也可以用 `storage_path` 函式來產生 Storage 目錄中指定檔案的完整路徑：

```php
$path = storage_path();

$path = storage_path('app/file.txt');
```


<a name="urls"></a>
## URL


<a name="method-action"></a>
#### `action()` {.collection-method}

`action` 函式能為指定的 Controller 動作產生 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

若該方法接受路由參數，則可將其作為第二個引數傳給該方法：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```


<a name="method-asset"></a>
#### `asset()` {.collection-method}

`asset` 函式能使用目前請求的 Scheme (HTTP 或 HTTPS) 來為素材 (Asset) 產生 URL：

```php
$url = asset('img/photo.jpg');
```

可以在 `.env` 檔中設定 `ASSET_URL` 變數來設定素材 URL 的主機。若是將素材存放在如 Amazon S3 或其他 CDN 等外部服務上，這個功能就很有用：

```php
// ASSET_URL=http://example.com/assets

$url = asset('img/photo.jpg'); // http://example.com/assets/img/photo.jpg
```


<a name="method-route"></a>
#### `route()` {.collection-method}

`route` 函式能為指定的[命名路由](/docs/{{version}}/routing#named-routes)產生 URL：

```php
$url = route('route.name');
```

若路由接受參數，可以將其作為第二個引數傳給該函式：

```php
$url = route('route.name', ['id' => 1]);
```

預設情況下，`route` 函式會產生絕對 URL。若想產生相對 URL，可傳入 `false` 作為函式的第三個引數：

```php
$url = route('route.name', ['id' => 1], false);
```


<a name="method-secure-asset"></a>
#### `secure_asset()` {.collection-method}

`secure_asset` 函式能使用 HTTPS 來為素材產生 URL：

```php
$url = secure_asset('img/photo.jpg');
```


<a name="method-secure-url"></a>
#### `secure_url()` {.collection-method}

`secure_url` 函式能為指定的路徑產生完整的 HTTPS URL。額外的 URL 片段可傳入函式的第二個引數：

```php
$url = secure_url('user/profile');

$url = secure_url('user/profile', [1]);
```


<a name="method-to-action"></a>
#### `to_action()` {.collection-method}

`to_action` 函式能為指定的 Controller 動作產生[重新導向的 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
use App\Http\Controllers\UserController;

return to_action([UserController::class, 'show'], ['user' => 1]);
```

如有需要，可將要指派給重新導向的 HTTP 狀態碼與任何額外的回應標頭作為第三與第四個引數傳給 `to_action` 方法：

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

`to_route` 函式能為指定的[命名路由](/docs/{{version}}/routing#named-routes)產生[重新導向的 HTTP 回應](/docs/{{version}}/responses#redirects)：

```php
return to_route('users.show', ['user' => 1]);
```

如有需要，可將要指派給重新導向的 HTTP 狀態碼與任何額外的回應標頭作為第三與第四個引數傳給 `to_route` 方法：

```php
return to_route('users.show', ['user' => 1], 302, ['X-Framework' => 'Laravel']);
```


<a name="method-uri"></a>
#### `uri()` {.collection-method}

`uri` 函式會為給定的 URI 產生一個 [fluent (流暢) 的 URI 實體](#uri)：

```php
$uri = uri('https://example.com')
    ->withPath('/users')
    ->withQuery(['page' => 1]);
```

如果給 `uri` 函式傳入一個包含可呼叫 Controller 與方法配對的陣列，該函式會為該 Controller 方法的路由路徑建立一個 `Uri` 實體：

```php
use App\Http\Controllers\UserController;

$uri = uri([UserController::class, 'show'], ['user' => $user]);
```

如果該 Controller 是可呼叫的 (invokable)，可以只提供 Controller 的類別名稱：

```php
use App\Http\Controllers\UserIndexController;

$uri = uri(UserIndexController::class);
```

如果給 `uri` 函式傳入的值與[命名路由](/docs/{{version}}/routing#named-routes)的名稱相符，則會為該路由的路徑產生一個 `Uri` 實體：

```php
$uri = uri('users.show', ['user' => $user]);
```


<a name="method-url"></a>
#### `url()` {.collection-method}

`url` 函式能為指定的路徑產生完整的 URL：

```php
$url = url('user/profile');

$url = url('user/profile', [1]);
```

若未提供路徑，則會回傳 `Illuminate\Routing\UrlGenerator` 實體：

```php
$current = url()->current();

$full = url()->full();

$previous = url()->previous();
```

想瞭解更多有關 `url` 函式的資訊，請參考 [URL 產生文件的說明](/docs/{{version}}/urls#generating-urls)。

<a name="miscellaneous"></a>
## 雜項


<a name="method-abort"></a>
#### `abort()` {.collection-method}

`abort` 函式會擲回一個[HTTP 例外狀況](/docs/{{version}}/errors#http-exceptions)，並由[例外處理器](/docs/{{version}}/errors#handling-exceptions)來 Render：

```php
abort(403);
```

我們也可以提供例外狀況的訊息，以及要傳送至瀏覽器的自訂 HTTP 回應標頭：

```php
abort(403, 'Unauthorized.', $headers);
```


<a name="method-abort-if"></a>
#### `abort_if()` {.collection-method}

`abort_if` 函式會在給定的布林運算式判斷為 `true` 時擲回 HTTP 例外狀況：

```php
abort_if(! Auth::user()->isAdmin(), 403);
```

與 `abort` 方法類似，我們也可以提供例外狀況的回應文字作為第三個引數，並提供自訂回應標頭的陣列作為第四個引數。


<a name="method-abort-unless"></a>
#### `abort_unless()` {.collection-method}

`abort_unless` 函式會在給定的布林運算式判斷為 `false` 時擲回 HTTP 例外狀況：

```php
abort_unless(Auth::user()->isAdmin(), 403);
```

與 `abort` 方法類似，我們也可以提供例外狀況的回應文字作為第三個引數，並提供自訂回應標頭的陣列作為第四個引數。


<a name="method-app"></a>
#### `app()` {.collection-method}

`app` 函式會回傳[服務容器](/docs/{{version}}/container)的實體：

```php
$container = app();
```

我們可以傳入 Class 或介面名稱來從容器中解析：

```php
$api = app('HelpSpot\API');
```


<a name="method-auth"></a>
#### `auth()` {.collection-method}

`auth` 函式會回傳 [Authenticator](/docs/{{version}}/authentication) 的實體。可將其作為 `Auth` Facade 的替代方案：

```php
$user = auth()->user();
```

若有需要，也可以指定要存取哪一個 Guard 實體：

```php
$user = auth('admin')->user();
```


<a name="method-back"></a>
#### `back()` {.collection-method}

`back` 函式會產生一個[重新導向的 HTTP 回應](/docs/{{version}}/responses#redirects)來將使用者導向至前一個位置：

```php
return back($status = 302, $headers = [], $fallback = '/');

return back();
```


<a name="method-bcrypt"></a>
#### `bcrypt()` {.collection-method}

`bcrypt` 函式會使用 Bcrypt 來[雜湊](/docs/{{version}}/hashing)給定的值。可將此函式作為 `Hash` Facade 的替代方案：

```php
$password = bcrypt('my-secret-password');
```


<a name="method-blank"></a>
#### `blank()` {.collection-method}

`blank` 函式可用來判斷給定的值是否為「空」(Blank)：

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

`blank` 的反向操作請參考 [filled](#method-filled) 函式。


<a name="method-broadcast"></a>
#### `broadcast()` {.collection-method}

`broadcast` 函式會將給定的[事件](/docs/{{version}}/events)[廣播](/docs/{{version}}/broadcasting)給其監聽器：

```php
broadcast(new UserRegistered($user));

broadcast(new UserRegistered($user))->toOthers();
```


<a name="method-broadcast-if"></a>
#### `broadcast_if()` {.collection-method}

`broadcast_if` 函式會在給定的布林運算式判斷為 `true` 時將給定的[事件](/docs/{{version}}/events)[廣播](/docs/{{version}}/broadcasting)給其監聽器：

```php
broadcast_if($user->isActive(), new UserRegistered($user));

broadcast_if($user->isActive(), new UserRegistered($user))->toOthers();
```


<a name="method-broadcast-unless"></a>
#### `broadcast_unless()` {.collection-method}

`broadcast_unless` 函式會在給定的布林運算式判斷為 `false` 時將給定的[事件](/docs/{{version}}/events)[廣播](/docs/{{version}}/broadcasting)給其監聽器：

```php
broadcast_unless($user->isBanned(), new UserRegistered($user));

broadcast_unless($user->isBanned(), new UserRegistered($user))->toOthers();
```


<a name="method-cache"></a>
#### `cache()` {.collection-method}

可使用 `cache` 函式來從[快取](/docs/{{version}}/cache)中取得值。若給定的索引鍵不存在於快取中，則會回傳選用的預設值：

```php
$value = cache('key');

$value = cache('key', 'default');
```

我們可以傳入一組索引鍵／值對的陣列來將項目新增至快取中。同時也應傳入秒數或一段時間來代表快取值應被視為有效的時長：

```php
cache(['key' => 'value'], 300);

cache(['key' => 'value'], now()->addSeconds(10));
```


<a name="method-class-uses-recursive"></a>
#### `class_uses_recursive()` {.collection-method}

`class_uses_recursive` 函式會回傳某個 Class 所使用的所有 Trait，其中也包含其所有上層 Class 所使用的 Trait：

```php
$traits = class_uses_recursive(App\Models\User::class);
```


<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 函式會從給定的值建立一個[集合 (Collection)](/docs/{{version}}/collections)的實體：

```php
$collection = collect(['Taylor', 'Abigail']);
```


<a name="method-config"></a>
#### `config()` {.collection-method}

`config` 函式可用來取得[設定](/docs/{{version}}/configuration)變數的值。設定值可使用「點 (.)」語法來存取，其中包含檔名與想存取的選項。我們也可以提供一個預設值，這個值會在設定選項不存在時回傳：

```php
$value = config('app.timezone');

$value = config('app.timezone', $default);
```

我們可以傳入一組索引鍵／值對的陣列來在執行時設定設定變數。不過請注意，此函式只會影響目前請求中的設定值，並不會更新實際的設定值：

```php
config(['app.debug' => true]);
```


<a name="method-context"></a>
#### `context()` {.collection-method}

`context` 函式可從目前的[情境 (Context)](/docs/{{version}}/context) 中取得值。我們也可以提供一個預設值，這個值會在情境索引鍵不存在時回傳：

```php
$value = context('trace_id');

$value = context('trace_id', $default);
```

我們可以傳入一組索引鍵／值對的陣列來設定情境值：

```php
use Illuminate\Support\Str;

context(['trace_id' => Str::uuid()->toString()]);
```


<a name="method-cookie"></a>
#### `cookie()` {.collection-method}

`cookie` 函式會建立一個新的 [Cookie](/docs/{{version}}/requests#cookies) 實體：

```php
$cookie = cookie('name', 'value', $minutes);
```


<a name="method-csrf-field"></a>
#### `csrf_field()` {.collection-method}

`csrf_field` 函式會產生一個包含 CSRF Token 值的 HTML `hidden` 輸入欄位。舉例來說，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
{{ csrf_field() }}
```


<a name="method-csrf-token"></a>
#### `csrf_token()` {.collection-method}

`csrf_token` 函式會擷取目前 CSRF Token 的值：

```php
$token = csrf_token();
```


<a name="method-decrypt"></a>
#### `decrypt()` {.collection-method}

`decrypt` 函式可[解密](/docs/{{version}}/encryption)給定的值。可將此函式作為 `Crypt` Facade 的替代方案：

```php
$password = decrypt($value);
```

`decrypt` 的反向操作請參考 [encrypt](#method-encrypt) 函式。


<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 函式會傾印 (Dump) 給定的變數並結束執行腳本：

```php
dd($value);

dd($value1, $value2, $value3, ...);
```

若不想在傾印變數後停止執行腳本，請改用 [dump](#method-dump) 函式。


<a name="method-dispatch"></a>
#### `dispatch()` {.collection-method}

`dispatch` 函式會將給定的 [Job](/docs/{{version}}/queues#creating-jobs) 推送至 Laravel 的 [Job 佇列](/docs/{{version}}/queues)上：

```php
dispatch(new App\Jobs\SendEmails);
```

<a name="method-dispatch-sync"></a>
#### `dispatch_sync()` {.collection-method}

`dispatch_sync` 函式會將給定的 Job 推送到 [sync](/docs/{{version}}/queues#synchronous-dispatching) 佇列，使其立即被處理：

```php
dispatch_sync(new App\Jobs\SendEmails);
```

<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 函式會傾印 (dump) 給定的變數：

```php
dump($value);

dump($value1, $value2, $value3, ...);
```

若想在傾印變數後停止執行腳本，請改用 [dd](#method-dd) 函式。

<a name="method-encrypt"></a>
#### `encrypt()` {.collection-method}

`encrypt` 函式會將給定的值 [加密](/docs/{{version}}/encryption)。可使用此函式來替代 `Crypt` facade：

```php
$secret = encrypt('my-secret-value');
```

有關 `encrypt` 的反向操作，請參考 [decrypt](#method-decrypt) 函式。

<a name="method-env"></a>
#### `env()` {.collection-method}

`env` 函式會擷取 [環境變數](/docs/{{version}}/configuration#environment-configuration) 的值，或回傳預設值：

```php
$env = env('APP_ENV');

$env = env('APP_ENV', 'production');
```

> [!WARNING]
> 在部署流程中若執行 `config:cache` 指令，請務必只在設定檔內呼叫 `env` 函式。設定檔被快取後，`.env` 檔就不會再被載入，且所有對 `env` 函式的呼叫都會回傳外部的環境變數，如伺服器層級或系統層級的環境變數，或回傳 `null`。

<a name="method-event"></a>
#### `event()` {.collection-method}

`event` 函式會將給定的 [Event](/docs/{{version}}/events) 分派 (Dispatch) 給其監聽器：

```php
event(new UserRegistered($user));
```

<a name="method-fake"></a>
#### `fake()` {.collection-method}

`fake` 函式會從容器中解析 [Faker](https://github.com/FakerPHP/Faker) 的 Singleton，在建立 Model Factory、資料庫填充 (Seeding)、測試、以及 View 原型時都很有用：

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

預設情況下，`fake` 函式會使用 `config/app.php` 設定檔中的 `app.faker_locale` 選項。一般來說，這個設定選項是透過 `APP_FAKER_LOCALE` 環境變數來設定的。也可將語系傳入 `fake` 函式來指定語系。每個語系都會解析出一個獨立的 Singleton：

```php
fake('nl_NL')->name()
```

<a name="method-filled"></a>
#### `filled()` {.collection-method}

`filled` 函式用來判斷給定的值是否「非空白」：

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

有關 `filled` 的反向操作，請參考 [blank](#method-blank) 函式。

<a name="method-info"></a>
#### `info()` {.collection-method}

`info` 函式會將資訊寫入到應用程式的 [Log](/docs/{{version}}/logging) 中：

```php
info('Some helpful information!');
```

也可以傳入一個包含脈絡資訊的陣列到該函式中：

```php
info('User login attempt failed.', ['id' => $user->id]);
```

<a name="method-literal"></a>
#### `literal()` {.collection-method}

`literal` 函式會建立一個新的 [stdClass](https://www.php.net/manual/en/class.stdclass.php) 執行個體，並將給定的具名引數作為屬性：

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

可使用 `logger` 函式來將 `debug` 層級的訊息寫入到 [Log](/docs/{{version}}/logging) 中：

```php
logger('Debug message');
```

也可以傳入一個包含脈絡資訊的陣列到該函式中：

```php
logger('User has logged in.', ['id' => $user->id]);
```

若未傳入任何值給函式，則會回傳一個 [Logger](/docs/{{version}}/logging) 執行個體：

```php
logger()->error('You are not allowed here.');
```

<a name="method-method-field"></a>
#### `method_field()` {.collection-method}

`method_field` 函式會產生一個 HTML `hidden` 輸入欄位，其中包含了表單的 HTTP 動詞的偽造值。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

```blade
<form method="POST">
    {{ method_field('DELETE') }}
</form>
```

<a name="method-now"></a>
#### `now()` {.collection-method}

`now` 函式會為目前時間建立一個新的 `Illuminate\Support\Carbon` 執行個體：

```php
$now = now();
```

<a name="method-old"></a>
#### `old()` {.collection-method}

`old` 函式會 [擷取](/docs/{{version}}/requests#retrieving-input)快閃 (Flash) 到 Session 中的 [舊輸入資料](/docs/{{version}}/requests#old-input)：

```php
$value = old('value');

$value = old('value', 'default');
```

由於傳給 `old` 函式的第二個引數「預設值」通常是 Eloquent Model 的屬性，因此 Laravel 允許直接將整個 Eloquent Model 作為 `old` 函式的第二個引數傳入。這樣做時，Laravel 會假設傳給 `old` 函式的第一個引數就是要作為「預設值」的 Eloquent 屬性名稱：

```blade
{{ old('name', $user->name) }}

// Is equivalent to...

{{ old('name', $user) }}
```

<a name="method-once"></a>
#### `once()` {.collection-method}

`once` 函式會執行給定的回呼函式，並在該請求的生命週期內將結果快取在記憶體中。之後所有使用相同回呼函式對 `once` 函式的呼叫，都會回傳先前快取的結果：

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

當 `once` 函式是從一個物件執行個體內執行時，快取的結果對該物件執行個體來說是唯一的：

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

`optional` 函式可接受任何引數，並讓我們能在該物件上存取屬性或呼叫方法。若給定的物件為 `null`，屬性與方法會回傳 `null` 而不是造成錯誤：

```php
return optional($user->address)->street;

{!! old('name', optional($user)->name) !!}
```

`optional` 函式也接受閉包作為其第二個引數。若傳入的第一個引數不為 null，就會叫用該閉包：

```php
return optional(User::find($id), function (User $user) {
    return $user->name;
});
```

<a name="method-policy"></a>
#### `policy()` {.collection-method}

`policy` 方法會為給定的類別擷取 [Policy](/docs/{{version}}/authorization#creating-policies) 執行個體：

```php
$policy = policy(App\Models\User::class);
```

<a name="method-redirect"></a>
#### `redirect()` {.collection-method}

`redirect` 函式會回傳一個 [重新導向 HTTP 回應](/docs/{{version}}/responses#redirects)，若呼叫時不帶任何引數，則會回傳 Redirector 執行個體：

```php
return redirect($to = null, $status = 302, $headers = [], $secure = null);

return redirect('/home');

return redirect()->route('route.name');
```

<a name="method-report"></a>
#### `report()` {.collection-method}

`report` 函式會使用 [例外處理常式](/docs/{{version}}/errors#handling-exceptions) 來回報例外：

```php
report($e);
```

`report` 函式也接受字串作為引數。當傳入字串給該函式時，該函式會建立一個例外，並將該字串作為其訊息：

```php
report('Something went wrong.');
```

<a name="method-report-if"></a>
#### `report_if()` {.collection-method}

若給定的布林運算式評估為 `true`，`report_if` 函式會使用[例外處理常式](/docs/{{version}}/errors#handling-exceptions)來回報例外：

```php
report_if($shouldReport, $e);

report_if($shouldReport, 'Something went wrong.');
```


<a name="method-report-unless"></a>
#### `report_unless()` {.collection-method}

若給定的布林運算式評估為 `false`，`report_unless` 函式會使用[例外處理常式](/docs/{{version}}/errors#handling-exceptions)來回報例外：

```php
report_unless($reportingDisabled, $e);

report_unless($reportingDisabled, 'Something went wrong.');
```


<a name="method-request"></a>
#### `request()` {.collection-method}

`request` 函式會回傳目前的[請求](/docs/{{version}}/requests)執行個體，或從目前的請求中取得某個輸入欄位的值：

```php
$request = request();

$value = request('key', $default);
```


<a name="method-rescue"></a>
#### `rescue()` {.collection-method}

`rescue` 函式會執行給定的閉包，並捕捉執行期間發生的任何例外。所有被捕捉到的例外都會被傳送到[例外處理常式](/docs/{{version}}/errors#handling-exceptions)；不過，請求會繼續處理：

```php
return rescue(function () {
    return $this->method();
});
```

也可以傳入第二個引數給 `rescue` 函式。這個引數會是「預設」值，在執行閉包時若發生例外則會回傳該值：

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

可以提供一個 `report` 引數給 `rescue` 函式，以決定是否應透過 `report` 函式回報該例外：

```php
return rescue(function () {
    return $this->method();
}, report: function (Throwable $throwable) {
    return $throwable instanceof InvalidArgumentException;
});
```


<a name="method-resolve"></a>
#### `resolve()` {.collection-method}

`resolve` 函式會使用[服務容器](/docs/{{version}}/container)將給定的 Class 或介面名稱解析為一個執行個體：

```php
$api = resolve('HelpSpot\API');
```


<a name="method-response"></a>
#### `response()` {.collection-method}

`response` 函式會建立一個[回應](/docs/{{version}}/responses)執行個體，或取得一個回應處理站 (Factory) 的執行個體：

```php
return response('Hello World', 200, $headers);

return response()->json(['foo' => 'bar'], 200, $headers);
```


<a name="method-retry"></a>
#### `retry()` {.collection-method}

`retry` 函式會嘗試執行給定的回呼，直到達到給定的最大嘗試次數上限為止。若該回呼沒有擲回例外，則會回傳其回傳值。若該回呼擲回例外，則會自動重試。若超過最大嘗試次數，則會擲回該例外：

```php
return retry(5, function () {
    // Attempt 5 times while resting 100ms between attempts...
}, 100);
```

若想手動計算每次嘗試之間要休眠的毫秒數，可以傳入一個閉包作為 `retry` 函式的第三個引數：

```php
use Exception;

return retry(5, function () {
    // ...
}, function (int $attempt, Exception $exception) {
    return $attempt * 100;
});
```

為方便起見，可以提供一個陣列作為 `retry` 函式的第一個引數。此陣列將用於決定後續每次嘗試之間要休眠多少毫秒：

```php
return retry([100, 200], function () {
    // Sleep for 100ms on first retry, 200ms on second retry...
});
```

若只要在特定條件下才重試，可以傳入一個閉包作為 `retry` 函式的第四個引數：

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

可以透過傳入一個鍵／值配對的陣列來設定值：

```php
session(['chairs' => 7, 'instruments' => 3]);
```

若未傳入任何值給函式，則會回傳 Session 儲存庫：

```php
$value = session()->get('key');

session()->put('key', $value);
```


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 函式接受兩個引數：一個任意的 `$value` 與一個閉包。`$value` 會被傳入閉包，然後由 `tap` 函式回傳。閉包的回傳值無關緊要：

```php
$user = tap(User::first(), function (User $user) {
    $user->name = 'Taylor';

    $user->save();
});
```

若未傳入閉包給 `tap` 函式，則可以在給定的 `$value` 上呼叫任何方法。呼叫的方法的回傳值將始終是 `$value`，無論該方法在其定義中實際回傳什麼。例如，Eloquent 的 `update` 方法通常回傳一個整數。但是，我們可以透過 `tap` 函式來鏈式呼叫 `update` 方法，強制該方法回傳模型本身：

```php
$user = tap($user)->update([
    'name' => $name,
    'email' => $email,
]);
```

要在一個類別中新增 `tap` 方法，可以將 `Illuminate\Support\Traits\Tappable` Trait 加入該類別中。該 Trait 的 `tap` 方法接受一個閉包作為其唯一引數。物件執行個體本身將被傳入該閉包，然後由 `tap` 方法回傳：

```php
return $user->tap(function (User $user) {
    // ...
});
```


<a name="method-throw-if"></a>
#### `throw_if()` {.collection-method}

若給定的布林運算式評估為 `true`，`throw_if` 函式會擲回給定的例外：

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

若給定的布林運算式評估為 `false`，`throw_unless` 函式會擲回給定的例外：

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

`today` 函式會為目前日期建立一個新的 `Illuminate\Support\Carbon` 執行個體：

```php
$today = today();
```


<a name="method-trait-uses-recursive"></a>
#### `trait_uses_recursive()` {.collection-method}

`trait_uses_recursive` 函式會回傳某個 Trait 所使用的所有 Trait：

```php
$traits = trait_uses_recursive(\Illuminate\Notifications\Notifiable::class);
```


<a name="method-transform"></a>
#### `transform()` {.collection-method}

如果給定的值不是[空值](#method-blank)，`transform` 函式會對該值執行一個閉包，然後回傳該閉包的回傳值：

```php
$callback = function (int $value) {
    return $value * 2;
};

$result = transform(5, $callback);

// 10
```

可以傳入一個預設值或閉包作為該函式的第三個引數。若給定的值是空值，則會回傳這個值：

```php
$result = transform(null, $callback, 'The value is blank');

// The value is blank
```


<a name="method-validator"></a>
#### `validator()` {.collection-method}

`validator` 函式會使用給定的引數建立一個新的[驗證器](/docs/{{version}}/validation)執行個體。可以將其作為 `Validator` Facade 的替代方案：

```php
$validator = validator($data, $rules, $messages);
```

<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 函式會回傳其收到的值。不過，若傳遞給該函式的是一個閉包，則會執行該閉包並回傳其回傳值：

```php
$result = value(true);

// true

$result = value(function () {
    return false;
});

// false
```

可以傳遞額外的引數給 `value` 函式。若第一個引數是閉包，則額外的參數會作為引數傳給該閉包，否則會被忽略：

```php
$result = value(function (string $name) {
    return $name;
}, 'Taylor');

// 'Taylor'
```

<a name="method-view"></a>
#### `view()` {.collection-method}

`view` 函式會取得一個 [view](/docs/{{version}}/views) 的實體：

```php
return view('auth.login');
```

<a name="method-with"></a>
#### `with()` {.collection-method}

`with` 函式會回傳其收到的值。若傳遞給該函式的第二個引數是閉包，則會執行該閉包並回傳其回傳值：

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

若給定的條件求值為 `true`，`when` 函式會回傳其收到的值。否則，會回傳 `null`。若傳遞給該函式的第二個引數是閉包，則會執行該閉包並回傳其回傳值：

```php
$value = when(true, 'Hello World');

$value = when(true, fn () => 'Hello World');
```

`when` 函式主要用於有條件地渲染 HTML 屬性：

```blade
<div {!! when($condition, 'wire:poll="calculate"') !!}>
    ...
</div>
```

<a name="other-utilities"></a>
## 其他工具


<a name="benchmarking"></a>
### 效能測試

有時候，你可能會想快速測試應用程式某些部分的效能。在這種情況下，你可以使用 `Benchmark` 輔助類別來測量給定的回呼函式完成所需的時間 (毫秒)：

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

預設情況下，給定的回呼函式會執行一次 (一次迭代)，其持續時間會顯示在瀏覽器或主控台中。

若要多次叫用回呼函式，你可以在方法的第二個引數中指定回呼函式應被叫用的迭代次數。當多次執行回呼函式時，`Benchmark` 類別會回傳所有迭代中執行回呼函式所需的平均時間 (毫秒)：

```php
Benchmark::dd(fn () => User::count(), iterations: 10); // 0.5 ms
```

有時候，你可能想在對回呼函式進行效能測試的同時，仍能取得該回呼函式回傳的值。`value` 方法會回傳一個元組 (tuple)，其中包含回呼函式回傳的值以及執行該回呼函式所需的時間 (毫秒)：

```php
[$count, $duration] = Benchmark::value(fn () => User::count());
```


<a name="dates"></a>
### 日期

Laravel 包含了 [Carbon](https://carbon.nesbot.com/docs/)，這是一個強大的日期和時間處理函式庫。要建立一個新的 `Carbon` 實體，你可以叫用 `now` 函式。這個函式在你的 Laravel 應用程式中是全域可用的：

```php
$now = now();
```

或者，你可以使用 `Illuminate\Support\Carbon` 類別來建立一個新的 `Carbon` 實體：

```php
use Illuminate\Support\Carbon;

$now = Carbon::now();
```

有關 Carbon 及其功能的詳細討論，請參閱 [Carbon 官方文件](https://carbon.nesbot.com/docs/)。


<a name="deferred-functions"></a>
### 延遲函式

雖然 Laravel 的 [佇列任務 (queued jobs)](/docs/{{version}}/queues) 讓你可以將任務排入佇列進行背景處理，但有時候你可能有一些簡單的任務想要延遲執行，而不想設定或維護一個長時間執行的佇列處理器。

延遲函式讓你可以將閉包的執行延遲到 HTTP 回應傳送給使用者之後，讓你的應用程式感覺快速且反應靈敏。要延遲閉包的執行，只需將閉包傳遞給 `Illuminate\Support\defer` 函式：

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

預設情況下，只有在叫用 `Illuminate\Support\defer` 的 HTTP 回應、Artisan 命令或佇列任務成功完成時，延遲函式才會被執行。這意味著如果請求導致 `4xx` 或 `5xx` HTTP 回應，延遲函式將不會被執行。如果你希望延遲函式總是執行，你可以在你的延遲函式上鏈結 `always` 方法：

```php
defer(fn () => Metrics::reportOrder($order))->always();
```

> [!WARNING]
> 如果你安裝了 [Swoole PHP 擴充套件](https://www.php.net/manual/en/book.swoole.php)，Laravel 的 `defer` 函式可能會與 Swoole 自己的全域 `defer` 函式衝突，導致網頁伺服器錯誤。請確保透過明確的命名空間來呼叫 Laravel 的 `defer` 輔助函式：`use function Illuminate\Support\defer;`


<a name="cancelling-deferred-functions"></a>
#### 取消延遲函式

如果你需要在延遲函式執行前取消它，你可以使用 `forget` 方法，透過其名稱來取消函式。要為延遲函式命名，請在 `Illuminate\Support\defer` 函式中提供第二個引數：

```php
defer(fn () => Metrics::report(), 'reportMetrics');

defer()->forget('reportMetrics');
```


<a name="disabling-deferred-functions-in-tests"></a>
#### 在測試中停用延遲函式

在撰寫測試時，停用延遲函式可能很有用。你可以在測試中呼叫 `withoutDefer`，指示 Laravel 立即叫用所有延遲函式：

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

如果你想為一個測試案例中的所有測試停用延遲函式，你可以在你的基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutDefer` 方法：

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
### Lottery

Laravel 的 Lottery 類別可用於根據一組給定的機率來執行回呼。當你只想對一小部分傳入的請求執行程式碼時，這特別有用：

```php
use Illuminate\Support\Lottery;

Lottery::odds(1, 20)
    ->winner(fn () => $user->won())
    ->loser(fn () => $user->lost())
    ->choose();
```

你可以將 Laravel 的 Lottery 類別與其他 Laravel 功能結合使用。例如，你可能只想向你的例外處理常式回報一小部分慢查詢。而且，由於 Lottery 類別是可呼叫的，我們可以將該類別的實體傳遞給任何接受可呼叫物件的方法：

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
#### 測試 Lottery

Laravel 提供了一些簡單的方法，讓你可以輕鬆地測試應用程式的 Lottery 叫用：

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

Laravel 的 `Pipeline` facade 提供了一個便利的方法，能將給定的輸入「管道化 (Pipe)」，使其通過一系列可呼叫 (Invokable) 的類別、閉包、或可呼叫函式，並讓每個類別都有機會來檢查或修改該輸入，並呼叫 Pipeline 中的下一個可呼叫函式：

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

如你所見，Pipeline 中的每個可呼叫類別或閉包都會被提供該輸入與一個 `$next` 閉包。呼叫 `$next` 閉包會呼叫 Pipeline 中的下一個可呼叫函式。你可能已經注意到了，這與[中介層](/docs/{{version}}/middleware)非常類似。

當 Pipeline 中的最後一個可呼叫函式呼叫了 `$next` 閉包時，會呼叫 `then` 方法中提供的可呼叫函式。一般來說，這個可呼叫函式只會回傳給定的輸入。為了方便，若你只想在處理完輸入後回傳該輸入，則可以使用 `thenReturn` 方法。

當然，如前所述，不只閉包能提供給 Pipeline。你也可以提供可呼叫的類別。若提供了類別名稱，該類別會透過 Laravel 的[服務容器](/docs/{{version}}/container)來實例化，以將依賴項注入至該可呼叫類別中：

```php
$user = Pipeline::send($user)
    ->through([
        GenerateProfilePhoto::class,
        ActivateSubscription::class,
        SendWelcomeEmail::class,
    ])
    ->thenReturn();
```

可以在 Pipeline 上呼叫 `withinTransaction` 方法，來自動將 Pipeline 中的所有步驟都包在單一資料庫交易中：

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

Laravel 的 `Sleep` 類別是 PHP 原生 `sleep` 與 `usleep` 函式的一個輕量級包裝，它提供了更好的可測試性，同時也為處理時間提供了一個對開發人員友善的 API：

```php
use Illuminate\Support\Sleep;

$waiting = true;

while ($waiting) {
    Sleep::for(1)->second();

    $waiting = /* ... */;
}
```

`Sleep` 類別提供了多種方法，可讓我們處理不同的時間單位：

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
Sleep::until(now()->addMinute());

// Alias of PHP's native "sleep" function...
Sleep::sleep(2);

// Alias of PHP's native "usleep" function...
Sleep::usleep(5000);
```

若要輕鬆地組合時間單位，可以使用 `and` 方法：

```php
Sleep::for(1)->second()->and(10)->milliseconds();
```

<a name="testing-sleep"></a>
#### 測試 Sleep

在測試使用 `Sleep` 類別或 PHP 原生 sleep 函式的程式碼時，你的測試會暫停執行。你可能也想到了，這會讓你的測試套件明顯變慢。舉例來說，假設我們在測試下列程式碼：

```php
$waiting = /* ... */;

$seconds = 1;

while ($waiting) {
    Sleep::for($seconds++)->seconds();

    $waiting = /* ... */;
}
```

一般來說，測試這段程式碼至少需要 1 秒鐘。幸運的是，`Sleep` 類別允許我們「偽造 (Fake)」sleep，讓我們的測試套件能保持快速：

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

在偽造 `Sleep` 類別時，會略過實際上的執行暫停，讓測試快上許多。

一旦 `Sleep` 類別被偽造後，就可以對預期應該發生的「sleep」進行判斷提示 (Assertion)。為了說明這點，讓我們假設我們在測試一段會暫停執行三次的程式碼，且每次暫停都會增加一秒。使用 `assertSequence` 方法，我們就可以判斷提示我們的程式碼「sleep」了正確的時間，同時維持測試的快速：

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

當然，`Sleep` 類別也提供了其他各種可在測試時使用的判斷提示：

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

有時候，在偽造的 sleep 發生時執行某個動作會很有用。為此，我們可以傳遞一個回呼函式給 `whenFakingSleep` 方法。在下列範例中，我們使用 Laravel 的[時間操作輔助函式](/docs/{{version}}/mocking#interacting-with-time)來立即將時間往前推進每個 sleep 的持續時間：

```php
use Carbon\CarbonInterval as Duration;

$this->freezeTime();

Sleep::fake();

Sleep::whenFakingSleep(function (Duration $duration) {
    // Progress time when faking sleep...
    $this->travel($duration->totalMilliseconds)->milliseconds();
});
```

由於推進時間是個常見的需求，因此 `fake` 方法可接受一個 `syncWithCarbon` 引數，可在測試中 sleep 時讓 Carbon 保持同步：

```php
Sleep::fake(syncWithCarbon: true);

$start = now();

Sleep::for(1)->second();

$start->diffForHumans(); // 1 second ago
```

Laravel 在內部會在需要暫停執行時使用 `Sleep` 類別。舉例來說，[retry](#method-retry) 輔助函式在 sleep 時就會使用 `Sleep` 類別，讓該輔助函式在使用時能有更好的可測試性。

<a name="timebox"></a>
### Timebox

Laravel 的 `Timebox` 類別能確保給定的回呼函式總是花費固定的時間來執行，就算其實際執行時間較短也一樣。這在密碼學操作與使用者身份驗證檢查等情境下特別有用，因為在這些情境中，攻擊者可能會利用執行時間的差異來推斷敏感資訊。

若執行時間超過了固定的持續時間，`Timebox` 則沒有效果。開發人員必須選擇一個夠長的時間作為固定持續時間，以應對最壞的情況。

call 方法接受一個閉包與一個以微秒為單位的時間限制，然後會執行該閉包並等到時間限制到達為止：

```php
use Illuminate\Support\Timebox;

(new Timebox)->call(function ($timebox) {
    // ...
}, microseconds: 10000);
```

若在閉包中擲回了例外，此類別會遵守定義的延遲，並在延遲後重新擲回該例外。

<a name="uri"></a>
### URI

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面，用來建立與操作 URI。該類別包裝了底層 League URI 套件提供的功能，並與 Laravel 的路由系統無縫整合。

你可以透過靜態方法輕鬆地建立一個 `Uri` 執行個體：

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
$uri = Uri::temporarySignedRoute('user.index', now()->addMinutes(5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// Generate a URI instance from the current request URL...
$uri = $request->uri();
```

一旦你有了 URI 執行個體，就可以流暢地對其進行修改：

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

`Uri` 類別也讓你能輕鬆地檢視底層 URI 的各種元件：

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

`Uri` 類別提供了數個可用來操作 URI 查詢字串的方法。`withQuery` 方法可用來將額外的查詢字串參數合併到現有的查詢字串中：

```php
$uri = $uri->withQuery(['sort' => 'name']);
```

`withQueryIfMissing` 方法可用來將額外的查詢字串參數合併到現有的查詢字串中，但前提是給定的鍵不存在於查詢字串中：

```php
$uri = $uri->withQueryIfMissing(['page' => 1]);
```

`replaceQuery` 方法可用來將現有的查詢字串完全取代為新的查詢字串：

```php
$uri = $uri->replaceQuery(['page' => 1]);
```

`pushOntoQuery` 方法可用來將額外的參數推入一個值為陣列的查詢字串參數中：

```php
$uri = $uri->pushOntoQuery('filter', ['active', 'pending']);
```

`withoutQuery` 方法可用來從查詢字串中移除參數：

```php
$uri = $uri->withoutQuery(['page']);
```

<a name="generating-responses-from-uris"></a>
#### 從 URI 產生回應

`redirect` 方法可用來產生一個導向至給定 URI 的 `RedirectResponse` 執行個體：

```php
$uri = Uri::of('https://example.com');

return $uri->redirect();
```

或者，你也可以直接從路由或控制器動作中回傳 `Uri` 執行個體，這將會自動產生一個重導向至所回傳 URI 的回應：

```php
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Uri;

Route::get('/redirect', function () {
    return Uri::to('/index')
        ->withQuery(['sort' => 'name']);
});
```