# Eloquent：修改器與轉換

- [介紹](#introduction)
- [取用器與修改器](#accessors-and-mutators)
    - [定義取用器](#defining-an-accessor)
    - [定義修改器](#defining-a-mutator)
- [屬性轉換](#attribute-casting)
    - [陣列與 JSON 轉換](#array-and-json-casting)
    - [日期轉換](#date-casting)
    - [Enum 轉換](#enum-casting)
    - [加密轉換](#encrypted-casting)
    - [查詢時轉換](#query-time-casting)
- [自訂轉換](#custom-casts)
    - [值物件轉換](#value-object-casting)
    - [陣列 / JSON 序列化](#array-json-serialization)
    - [僅傳入轉換](#inbound-casting)
    - [轉換參數](#cast-parameters)
    - [比較轉換值](#comparing-cast-values)
    - [可轉換物件 (Castable)](#castables)

<a name="introduction"></a>
## 介紹

取用器 (Accessor)、修改器 (Mutator)、以及屬性轉換 (Attribute Casting) 可讓你在 Model 實體上取用或設定 Eloquent 屬性值時對其進行轉換。舉例來說，我們可能會想在資料庫內儲存某個值時使用 [Laravel 加密器](/docs/{{version}}/encryption) 來加密，然後在 Eloquent Model 上取用該屬性時自動解密。或者，我們也可能想在透過 Eloquent Model 取用時，將資料庫內儲存的 JSON 字串轉換為陣列。

<a name="accessors-and-mutators"></a>
## 取用器與修改器

<a name="defining-an-accessor"></a>
### 定義取用器

取用器 (Accessor) 會在 Eloquent 屬性被取用時對其值進行轉換。若要定義取用器，請在 Model 上建立一個 protected 方法來代表要取用的屬性。若適用，該方法的名稱應對應至底層 Model 屬性 / 資料庫欄位的「駝峰式命名 (camel case)」。

在本範例中，我們要為 `first_name` 屬性定義一個取用器。當嘗試取用 `first_name` 屬性的值時，Eloquent 會自動呼叫該取用器。所有屬性取用器 / 修改器方法都必須宣告 `Illuminate\Database\Eloquent\Casts\Attribute` 的回傳型別提示：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the user's first name.
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
        );
    }
}
```

所有取用器方法都會回傳一個 `Attribute` 實體，該實體定義了該屬性將如何被取用，並可選擇性地定義其如何被修改。在本範例中，我們只定義了該屬性如何被取用。為此，我們為 `Attribute` 類別的建構函式提供了 `get` 引數。

如你所見，欄位的原始值會被傳給取用器，讓你能操作並回傳該值。若要取用該取用器的值，只要在 Model 實體上取用 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$firstName = $user->first_name;
```

> [!NOTE]
> 若想將這些計算出來的值加到 Model 的陣列 / JSON 表示中，[則需要將其附加 (Append)](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。

<a name="building-value-objects-from-multiple-attributes"></a>
#### 從多個屬性建立值物件

有時候，你的取用器可能需要將多個 Model 屬性轉換為單一的「值物件 (Value Object)」。為此，你的 `get` 閉包可接受第二個引數 `$attributes`，該引數會被自動提供給閉包，其中會包含一個包含 Model 目前所有屬性的陣列：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    );
}
```

<a name="accessor-caching"></a>
#### 取用器快取

當從取用器回傳值物件時，對該值物件所做的任何變更都會在 Model 儲存前自動同步回 Model。這是可行的，因為 Eloquent 會保留取用器回傳的實體，因此每次叫用取用器時都能回傳相同的實體：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Line 1 Value';
$user->address->lineTwo = 'Updated Address Line 2 Value';

$user->save();
```

不過，有時候你可能會想為字串與布林值等原始型別值啟用快取，特別是在這些值的計算量很大的時候。為此，可在定義取用器時叫用 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

若想停用屬性的物件快取行為，可在定義屬性時叫用 `withoutObjectCaching` 方法：

```php
/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    )->withoutObjectCaching();
}
```

<a name="defining-a-mutator"></a>
### 定義修改器

修改器 (Mutator) 會在 Eloquent 屬性被設定時對其值進行轉換。若要定義修改器，可在定義屬性時提供 `set` 引數。我們來為 `first_name` 屬性定義一個修改器。當我們嘗試在 Model 上設定 `first_name` 屬性的值時，這個修改器會被自動呼叫：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Interact with the user's first name.
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }
}
```

修改器的閉包會收到要被設定在屬性上的值，讓你能操作該值並回傳操作後的值。若要使用我們的修改器，只需要在 Eloquent Model 上設定 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$user->first_name = 'Sally';
```

在本範例中，`set` 回呼會以 `Sally` 這個值被呼叫。接著，修改器會對該名稱套用 `strtolower` 函式，並將結果設定在 Model 內部的 `$attributes` 陣列中。

<a name="mutating-multiple-attributes"></a>
#### 修改多個屬性

有時候，你的修改器可能需要在底層 Model 上設定多個屬性。為此，可從 `set` 閉包回傳一個陣列。陣列中的每個索引鍵都應對應至與該 Model 關聯的底層屬性 / 資料庫欄位：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
        set: fn (Address $value) => [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ],
    );
}
```

<a name="attribute-casting"></a>
## 屬性轉換

屬性轉換提供了類似於取用器與修改器的功能，且不需要在 Model 上定義任何額外的方法。反之，Model 的 `casts` 方法提供了一個方便的方法，來將屬性轉換為常用的資料類型。

`casts` 方法應回傳一個陣列，其中 Key 為要轉換的屬性名稱，而 Value 則為要將該欄位轉換過去的類型。支援的轉換類型有：

<div class="content-list" markdown="1">

- `array`
- `AsFluent::class`
- `AsStringable::class`
- `AsUri::class`
- `boolean`
- `collection`
- `date`
- `datetime`
- `immutable_date`
- `immutable_datetime`
- <code>decimal:&lt;precision&gt;</code>
- `double`
- `encrypted`
- `encrypted:array`
- `encrypted:collection`
- `encrypted:object`
- `float`
- `hashed`
- `integer`
- `object`
- `real`
- `string`
- `timestamp`

</div>

為了示範屬性轉換，我們來將 `is_admin` 屬性轉換為布林值。該屬性在資料庫中是以整數 (`0` 或 `1`) 儲存的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'is_admin' => 'boolean',
        ];
    }
}
```

定義好轉換後，在存取 `is_admin` 屬性時，就算底層的值在資料庫中是以整數儲存，該屬性也一律會被轉換為布林值：

```php
$user = App\Models\User::find(1);

if ($user->is_admin) {
    // ...
}
```

若需要在執行階段新增一個新的、暫時的轉換，可使用 `mergeCasts` 方法。這些轉換定義會被加到 Model 上任何已定義的轉換中：

```php
$user->mergeCasts([
    'is_admin' => 'integer',
    'options' => 'object',
]);
```

> [!WARNING]
> `null` 的屬性不會被轉換。此外，絕對不要定義與關聯名稱相同的轉換 (或屬性)，也絕對不要將轉換指派給 Model 的主鍵。


<a name="stringable-casting"></a>
#### Stringable 轉換

可使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 轉換類別，來將 Model 屬性轉換為一個[流暢的 Illuminate\Support\Stringable 物件](/docs/{{version}}/strings#fluent-strings-method-list)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\AsStringable;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'directory' => AsStringable::class,
        ];
    }
}
```

<a name="array-and-json-casting"></a>
### 陣列與 JSON 轉換

`array` 轉換在處理儲存為序列化 JSON 的欄位時特別有用。例如，若資料庫中有個 `JSON` 或 `TEXT` 型別的欄位，其中包含了序列化的 JSON，只要為該屬性加上 `array` 轉換，在 Eloquent Model 上存取該屬性時，它就會自動被反序列化為 PHP 陣列：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => 'array',
        ];
    }
}
```

定義好轉換後，便可存取 `options` 屬性，而它會自動從 JSON 反序列化為 PHP 陣列。在設定 `options` 屬性的值時，給定的陣列會自動被序列化回 JSON 以進行儲存：

```php
use App\Models\User;

$user = User::find(1);

$options = $user->options;

$options['key'] = 'value';

$user->options = $options;

$user->save();
```

若要以更簡潔的語法更新 JSON 屬性中的單一欄位，可以[將該屬性設為可大量指派](/docs/{{version}}/eloquent#mass-assignment-json-columns)並在呼叫 `update` 方法時使用 `->` 運算子：

```php
$user = User::find(1);

$user->update(['options->key' => 'value']);
```

<a name="json-and-unicode"></a>
#### JSON 與 Unicode

若想將陣列屬性以未逸脫 (unescaped) 的 Unicode 字元儲存為 JSON，可以使用 `json:unicode` 轉換：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => 'json:unicode',
    ];
}
```

<a name="array-object-and-collection-casting"></a>
#### 陣列物件與 Collection 轉換

雖然標準的 `array` 轉換對許多應用程式來說已經足夠，但它仍有一些缺點。由於 `array` 轉換回傳的是一個原生型別，因此無法直接修改陣列中的某個 Offset (位移)。例如，下列程式碼會觸發一個 PHP 錯誤：

```php
$user = User::find(1);

$user->options['key'] = $value;
```

為了解決這個問題，Laravel 提供了一個 `AsArrayObject` 轉換，能將你的 JSON 屬性轉換為 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 類別。這個功能是使用 Laravel 的[自訂轉換](#custom-casts)實作來達成的，它讓 Laravel 能智慧地快取並轉換經修改的物件，進而使個別 Offset 得以在不觸發 PHP 錯誤的情況下被修改。若要使用 `AsArrayObject` 轉換，只要將其指派給屬性即可：

```php
use Illuminate\Database\Eloquent\Casts\AsArrayObject;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsArrayObject::class,
    ];
}
```

同樣地，Laravel 也提供了一個 `AsCollection` 轉換，可將你的 JSON 屬性轉換為 Laravel 的 [Collection](/docs/{{version}}/collections) 實體：

```php
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::class,
    ];
}
```

若希望 `AsCollection` 轉換實體化的是自訂的 Collection 類別，而非 Laravel 的基礎 Collection 類別，可以提供該 Collection 類別的名稱作為轉換參數：

```php
use App\Collections\OptionCollection;
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::using(OptionCollection::class),
    ];
}
```

`of` 方法可用來指示 Collection 項目應透過 Collection 的 [mapInto 方法](/docs/{{version}}/collections#method-mapinto)對應至給定的類別：

```php
use App\ValueObjects\Option;
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::of(Option::class)
    ];
}
```

當將 Collection 對應至物件時，該物件應實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面，以定義其物件實體應如何被序列化為 JSON 並存入資料庫：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Support\Arrayable;
use JsonSerializable;

class Option implements Arrayable, JsonSerializable
{
    public string $name;
    public mixed $value;
    public bool $isLocked;

    /**
     * Create a new Option instance.
     */
    public function __construct(array $data)
    {
        $this->name = $data['name'];
        $this->value = $data['value'];
        $this->isLocked = $data['is_locked'];
    }

    /**
     * Get the instance as an array.
     *
     * @return array{name: string, data: string, is_locked: bool}
     */
    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'value' => $this->value,
            'is_locked' => $this->isLocked,
        ];
    }

    /**
     * Specify the data which should be serialized to JSON.
     *
     * @return array{name: string, data: string, is_locked: bool}
     */
    public function jsonSerialize(): array
    {
        return $this->toArray();
    }
}
```

