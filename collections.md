# 集合 (Collections)

- [簡介](#introduction)
    - [建立集合](#creating-collections)
    - [擴充集合](#extending-collections)
- [可用方法](#available-methods)
- [高階訊息](#higher-order-messages)
- [惰性集合 (Lazy Collections)](#lazy-collections)
    - [簡介](#lazy-collection-introduction)
    - [建立惰性集合](#creating-lazy-collections)
    - [Enumerable 契約](#the-enumerable-contract)
    - [惰性集合方法](#lazy-collection-methods)

<a name="introduction"></a>
## 簡介

`Illuminate\Support\Collection` 類別為處理資料陣列提供了一個流暢且方便的封裝。例如，請看以下程式碼。我們將使用 `collect` 輔助函式從陣列建立一個新的 Collection 實例，對每個元素執行 `strtoupper` 函式，然後移除所有空元素：

    $collection = collect(['taylor', 'abigail', null])->map(function (?string $name) {
        return strtoupper($name);
    })->reject(function (string $name) {
        return empty($name);
    });

如您所見，`Collection` 類別允許您鏈接其方法，以流暢地對底層陣列進行映射和歸約。通常，Collection 是不可變的，這表示每個 `Collection` 方法都會回傳一個全新的 `Collection` 實例。

<a name="creating-collections"></a>
### 建立集合

如上所述，`collect` 輔助函式會為給定的陣列回傳一個新的 `Illuminate\Support\Collection` 實例。因此，建立 Collection 就像這樣簡單：

    $collection = collect([1, 2, 3]);

> [!NOTE]
> [Eloquent](/docs/{{version}}/eloquent) 查詢的結果總是會以 `Collection` 實例的形式回傳。

<a name="extending-collections"></a>
### 擴充集合

Collection 是「可巨集化的 (macroable)」，這讓您可以在執行時為 `Collection` 類別新增額外的方法。`Illuminate\Support\Collection` 類別的 `macro` 方法接受一個閉包，該閉包將在您的巨集被呼叫時執行。巨集閉包可以透過 `$this` 存取 Collection 的其他方法，就像它是 Collection 類別的真實方法一樣。例如，以下程式碼為 `Collection` 類別新增了一個 `toUpper` 方法：

    use Illuminate\Support\Collection;
    use Illuminate\Support\Str;

    Collection::macro('toUpper', function () {
        return $this->map(function (string $value) {
            return Str::upper($value);
        });
    });

    $collection = collect(['first', 'second']);

    $upper = $collection->toUpper();

    // ['FIRST', 'SECOND']

通常，您應該在 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法中宣告 Collection 巨集。

<a name="macro-arguments"></a>
#### 巨集引數

如有必要，您可以定義接受額外引數的巨集：

    use Illuminate\Support\Collection;
    use Illuminate\Support\Facades\Lang;

    Collection::macro('toLocale', function (string $locale) {
        return $this->map(function (string $value) use ($locale) {
            return Lang::get($value, [], $locale);
        });
    });

    $collection = collect(['first', 'second']);

    $translated = $collection->toLocale('es');

<a name="available-methods"></a>
## 可用方法

在接下來的 Collection 文件中，我們將討論 `Collection` 類別中可用的每個方法。請記住，所有這些方法都可以鏈接起來，以流暢地操作底層陣列。此外，幾乎每個方法都會回傳一個新的 `Collection` 實例，讓您在必要時保留 Collection 的原始副本：

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

<div class="collection-method-list" markdown="1">

[after](#method-after)
[all](#method-all)
[average](#method-average)
[avg](#method-avg)
[before](#method-before)
[chunk](#method-chunk)
[chunkWhile](#method-chunkwhile)
[collapse](#method-collapse)
[collapseWithKeys](#method-collapsewithkeys)
[collect](#method-collect)
[combine](#method-combine)
[concat](#method-concat)
[contains](#method-contains)
[containsOneItem](#method-containsoneitem)
[containsStrict](#method-containsstrict)
[count](#method-count)
[countBy](#method-countBy)
[crossJoin](#method-crossjoin)
[dd](#method-dd)
[diff](#method-diff)
[diffAssoc](#method-diffassoc)
[diffAssocUsing](#method-diffassocusing)
[diffKeys](#method-diffkeys)
[doesntContain](#method-doesntcontain)
[dot](#method-dot)
[dump](#method-dump)
[duplicates](#method-duplicates)
[duplicatesStrict](#method-duplicatesstrict)
[each](#method-each)
[eachSpread](#method-eachspread)
[ensure](#method-ensure)
[every](#method-every)
[except](#method-except)
[filter](#method-filter)
[first](#method-first)
[firstOrFail](#method-first-or-fail)
[firstWhere](#method-first-where)
[flatMap](#method-flatmap)
[flatten](#method-flatten)
[flip](#method-flip)
[forget](#method-forget)
[forPage](#method-forpage)
[get](#method-get)
[groupBy](#method-groupby)
[has](#method-has)
[hasAny](#method-hasany)
[implode](#method-implode)
[intersect](#method-intersect)
[intersectUsing](#method-intersectusing)
[intersectAssoc](#method-intersectAssoc)
[intersectAssocUsing](#method-intersectassocusing)
[intersectByKeys](#method-intersectbykeys)
[isEmpty](#method-isempty)
[isNotEmpty](#method-isnotempty)
[join](#method-join)
[keyBy](#method-keyby)
[keys](#method-keys)
[last](#method-last)
[lazy](#method-lazy)
[macro](#method-macro)
[make](#method-make)
[map](#method-map)
[mapInto](#method-mapinto)
[mapSpread](#method-mapspread)
[mapToGroups](#method-maptogroups)
[mapWithKeys](#method-mapwithkeys)
[max](#method-max)
[median](#method-median)
[merge](#method-merge)
[mergeRecursive](#method-mergerecursive)
[min](#method-min)
[mode](#method-mode)
[multiply](#method-multiply)
[nth](#method-nth)
[only](#method-only)
[pad](#method-pad)
[partition](#method-partition)
[percentage](#method-percentage)
[pipe](#method-pipe)
[pipeInto](#method-pipeinto)
[pipeThrough](#method-pipethrough)
[pluck](#method-pluck)
[pop](#method-pop)
[prepend](#method-prepend)
[pull](#method-pull)
[push](#method-push)
[put](#method-put)
[random](#method-random)
[range](#method-range)
[reduce](#method-reduce)
[reduceSpread](#method-reduce-spread)
[reject](#method-reject)
[replace](#method-replace)
[replaceRecursive](#method-replacerecursive)
[reverse](#method-reverse)
[search](#method-search)
[select](#method-select)
[shift](#method-shift)
[shuffle](#method-shuffle)
[skip](#method-skip)
[skipUntil](#method-skipuntil)
[skipWhile](#method-skipwhile)
[slice](#method-slice)
[sliding](#method-sliding)
[sole](#method-sole)
[some](#method-some)
[sort](#method-sort)
[sortBy](#method-sortby)
[sortByDesc](#method-sortbydesc)
[sortDesc](#method-sortdesc)
[sortKeys](#method-sortkeys)
[sortKeysDesc](#method-sortkeysdesc)
[sortKeysUsing](#method-sortkeysusing)
[splice](#method-splice)
[split](#method-split)
[splitIn](#method-splitin)
[sum](#method-sum)
[take](#method-take)
[takeUntil](#method-takeuntil)
[takeWhile](#method-takewhile)
[tap](#method-tap)
[times](#method-times)
[toArray](#method-toarray)
[toJson](#method-tojson)
[transform](#method-transform)
[undot](#method-undot)
[union](#method-union)
[unique](#method-unique)
[uniqueStrict](#method-uniquestrict)
[unless](#method-unless)
[unlessEmpty](#method-unlessempty)
[unlessNotEmpty](#method-unlessnotempty)
[unwrap](#method-unwrap)
[value](#method-value)
[values](#method-values)
[when](#method-when)
[whenEmpty](#method-whenempty)
[whenNotEmpty](#method-whennotempty)
[where](#method-where)
[whereStrict](#method-wherestrict)
[whereBetween](#method-wherebetween)
[whereIn](#method-wherein)
[whereInStrict](#method-whereinstrict)
[whereInstanceOf](#method-whereinstanceof)
[whereNotBetween](#method-wherenotbetween)
[whereNotIn](#method-wherenotin)
[whereNotInStrict](#method-wherenotinstrict)
[whereNotNull](#method-wherenotnull)
[whereNull](#method-wherenull)
[wrap](#method-wrap)
[zip](#method-zip)

</div>

<a name="method-listing"></a>
## 方法列表

<style>
    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>

<a name="method-after"></a>
#### `after()` {.collection-method .first-collection-method}

`after` 方法會回傳給定項目之後的項目。如果找不到給定項目或它是最後一個項目，則回傳 `null`：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->after(3);

    // 4

    $collection->after(5);

    // null

此方法使用「寬鬆」比較來搜尋給定項目，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比較，您可以向方法提供 `strict` 引數：

    collect([2, 4, 6, 8])->after('4', strict: true);

    // null

或者，您可以提供自己的閉包來搜尋第一個通過給定真值測試的項目：

    collect([2, 4, 6, 8])->after(function (int $item, int $key) {
        return $item > 5;
    });

    // 8

<a name="method-all"></a>
#### `all()` {.collection-method}

`all` 方法會回傳 Collection 所代表的底層陣列：

    collect([1, 2, 3])->all();

    // [1, 2, 3]

<a name="method-average"></a>
#### `average()` {.collection-method}

[`avg`](#method-avg) 方法的別名。

<a name="method-avg"></a>
#### `avg()` {.collection-method}

`avg` 方法會回傳給定鍵的[平均值](https://en.wikipedia.org/wiki/Average)：

    $average = collect([
        ['foo' => 10],
        ['foo' => 10],
        ['foo' => 20],
        ['foo' => 40]
    ])->avg('foo');

    // 20

    $average = collect([1, 1, 2, 4])->avg();

    // 2

<a name="method-before"></a>
#### `before()` {.collection-method}

`before` 方法與 [`after`](#method-after) 方法相反。它會回傳給定項目之前的項目。如果找不到給定項目或它是第一個項目，則回傳 `null`：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->before(3);

    // 2

    $collection->before(1);

    // null

    collect([2, 4, 6, 8])->before('4', strict: true);

    // null

    collect([2, 4, 6, 8])->before(function (int $item, int $key) {
        return $item > 5;
    });

    // 4

<a name="method-chunk"></a>
#### `chunk()` {.collection-method}

`chunk` 方法會將 Collection 分割成多個指定大小的較小 Collection：

    $collection = collect([1, 2, 3, 4, 5, 6, 7]);

    $chunks = $collection->chunk(4);

    $chunks->all();

    // [[1, 2, 3, 4], [5, 6, 7]]

此方法在 [視圖](/docs/{{version}}/views) 中使用網格系統（例如 [Bootstrap](https://getbootstrap.com/docs/5.3/layout/grid/)）時特別有用。例如，假設您有一個 [Eloquent](/docs/{{version}}/eloquent) 模型 Collection，您想在網格中顯示：

```blade
 @foreach ($products->chunk(3) as $chunk)
    <div class="row">
        @foreach ($chunk as $product)
            <div class="col-xs-4">{{ $product->name }}</div>
        @endforeach
    </div>
 @endforeach
```

<a name="method-chunkwhile"></a>
#### `chunkWhile()` {.collection-method}

`chunkWhile` 方法會根據給定回呼的評估，將 Collection 分割成多個較小的 Collection。傳遞給閉包的 `$chunk` 變數可用於檢查前一個元素：

    $collection = collect(str_split('AABBCCCD'));

    $chunks = $collection->chunkWhile(function (string $value, int $key, Collection $chunk) {
        return $value === $chunk->last();
    });

    $chunks->all();

    // [['A', 'A'], ['B', 'B'], ['C', 'C', 'C'], ['D']]

<a name="method-collapse"></a>
#### `collapse()` {.collection-method}

`collapse` 方法會將陣列的 Collection 摺疊成單一的扁平 Collection：

    $collection = collect([
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ]);

    $collapsed = $collection->collapse();

    $collapsed->all();

    // [1, 2, 3, 4, 5, 6, 7, 8, 9]

<a name="method-collapsewithkeys"></a>
#### `collapseWithKeys()` {.collection-method}

`collapseWithKeys` 方法會將陣列或 Collection 的 Collection 扁平化為單一 Collection，同時保留原始鍵：

    $collection = collect([
      ['first'  => collect([1, 2, 3])],
      ['second' => [4, 5, 6]],
      ['third'  => collect([7, 8, 9])]
    ]);


    $collapsed = $collection->collapseWithKeys();

    $collapsed->all();

    // [
    //     'first'  => [1, 2, 3],
    //     'second' => [4, 5, 6],
    //     'third'  => [7, 8, 9],
    // ]

<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 方法會回傳一個新的 `Collection` 實例，其中包含 Collection 中目前的項目：

    $collectionA = collect([1, 2, 3]);

    $collectionB = $collectionA->collect();

    $collectionB->all();

    // [1, 2, 3]

`collect` 方法主要用於將 [惰性集合](#lazy-collections) 轉換為標準的 `Collection` 實例：

    $lazyCollection = LazyCollection::make(function () {
        yield 1;
        yield 2;
        yield 3;
    });

    $collection = $lazyCollection->collect();

    $collection::class;

    // 'Illuminate\Support\Collection'

    $collection->all();

    // [1, 2, 3]

> [!NOTE]
> 當您有一個 `Enumerable` 實例並需要一個非惰性 Collection 實例時，`collect` 方法特別有用。由於 `collect()` 是 `Enumerable` 契約的一部分，您可以安全地使用它來取得 `Collection` 實例。

<a name="method-combine"></a>
#### `combine()` {.collection-method}

`combine` 方法會將 Collection 的值作為鍵，與另一個陣列或 Collection 的值組合起來：

    $collection = collect(['name', 'age']);

    $combined = $collection->combine(['George', 29]);

    $combined->all();

    // ['name' => 'George', 'age' => 29]

<a name="method-concat"></a>
#### `concat()` {.collection-method}

`concat` 方法會將給定 `array` 或 Collection 的值附加到另一個 Collection 的末尾：

    $collection = collect(['John Doe']);

    $concatenated = $collection->concat(['Jane Doe'])->concat(['name' => 'Johnny Doe']);

    $concatenated->all();

    // ['John Doe', 'Jane Doe', 'Johnny Doe']

`concat` 方法會對附加到原始 Collection 的項目進行數字重新索引。若要在關聯 Collection 中保留鍵，請參閱 [merge](#method-merge) 方法。

<a name="method-contains"></a>
#### `contains()` {.collection-method}

`contains` 方法會判斷 Collection 是否包含給定項目。您可以將閉包傳遞給 `contains` 方法，以判斷 Collection 中是否存在符合給定真值測試的元素：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->contains(function (int $value, int $key) {
        return $value > 5;
    });

    // false

或者，您可以將字串傳遞給 `contains` 方法，以判斷 Collection 是否包含給定項目值：

    $collection = collect(['name' => 'Desk', 'price' => 100]);

    $collection->contains('Desk');

    // true

    $collection->contains('New York');

    // false

您也可以將鍵/值對傳遞給 `contains` 方法，這將判斷給定對是否存在於 Collection 中：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->contains('product', 'Bookcase');

    // false

`contains` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。使用 [`containsStrict`](#method-containsstrict) 方法以使用「嚴格」比較進行篩選。

`contains` 的反向操作，請參閱 [doesntContain](#method-doesntcontain) 方法。

<a name="method-containsoneitem"></a>
#### `containsOneItem()` {.collection-method}

`containsOneItem` 方法會判斷 Collection 是否包含單一項目：

    collect([])->containsOneItem();

    // false

    collect(['1'])->containsOneItem();

    // true

    collect(['1', '2'])->containsOneItem();

    // false

<a name="method-containsstrict"></a>
#### `containsStrict()` {.collection-method}

此方法與 [`contains`](#method-contains) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-contains) 時，此方法的行為會被修改。

<a name="method-count"></a>
#### `count()` {.collection-method}

`count` 方法會回傳 Collection 中項目的總數：

    $collection = collect([1, 2, 3, 4]);

    $collection->count();

    // 4

<a name="method-countBy"></a>
#### `countBy()` {.collection-method}

`countBy` 方法會計算 Collection 中值的出現次數。預設情況下，此方法會計算每個元素的出現次數，讓您可以計算 Collection 中某些「類型」元素的數量：

    $collection = collect([1, 2, 2, 2, 3]);

    $counted = $collection->countBy();

    $counted->all();

    // [1 => 1, 2 => 3, 3 => 1]

您可以將閉包傳遞給 `countBy` 方法，以根據自訂值計算所有項目：

    $collection = collect(['alice @gmail.com', 'bob @yahoo.com', 'carlos @gmail.com']);

    $counted = $collection->countBy(function (string $email) {
        return substr(strrchr($email, " @"), 1);
    });

    $counted->all();

    // ['gmail.com' => 2, 'yahoo.com' => 1]

<a name="method-crossjoin"></a>
#### `crossJoin()` {.collection-method}

`crossJoin` 方法會將 Collection 的值與給定陣列或 Collection 的值進行交叉連接，回傳一個包含所有可能排列的笛卡爾積：

    $collection = collect([1, 2]);

    $matrix = $collection->crossJoin(['a', 'b']);

    $matrix->all();

    /*
        [
            [1, 'a'],
            [1, 'b'],
            [2, 'a'],
            [2, 'b'],
        ]
    */

    $collection = collect([1, 2]);

    $matrix = $collection->crossJoin(['a', 'b'], ['I', 'II']);

    $matrix->all();

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

<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 方法會傾印 Collection 的項目並結束腳本執行：

    $collection = collect(['John Doe', 'Jane Doe']);

    $collection->dd();

    /*
        Collection {
            #items: array:2 [
                0 => "John Doe"
                1 => "Jane Doe"
            ]
        }
    */

如果您不想停止執行腳本，請改用 [`dump`](#method-dump) 方法。

<a name="method-diff"></a>
#### `diff()` {.collection-method}

`diff` 方法會根據其值將 Collection 與另一個 Collection 或純 PHP `array` 進行比較。此方法將回傳原始 Collection 中不存在於給定 Collection 中的值：

    $collection = collect([1, 2, 3, 4, 5]);

    $diff = $collection->diff([2, 4, 6, 8]);

    $diff->all();

    // [1, 3, 5]

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-diff) 時，此方法的行為會被修改。

<a name="method-diffassoc"></a>
#### `diffAssoc()` {.collection-method}

`diffAssoc` 方法會根據其鍵和值將 Collection 與另一個 Collection 或純 PHP `array` 進行比較。此方法將回傳原始 Collection 中不存在於給定 Collection 中的鍵/值對：

    $collection = collect([
        'color' => 'orange',
        'type' => 'fruit',
        'remain' => 6,
    ]);

    $diff = $collection->diffAssoc([
        'color' => 'yellow',
        'type' => 'fruit',
        'remain' => 3,
        'used' => 6,
    ]);

    $diff->all();

    // ['color' => 'orange', 'remain' => 6]

<a name="method-diffassocusing"></a>
#### `diffAssocUsing()` {.collection-method}

與 `diffAssoc` 不同，`diffAssocUsing` 接受一個使用者提供的回呼函式用於索引比較：

    $collection = collect([
        'color' => 'orange',
        'type' => 'fruit',
        'remain' => 6,
    ]);

    $diff = $collection->diffAssocUsing([
        'Color' => 'yellow',
        'Type' => 'fruit',
        'Remain' => 3,
    ], 'strnatcasecmp');

    $diff->all();

    // ['color' => 'orange', 'remain' => 6]

回呼必須是一個比較函式，回傳一個小於、等於或大於零的整數。有關更多資訊，請參閱 PHP 文件中關於 [`array_diff_uassoc`](https://www.php.net/array_diff_uassoc#refsect1-function.array-diff-uassoc-parameters) 的部分，這是 `diffAssocUsing` 方法內部使用的 PHP 函式。

<a name="method-diffkeys"></a>
#### `diffKeys()` {.collection-method}

`diffKeys` 方法會根據其鍵將 Collection 與另一個 Collection 或純 PHP `array` 進行比較。此方法將回傳原始 Collection 中不存在於給定 Collection 中的鍵/值對：

    $collection = collect([
        'one' => 10,
        'two' => 20,
        'three' => 30,
        'four' => 40,
        'five' => 50,
    ]);

    $diff = $collection->diffKeys([
        'two' => 2,
        'four' => 4,
        'six' => 6,
        'eight' => 8,
    ]);

    $diff->all();

    // ['one' => 10, 'three' => 30, 'five' => 50]

<a name="method-doesntcontain"></a>
#### `doesntContain()` {.collection-method}

`doesntContain` 方法會判斷 Collection 是否不包含給定項目。您可以將閉包傳遞給 `doesntContain` 方法，以判斷 Collection 中是否存在不符合給定真值測試的元素：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->doesntContain(function (int $value, int $key) {
        return $value < 5;
    });

    // false

或者，您可以將字串傳遞給 `doesntContain` 方法，以判斷 Collection 是否不包含給定項目值：

    $collection = collect(['name' => 'Desk', 'price' => 100]);

    $collection->doesntContain('Table');

    // true

    $collection->doesntContain('Desk');

    // false

您也可以將鍵/值對傳遞給 `doesntContain` 方法，這將判斷給定對是否存在於 Collection 中：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->doesntContain('product', 'Bookcase');

    // true

`doesntContain` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。

<a name="method-dot"></a>
#### `dot()` {.collection-method}

`dot` 方法會將多維 Collection 扁平化為單層 Collection，並使用「點」符號表示深度：

    $collection = collect(['products' => ['desk' => ['price' => 100]]]);

    $flattened = $collection->dot();

    $flattened->all();

    // ['products.desk.price' => 100]

<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 方法會傾印 Collection 的項目：

    $collection = collect(['John Doe', 'Jane Doe']);

    $collection->dump();

    /*
        Collection {
            #items: array:2 [
                0 => "John Doe"
                1 => "Jane Doe"
            ]
        }
    */

如果您想在傾印 Collection 後停止執行腳本，請改用 [`dd`](#method-dd) 方法。

<a name="method-duplicates"></a>
#### `duplicates()` {.collection-method}

`duplicates` 方法會從 Collection 中檢索並回傳重複的值：

    $collection = collect(['a', 'b', 'a', 'c', 'b']);

    $collection->duplicates();

    // [2 => 'a', 4 => 'b']

如果 Collection 包含陣列或物件，您可以傳遞您希望檢查重複值的屬性鍵：

    $employees = collect([
        ['email' => 'abigail @example.com', 'position' => 'Developer'],
        ['email' => 'james @example.com', 'position' => 'Designer'],
        ['email' => 'victoria @example.com', 'position' => 'Developer'],
    ]);

    $employees->duplicates('position');

    // [2 => 'Developer']

<a name="method-duplicatesstrict"></a>
#### `duplicatesStrict()` {.collection-method}

此方法與 [`duplicates`](#method-duplicates) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

<a name="method-each"></a>
#### `each()` {.collection-method}

`each` 方法會迭代 Collection 中的項目，並將每個項目傳遞給閉包：

    $collection = collect([1, 2, 3, 4]);

    $collection->each(function (int $item, int $key) {
        // ...
    });

如果您想停止迭代項目，您可以從閉包中回傳 `false`：

    $collection->each(function (int $item, int $key) {
        if (/* condition */) {
            return false;
        }
    });

<a name="method-eachspread"></a>
#### `eachSpread()` {.collection-method}

`eachSpread` 方法會迭代 Collection 的項目，將每個巢狀項目值傳遞給給定的回呼：

    $collection = collect([['John Doe', 35], ['Jane Doe', 33]]);

    $collection->eachSpread(function (string $name, int $age) {
        // ...
    });

您可以從回呼中回傳 `false` 來停止迭代項目：

    $collection->eachSpread(function (string $name, int $age) {
        return false;
    });

<a name="method-ensure"></a>
#### `ensure()` {.collection-method}

`ensure` 方法可用於驗證 Collection 的所有元素是否為給定類型或類型列表。否則，將拋出 `UnexpectedValueException`：

    return $collection->ensure(User::class);

    return $collection->ensure([User::class, Customer::class]);

也可以指定原始類型，例如 `string`、`int`、`float`、`bool` 和 `array`：

    return $collection->ensure('int');

> [!WARNING]
> `ensure` 方法不保證以後不會將不同類型的元素新增到 Collection 中。

<a name="method-every"></a>
#### `every()` {.collection-method}

`every` 方法可用於驗證 Collection 的所有元素是否通過給定真值測試：

    collect([1, 2, 3, 4])->every(function (int $value, int $key) {
        return $value > 2;
    });

    // false

如果 Collection 為空，`every` 方法將回傳 `true`：

    $collection = collect([]);

    $collection->every(function (int $value, int $key) {
        return $value > 2;
    });

    // true

<a name="method-except"></a>
#### `except()` {.collection-method}

`except` 方法會回傳 Collection 中除指定鍵之外的所有項目：

    $collection = collect(['product_id' => 1, 'price' => 100, 'discount' => false]);

    $filtered = $collection->except(['price', 'discount']);

    $filtered->all();

    // ['product_id' => 1]

`except` 的反向操作，請參閱 [only](#method-only) 方法。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-except) 時，此方法的行為會被修改。

<a name="method-filter"></a>
#### `filter()` {.collection-method}

`filter` 方法會使用給定回呼篩選 Collection，只保留通過給定真值測試的項目：

    $collection = collect([1, 2, 3, 4]);

    $filtered = $collection->filter(function (int $value, int $key) {
        return $value > 2;
    });

    $filtered->all();

    // [3, 4]

如果未提供回呼，則 Collection 中所有等同於 `false` 的項目都將被移除：

    $collection = collect([1, 2, 3, null, false, '', 0, []]);

    $collection->filter()->all();

    // [1, 2, 3]

`filter` 的反向操作，請參閱 [reject](#method-reject) 方法。

<a name="method-first"></a>
#### `first()` {.collection-method}

`first` 方法會回傳 Collection 中第一個通過給定真值測試的元素：

    collect([1, 2, 3, 4])->first(function (int $value, int $key) {
        return $value > 2;
    });

    // 3

您也可以不帶引數呼叫 `first` 方法以取得 Collection 中的第一個元素。如果 Collection 為空，則回傳 `null`：

    collect([1, 2, 3, 4])->first();

    // 1

<a name="method-first-or-fail"></a>
#### `firstOrFail()` {.collection-method}

`firstOrFail` 方法與 `first` 方法相同；但是，如果找不到結果，將拋出 `Illuminate\Support\ItemNotFoundException` 異常：

    collect([1, 2, 3, 4])->firstOrFail(function (int $value, int $key) {
        return $value > 5;
    });

    // Throws ItemNotFoundException...

您也可以不帶引數呼叫 `firstOrFail` 方法以取得 Collection 中的第一個元素。如果 Collection 為空，將拋出 `Illuminate\Support\ItemNotFoundException` 異常：

    collect([])->firstOrFail();

    // Throws ItemNotFoundException...

<a name="method-first-where"></a>
#### `firstWhere()` {.collection-method}

`firstWhere` 方法會回傳 Collection 中第一個具有給定鍵/值對的元素：

    $collection = collect([
        ['name' => 'Regena', 'age' => null],
        ['name' => 'Linda', 'age' => 14],
        ['name' => 'Diego', 'age' => 23],
        ['name' => 'Linda', 'age' => 84],
    ]);

    $collection->firstWhere('name', 'Linda');

    // ['name' => 'Linda', 'age' => 14]

您也可以使用比較運算子呼叫 `firstWhere` 方法：

    $collection->firstWhere('age', '>=', 18);

    // ['name' => 'Diego', 'age' => 23]

與 [where](#method-where) 方法一樣，您可以向 `firstWhere` 方法傳遞一個引數。在此情境下，`firstWhere` 方法將回傳給定項目鍵的值為「真值」的第一個項目：

    $collection->firstWhere('age');

    // ['name' => 'Linda', 'age' => 14]

<a name="method-flatmap"></a>
#### `flatMap()` {.collection-method}

`flatMap` 方法會迭代 Collection 並將每個值傳遞給給定的閉包。閉包可以自由地修改項目並回傳它，從而形成一個新的修改項目 Collection。然後，陣列會被扁平化一層：

    $collection = collect([
        ['name' => 'Sally'],
        ['school' => 'Arkansas'],
        ['age' => 28]
    ]);

    $flattened = $collection->flatMap(function (array $values) {
        return array_map('strtoupper', $values);
    });

    $flattened->all();

    // ['name' => 'SALLY', 'school' => 'ARKANSAS', 'age' => '28'];

<a name="method-flatten"></a>
#### `flatten()` {.collection-method}

`flatten` 方法會將多維 Collection 扁平化為單一維度：

    $collection = collect([
        'name' => 'taylor',
        'languages' => [
            'php', 'javascript'
        ]
    ]);

    $flattened = $collection->flatten();

    $flattened->all();

    // ['taylor', 'php', 'javascript'];

如有必要，您可以向 `flatten` 方法傳遞一個「深度」引數：

    $collection = collect([
        'Apple' => [
            [
                'name' => 'iPhone 6S',
                'brand' => 'Apple'
            ],
        ],
        'Samsung' => [
            [
                'name' => 'Galaxy S7',
                'brand' => 'Samsung'
            ],
        ],
    ]);

    $products = $collection->flatten(1);

    $products->values()->all();

    /*
        [
            ['name' => 'iPhone 6S', 'brand' => 'Apple'],
            ['name' => 'Galaxy S7', 'brand' => 'Samsung'],
        ]
    */

在此範例中，如果沒有提供深度就呼叫 `flatten`，也會扁平化巢狀陣列，導致 `['iPhone 6S', 'Apple', 'Galaxy S7', 'Samsung']`。提供深度可讓您指定巢狀陣列將被扁平化的層數。

<a name="method-flip"></a>
#### `flip()` {.collection-method}

`flip` 方法會將 Collection 的鍵與其對應的值交換：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $flipped = $collection->flip();

    $flipped->all();

    // ['taylor' => 'name', 'laravel' => 'framework']

<a name="method-forget"></a>
#### `forget()` {.collection-method}

`forget` 方法會根據其鍵從 Collection 中移除項目：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    // Forget a single key...
    $collection->forget('name');

    // ['framework' => 'laravel']

    // Forget multiple keys...
    $collection->forget(['name', 'framework']);

    // []

> [!WARNING]
> 與大多數其他 Collection 方法不同，`forget` 不會回傳新的修改 Collection；它會修改並回傳其被呼叫的 Collection。

<a name="method-forpage"></a>
#### `forPage()` {.collection-method}

`forPage` 方法會回傳一個新的 Collection，其中包含指定頁碼上的項目。此方法接受頁碼作為第一個引數，每頁顯示的項目數作為第二個引數：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9]);

    $chunk = $collection->forPage(2, 3);

    $chunk->all();

    // [4, 5, 6]

<a name="method-get"></a>
#### `get()` {.collection-method}

`get` 方法會回傳給定鍵的項目。如果鍵不存在，則回傳 `null`：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $value = $collection->get('name');

    // taylor

您可以選擇將預設值作為第二個引數傳遞：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $value = $collection->get('age', 34);

    // 34

您甚至可以將回呼作為方法的預設值傳遞。如果指定的鍵不存在，則回傳回呼的結果：

    $collection->get('email', function () {
        return 'taylor @example.com';
    });

    // taylor @example.com

<a name="method-groupby"></a>
#### `groupBy()` {.collection-method}

`groupBy` 方法會根據給定鍵對 Collection 的項目進行分組：

    $collection = collect([
        ['account_id' => 'account-x10', 'product' => 'Chair'],
        ['account_id' => 'account-x10', 'product' => 'Bookcase'],
        ['account_id' => 'account-x11', 'product' => 'Desk'],
    ]);

    $grouped = $collection->groupBy('account_id');

    $grouped->all();

    /*
        [
            'account-x10' => [
                ['account_id' => 'account-x10', 'product' => 'Chair'],
                ['account_id' => 'account-x10', 'product' => 'Bookcase'],
            ],
            'account-x11' => [
                ['account_id' => 'account-x11', 'product' => 'Desk'],
            ],
        ]
    */

您可以傳遞回呼而不是字串 `key`。回呼應該回傳您希望用作分組鍵的值：

    $grouped = $collection->groupBy(function (array $item, int $key) {
        return substr($item['account_id'], -3);
    });

    $grouped->all();

    /*
        [
            'x10' => [
                ['account_id' => 'account-x10', 'product' => 'Chair'],
                ['account_id' => 'account-x10', 'product' => 'Bookcase'],
            ],
            'x11' => [
                ['account_id' => 'account-x11', 'product' => 'Desk'],
            ],
        ]
    */

可以將多個分組條件作為陣列傳遞。每個陣列元素將應用於多維陣列中的相應層級：

    $data = new Collection([
        10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
        20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
        30 => ['user' => 3, 'skill' => 2, 'roles' => ['Role_1']],
        40 => ['user' => 4, 'skill' => 2, 'roles' => ['Role_2']],
    ]);

    $result = $data->groupBy(['skill', function (array $item) {
        return $item['roles'];
    }], preserveKeys: true);

    /*
    [
        1 => [
            'Role_1' => [
                10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
                20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
            ],
            'Role_2' => [
                20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
            ],
            'Role_3' => [
                10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
            ],
        ],
        2 => [
            'Role_1' => [
                30 => ['user' => 3, 'skill' => 2, 'roles' => ['Role_1']],
            ],
            'Role_2' => [
                40 => ['user' => 4, 'skill' => 2, 'roles' => ['Role_2']],
            ],
        ],
    ];
    */

<a name="method-has"></a>
#### `has()` {.collection-method}

`has` 方法會判斷 Collection 中是否存在給定鍵：

    $collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

    $collection->has('product');

    // true

    $collection->has(['product', 'amount']);

    // true

    $collection->has(['amount', 'price']);

    // false

<a name="method-hasany"></a>
#### `hasAny()` {.collection-method}

`hasAny` 方法會判斷 Collection 中是否存在任何給定鍵：

    $collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

    $collection->hasAny(['product', 'price']);

    // true

    $collection->hasAny(['name', 'price']);

    // false

<a name="method-implode"></a>
#### `implode()` {.collection-method}

`implode` 方法會將 Collection 中的項目連接起來。其引數取決於 Collection 中項目的類型。如果 Collection 包含陣列或物件，您應該傳遞您希望連接的屬性鍵，以及您希望放在值之間的「膠合」字串：

    $collection = collect([
        ['account_id' => 1, 'product' => 'Desk'],
        ['account_id' => 2, 'product' => 'Chair'],
    ]);

    $collection->implode('product', ', ');

    // Desk, Chair

如果 Collection 包含簡單的字串或數值，您應該將「膠合」作為唯一引數傳遞給方法：

    collect([1, 2, 3, 4, 5])->implode('-');

    // '1-2-3-4-5'

如果您想格式化被連接的值，您可以將閉包傳遞給 `implode` 方法：

    $collection->implode(function (array $item, int $key) {
        return strtoupper($item['product']);
    }, ', ');

    // DESK, CHAIR

<a name="method-intersect"></a>
#### `intersect()` {.collection-method}

`intersect` 方法會從原始 Collection 中移除任何不存在於給定 `array` 或 Collection 中的值。結果 Collection 將保留原始 Collection 的鍵：

    $collection = collect(['Desk', 'Sofa', 'Chair']);

    $intersect = $collection->intersect(['Desk', 'Chair', 'Bookcase']);

    $intersect->all();

    // [0 => 'Desk', 2 => 'Chair']

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-intersect) 時，此方法的行為會被修改。

<a name="method-intersectusing"></a>
#### `intersectUsing()` {.collection-method}

`intersectUsing` 方法會從原始 Collection 中移除任何不存在於給定 `array` 或 Collection 中的值，並使用自訂回呼來比較這些值。結果 Collection 將保留原始 Collection 的鍵：

    $collection = collect(['Desk', 'Sofa', 'Chair']);

    $intersect = $collection->intersectUsing(['desk', 'chair', 'bookcase'], function ($a, $b) {
        return strcasecmp($a, $b);
    });

    $intersect->all();

    // [0 => 'Desk', 2 => 'Chair']

<a name="method-intersectAssoc"></a>
#### `intersectAssoc()` {.collection-method}

`intersectAssoc` 方法會將原始 Collection 與另一個 Collection 或 `array` 進行比較，回傳所有給定 Collection 中都存在的鍵/值對：

    $collection = collect([
        'color' => 'red',
        'size' => 'M',
        'material' => 'cotton'
    ]);

    $intersect = $collection->intersectAssoc([
        'color' => 'blue',
        'size' => 'M',
        'material' => 'polyester'
    ]);

    $intersect->all();

    // ['size' => 'M']

<a name="method-intersectassocusing"></a>
#### `intersectAssocUsing()` {.collection-method}

`intersectAssocUsing` 方法會將原始 Collection 與另一個 Collection 或 `array` 進行比較，回傳兩者都存在的鍵/值對，並使用自訂比較回呼來判斷鍵和值的相等性：

    $collection = collect([
        'color' => 'red',
        'Size' => 'M',
        'material' => 'cotton',
    ]);

    $intersect = $collection->intersectAssocUsing([
        'color' => 'blue',
        'size' => 'M',
        'material' => 'polyester',
    ], function ($a, $b) {
        return strcasecmp($a, $b);
    });

    $intersect->all();

    // ['Size' => 'M']

<a name="method-intersectbykeys"></a>
#### `intersectByKeys()` {.collection-method}

`intersectByKeys` 方法會從原始 Collection 中移除任何不存在於給定 `array` 或 Collection 中的鍵及其對應的值：

    $collection = collect([
        'serial' => 'UX301', 'type' => 'screen', 'year' => 2009,
    ]);

    $intersect = $collection->intersectByKeys([
        'reference' => 'UX404', 'type' => 'tab', 'year' => 2011,
    ]);

    $intersect->all();

    // ['type' => 'screen', 'year' => 2009]

<a name="method-isempty"></a>
#### `isEmpty()` {.collection-method}

`isEmpty` 方法會回傳 `true` 如果 Collection 為空；否則，回傳 `false`：

    collect([])->isEmpty();

    // true

<a name="method-isnotempty"></a>
#### `isNotEmpty()` {.collection-method}

`isNotEmpty` 方法會回傳 `true` 如果 Collection 不為空；否則，回傳 `false`：

    collect([])->isNotEmpty();

    // false

<a name="method-join"></a>
#### `join()` {.collection-method}

`join` 方法會將 Collection 的值與字串連接起來。使用此方法的第二個引數，您還可以指定最後一個元素應如何附加到字串：

    collect(['a', 'b', 'c'])->join(', '); // 'a, b, c'
    collect(['a', 'b', 'c'])->join(', ', ', and '); // 'a, b, and c'
    collect(['a', 'b'])->join(', ', ' and '); // 'a and b'
    collect(['a'])->join(', ', ' and '); // 'a'
    collect([])->join(', ', ' and '); // ''

<a name="method-keyby"></a>
#### `keyBy()` {.collection-method}

`keyBy` 方法會根據給定鍵對 Collection 進行鍵控。如果多個項目具有相同的鍵，則只有最後一個會出現在新的 Collection 中：

    $collection = collect([
        ['product_id' => 'prod-100', 'name' => 'Desk'],
        ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]);

    $keyed = $collection->keyBy('product_id');

    $keyed->all();

    /*
        [
            'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
            'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
        ]
    */

您也可以將回呼傳遞給方法。回呼應該回傳用於鍵控 Collection 的值：

    $keyed = $collection->keyBy(function (array $item, int $key) {
        return strtoupper($item['product_id']);
    });

    $keyed->all();

    /*
        [
            'PROD-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
            'PROD-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
        ]
    */

<a name="method-keys"></a>
#### `keys()` {.collection-method}

`keys` 方法會回傳 Collection 的所有鍵：

    $collection = collect([
        'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
        'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]);

    $keys = $collection->keys();

    $keys->all();

    // ['prod-100', 'prod-200']

<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 方法會回傳 Collection 中最後一個通過給定真值測試的元素：

    collect([1, 2, 3, 4])->last(function (int $value, int $key) {
        return $value < 3;
    });

    // 2

您也可以不帶引數呼叫 `last` 方法以取得 Collection 中的最後一個元素。如果 Collection 為空，則回傳 `null`：

    collect([1, 2, 3, 4])->last();

    // 4

<a name="method-lazy"></a>
#### `lazy()` {.collection-method}

`lazy` 方法會從底層項目陣列回傳一個新的 [`LazyCollection`](#lazy-collections) 實例：

    $lazyCollection = collect([1, 2, 3, 4])->lazy();

    $lazyCollection::class;

    // Illuminate\Support\LazyCollection

    $lazyCollection->all();

    // [1, 2, 3, 4]

當您需要對包含許多項目的巨大 `Collection` 執行轉換時，這特別有用：

    $count = $hugeCollection
        ->lazy()
        ->where('country', 'FR')
        ->where('balance', '>', '100')
        ->count();

透過將 Collection 轉換為 `LazyCollection`，我們避免了分配大量額外記憶體。儘管原始 Collection 仍然將其值保留在記憶體中，但後續的篩選不會。因此，在篩選 Collection 的結果時，幾乎不會分配額外記憶體。

<a name="method-macro"></a>
#### `macro()` {.collection-method}

靜態 `macro` 方法允許您在執行時為 `Collection` 類別新增方法。有關更多資訊，請參閱 [擴充集合](#extending-collections) 的文件。

<a name="method-make"></a>
#### `make()` {.collection-method}

靜態 `make` 方法會建立一個新的 Collection 實例。請參閱 [建立集合](#creating-collections) 部分。

<a name="method-map"></a>
#### `map()` {.collection-method}

`map` 方法會迭代 Collection 並將每個值傳遞給給定的回呼。回呼可以自由地修改項目並回傳它，從而形成一個新的修改項目 Collection：

    $collection = collect([1, 2, 3, 4, 5]);

    $multiplied = $collection->map(function (int $item, int $key) {
        return $item * 2;
    });

    $multiplied->all();

    // [2, 4, 6, 8, 10]

> [!WARNING]
> 與大多數其他 Collection 方法一樣，`map` 會回傳一個新的 Collection 實例；它不會修改其被呼叫的 Collection。如果您想轉換原始 Collection，請使用 [`transform`](#method-transform) 方法。

<a name="method-mapinto"></a>
#### `mapInto()` {.collection-method}

`mapInto()` 方法會迭代 Collection，透過將值傳遞給建構函式來建立給定類別的新實例：

    class Currency
    {
        /**
         * Create a new currency instance.
         */
        function __construct(
            public string $code,
        ) {}
    }

    $collection = collect(['USD', 'EUR', 'GBP']);

    $currencies = $collection->mapInto(Currency::class);

    $currencies->all();

    // [Currency('USD'), Currency('EUR'), Currency('GBP')]

<a name="method-mapspread"></a>
#### `mapSpread()` {.collection-method}

`mapSpread` 方法會迭代 Collection 的項目，將每個巢狀項目值傳遞給給定的閉包。閉包可以自由地修改項目並回傳它，從而形成一個新的修改項目 Collection：

    $collection = collect([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

    $chunks = $collection->chunk(2);

    $sequence = $chunks->mapSpread(function (int $even, int $odd) {
        return $even + $odd;
    });

    $sequence->all();

    // [1, 5, 9, 13, 17]

<a name="method-maptogroups"></a>
#### `mapToGroups()` {.collection-method}

`mapToGroups` 方法會根據給定的閉包對 Collection 的項目進行分組。閉包應該回傳一個包含單一鍵/值對的關聯陣列，從而形成一個新的分組值 Collection：

    $collection = collect([
        [
            'name' => 'John Doe',
            'department' => 'Sales',
        ],
        [
            'name' => 'Jane Doe',
            'department' => 'Sales',
        ],
        [
            'name' => 'Johnny Doe',
            'department' => 'Marketing',
        ]
    ]);

    $grouped = $collection->mapToGroups(function (array $item, int $key) {
        return [$item['department'] => $item['name']];
    });

    $grouped->all();

    /*
        [
            'Sales' => ['John Doe', 'Jane Doe'],
            'Marketing' => ['Johnny Doe'],
        ]
    */

    $grouped->get('Sales')->all();

    // ['John Doe', 'Jane Doe']

<a name="method-mapwithkeys"></a>
#### `mapWithKeys()` {.collection-method}

`mapWithKeys` 方法會迭代 Collection 並將每個值傳遞給給定的回呼。回呼應該回傳一個包含單一鍵/值對的關聯陣列：

    $collection = collect([
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
    ]);

    $keyed = $collection->mapWithKeys(function (array $item, int $key) {
        return [$item['email'] => $item['name']];
    });

    $keyed->all();

    /*
        [
            'john @example.com' => 'John',
            'jane @example.com' => 'Jane',
        ]
    */

<a name="method-max"></a>
#### `max()` {.collection-method}

`max` 方法會回傳給定鍵的最大值：

    $max = collect([
        ['foo' => 10],
        ['foo' => 20]
    ])->max('foo');

    // 20

    $max = collect([1, 2, 3, 4, 5])->max();

    // 5

<a name="method-median"></a>
#### `median()` {.collection-method}

`median` 方法會回傳給定鍵的[中位數](https://en.wikipedia.org/wiki/Median)值：

    $median = collect([
        ['foo' => 10],
        ['foo' => 10],
        ['foo' => 20],
        ['foo' => 40]
    ])->median('foo');

    // 15

    $median = collect([1, 1, 2, 4])->median();

    // 1.5

<a name="method-merge"></a>
#### `merge()` {.collection-method}

`merge` 方法會將給定陣列或 Collection 與原始 Collection 合併。如果給定項目中的字串鍵與原始 Collection 中的字串鍵匹配，則給定項目的值將覆寫原始 Collection 中的值：

    $collection = collect(['product_id' => 1, 'price' => 100]);

    $merged = $collection->merge(['price' => 200, 'discount' => false]);

    $merged->all();

    // ['product_id' => 1, 'price' => 200, 'discount' => false]

如果給定項目的鍵是數字，則這些值將附加到 Collection 的末尾：

    $collection = collect(['Desk', 'Chair']);

    $merged = $collection->merge(['Bookcase', 'Door']);

    $merged->all();

    // ['Desk', 'Chair', 'Bookcase', 'Door']

<a name="method-mergerecursive"></a>
#### `mergeRecursive()` {.collection-method}

`mergeRecursive` 方法會將給定陣列或 Collection 與原始 Collection 遞迴合併。如果給定項目中的字串鍵與原始 Collection 中的字串鍵匹配，則這些鍵的值將合併到一個陣列中，並遞迴執行此操作：

    $collection = collect(['product_id' => 1, 'price' => 100]);

    $merged = $collection->mergeRecursive([
        'product_id' => 2,
        'price' => 200,
        'discount' => false
    ]);

    $merged->all();

    // ['product_id' => [1, 2], 'price' => [100, 200], 'discount' => false]

<a name="method-min"></a>
#### `min()` {.collection-method}

`min` 方法會回傳給定鍵的最小值：

    $min = collect([['foo' => 10], ['foo' => 20]])->min('foo');

    // 10

    $min = collect([1, 2, 3, 4, 5])->min();

    // 1

<a name="method-mode"></a>
#### `mode()` {.collection-method}

`mode` 方法會回傳給定鍵的[眾數](https://en.wikipedia.org/wiki/Mode_(statistics))值：

    $mode = collect([
        ['foo' => 10],
        ['foo' => 10],
        ['foo' => 20],
        ['foo' => 40]
    ])->mode('foo');

    // [10]

    $mode = collect([1, 1, 2, 4])->mode();

    // [1]

    $mode = collect([1, 1, 2, 2])->mode();

    // [1, 2]

<a name="method-multiply"></a>
#### `multiply()` {.collection-method}

`multiply` 方法會建立 Collection 中所有項目的指定數量副本：

```php
$users = collect([
    ['name' => 'User #1', 'email' => 'user1 @example.com'],
    ['name' => 'User #2', 'email' => 'user2 @example.com'],
])->multiply(3);

/*
    [
        ['name' => 'User #1', 'email' => 'user1 @example.com'],
        ['name' => 'User #2', 'email' => 'user2 @example.com'],
        ['name' => 'User #1', 'email' => 'user1 @example.com'],
        ['name' => 'User #2', 'email' => 'user2 @example.com'],
        ['name' => 'User #1', 'email' => 'user1 @example.com'],
        ['name' => 'User #2', 'email' => 'user2 @example.com'],
    ]
*/
```

<a name="method-nth"></a>
#### `nth()` {.collection-method}

`nth` 方法會建立一個由每 n 個元素組成的新 Collection：

    $collection = collect(['a', 'b', 'c', 'd', 'e', 'f']);

    $collection->nth(4);

    // ['a', 'e']

您可以選擇將起始偏移量作為第二個引數傳遞：

    $collection->nth(4, 1);

    // ['b', 'f']

<a name="method-only"></a>
#### `only()` {.collection-method}

`only` 方法會回傳 Collection 中具有指定鍵的項目：

    $collection = collect([
        'product_id' => 1,
        'name' => 'Desk',
        'price' => 100,
        'discount' => false
    ]);

    $filtered = $collection->only(['product_id', 'name']);

    $filtered->all();

    // ['product_id' => 1, 'name' => 'Desk']

`only` 的反向操作，請參閱 [except](#method-except) 方法。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-only) 時，此方法的行為會被修改。

<a name="method-pad"></a>
#### `pad()` {.collection-method}

`pad` 方法會用給定值填充陣列，直到陣列達到指定大小。此方法的行為類似於 [array_pad](https://secure.php.net/manual/en/function.array-pad.php) PHP 函式。

若要向左填充，您應該指定一個負數大小。如果給定大小的絕對值小於或等於陣列的長度，則不會進行填充：

    $collection = collect(['A', 'B', 'C']);

    $filtered = $collection->pad(5, 0);

    $filtered->all();

    // ['A', 'B', 'C', 0, 0]

    $filtered = $collection->pad(-5, 0);

    $filtered->all();

    // [0, 0, 'A', 'B', 'C']

<a name="method-partition"></a>
#### `partition()` {.collection-method}

`partition` 方法可以與 PHP 陣列解構結合使用，以將通過給定真值測試的元素與未通過的元素分開：

    $collection = collect([1, 2, 3, 4, 5, 6]);

    [$underThree, $equalOrAboveThree] = $collection->partition(function (int $i) {
        return $i < 3;
    });

    $underThree->all();

    // [1, 2]

    $equalOrAboveThree->all();

    // [3, 4, 5, 6]

<a name="method-percentage"></a>
#### `percentage()` {.collection-method}

`percentage` 方法可用於快速判斷 Collection 中通過給定真值測試的項目百分比：

```php
$collection = collect([1, 1, 2, 2, 2, 3]);

$percentage = $collection->percentage(fn ($value) => $value === 1);

// 33.33
```

預設情況下，百分比將四捨五入到小數點後兩位。但是，您可以透過向方法提供第二個引數來自訂此行為：

```php
$percentage = $collection->percentage(fn ($value) => $value === 1, precision: 3);

// 33.333
```

<a name="method-pipe"></a>
#### `pipe()` {.collection-method}

`pipe` 方法會將 Collection 傳遞給給定的閉包並回傳執行閉包的結果：

    $collection = collect([1, 2, 3]);

    $piped = $collection->pipe(function (Collection $collection) {
        return $collection->sum();
    });

    // 6

<a name="method-pipeinto"></a>
#### `pipeInto()` {.collection-method}

`pipeInto` 方法會建立給定類別的新實例，並將 Collection 傳遞給建構函式：

    class ResourceCollection
    {
        /**
         * Create a new ResourceCollection instance.
         */
        public function __construct(
            public Collection $collection,
        ) {}
    }

    $collection = collect([1, 2, 3]);

    $resource = $collection->pipeInto(ResourceCollection::class);

    $resource->collection->all();

    // [1, 2, 3]

<a name="method-pipethrough"></a>
#### `pipeThrough()` {.collection-method}

`pipeThrough` 方法會將 Collection 傳遞給給定的閉包陣列並回傳執行閉包的結果：

    use Illuminate\Support\Collection;

    $collection = collect([1, 2, 3]);

    $result = $collection->pipeThrough([
        function (Collection $collection) {
            return $collection->merge([4, 5]);
        },
        function (Collection $collection) {
            return $collection->sum();
        },
    ]);

    // 15

<a name="method-pluck"></a>
#### `pluck()` {.collection-method}

`pluck` 方法會檢索給定鍵的所有值：

    $collection = collect([
        ['product_id' => 'prod-100', 'name' => 'Desk'],
        ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]);

    $plucked = $collection->pluck('name');

    $plucked->all();

    // ['Desk', 'Chair']

您還可以指定您希望結果 Collection 如何鍵控：

    $plucked = $collection->pluck('name', 'product_id');

    $plucked->all();

    // ['prod-100' => 'Desk', 'prod-200' => 'Chair']

`pluck` 方法也支援使用「點」符號檢索巢狀值：

    $collection = collect([
        [
            'name' => 'Laracon',
            'speakers' => [
                'first_day' => ['Rosa', 'Judith'],
            ],
        ],
        [
            'name' => 'VueConf',
            'speakers' => [
                'first_day' => ['Abigail', 'Joey'],
            ],
        ],
    ]);

    $plucked = $collection->pluck('speakers.first_day');

    $plucked->all();

    // [['Rosa', 'Judith'], ['Abigail', 'Joey']]

如果存在重複的鍵，則最後一個匹配的元素將插入到被提取的 Collection 中：

    $collection = collect([
        ['brand' => 'Tesla',  'color' => 'red'],
        ['brand' => 'Pagani', 'color' => 'white'],
        ['brand' => 'Tesla',  'color' => 'black'],
        ['brand' => 'Pagani', 'color' => 'orange'],
    ]);

    $plucked = $collection->pluck('color', 'brand');

    $plucked->all();

    // ['Tesla' => 'black', 'Pagani' => 'orange']

<a name="method-pop"></a>
#### `pop()` {.collection-method}

`pop` 方法會從 Collection 中移除並回傳最後一個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->pop();

    // 5

    $collection->all();

    // [1, 2, 3, 4]

您可以將整數傳遞給 `pop` 方法，以從 Collection 的末尾移除並回傳多個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->pop(3);

    // collect([5, 4, 3])

    $collection->all();

    // [1, 2]

<a name="method-prepend"></a>
#### `prepend()` {.collection-method}

`prepend` 方法會將項目新增到 Collection 的開頭：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->prepend(0);

    $collection->all();

    // [0, 1, 2, 3, 4, 5]

您還可以傳遞第二個引數來指定預置項目的鍵：

    $collection = collect(['one' => 1, 'two' => 2]);

    $collection->prepend(0, 'zero');

    $collection->all();

    // ['zero' => 0, 'one' => 1, 'two' => 2]

<a name="method-pull"></a>
#### `pull()` {.collection-method}

`pull` 方法會根據其鍵從 Collection 中移除並回傳項目：

    $collection = collect(['product_id' => 'prod-100', 'name' => 'Desk']);

    $collection->pull('name');

    // 'Desk'

    $collection->all();

    // ['product_id' => 'prod-100']

<a name="method-push"></a>
#### `push()` {.collection-method}

`push` 方法會將項目附加到 Collection 的末尾：

    $collection = collect([1, 2, 3, 4]);

    $collection->push(5);

    $collection->all();

    // [1, 2, 3, 4, 5]

<a name="method-put"></a>
#### `put()` {.collection-method}

`put` 方法會設定 Collection 中的給定鍵和值：

    $collection = collect(['product_id' => 1, 'name' => 'Desk']);

    $collection->put('price', 100);

    $collection->all();

    // ['product_id' => 1, 'name' => 'Desk', 'price' => 100]

<a name="method-random"></a>
#### `random()` {.collection-method}

`random` 方法會從 Collection 中回傳一個隨機項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->random();

    // 4 - (隨機檢索)

您可以將整數傳遞給 `random` 以指定您希望隨機檢索多少個項目。當明確傳遞您希望接收的項目數量時，總是會回傳一個項目 Collection：

    $random = $collection->random(3);

    $random->all();

    // [2, 4, 5] - (隨機檢索)

如果 Collection 實例的項目少於請求的數量，`random` 方法將拋出 `InvalidArgumentException`。

`random` 方法也接受一個閉包，該閉包將接收當前的 Collection 實例：

    use Illuminate\Support\Collection;

    $random = $collection->random(fn (Collection $items) => min(10, count($items)));

    $random->all();

    // [1, 2, 3, 4, 5] - (隨機檢索)

<a name="method-range"></a>
#### `range()` {.collection-method}

`range` 方法會回傳一個包含指定範圍內整數的 Collection：

    $collection = collect()->range(3, 6);

    $collection->all();

    // [3, 4, 5, 6]

<a name="method-reduce"></a>
#### `reduce()` {.collection-method}

`reduce` 方法會將 Collection 歸約為單一值，將每次迭代的結果傳遞給後續迭代：

    $collection = collect([1, 2, 3]);

    $total = $collection->reduce(function (?int $carry, int $item) {
        return $carry + $item;
    });

    // 6

第一次迭代的 `$carry` 值為 `null`；但是，您可以透過向 `reduce` 傳遞第二個引數來指定其初始值：

    $collection->reduce(function (int $carry, int $item) {
        return $carry + $item;
    }, 4);

    // 10

`reduce` 方法還會將關聯 Collection 中的陣列鍵傳遞給給定的回呼：

    $collection = collect([
        'usd' => 1400,
        'gbp' => 1200,
        'eur' => 1000,
    ]);

    $ratio = [
        'usd' => 1,
        'gbp' => 1.37,
        'eur' => 1.22,
    ];

    $collection->reduce(function (int $carry, int $value, int $key) use ($ratio) {
        return $carry + ($value * $ratio[$key]);
    });

    // 4264

<a name="method-reduce-spread"></a>
#### `reduceSpread()` {.collection-method}

`reduceSpread` 方法會將 Collection 歸約為一個值陣列，將每次迭代的結果傳遞給後續迭代。此方法類似於 `reduce` 方法；但是，它可以接受多個初始值：

    [$creditsRemaining, $batch] = Image::where('status', 'unprocessed')
        ->get()
        ->reduceSpread(function (int $creditsRemaining, Collection $batch, Image $image) {
            if ($creditsRemaining >= $image->creditsRequired()) {
                $batch->push($image);

                $creditsRemaining -= $image->creditsRequired();
            }

            return [$creditsRemaining, $batch];
        }, $creditsAvailable, collect());

<a name="method-reject"></a>
#### `reject()` {.collection-method}

`reject` 方法會使用給定的閉包篩選 Collection。如果項目應該從結果 Collection 中移除，則閉包應該回傳 `true`：

    $collection = collect([1, 2, 3, 4]);

    $filtered = $collection->reject(function (int $value, int $key) {
        return $value > 2;
    });

    $filtered->all();

    // [1, 2]

`reject` 方法的反向操作，請參閱 [`filter`](#method-filter) 方法。

<a name="method-replace"></a>
#### `replace()` {.collection-method}

`replace` 方法的行為類似於 `merge`；但是，除了覆寫具有字串鍵的匹配項目之外，`replace` 方法還會覆寫 Collection 中具有匹配數字鍵的項目：

    $collection = collect(['Taylor', 'Abigail', 'James']);

    $replaced = $collection->replace([1 => 'Victoria', 3 => 'Finn']);

    $replaced->all();

    // ['Taylor', 'Victoria', 'James', 'Finn']

<a name="method-replacerecursive"></a>
#### `replaceRecursive()` {.collection-method}

此方法的行為類似於 `replace`，但它會遞迴進入陣列並將相同的替換過程應用於內部值：

    $collection = collect([
        'Taylor',
        'Abigail',
        [
            'James',
            'Victoria',
            'Finn'
        ]
    ]);

    $replaced = $collection->replaceRecursive([
        'Charlie',
        2 => [1 => 'King']
    ]);

    $replaced->all();

    // ['Charlie', 'Abigail', ['James', 'King', 'Finn']]

<a name="method-reverse"></a>
#### `reverse()` {.collection-method}

`reverse` 方法會反轉 Collection 項目的順序，同時保留原始鍵：

    $collection = collect(['a', 'b', 'c', 'd', 'e']);

    $reversed = $collection->reverse();

    $reversed->all();

    /*
        [
            4 => 'e',
            3 => 'd',
            2 => 'c',
            1 => 'b',
            0 => 'a',
        ]
    */

<a name="method-search"></a>
#### `search()` {.collection-method}

`search` 方法會在 Collection 中搜尋給定值，如果找到則回傳其鍵。如果找不到項目，則回傳 `false`：

    $collection = collect([2, 4, 6, 8]);

    $collection->search(4);

    // 1

搜尋是使用「寬鬆」比較完成的，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比較，請將 `true` 作為第二個引數傳遞給方法：

    collect([2, 4, 6, 8])->search('4', strict: true);

    // false

或者，您可以提供自己的閉包來搜尋第一個通過給定真值測試的項目：

    collect([2, 4, 6, 8])->search(function (int $item, int $key) {
        return $item > 5;
    });

    // 2

<a name="method-select"></a>
#### `select()` {.collection-method}

`select` 方法會從 Collection 中選取給定鍵，類似於 SQL `SELECT` 陳述式：

```php
$users = collect([
    ['name' => 'Taylor Otwell', 'role' => 'Developer', 'status' => 'active'],
    ['name' => 'Victoria Faith', 'role' => 'Researcher', 'status' => 'active'],
]);

$users->select(['name', 'role']);

/*
    [
        ['name' => 'Taylor Otwell', 'role' => 'Developer'],
        ['name' => 'Victoria Faith', 'role' => 'Researcher'],
    ],
*/
```

<a name="method-shift"></a>
#### `shift()` {.collection-method}

`shift` 方法會從 Collection 中移除並回傳第一個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->shift();

    // 1

    $collection->all();

    // [2, 3, 4, 5]

您可以將整數傳遞給 `shift` 方法，以從 Collection 的開頭移除並回傳多個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->shift(3);

    // collect([1, 2, 3])

    $collection->all();

    // [4, 5]

<a name="method-shuffle"></a>
#### `shuffle()` {.collection-method}

`shuffle` 方法會隨機打亂 Collection 中的項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $shuffled = $collection->shuffle();

    $shuffled->all();

    // [3, 2, 5, 1, 4] - (隨機產生)

<a name="method-skip"></a>
#### `skip()` {.collection-method}

`skip` 方法會回傳一個新的 Collection，其中從 Collection 的開頭移除了指定數量的元素：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $collection = $collection->skip(4);

    $collection->all();

    // [5, 6, 7, 8, 9, 10]

<a name="method-skipuntil"></a>
#### `skipUntil()` {.collection-method}

`skipUntil` 方法會跳過 Collection 中的項目，直到給定回呼回傳 `true`。一旦回呼回傳 `true`，Collection 中所有剩餘的項目將作為一個新的 Collection 回傳：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipUntil(function (int $item) {
        return $item >= 3;
    });

    $subset->all();

    // [3, 4]

您也可以將簡單值傳遞給 `skipUntil` 方法，以跳過所有項目直到找到給定值：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipUntil(3);

    $subset->all();

    // [3, 4]

> [!WARNING]
> 如果找不到給定值或回呼從未回傳 `true`，`skipUntil` 方法將回傳一個空 Collection。

<a name="method-skipwhile"></a>
#### `skipWhile()` {.collection-method}

`skipWhile` 方法會跳過 Collection 中的項目，直到給定回呼回傳 `false`。一旦回呼回傳 `false`，Collection 中所有剩餘的項目將作為一個新的 Collection 回傳：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipWhile(function (int $item) {
        return $item <= 3;
    });

    $subset->all();

    // [4]

> [!WARNING]
> 如果回呼從未回傳 `false`，`skipWhile` 方法將回傳一個空 Collection。

<a name="method-slice"></a>
#### `slice()` {.collection-method}

`slice` 方法會回傳從給定索引開始的 Collection 切片：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $slice = $collection->slice(4);

    $slice->all();

    // [5, 6, 7, 8, 9, 10]

如果您想限制回傳切片的大小，請將所需大小作為第二個引數傳遞給方法：

    $slice = $collection->slice(4, 2);

    $slice->all();

    // [5, 6]

回傳的切片預設會保留鍵。如果您不想保留原始鍵，可以使用 [`values`](#method-values) 方法重新索引它們。

<a name="method-sliding"></a>
#### `sliding()` {.collection-method}

`sliding` 方法會回傳一個新的 Collection，其中包含代表 Collection 中項目「滑動視窗」視圖的區塊：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunks = $collection->sliding(2);

    $chunks->toArray();

    // [[1, 2], [2, 3], [3, 4], [4, 5]]

這與 [`eachSpread`](#method-eachspread) 方法結合使用時特別有用：

    $transactions->sliding(2)->eachSpread(function (Collection $previous, Collection $current) {
        $current->total = $previous->total + $current->amount;
    });

您可以選擇傳遞第二個「步長」值，它決定了每個區塊第一個項目之間的距離：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunks = $collection->sliding(3, step: 2);

    $chunks->toArray();

    // [[1, 2, 3], [3, 4, 5]]

<a name="method-sole"></a>
#### `sole()` {.collection-method}

`sole` 方法會回傳 Collection 中第一個通過給定真值測試的元素，但前提是真值測試只匹配一個元素：

    collect([1, 2, 3, 4])->sole(function (int $value, int $key) {
        return $value === 2;
    });

    // 2

您也可以將鍵/值對傳遞給 `sole` 方法，這將回傳 Collection 中第一個匹配給定對的元素，但前提是只匹配一個元素：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->sole('product', 'Chair');

    // ['product' => 'Chair', 'price' => 100]

或者，您也可以不帶引數呼叫 `sole` 方法，以在 Collection 中只有一個元素時取得第一個元素：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
    ]);

    $collection->sole();

    // ['product' => 'Desk', 'price' => 200]

如果 Collection 中沒有應由 `sole` 方法回傳的元素，將拋出 `\Illuminate\Collections\ItemNotFoundException` 異常。如果有多個元素應回傳，將拋出 `\Illuminate\Collections\MultipleItemsFoundException`。

<a name="method-some"></a>
#### `some()` {.collection-method}

[`contains`](#method-contains) 方法的別名。

<a name="method-sort"></a>
#### `sort()` {.collection-method}

`sort` 方法會對 Collection 進行排序。排序後的 Collection 會保留原始陣列鍵，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續編號的索引：

    $collection = collect([5, 3, 1, 2, 4]);

    $sorted = $collection->sort();

    $sorted->values()->all();

    // [1, 2, 3, 4, 5]

如果您的排序需求更進階，您可以將回呼傳遞給 `sort`，其中包含您自己的演算法。請參閱 PHP 文件中關於 [`uasort`](https://secure.php.net/manual/en/function.uasort.php#refsect1-function.uasort-parameters) 的部分，這是 Collection 的 `sort` 方法內部使用的 PHP 函式。

> [!NOTE]
> 如果您需要對巢狀陣列或物件的 Collection 進行排序，請參閱 [`sortBy`](#method-sortby) 和 [`sortByDesc`](#method-sortbydesc) 方法。

<a name="method-sortby"></a>
#### `sortBy()` {.collection-method}

`sortBy` 方法會根據給定鍵對 Collection 進行排序。排序後的 Collection 會保留原始陣列鍵，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續編號的索引：

    $collection = collect([
        ['name' => 'Desk', 'price' => 200],
        ['name' => 'Chair', 'price' => 100],
        ['name' => 'Bookcase', 'price' => 150],
    ]);

    $sorted = $collection->sortBy('price');

    $sorted->values()->all();

    /*
        [
            ['name' => 'Chair', 'price' => 100],
            ['name' => 'Bookcase', 'price' => 150],
            ['name' => 'Desk', 'price' => 200],
        ]
    */

`sortBy` 方法接受 [排序旗標](https://www.php.net/manual/en/function.sort.php) 作為其第二個引數：

    $collection = collect([
        ['title' => 'Item 1'],
        ['title' => 'Item 12'],
        ['title' => 'Item 3'],
    ]);

    $sorted = $collection->sortBy('title', SORT_NATURAL);

    $sorted->values()->all();

    /*
        [
            ['title' => 'Item 1'],
            ['title' => 'Item 3'],
            ['title' => 'Item 12'],
        ]
    */

或者，您可以傳遞自己的閉包來決定如何排序 Collection 的值：

    $collection = collect([
        ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
        ['name' => 'Chair', 'colors' => ['Black']],
        ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
    ]);

    $sorted = $collection->sortBy(function (array $product, int $key) {
        return count($product['colors']);
    });

    $sorted->values()->all();

    /*
        [
            ['name' => 'Chair', 'colors' => ['Black']],
            ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
            ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
        ]
    */

如果您想根據多個屬性對 Collection 進行排序，您可以將排序操作陣列傳遞給 `sortBy` 方法。每個排序操作都應該是一個陣列，其中包含您希望排序的屬性以及所需排序的方向：

    $collection = collect([
        ['name' => 'Taylor Otwell', 'age' => 34],
        ['name' => 'Abigail Otwell', 'age' => 30],
        ['name' => 'Taylor Otwell', 'age' => 36],
        ['name' => 'Abigail Otwell', 'age' => 32],
    ]);

    $sorted = $collection->sortBy([
        ['name', 'asc'],
        ['age', 'desc'],
    ]);

    $sorted->values()->all();

    /*
        [
            ['name' => 'Abigail Otwell', 'age' => 32],
            ['name' => 'Abigail Otwell', 'age' => 30],
            ['name' => 'Taylor Otwell', 'age' => 36],
            ['name' => 'Taylor Otwell', 'age' => 34],
        ]
    */

當根據多個屬性對 Collection 進行排序時，您還可以提供定義每個排序操作的閉包：

    $collection = collect([
        ['name' => 'Taylor Otwell', 'age' => 34],
        ['name' => 'Abigail Otwell', 'age' => 30],
        ['name' => 'Taylor Otwell', 'age' => 36],
        ['name' => 'Abigail Otwell', 'age' => 32],
    ]);

    $sorted = $collection->sortBy([
        fn (array $a, array $b) => $a['name'] <=> $b['name'],
        fn (array $a, array $b) => $b['age'] <=> $a['age'],
    ]);

    $sorted->values()->all();

    /*
        [
            ['name' => 'Abigail Otwell', 'age' => 32],
            ['name' => 'Abigail Otwell', 'age' => 30],
            ['name' => 'Taylor Otwell', 'age' => 36],
            ['name' => 'Taylor Otwell', 'age' => 34],
        ]
    */

<a name="method-sortbydesc"></a>
#### `sortByDesc()` {.collection-method}

此方法與 [`sortBy`](#method-sortby) 方法具有相同的簽名，但會以相反的順序對 Collection 進行排序。

<a name="method-sortdesc"></a>
#### `sortDesc()` {.collection-method}

此方法會以與 [`sort`](#method-sort) 方法相反的順序對 Collection 進行排序：

    $collection = collect([5, 3, 1, 2, 4]);

    $sorted = $collection->sortDesc();

    $sorted->values()->all();

    // [5, 4, 3, 2, 1]

與 `sort` 不同，您不能將閉包傳遞給 `sortDesc`。相反，您應該使用 [`sort`](#method-sort) 方法並反轉您的比較。

<a name="method-sortkeys"></a>
#### `sortKeys()` {.collection-method}

`sortKeys` 方法會根據底層關聯陣列的鍵對 Collection 進行排序：

    $collection = collect([
        'id' => 22345,
        'first' => 'John',
        'last' => 'Doe',
    ]);

    $sorted = $collection->sortKeys();

    $sorted->all();

    /*
        [
            'first' => 'John',
            'id' => 22345,
            'last' => 'Doe',
        ]
    */

<a name="method-sortkeysdesc"></a>
#### `sortKeysDesc()` {.collection-method}

此方法與 [`sortKeys`](#method-sortkeys) 方法具有相同的簽名，但會以相反的順序對 Collection 進行排序。

<a name="method-sortkeysusing"></a>
#### `sortKeysUsing()` {.collection-method}

`sortKeysUsing` 方法會使用回呼根據底層關聯陣列的鍵對 Collection 進行排序：

    $collection = collect([
        'ID' => 22345,
        'first' => 'John',
        'last' => 'Doe',
    ]);

    $sorted = $collection->sortKeysUsing('strnatcasecmp');

    $sorted->all();

    /*
        [
            'first' => 'John',
            'ID' => 22345,
            'last' => 'Doe',
        ]
    */

回呼必須是一個比較函式，回傳一個小於、等於或大於零的整數。有關更多資訊，請參閱 PHP 文件中關於 [`uksort`](https://www.php.net/manual/en/function.uksort.php#refsect1-function.uksort-parameters) 的部分，這是 `sortKeysUsing` 方法內部使用的 PHP 函式。

<a name="method-splice"></a>
#### `splice()` {.collection-method}

`splice` 方法會移除並回傳從指定索引開始的項目切片：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2);

    $chunk->all();

    // [3, 4, 5]

    $collection->all();

    // [1, 2]

您可以傳遞第二個引數來限制結果 Collection 的大小：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2, 1);

    $chunk->all();

    // [3]

    $collection->all();

    // [1, 2, 4, 5]

此外，您可以傳遞第三個引數，其中包含新項目以替換從 Collection 中移除的項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2, 1, [10, 11]);

    $chunk->all();

    // [3]

    $collection->all();

    // [1, 2, 10, 11, 4, 5]

<a name="method-split"></a>
#### `split()` {.collection-method}

`split` 方法會將 Collection 分割成指定數量的組：

    $collection = collect([1, 2, 3, 4, 5]);

    $groups = $collection->split(3);

    $groups->all();

    // [[1, 2], [3, 4], [5]]

<a name="method-splitin"></a>
#### `splitIn()` {.collection-method}

`splitIn` 方法會將 Collection 分割成指定數量的組，在將剩餘部分分配給最後一個組之前，完全填充非終端組：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $groups = $collection->splitIn(3);

    $groups->all();

    // [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]

<a name="method-sum"></a>
#### `sum()` {.collection-method}

`sum` 方法會回傳 Collection 中所有項目的總和：

    collect([1, 2, 3, 4, 5])->sum();

    // 15

如果 Collection 包含巢狀陣列或物件，您應該傳遞一個鍵，該鍵將用於確定要加總的值：

    $collection = collect([
        ['name' => 'JavaScript: The Good Parts', 'pages' => 176],
        ['name' => 'JavaScript: The Definitive Guide', 'pages' => 1096],
    ]);

    $collection->sum('pages');

    // 1272

此外，您可以傳遞自己的閉包來確定 Collection 中要加總的值：

    $collection = collect([
        ['name' => 'Chair', 'colors' => ['Black']],
        ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
        ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
    ]);

    $collection->sum(function (array $product) {
        return count($product['colors']);
    });

    // 6

<a name="method-take"></a>
#### `take()` {.collection-method}

`take` 方法會回傳一個包含指定數量項目數的新 Collection：

    $collection = collect([0, 1, 2, 3, 4, 5]);

    $chunk = $collection->take(3);

    $chunk->all();

    // [0, 1, 2]

您還可以傳遞一個負整數，以從 Collection 的末尾取得指定數量的項目：

    $collection = collect([0, 1, 2, 3, 4, 5]);

    $chunk = $collection->take(-2);

    $chunk->all();

    // [4, 5]

<a name="method-takeuntil"></a>
#### `takeUntil()` {.collection-method}

`takeUntil` 方法會回傳 Collection 中的項目，直到給定回呼回傳 `true`：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeUntil(function (int $item) {
        return $item >= 3;
    });

    $subset->all();

    // [1, 2]

您也可以將簡單值傳遞給 `takeUntil` 方法，以取得項目直到找到給定值：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeUntil(3);

    $subset->all();

    // [1, 2]

> [!WARNING]
> 如果找不到給定值或回呼從未回傳 `true`，`takeUntil` 方法將回傳 Collection 中的所有項目。

<a name="method-takewhile"></a>
#### `takeWhile()` {.collection-method}

`takeWhile` 方法會回傳 Collection 中的項目，直到給定回呼回傳 `false`：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeWhile(function (int $item) {
        return $item < 3;
    });

    $subset->all();

    // [1, 2]

> [!WARNING]
> 如果回呼從未回傳 `false`，`takeWhile` 方法將回傳 Collection 中的所有項目。

<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 方法會將 Collection 傳遞給給定的回呼，讓您可以在特定點「輕觸」Collection 並對項目執行某些操作，同時不影響 Collection 本身。然後 `tap` 方法會回傳 Collection：

    collect([2, 4, 3, 1, 5])
        ->sort()
        ->tap(function (Collection $collection) {
            Log::debug('Values after sorting', $collection->values()->all());
        })
        ->shift();

    // 1

<a name="method-times"></a>
#### `times()` {.collection-method}

靜態 `times` 方法會透過呼叫給定閉包指定次數來建立一個新的 Collection：

    $collection = Collection::times(10, function (int $number) {
        return $number * 9;
    });

    $collection->all();

    // [9, 18, 27, 36, 45, 54, 63, 72, 81, 90]

<a name="method-toarray"></a>
#### `toArray()` {.collection-method}

`toArray` 方法會將 Collection 轉換為純 PHP `array`。如果 Collection 的值是 [Eloquent](/docs/{{version}}/eloquent) 模型，則模型也將轉換為陣列：

    $collection = collect(['name' => 'Desk', 'price' => 200]);

    $collection->toArray();

    /*
        [
            ['name' => 'Desk', 'price' => 200],
        ]
    */

> [!WARNING]
> `toArray` 還會將 Collection 中所有是 `Arrayable` 實例的巢狀物件轉換為陣列。如果您想取得 Collection 底層的原始陣列，請改用 [`all`](#method-all) 方法。

<a name="method-tojson"></a>
#### `toJson()` {.collection-method}

`toJson` 方法會將 Collection 轉換為 JSON 序列化字串：

    $collection = collect(['name' => 'Desk', 'price' => 200]);

    $collection->toJson();

    // '{"name":"Desk", "price":200}'

<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 方法會迭代 Collection 並使用 Collection 中的每個項目呼叫給定回呼。Collection 中的項目將被回呼回傳的值替換：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->transform(function (int $item, int $key) {
        return $item * 2;
    });

    $collection->all();

    // [2, 4, 6, 8, 10]

> [!WARNING]
> 與大多數其他 Collection 方法不同，`transform` 會修改 Collection 本身。如果您希望建立一個新的 Collection，請使用 [`map`](#method-map) 方法。

<a name="method-undot"></a>
#### `undot()` {.collection-method}

`undot` 方法會將使用「點」符號的單維 Collection 展開為多維 Collection：

    $person = collect([
        'name.first_name' => 'Marie',
        'name.last_name' => 'Valentine',
        'address.line_1' => '2992 Eagle Drive',
        'address.line_2' => '',
        'address.suburb' => 'Detroit',
        'address.state' => 'MI',
        'address.postcode' => '48219'
    ]);

    $person = $person->undot();

    $person->toArray();

    /*
        [
            "name" => [
                "first_name" => "Marie",
                "last_name" => "Valentine",
            ],
            "address" => [
                "line_1" => "2992 Eagle Drive",
                "line_2" => "",
                "suburb" => "Detroit",
                "state" => "MI",
                "postcode" => "48219",
            ],
        ]
    */

<a name="method-union"></a>
#### `union()` {.collection-method}

`union` 方法會將給定陣列新增到 Collection。如果給定陣列包含原始 Collection 中已有的鍵，則原始 Collection 的值將優先：

    $collection = collect([1 => ['a'], 2 => ['b']]);

    $union = $collection->union([3 => ['c'], 1 => ['d']]);

    $union->all();

    // [1 => ['a'], 2 => ['b'], 3 => ['c']]

<a name="method-unique"></a>
#### `unique()` {.collection-method}

`unique` 方法會回傳 Collection 中所有唯一項目。回傳的 Collection 會保留原始陣列鍵，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續編號的索引：

    $collection = collect([1, 1, 2, 2, 3, 4, 2]);

    $unique = $collection->unique();

    $unique->values()->all();

    // [1, 2, 3, 4]

處理巢狀陣列或物件時，您可以指定用於確定唯一性的鍵：

    $collection = collect([
        ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
        ['name' => 'iPhone 5', 'brand' => 'Apple', 'type' => 'phone'],
        ['name' => 'Apple Watch', 'brand' => 'Apple', 'type' => 'watch'],
        ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
        ['name' => 'Galaxy Gear', 'brand' => 'Samsung', 'type' => 'watch'],
    ]);

    $unique = $collection->unique('brand');

    $unique->values()->all();

    /*
        [
            ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
            ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
        ]
    */

最後，您還可以將自己的閉包傳遞給 `unique` 方法，以指定哪個值應該決定項目的唯一性：

    $unique = $collection->unique(function (array $item) {
        return $item['brand'].$item['type'];
    });

    $unique->values()->all();

    /*
        [
            ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
            ['name' => 'Apple Watch', 'brand' => 'Apple', 'type' => 'watch'],
            ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
            ['name' => 'Galaxy Gear', 'brand' => 'Samsung', 'type' => 'watch'],
        ]
    */

`unique` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。使用 [`uniqueStrict`](#method-uniquestrict) 方法以使用「嚴格」比較進行篩選。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-unique) 時，此方法的行為會被修改。

<a name="method-uniquestrict"></a>
#### `uniqueStrict()` {.collection-method}

此方法與 [`unique`](#method-unique) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

<a name="method-unless"></a>
#### `unless()` {.collection-method}

`unless` 方法將執行給定回呼，除非傳遞給方法的第一個引數評估為 `true`：

    $collection = collect([1, 2, 3]);

    $collection->unless(true, function (Collection $collection) {
        return $collection->push(4);
    });

    $collection->unless(false, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

可以將第二個回呼傳遞給 `unless` 方法。當傳遞給 `unless` 方法的第一個引數評估為 `true` 時，將執行第二個回呼：

    $collection = collect([1, 2, 3]);

    $collection->unless(true, function (Collection $collection) {
        return $collection->push(4);
    }, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

`unless` 的反向操作，請參閱 [`when`](#method-when) 方法。

<a name="method-unlessempty"></a>
#### `unlessEmpty()` {.collection-method}

[`whenNotEmpty`](#method-whennotempty) 方法的別名。

<a name="method-unlessnotempty"></a>
#### `unlessNotEmpty()` {.collection-method}

[`whenEmpty`](#method-whenempty) 方法的別名。

<a name="method-unwrap"></a>
#### `unwrap()` {.collection-method}

靜態 `unwrap` 方法會在適用時從給定值回傳 Collection 的底層項目：

    Collection::unwrap(collect('John Doe'));

    // ['John Doe']

    Collection::unwrap(['John Doe']);

    // ['John Doe']

    Collection::unwrap('John Doe');

    // 'John Doe'

<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 方法會從 Collection 的第一個元素中檢索給定值：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Speaker', 'price' => 400],
    ]);

    $value = $collection->value('price');

    // 200

<a name="method-values"></a>
#### `values()` {.collection-method}

`values` 方法會回傳一個新的 Collection，其中鍵已重設為連續整數：

    $collection = collect([
        10 => ['product' => 'Desk', 'price' => 200],
        11 => ['product' => 'Desk', 'price' => 200],
    ]);

    $values = $collection->values();

    $values->all();

    /*
        [
            0 => ['product' => 'Desk', 'price' => 200],
            1 => ['product' => 'Desk', 'price' => 200],
        ]
    */

<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 方法將在傳遞給方法的第一個引數評估為 `true` 時執行給定回呼。Collection 實例和傳遞給 `when` 方法的第一個引數將提供給閉包：

    $collection = collect([1, 2, 3]);

    $collection->when(true, function (Collection $collection, int $value) {
        return $collection->push(4);
    });

    $collection->when(false, function (Collection $collection, int $value) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 4]

可以將第二個回呼傳遞給 `when` 方法。當傳遞給 `when` 方法的第一個引數評估為 `false` 時，將執行第二個回呼：

    $collection = collect([1, 2, 3]);

    $collection->when(false, function (Collection $collection, int $value) {
        return $collection->push(4);
    }, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

`when` 的反向操作，請參閱 [`unless`](#method-unless) 方法。

<a name="method-whenempty"></a>
#### `whenEmpty()` {.collection-method}

`whenEmpty` 方法將在 Collection 為空時執行給定回呼：

    $collection = collect(['Michael', 'Tom']);

    $collection->whenEmpty(function (Collection $collection) {
        return $collection->push('Adam');
    });

    $collection->all();

    // ['Michael', 'Tom']


    $collection = collect();

    $collection->whenEmpty(function (Collection $collection) {
        return $collection->push('Adam');
    });

    $collection->all();

    // ['Adam']

可以將第二個閉包傳遞給 `whenEmpty` 方法，該閉包將在 Collection 不為空時執行：

    $collection = collect(['Michael', 'Tom']);

    $collection->whenEmpty(function (Collection $collection) {
        return $collection->push('Adam');
    }, function (Collection $collection) {
        return $collection->push('Taylor');
    });

    $collection->all();

    // ['Michael', 'Tom', 'Taylor']

`whenEmpty` 的反向操作，請參閱 [`whenNotEmpty`](#method-whennotempty) 方法。

<a name="method-whennotempty"></a>
#### `whenNotEmpty()` {.collection-method}

`whenNotEmpty` 方法將在 Collection 不為空時執行給定回呼：

    $collection = collect(['michael', 'tom']);

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    });

    $collection->all();

    // ['michael', 'tom', 'adam']


    $collection = collect();

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    });

    $collection->all();

    // []

可以將第二個閉包傳遞給 `whenNotEmpty` 方法，該閉包將在 Collection 為空時執行：

    $collection = collect();

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    }, function (Collection $collection) {
        return $collection->push('taylor');
    });

    $collection->all();

    // ['taylor']

`whenNotEmpty` 的反向操作，請參閱 [`whenEmpty`](#method-whenempty) 方法。

<a name="method-where"></a>
#### `where()` {.collection-method}

`where` 方法會根據給定鍵/值對篩選 Collection：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Door', 'price' => 100],
    ]);

    $filtered = $collection->where('price', 100);

    $filtered->all();

    /*
        [
            ['product' => 'Chair', 'price' => 100],
            ['product' => 'Door', 'price' => 100],
        ]
    */

`where` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。使用 [`whereStrict`](#method-wherestrict) 方法以使用「嚴格」比較進行篩選。

您可以選擇將比較運算子作為第二個參數傳遞。支援的運算子有：'===', '!==', '!=', '==', '=', '<>', '>', '<', '>=', 和 '<='：

    $collection = collect([
        ['name' => 'Jim', 'deleted_at' => '2019-01-01 00:00:00'],
        ['name' => 'Sally', 'deleted_at' => '2019-01-02 00:00:00'],
        ['name' => 'Sue', 'deleted_at' => null],
    ]);

    $filtered = $collection->where('deleted_at', '!=', null);

    $filtered->all();

    /*
        [
            ['name' => 'Jim', 'deleted_at' => '2019-01-01 00:00:00'],
            ['name' => 'Sally', 'deleted_at' => '2019-01-02 00:00:00'],
        ]
    */

<a name="method-wherestrict"></a>
#### `whereStrict()` {.collection-method}

此方法與 [`where`](#method-where) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

<a name="method-wherebetween"></a>
#### `whereBetween()` {.collection-method}

`whereBetween` 方法會透過判斷指定項目值是否在給定範圍內來篩選 Collection：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 80],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Pencil', 'price' => 30],
        ['product' => 'Door', 'price' => 100],
    ]);

    $filtered = $collection->whereBetween('price', [100, 200]);

    $filtered->all();

    /*
        [
            ['product' => 'Desk', 'price' => 200],
            ['product' => 'Bookcase', 'price' => 150],
            ['product' => 'Door', 'price' => 100],
        ]
    */

<a name="method-wherein"></a>
#### `whereIn()` {.collection-method}

`whereIn` 方法會從 Collection 中移除不具有包含在給定陣列中的指定項目值的元素：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Door', 'price' => 100],
    ]);

    $filtered = $collection->whereIn('price', [150, 200]);

    $filtered->all();

    /*
        [
            ['product' => 'Desk', 'price' => 200],
            ['product' => 'Bookcase', 'price' => 150],
        ]
    */

`whereIn` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。使用 [`whereInStrict`](#method-whereinstrict) 方法以使用「嚴格」比較進行篩選。

<a name="method-whereinstrict"></a>
#### `whereInStrict()` {.collection-method}

此方法與 [`whereIn`](#method-wherein) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

<a name="method-whereinstanceof"></a>
#### `whereInstanceOf()` {.collection-method}

`whereInstanceOf` 方法會根據給定類別類型篩選 Collection：

    use App\Models\User;
    use App\Models\Post;

    $collection = collect([
        new User,
        new User,
        new Post,
    ]);

    $filtered = $collection->whereInstanceOf(User::class);

    $filtered->all();

    // [App\Models\User, App\Models\User]

<a name="method-wherenotbetween"></a>
#### `whereNotBetween()` {.collection-method}

`whereNotBetween` 方法會透過判斷指定項目值是否在給定範圍之外來篩選 Collection：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 80],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Pencil', 'price' => 30],
        ['product' => 'Door', 'price' => 100],
    ]);

    $filtered = $collection->whereNotBetween('price', [100, 200]);

    $filtered->all();

    /*
        [
            ['product' => 'Chair', 'price' => 80],
            ['product' => 'Pencil', 'price' => 30],
        ]
    */

<a name="method-wherenotin"></a>
#### `whereNotIn()` {.collection-method}

`whereNotIn` 方法會從 Collection 中移除具有包含在給定陣列中的指定項目值的元素：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Door', 'price' => 100],
    ]);

    $filtered = $collection->whereNotIn('price', [150, 200]);

    $filtered->all();

    /*
        [
            ['product' => 'Chair', 'price' => 100],
            ['product' => 'Door', 'price' => 100],
        ]
    */

`whereNotIn` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與相同值的整數相等。使用 [`whereNotInStrict`](#method-wherenotinstrict) 方法以使用「嚴格」比較進行篩選。

<a name="method-wherenotinstrict"></a>
#### `whereNotInStrict()` {.collection-method}

此方法與 [`whereNotIn`](#method-wherenotin) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。

<a name="method-wherenotnull"></a>
#### `whereNotNull()` {.collection-method}

`whereNotNull` 方法會從 Collection 中回傳給定鍵不為 `null` 的項目：

    $collection = collect([
        ['name' => 'Desk'],
        ['name' => null],
        ['name' => 'Bookcase'],
    ]);

    $filtered = $collection->whereNotNull('name');

    $filtered->all();

    /*
        [
            ['name' => 'Desk'],
            ['name' => 'Bookcase'],
        ]
    */

<a name="method-wherenull"></a>
#### `whereNull()` {.collection-method}

`whereNull` 方法會從 Collection 中回傳給定鍵為 `null` 的項目：

    $collection = collect([
        ['name' => 'Desk'],
        ['name' => null],
        ['name' => 'Bookcase'],
    ]);

    $filtered = $collection->whereNull('name');

    $filtered->all();

    /*
        [
            ['name' => null],
        ]
    */

<a name="method-wrap"></a>
#### `wrap()` {.collection-method}

靜態 `wrap` 方法會在適用時將給定值包裝在 Collection 中：

    use Illuminate\Support\Collection;

    $collection = Collection::wrap('John Doe');

    $collection->all();

    // ['John Doe']

    $collection = Collection::wrap(['John Doe']);

    $collection->all();

    // ['John Doe']

    $collection = Collection::wrap(collect('John Doe'));

    $collection->all();

    // ['John Doe']

<a name="method-zip"></a>
#### `zip()` {.collection-method}

`zip` 方法會將給定陣列的值與原始 Collection 的值在對應索引處合併：

    $collection = collect(['Chair', 'Desk']);

    $zipped = $collection->zip([100, 200]);

    $zipped->all();

    // [['Chair', 100], ['Desk', 200]]

<a name="higher-order-messages"></a>
## 高階訊息

Collection 還支援「高階訊息」，這是對 Collection 執行常見操作的捷徑。提供高階訊息的 Collection 方法有：[`average`](#method-average)、[`avg`](#method-avg)、[`contains`](#method-contains)、[`each`](#method-each)、[`every`](#method-every)、[`filter`](#method-filter)、[`first`](#method-first)、[`flatMap`](#method-flatmap)、[`groupBy`](#method-groupby)、[`keyBy`](#method-keyby)、[`map`](#method-map)、[`max`](#method-max)、[`min`](#method-min)、[`partition`](#method-partition)、[`reject`](#method-reject)、[`skipUntil`](#method-skipuntil)、[`skipWhile`](#method-skipwhile)、[`some`](#method-some)、[`sortBy`](#method-sortby)、[`sortByDesc`](#method-sortbydesc)、[`sum`](#method-sum)、[`takeUntil`](#method-takeuntil)、[`takeWhile`](#method-takewhile) 和 [`unique`](#method-unique)。

每個高階訊息都可以作為 Collection 實例上的動態屬性存取。例如，讓我們使用 `each` 高階訊息來呼叫 Collection 中每個物件的方法：

    use App\Models\User;

    $users = User::where('votes', '>', 500)->get();

    $users->each->markAsVip();

同樣地，我們可以使用 `sum` 高階訊息來收集使用者 Collection 的總「votes」數：

    $users = User::where('group', 'Development')->get();

    return $users->sum->votes;

<a name="lazy-collections"></a>
## 惰性集合 (Lazy Collections)

<a name="lazy-collection-introduction"></a>
### 簡介

> [!WARNING]
> 在深入了解 Laravel 的惰性 Collection 之前，請花一些時間熟悉 [PHP 生成器](https://www.php.net/manual/en/language.generators.overview.php)。

為了補充已經強大的 `Collection` 類別，`LazyCollection` 類別利用 PHP 的 [生成器](https://www.php.net/manual/en/language.generators.overview.php) 讓您能夠處理非常大的資料集，同時保持記憶體使用量低。

例如，假設您的應用程式需要處理一個多 GB 的日誌檔，同時利用 Laravel 的 Collection 方法來解析日誌。惰性 Collection 可以用於一次只將檔案的一小部分保留在記憶體中，而不是一次將整個檔案讀取到記憶體中：

    use App\Models\LogEntry;
    use Illuminate\Support\LazyCollection;

    LazyCollection::make(function () {
        $handle = fopen('log.txt', 'r');

        while (($line = fgets($handle)) !== false) {
            yield $line;
        }
    })->chunk(4)->map(function (array $lines) {
        return LogEntry::fromLines($lines);
    })->each(function (LogEntry $logEntry) {
        // Process the log entry...
    });

或者，假設您需要迭代 10,000 個 Eloquent 模型。使用傳統的 Laravel Collection 時，所有 10,000 個 Eloquent 模型必須同時載入到記憶體中：

    use App\Models\User;

    $users = User::all()->filter(function (User $user) {
        return $user->id > 500;
    });

然而，查詢建構器的 `cursor` 方法會回傳一個 `LazyCollection` 實例。這讓您仍然只對資料庫執行單一查詢，但一次只將一個 Eloquent 模型載入到記憶體中。在此範例中，`filter` 回呼直到我們實際單獨迭代每個使用者時才會執行，從而大幅減少記憶體使用量：

    use App\Models\User;

    $users = User::cursor()->filter(function (User $user) {
        return $user->id > 500;
    });

    foreach ($users as $user) {
        echo $user->id;
    }

<a name="creating-lazy-collections"></a>
### 建立惰性集合

若要建立惰性 Collection 實例，您應該將 PHP 生成器函式傳遞給 Collection 的 `make` 方法：

    use Illuminate\Support\LazyCollection;

    LazyCollection::make(function () {
        $handle = fopen('log.txt', 'r');

        while (($line = fgets($handle)) !== false) {
            yield $line;
        }
    });

<a name="the-enumerable-contract"></a>
### Enumerable 契約

`Collection` 類別中幾乎所有可用的方法也都在 `LazyCollection` 類別中可用。這兩個類別都實作了 `Illuminate\Support\Enumerable` 契約，該契約定義了以下方法：

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

<div class="collection-method-list" markdown="1">

[all](#method-all)
[average](#method-average)
[avg](#method-avg)
[chunk](#method-chunk)
[chunkWhile](#method-chunkwhile)
[collapse](#method-collapse)
[collect](#method-collect)
[combine](#method-combine)
[concat](#method-concat)
[contains](#method-contains)
[containsStrict](#method-containsstrict)
[count](#method-count)
[countBy](#method-countBy)
[crossJoin](#method-crossjoin)
[dd](#method-dd)
[diff](#method-diff)
[diffAssoc](#method-diffassoc)
[diffKeys](#method-diffkeys)
[dump](#method-dump)
[duplicates](#method-duplicates)
[duplicatesStrict](#method-duplicatesstrict)
[each](#method-each)
[eachSpread](#method-eachspread)
[every](#method-every)
[except](#method-except)
[filter](#method-filter)
[first](#method-first)
[firstOrFail](#method-first-or-fail)
[firstWhere](#method-first-where)
[flatMap](#method-flatmap)
[flatten](#method-flatten)
[flip](#method-flip)
[forPage](#method-forpage)
[get](#method-get)
[groupBy](#method-groupby)
[has](#method-has)
[implode](#method-implode)
[intersect](#method-intersect)
[intersectAssoc](#method-intersectAssoc)
[intersectByKeys](#method-intersectbykeys)
[isEmpty](#method-isempty)
[isNotEmpty](#method-isnotempty)
[join](#method-join)
[keyBy](#method-keyby)
[keys](#method-keys)
[last](#method-last)
[macro](#method-macro)
[make](#method-make)
[map](#method-map)
[mapInto](#method-mapinto)
[mapSpread](#method-mapspread)
[mapToGroups](#method-maptogroups)
[mapWithKeys](#method-mapwithkeys)
[max](#method-max)
[median](#method-median)
[merge](#method-merge)
[mergeRecursive](#method-mergerecursive)
[min](#method-min)
[mode](#method-mode)
[nth](#method-nth)
[only](#method-only)
[pad](#method-pad)
[partition](#method-partition)
[pipe](#method-pipe)
[pluck](#method-pluck)
[random](#method-random)
[reduce](#method-reduce)
[reject](#method-reject)
[replace](#method-replace)
[replaceRecursive](#method-replacerecursive)
[reverse](#method-reverse)
[search](#method-search)
[shuffle](#method-shuffle)
[skip](#method-skip)
[slice](#method-slice)
[sole](#method-sole)
[some](#method-some)
[sort](#method-sort)
[sortBy](#method-sortby)
[sortByDesc](#method-sortbydesc)
[sortKeys](#method-sortkeys)
[sortKeysDesc](#method-sortkeysdesc)
[split](#method-split)
[sum](#method-sum)
[take](#method-take)
[tap](#method-tap)
[times](#method-times)
[toArray](#method-toarray)
[toJson](#method-tojson)
[union](#method-union)
[unique](#method-unique)
[uniqueStrict](#method-uniquestrict)
[unless](#method-unless)
[unlessEmpty](#method-unlessempty)
[unlessNotEmpty](#method-unlessnotempty)
[unwrap](#method-unwrap)
[values](#method-values)
[when](#method-when)
[whenEmpty](#method-whenempty)
[whenNotEmpty](#method-whennotempty)
[where](#method-where)
[whereStrict](#method-wherestrict)
[whereBetween](#method-wherebetween)
[whereIn](#method-wherein)
[whereInStrict](#method-whereinstrict)
[whereInstanceOf](#method-whereinstanceof)
[whereNotBetween](#method-wherenotbetween)
[whereNotIn](#method-wherenotin)
[whereNotInStrict](#method-wherenotinstrict)
[wrap](#method-wrap)
[zip](#method-zip)

</div>

> [!WARNING]
> 變更 Collection 的方法（例如 `shift`、`pop`、`prepend` 等）在 `LazyCollection` 類別中**不可用**。

<a name="lazy-collection-methods"></a>
### 惰性集合方法

除了 `Enumerable` 契約中定義的方法之外，`LazyCollection` 類別還包含以下方法：

<a name="method-takeUntilTimeout"></a>
#### `takeUntilTimeout()` {.collection-method}

`takeUntilTimeout` 方法會回傳一個新的惰性 Collection，該 Collection 將枚舉值直到指定時間。在此時間之後，Collection 將停止枚舉：

    $lazyCollection = LazyCollection::times(INF)
        ->takeUntilTimeout(now()->addMinute());

    $lazyCollection->each(function (int $number) {
        dump($number);

        sleep(1);
    });

    // 1
    // 2
    // ...
    // 58
    // 59

為了說明此方法的用法，假設一個應用程式使用 cursor 從資料庫提交發票。您可以定義一個每 15 分鐘執行一次的 [排程任務](/docs/{{version}}/scheduling)，並且只處理最多 14 分鐘的發票：

    use App\Models\Invoice;
    use Illuminate\Support\Carbon;

    Invoice::pending()->cursor()
        ->takeUntilTimeout(
            Carbon::createFromTimestamp(LARAVEL_START)->add(14, 'minutes')
        )
        ->each(fn (Invoice $invoice) => $invoice->submit());

<a name="method-tapEach"></a>
#### `tapEach()` {.collection-method}

雖然 `each` 方法會立即為 Collection 中的每個項目呼叫給定回呼，但 `tapEach` 方法只會在項目一個接一個地從列表中取出時呼叫給定回呼：

    // Nothing has been dumped so far...
    $lazyCollection = LazyCollection::times(INF)->tapEach(function (int $value) {
        dump($value);
    });

    // Three items are dumped...
    $array = $lazyCollection->take(3)->all();

    // 1
    // 2
    // 3

<a name="method-throttle"></a>
#### `throttle()` {.collection-method}

`throttle` 方法將限制惰性 Collection，使得每個值在指定秒數後回傳。此方法對於您可能與限制傳入請求速率的外部 API 互動的情況特別有用：

```php
use App\Models\User;

User::where('vip', true)
    ->cursor()
    ->throttle(seconds: 1)
    ->each(function (User $user) {
        // Call external API...
    });
```

<a name="method-remember"></a>
#### `remember()` {.collection-method}

`remember` 方法會回傳一個新的惰性 Collection，該 Collection 將記住已枚舉的任何值，並且在後續 Collection 枚舉時不會再次檢索它們：

    // No query has been executed yet...
    $users = User::cursor()->remember();

    // The query is executed...
    // The first 5 users are hydrated from the database...
    $users->take(5)->all();

    // First 5 users come from the collection's cache...
    // The rest are hydrated from the database...
    $users->take(20)->all();

