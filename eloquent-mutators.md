# Eloquent：Mutators 與型別轉換

- [簡介](#introduction)
- [存取器與修改器](#accessors-and-mutators)
    - [定義存取器](#defining-an-accessor)
    - [定義修改器](#defining-a-mutator)
- [屬性型別轉換](#attribute-casting)
    - [陣列與 JSON 型別轉換](#array-and-json-casting)
    - [日期型別轉換](#date-casting)
    - [Enum 型別轉換](#enum-casting)
    - [加密型別轉換](#encrypted-casting)
    - [查詢時型別轉換](#query-time-casting)
- [自訂型別轉換](#custom-casts)
    - [值物件型別轉換](#value-object-casting)
    - [陣列 / JSON 序列化](#array-json-serialization)
    - [寫入型別轉換](#inbound-casting)
    - [型別轉換參數](#cast-parameters)
    - [Castables](#castables)

<a name="introduction"></a>
## 簡介

存取器 (Accessors)、修改器 (Mutators) 與屬性型別轉換 (attribute casting) 允許您在模型實例上擷取或設定 Eloquent 屬性值時，對其進行轉換。例如，您可能希望使用 [Laravel encrypter](/docs/{{version}}/encryption) 在值儲存到資料庫時對其進行加密，然後在 Eloquent 模型上存取該屬性時自動解密。或者，您可能希望在透過 Eloquent 模型存取資料庫中儲存的 JSON 字串時，將其轉換為陣列。

<a name="accessors-and-mutators"></a>
## 存取器與修改器

<a name="defining-an-accessor"></a>
### 定義存取器

存取器會在 Eloquent 屬性值被存取時，對其進行轉換。要定義存取器，請在模型上建立一個 protected 方法來代表可存取的屬性。當適用時，此方法的名稱應與其真實的底層模型屬性 / 資料庫欄位的「駝峰式命名 (camel case)」表示法相對應。

在此範例中，我們將為 `first_name` 屬性定義一個存取器。當嘗試擷取 `first_name` 屬性的值時，Eloquent 將自動調用該存取器。所有屬性存取器 / 修改器方法都必須宣告 `Illuminate\Database\Eloquent\Casts\Attribute` 的回傳型別提示：

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

所有存取器方法都會回傳一個 `Attribute` 實例，該實例定義了屬性將如何被存取，並且可選地被修改。在此範例中，我們只定義了屬性將如何被存取。為此，我們向 `Attribute` 類別的建構式提供了 `get` 引數。

如您所見，欄位的原始值會傳遞給存取器，讓您可以操作並回傳該值。要存取存取器的值，您可以直接在模型實例上存取 `first_name` 屬性：

    use App\Models\User;

    $user = User::find(1);

    $firstName = $user->first_name;

> [!NOTE]  
> 如果您希望這些計算值被添加到模型的陣列 / JSON 表示中，[您將需要附加它們](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。

<a name="building-value-objects-from-multiple-attributes"></a>
#### 從多個屬性建立值物件

有時您的存取器可能需要將多個模型屬性轉換成單一的「值物件 (value object)」。為此，您的 `get` 閉包可以接受第二個 `$attributes` 引數，該引數將自動提供給閉包，並包含模型目前所有屬性的陣列：

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
#### 存取器快取

當存取器回傳值物件時，對該值物件所做的任何變更都會在模型儲存之前自動同步回模型。這之所以可能，是因為 Eloquent 會保留存取器回傳的實例，這樣它就可以在每次調用存取器時回傳相同的實例：

    use App\Models\User;

    $user = User::find(1);

    $user->address->lineOne = 'Updated Address Line 1 Value';
    $user->address->lineTwo = 'Updated Address Line 2 Value';

    $user->save();

然而，您有時可能希望為字串和布林值等原始值 (primitive values) 啟用快取，特別是當它們是計算密集型的時候。為此，您可以在定義存取器時調用 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

如果您想禁用屬性的物件快取行為，您可以在定義屬性時調用 `withoutObjectCaching` 方法：

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

修改器會在 Eloquent 屬性值被設定時，對其進行轉換。要定義修改器，您可以在定義屬性時提供 `set` 引數。讓我們為 `first_name` 屬性定義一個修改器。當我們嘗試在模型上設定 `first_name` 屬性的值時，此修改器將自動被調用：

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

修改器閉包將接收正在設定到屬性上的值，讓您可以操作值並回傳操作過的值。要使用我們的修改器，我們只需在 Eloquent 模型上設定 `first_name` 屬性：

    use App\Models\User;

    $user = User::find(1);

    $user->first_name = 'Sally';

在此範例中，`set` 回調將以值 `Sally` 被調用。修改器隨後將對名稱應用 `strtolower` 函數，並將其結果值設定在模型的內部 `$attributes` 陣列中。

<a name="mutating-multiple-attributes"></a>
#### 修改多個屬性

有時您的修改器可能需要在底層模型上設定多個屬性。為此，您可以從 `set` 閉包回傳一個陣列。陣列中的每個鍵都應該與模型相關聯的底層屬性 / 資料庫欄位相對應：

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
## 屬性型別轉換

屬性型別轉換 (Attribute casting) 提供了與存取器 (accessors) 和修改器 (mutators) 類似的功能，而無需在你的模型 (model) 上定義任何額外方法。相反地，你的模型中的 `casts` 方法提供了一種便捷的方式，將屬性轉換為常見的資料型別。

`casts` 方法應該回傳一個陣列 (array)，其中鍵 (key) 是要進行型別轉換的屬性名稱，而值 (value) 則是你想將該欄位轉換成的型別。支援的型別轉換型別有：

<div class="content-list" markdown="1">

- `array`
- `AsStringable::class`
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

為了示範屬性型別轉換，我們將 `is_admin` 屬性進行型別轉換，它在資料庫中儲存為整數 (`0` 或 `1`)，將其轉換為布林值 (boolean value)：

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

定義型別轉換後，當你存取 `is_admin` 屬性時，它將始終被轉換為布林值，即使其底層值在資料庫中儲存為整數：

    $user = App\Models\User::find(1);

    if ($user->is_admin) {
        // ...
    }

如果你需要於執行階段 (runtime) 新增一個新的暫時性型別轉換，你可以使用 `mergeCasts` 方法。這些型別轉換定義將會被新增到模型上任何已定義的型別轉換中：

    $user->mergeCasts([
        'is_admin' => 'integer',
        'options' => 'object',
    ]);

> [!WARNING]  
> 為 `null` 的屬性將不會被型別轉換。此外，你不應定義與關聯 (relationship) 同名的型別轉換 (或屬性)，也不應將型別轉換指派給模型 (model) 的主鍵 (primary key)。

<a name="stringable-casting"></a>
#### Stringable 型別轉換

你可以使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 型別轉換類別 (cast class) 將模型屬性轉換為一個 [fluent `Illuminate\Support\Stringable` 物件](/docs/{{version}}/strings#fluent-strings-method-list)：

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

<a name="array-and-json-casting"></a>
### 陣列與 JSON 型別轉換

`array` 型別轉換在處理儲存為序列化 JSON 的欄位時特別有用。例如，如果你的資料庫中有一個包含序列化 JSON 的 `JSON` 或 `TEXT` 欄位型別 (field type)，將 `array` 型別轉換添加到該屬性後，當你在 Eloquent 模型上存取它時，它將自動將該屬性反序列化 (deserialize) 為 PHP 陣列：

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

一旦定義了型別轉換，你就可以存取 `options` 屬性，它將自動從 JSON 反序列化 (deserialize) 為 PHP 陣列。當你設定 `options` 屬性的值時，給定的陣列將自動序列化 (serialize) 回 JSON 以供儲存：

    use App\Models\User;

    $user = User::find(1);

    $options = $user->options;

    $options['key'] = 'value';

    $user->options = $options;

    $user->save();

若要以更簡潔的語法更新 JSON 屬性的單一欄位，你可以將該屬性 [設為可批量賦值 (mass assignable)](/docs/{{version}}/eloquent#mass-assignment-json-columns)，並在呼叫 `update` 方法時使用 `->` 運算子 (operator)：

    $user = User::find(1);

    $user->update(['options->key' => 'value']);

<a name="array-object-and-collection-casting"></a>
#### 陣列物件與集合型別轉換

儘管標準的 `array` 型別轉換對於許多應用程式來說已經足夠，但它確實存在一些缺點。由於 `array` 型別轉換回傳的是一個基本型別 (primitive type)，因此無法直接變更陣列的偏移量 (offset)。例如，以下程式碼將觸發一個 PHP 錯誤：

    $user = User::find(1);

    $user->options['key'] = $value;

為了解決這個問題，Laravel 提供了 `AsArrayObject` 型別轉換，它將你的 JSON 屬性轉換為一個 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 類別。此功能是透過 Laravel 的 [自訂型別轉換](#custom-casts) 實現的，它允許 Laravel 智慧地快取 (cache) 並轉換被修改的物件 (object)，使得個別的偏移量 (offset) 可以被修改而不會觸發 PHP 錯誤。若要使用 `AsArrayObject` 型別轉換，只需將其指派給一個屬性即可：

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

同樣地，Laravel 提供了 `AsCollection` 型別轉換，它將你的 JSON 屬性轉換為一個 Laravel [Collection](/docs/{{version}}/collections) 實例 (instance)：

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

如果你希望 `AsCollection` 型別轉換實例化 (instantiate) 一個自訂的 Collection 類別，而不是 Laravel 的基本 Collection 類別，你可以將 Collection 類別名稱作為型別轉換參數 (cast argument) 提供：

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

<a name="date-casting"></a>
### 日期型別轉換

預設情況下，Eloquent 會將 `created_at` 和 `updated_at` 欄位轉換為 [Carbon](https://github.com/briannesbitt/Carbon) 實例。Carbon 擴展了 PHP `DateTime` 類別並提供了一系列實用的方法。您可以透過在模型的 `casts` 方法中定義額外的日期型別轉換來轉換其他日期屬性。通常，日期應使用 `datetime` 或 `immutable_datetime` 型別轉換來轉換。

定義 `date` 或 `datetime` 型別轉換時，您還可以指定日期的格式。此格式將在模型被 [序列化為陣列或 JSON](/docs/{{version}}/eloquent-serialization) 時使用：

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

當欄位被轉換為日期時，您可以將對應的模型屬性值設定為 UNIX 時間戳記、日期字串 (`Y-m-d`)、日期時間字串或 `DateTime` / `Carbon` 實例。日期的值將被正確轉換並儲存到資料庫中。

您可以透過在模型上定義 `serializeDate` 方法來自訂所有模型日期的預設序列化格式。此方法不會影響日期在資料庫中儲存時的格式：

    /**
     * Prepare a date for array / JSON serialization.
     */
    protected function serializeDate(DateTimeInterface $date): string
    {
        return $date->format('Y-m-d');
    }

若要指定實際儲存模型日期到資料庫時應使用的格式，您應該在模型上定義 `$dateFormat` 屬性：

    /**
     * The storage format of the model's date columns.
     *
     * @var string
     */
    protected $dateFormat = 'U';


<a name="date-casting-and-timezones"></a>
#### 日期型別轉換、序列化與時區

預設情況下，`date` 和 `datetime` 型別轉換會將日期序列化為 UTC ISO-8601 日期字串 (`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`)，而無論應用程式 `timezone` 設定選項中指定的時區為何。強烈建議您始終使用此序列化格式，並透過不將應用程式的 `timezone` 設定選項從其預設的 `UTC` 值更改為其他值來將應用程式的日期儲存在 UTC 時區。在整個應用程式中一致地使用 UTC 時區將提供與其他 PHP 和 JavaScript 中編寫的日期操作函式庫的最高互通性。

如果對 `date` 或 `datetime` 型別轉換應用了自訂格式，例如 `datetime:Y-m-d H:i:s`，則 Carbon 實例的內部時區將在日期序列化期間使用。通常，這將是應用程式 `timezone` 設定選項中指定的時區。但是，請務必注意，`timestamp` 欄位（例如 `created_at` 和 `updated_at`）不受此行為的影響，並且無論應用程式的時區設定如何，它們始終以 UTC 格式化。


<a name="enum-casting"></a>
### Enum 型別轉換

Eloquent 也允許您將屬性值轉換為 PHP [Enums](https://www.php.net/manual/en/language.enumerations.backed.php)。為此，您可以在模型的 `casts` 方法中指定要轉換的屬性及 Enum：

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

一旦您在模型上定義了型別轉換，當您與該屬性互動時，指定的屬性將自動在 Enum 之間進行型別轉換：

    if ($server->status == ServerStatus::Provisioned) {
        $server->status = ServerStatus::Ready;

        $server->save();
    }


<a name="casting-arrays-of-enums"></a>
#### Enum 陣列型別轉換

有時您可能需要模型在單一欄位中儲存一個 Enum 值陣列。為此，您可以使用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 型別轉換：

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


<a name="encrypted-casting"></a>
### 加密型別轉換

`encrypted` 型別轉換將使用 Laravel 內建的[加密](/docs/{{version}}/encryption)功能加密模型的屬性值。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject` 和 `AsEncryptedCollection` 型別轉換的功能與其未加密的對應型別轉換相同；但是，正如您所預期的，當儲存在資料庫中時，底層值會被加密。

由於加密文字的最終長度不可預測且比其純文字對應項更長，請確保相關的資料庫欄位是 `TEXT` 型別或更大的型別。此外，由於值在資料庫中是加密的，您將無法查詢或搜尋加密的屬性值。


<a name="key-rotation"></a>
#### 密鑰輪換

您可能知道，Laravel 使用應用程式 `app` 設定檔中指定的 `key` 設定值來加密字串。通常，此值對應於 `APP_KEY` 環境變數的值。如果您需要輪換應用程式的加密密鑰，您將需要使用新密鑰手動重新加密您的加密屬性。


<a name="query-time-casting"></a>
### 查詢時型別轉換

有時您可能需要在執行查詢時應用型別轉換，例如從表格中選擇原始值時。例如，考慮以下查詢：

    use App\Models\Post;
    use App\Models\User;

    $users = User::select([
        'users.*',
        'last_posted_at' => Post::selectRaw('MAX(created_at)')
            ->whereColumn('user_id', 'users.id')
    ])->get();

此查詢結果中的 `last_posted_at` 屬性將是一個簡單的字串。如果我們可以在執行查詢時將 `datetime` 型別轉換應用於此屬性，那將會很棒。幸運的是，我們可以透過使用 `withCasts` 方法來實現此目的：

    $users = User::select([
        'users.*',
        'last_posted_at' => Post::selectRaw('MAX(created_at)')
            ->whereColumn('user_id', 'users.id')
    ])->withCasts([
        'last_posted_at' => 'datetime'
    ])->get();

<a name="custom-casts"></a>
## 自訂型別轉換

Laravel 擁有各種內建且實用的型別轉換型態；然而，你偶爾可能需要定義自己的型別轉換型態。要建立一個型別轉換，請執行 `make:cast` Artisan 命令。新的型別轉換類別會被放置在 `app/Casts` 目錄中：

```shell
php artisan make:cast Json
```

所有自訂型別轉換類別都實作了 `CastsAttributes` 介面。實作此介面的類別必須定義 `get` 與 `set` 方法。`get` 方法負責將資料庫中的原始值轉換為已型別轉換的值，而 `set` 方法應將已型別轉換的值轉換為可儲存到資料庫的原始值。舉例來說，我們將重新實作內建的 `json` 型別轉換型態作為自訂型別轉換型態：

    <?php

    namespace App\Casts;

    use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
    use Illuminate\Database\Eloquent\Model;

    class Json implements CastsAttributes
    {
        /**
         * Cast the given value.
         *
         * @param  array<string, mixed>  $attributes
         * @return array<string, mixed>
         */
        public function get(Model $model, string $key, mixed $value, array $attributes): array
        {
            return json_decode($value, true);
        }

        /**
         * Prepare the given value for storage.
         *
         * @param  array<string, mixed>  $attributes
         */
        public function set(Model $model, string $key, mixed $value, array $attributes): string
        {
            return json_encode($value);
        }
    }

一旦你定義了自訂型別轉換型態，就可以使用其類別名稱將其附加到模型屬性：

    <?php

    namespace App\Models;

    use App\Casts\Json;
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
                'options' => Json::class,
            ];
        }
    }


<a name="value-object-casting"></a>
### 值物件型別轉換

你不僅限於將值型別轉換為基本型別。你也可以將值型別轉換為物件。定義將值型別轉換為物件的自訂型別轉換與型別轉換為基本型別非常相似；然而，`set` 方法應回傳鍵/值對的陣列，這些鍵/值對將用於在模型上設定原始、可儲存的值。

舉例來說，我們將定義一個自訂型別轉換類別，它將多個模型值型別轉換為單一 `Address` 值物件。我們假設 `Address` 值物件有兩個公開屬性：`lineOne` 和 `lineTwo`：

    <?php

    namespace App\Casts;

    use App\ValueObjects\Address as AddressValueObject;
    use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
    use Illuminate\Database\Eloquent\Model;
    use InvalidArgumentException;

    class Address implements CastsAttributes
    {
        /**
         * Cast the given value.
         *
         * @param  array<string, mixed>  $attributes
         */
        public function get(Model $model, string $key, mixed $value, array $attributes): AddressValueObject
        {
            return new AddressValueObject(
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
        public function set(Model $model, string $key, mixed $value, array $attributes): array
        {
            if (! $value instanceof AddressValueObject) {
                throw new InvalidArgumentException('The given value is not an Address instance.');
            }

            return [
                'address_line_one' => $value->lineOne,
                'address_line_two' => $value->lineTwo,
            ];
        }
    }

當型別轉換為值物件時，對值物件所做的任何更改都會在模型儲存之前自動同步回模型：

    use App\Models\User;

    $user = User::find(1);

    $user->address->lineOne = 'Updated Address Value';

    $user->save();

> [!NOTE]  
> 如果你打算將包含值物件的 Eloquent 模型序列化為 JSON 或陣列，你應該在值物件上實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面。


<a name="value-object-caching"></a>
#### 值物件快取

當型別轉換為值物件的屬性被解析時，它們會由 Eloquent 快取。因此，如果再次存取該屬性，將會回傳相同的物件實例。

如果你想停用自訂型別轉換類別的物件快取行為，你可以在自訂型別轉換類別上宣告一個公開的 `withoutObjectCaching` 屬性：

```php
class Address implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```


<a name="array-json-serialization"></a>
### 陣列 / JSON 序列化

當 Eloquent 模型透過 `toArray` 與 `toJson` 方法轉換為陣列或 JSON 時，只要你的自訂型別轉換值物件實作了 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面，它們通常也會被序列化。然而，當使用第三方函式庫提供的值物件時，你可能無法將這些介面新增到物件中。

因此，你可以指定你的自訂型別轉換類別將負責序列化值物件。為此，你的自訂型別轉換類別應該實作 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 介面。此介面規定你的類別應該包含一個 `serialize` 方法，它應該回傳值物件的序列化形式：

    /**
     * Get the serialized representation of the value.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function serialize(Model $model, string $key, mixed $value, array $attributes): string
    {
        return (string) $value;
    }


<a name="inbound-casting"></a>
### 寫入型別轉換

偶爾，你可能需要編寫一個自訂型別轉換類別，它只轉換設定到模型上的值，並且在從模型中取回屬性時不執行任何操作。

僅用於寫入的自訂型別轉換應實作 `CastsInboundAttributes` 介面，此介面只要求定義一個 `set` 方法。`make:cast` Artisan 命令可以搭配 `--inbound` 選項呼叫，以生成僅用於寫入的型別轉換類別：

```shell
php artisan make:cast Hash --inbound
```

僅用於寫入的經典範例是「雜湊」型別轉換。例如，我們可以定義一個型別轉換，它透過給定的演算法對寫入值進行雜湊處理：

    <?php

    namespace App\Casts;

    use Illuminate\Contracts\Database\Eloquent\CastsInboundAttributes;
    use Illuminate\Database\Eloquent\Model;

    class Hash implements CastsInboundAttributes
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
        public function set(Model $model, string $key, mixed $value, array $attributes): string
        {
            return is_null($this->algorithm)
                ? bcrypt($value)
                : hash($this->algorithm, $value);
        }
    }


<a name="cast-parameters"></a>
### 型別轉換參數

將自訂型別轉換附加到模型時，型別轉換參數可以透過使用 `:` 字元將其與類別名稱分開來指定，並以逗號分隔多個參數。這些參數將傳遞給型別轉換類別的建構子：

    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'secret' => Hash::class.':sha256',
        ];
    }

<a name="castables"></a>
### Castables

您可能會希望應用程式的值物件 (value objects) 能定義自己的自訂型別轉換類別 (custom cast classes)。與其將自訂型別轉換類別附加到您的 Model，您也可以改為附加一個實作 `Illuminate\Contracts\Database\Eloquent\Castable` 介面的值物件類別：

    use App\ValueObjects\Address;

    protected function casts(): array
    {
        return [
            'address' => Address::class,
        ];
    }

實作 `Castable` 介面的物件必須定義一個 `castUsing` 方法，該方法會回傳負責 `Castable` 類別型別轉換的自訂型別轉換器類別名稱：

    <?php

    namespace App\ValueObjects;

    use Illuminate\Contracts\Database\Eloquent\Castable;
    use App\Casts\Address as AddressCast;

    class Address implements Castable
    {
        /**
         * Get the name of the caster class to use when casting from / to this cast target.
         *
         * @param  array<string, mixed>  $arguments
         */
        public static function castUsing(array $arguments): string
        {
            return AddressCast::class;
        }
    }

當使用 `Castable` 類別時，您仍可在 `casts` 方法定義中提供引數。這些引數會傳遞給 `castUsing` 方法：

    use App\ValueObjects\Address;

    protected function casts(): array
    {
        return [
            'address' => Address::class.':argument',
        ];
    }


<a name="anonymous-cast-classes"></a>
#### Castables 與匿名型別轉換類別

結合「Castables」與 PHP 的 [匿名類別](https://www.php.net/manual/en/language.oop5.anonymous.php)，您可以將值物件 (value object) 及其型別轉換邏輯定義為單一的可型別轉換物件。要實現這一點，請從您的值物件的 `castUsing` 方法中回傳一個匿名類別。此匿名類別應實作 `CastsAttributes` 介面：

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
                public function get(Model $model, string $key, mixed $value, array $attributes): Address
                {
                    return new Address(
                        $attributes['address_line_one'],
                        $attributes['address_line_two']
                    );
                }

                public function set(Model $model, string $key, mixed $value, array $attributes): array
                {
                    return [
                        'address_line_one' => $value->lineOne,
                        'address_line_two' => $value->lineTwo,
                    ];
                }
            };
        }
    }