<a name="date-casting"></a>
### 日期轉換

在預設情況下，Eloquent 會將 `created_at` 與 `updated_at` 欄位轉換為 [Carbon](https://github.com/briannesbitt/Carbon) 的實體。Carbon 繼承了 PHP 的 `DateTime` 類別，並提供了一系列有用的方法。我們可以在 Model 的 `casts` 方法中定義額外的日期轉換，以轉換其他日期屬性。一般來說，日期應使用 `datetime` 或 `immutable_datetime` 型別來轉換。

定義 `date` 或 `datetime` 轉換時，也可以指定日期的格式。此格式會在 [Model 序列化為陣列或 JSON](/docs/{{version}}/eloquent-serialization) 時使用：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'created_at' => 'datetime:Y-m-d',
    ];
}
```

當某個欄位被轉換為日期時，可以將對應 Model 屬性的值設為 UNIX 時間戳、日期字串 (`Y-m-d`)、日期時間字串、或是 `DateTime` / `Carbon` 的實體。日期的值會被正確地轉換並存入資料庫。

我們可以定義 Model 上的 `serializeDate` 方法來自訂該 Model 上所有日期的預設序列化格式。此方法不會影響日期在存入資料庫時的格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

若要指定 Model 的日期在實際存入資料庫時應使用的格式，則應在 Model 上定義 `$dateFormat` 屬性：

```php
/**
 * The storage format of the model's date columns.
 *
 * @var string
 */
protected $dateFormat = 'U';
```


<a name="date-casting-and-timezones"></a>
#### 日期轉換、序列化、與時區

在預設情況下，`date` 與 `datetime` 轉換會將日期序列化為 UTC ISO-8601 日期字串 (`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`)，而不會去管應用程式設定檔 `timezone` 選項中指定的時區為何。我們強烈建議一律使用這個序列化格式，並將應用程式的 `timezone` 設定選項保留為預設的 `UTC` 值，藉此將應用程式的日期儲存在 UTC 時區。在整個應用程式中一致地使用 UTC 時區，可為我們提供與其他 PHP 或 JavaScript 日期處理函式庫最大程度的互通性。

若有為 `date` 或 `datetime` 轉換套用自訂格式 (如 `datetime:Y-m-d H:i:s`)，則在日期序列化時，會使用 Carbon 實體內部的時區。一般來說，這個時區會是應用程式設定檔 `timezone` 中指定的時區。不過，請務必注意，`timestamp` 欄位 (如 `created_at` 與 `updated_at`) 並不受此行為影響，且不論應用程式的時區設定為何，都一律會以 UTC 格式化。


<a name="enum-casting"></a>
### Enum 轉換

Eloquent 也允許我們將屬性值轉換為 PHP 的 [Enum](https://www.php.net/manual/en/language.enumerations.backed.php)。為此，可在 Model 的 `casts` 方法中指定要轉換的屬性與 Enum：

```php
use App\Enums\ServerStatus;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'status' => ServerStatus::class,
    ];
}
```

在 Model 上定義好轉換後，在與該屬性互動時，指定的屬性就會自動被轉換為 Enum，或由 Enum 轉回：

```php
if ($server->status == ServerStatus::Provisioned) {
    $server->status = ServerStatus::Ready;

    $server->save();
}
```


<a name="casting-arrays-of-enums"></a>
#### 轉換 Enum 陣列

有時候，我們可能需要在單一欄位中儲存一個 Enum 值陣列。為此，可利用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 轉換：

```php
use App\Enums\ServerStatus;
use Illuminate\Database\Eloquent\Casts\AsEnumCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'statuses' => AsEnumCollection::of(ServerStatus::class),
    ];
}
```


<a name="encrypted-casting"></a>
### 加密轉換

`encrypted` 轉換會使用 Laravel 內建的[加密](/docs/{{version}}/encryption)功能來將 Model 屬性值加密。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject`、`AsEncryptedCollection` 等轉換的運作方式都與其未加密的版本類似；不過，正如其名，在存入資料庫時，其底層的值會被加密。

由於加密後文字的最終長度是不可預測的，且會比其純文字版本還長，請確定對應的資料庫欄位是 `TEXT` 或更大的型別。此外，由於這些值在資料庫中是加密的，因此將無法查詢或搜尋已加密的屬性值。


<a name="key-rotation"></a>
#### 金鑰輪替

讀者可能知道，Laravel 會使用應用程式的 `app` 設定檔中指定的 `key` 設定值來加密字串。一般來說，這個值會對應到 `APP_KEY` 環境變數的值。若需要輪替應用程式的加密金鑰，則需要手動使用新的金鑰來重新加密這些已加密的屬性。


<a name="query-time-casting"></a>
### 查詢時轉換

有時候，我們可能需要在執行查詢時套用轉換，例如，從資料表中選取一個 Raw (原始) 值時。舉例來說，請參考下列查詢：

```php
use App\Models\Post;
use App\Models\User;

$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->get();
```

這個查詢結果上的 `last_posted_at` 屬性會是一個簡單的字串。若能在執行查詢時為這個屬性套用 `datetime` 轉換就再好不過了。值得慶幸的是，我們可以使用 `withCasts` 方法來達成這個目的：

```php
$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->withCasts([
    'last_posted_at' => 'datetime'
])->get();
```

<a name="custom-casts"></a>
## 自訂轉換

Laravel 有許多內建且實用的轉換型別；不過，有時候可能也需要定義自己的轉換型別。若要建立轉換，請執行 `make:cast` 這個 Artisan 指令。新的轉換類別會被放在 `app/Casts` 目錄下：

```shell
php artisan make:cast AsJson
```

所有自訂轉換類別都實作了 `CastsAttributes` 介面。實作此介面的類別必須定義 `get` 與 `set` 方法。`get` 方法負責將資料庫中的原始值轉換為轉換後的值，而 `set` 方法則應將轉換後的值轉換為可存入資料庫的原始值。舉例來說，我們來將內建的 `json` 轉換型別重新實作為一個自訂轉換型別：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

class AsJson implements CastsAttributes
{
    /**
     * Cast the given value.
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function get(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): array {
        return json_decode($value, true);
    }

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): string {
        return json_encode($value);
    }
}
```

定義好自訂轉換型別後，就可以使用其類別名稱將該型別附加到 Model 的屬性上：

```php
<?php

namespace App\Models;

use App\Casts\AsJson;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => AsJson::class,
        ];
    }
}
```

<a name="value-object-casting"></a>
### 值物件轉換

不僅能將值轉換為原始型別，也可以將值轉換為物件。定義將值轉換為物件的自訂轉換與轉換為原始型別非常類似；不過，若值物件包含了多個資料庫欄位，則 `set` 方法必須回傳一個鍵／值對陣列，用來在 Model 上設定可儲存的原始值。若值物件只影響單一欄位，則只需回傳可儲存的值即可。

舉例來說，我們來定義一個自訂轉換類別，用來將多個 Model 值轉換為單一的 `Address` 值物件。我們假設 `Address` 值物件有 `lineOne` 與 `lineTwo` 這兩個公開屬性：

```php
<?php

namespace App\Casts;

use App\ValueObjects\Address;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use InvalidArgumentException;

class AsAddress implements CastsAttributes
{
    /**
     * Cast the given value.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function get(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): Address {
        return new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two']
        );
    }

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, string>
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): array {
        if (! $value instanceof Address) {
            throw new InvalidArgumentException('The given value is not an Address instance.');
        }

        return [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ];
    }
}
```

當轉換為值物件時，對該值物件所做的任何變更，都會在 Model 儲存前自動同步回 Model：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Value';

$user->save();
```

> [!NOTE]
> 若打算將包含值物件的 Eloquent Model 序列化為 JSON 或陣列，則應在該值物件上實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面。

<a name="value-object-caching"></a>
#### 值物件快取

當被轉換為值物件的屬性被解析時，Eloquent 會快取這些物件。因此，若再次存取該屬性，就會回傳同一個物件實體。

若想停用自訂轉換類別的物件快取行為，可以在自訂轉換類別上宣告一個公開的 `withoutObjectCaching` 屬性：

```php
class AsAddress implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```

<a name="array-json-serialization"></a>
### 陣列 / JSON 序列化

當使用 `toArray` 與 `toJson` 方法將 Eloquent Model 轉換為陣列或 JSON 時，只要自訂轉換的值物件有實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面，通常也會一併被序列化。不過，當使用第三方函式庫提供的值物件時，可能無法將這些介面新增到物件上。

因此，可以指定由自訂轉換類別來負責序列化該值物件。若要這麼做，自訂轉換類別應實作 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 介面。此介面規定類別中應包含一個 `serialize` 方法，該方法應回傳值物件的序列化形式：

```php
/**
 * Get the serialized representation of the value.
 *
 * @param  array<string, mixed>  $attributes
 */
public function serialize(
    Model $model,
    string $key,
    mixed $value,
    array $attributes,
): string {
    return (string) $value;
}
```

<a name="inbound-casting"></a>
### 僅傳入轉換

有時候，可能需要撰寫一個自訂轉換類別，該類別只在設定 Model 屬性時轉換值，而在從 Model 取出屬性時不執行任何操作。

僅傳入的自訂轉換應實作 `CastsInboundAttributes` 介面，該介面僅要求定義 `set` 方法。可使用 `--inbound` 選項來叫用 `make:cast` Artisan 指令，以產生一個僅傳入的轉換類別：

```shell
php artisan make:cast AsHash --inbound
```

「雜湊 (hashing)」轉換是僅傳入轉換的一個典型範例。舉例來說，我們可以定義一個轉換，透過給定的演算法來雜湊傳入的值：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsInboundAttributes;
use Illuminate\Database\Eloquent\Model;

class AsHash implements CastsInboundAttributes
{
    /**
     * Create a new cast class instance.
     */
    public function __construct(
        protected string|null $algorithm = null,
    ) {}

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): string {
        return is_null($this->algorithm)
            ? bcrypt($value)
            : hash($this->algorithm, $value);
    }
}
```

<a name="cast-parameters"></a>
### 轉換參數

將自訂轉換附加到 Model 上時，可以使用 `:` 字元將參數與類別名稱分開，並用逗號分隔多個參數來指定轉換參數。這些參數會被傳遞給該轉換類別的建構函式：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'secret' => AsHash::class.':sha256',
    ];
}
```

<a name="comparing-cast-values"></a>
### 比較轉換值

若想定義兩個給定的轉換值應如何比較來判斷其是否已變更，則自訂的轉換類別可實作 `Illuminate\Contracts\Database\Eloquent\ComparesCastableAttributes` 介面。這樣一來，我們就能精細地控制 Eloquent 認定哪些值已變更，並在更新 Model 時將其儲存至資料庫。

該介面規定，類別中應包含一個 `compare` 方法，且若給定的值被視為相等，則該方法應回傳 `true`：

```php
/**
 * Determine if the given values are equal.
 *
 * @param  \Illuminate\Database\Eloquent\Model  $model
 * @param  string  $key
 * @param  mixed  $firstValue
 * @param  mixed  $secondValue
 * @return bool
 */
public function compare(
    Model $model,
    string $key,
    mixed $firstValue,
    mixed $secondValue
): bool {
    return $firstValue === $secondValue;
}
```

<a name="castables"></a>
### 可轉換物件 (Castable)

有時候，我們會想讓應用程式中的值物件能定義自己的自訂轉換類別。比起將自訂轉換類別附加到 Model 上，我們也可以改為附加一個有實作 `Illuminate\Contracts\Database\Eloquent\Castable` 介面的值物件類別：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class,
    ];
}
```

實作 `Castable` 介面的物件必須定義一個 `castUsing` 方法，該方法會回傳自訂轉換器 (Caster) 類別的類別名稱，而該轉換器則負責處理與這個 `Castable` 類別之間的雙向轉換：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use App\Casts\AsAddress;

class Address implements Castable
{
    /**
     * Get the name of the caster class to use when casting from / to this cast target.
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): string
    {
        return AsAddress::class;
    }
}
```

在使用 `Castable` 類別時，我們還是可以在 `casts` 方法的定義中提供引數。這些引數會被傳遞給 `castUsing` 方法：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class.':argument',
    ];
}
```

<a name="anonymous-cast-classes"></a>
#### 可轉換物件與匿名轉換類別

透過將「可轉換物件 (castable)」與 PHP 的[匿名類別](https://www.php.net/manual/en/language.oop5.anonymous.php)結合，我們就可以將值物件與其轉換邏輯定義為單一一個可轉換物件。若要這麼做，只要在值物件的 `castUsing` 方法中回傳一個匿名類別即可。該匿名類別應實作 `CastsAttributes` 介面：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

class Address implements Castable
{
    // ...

    /**
     * Get the caster class to use when casting from / to this cast target.
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): CastsAttributes
    {
        return new class implements CastsAttributes
        {
            public function get(
                Model $model,
                string $key,
                mixed $value,
                array $attributes,
            ): Address {
                return new Address(
                    $attributes['address_line_one'],
                    $attributes['address_line_two']
                );
            }

            public function set(
                Model $model,
                string $key,
                mixed $value,
                array $attributes,
            ): array {
                return [
                    'address_line_one' => $value->lineOne,
                    'address_line_two' => $value->lineTwo,
                ];
            }
        };
    }
}
```