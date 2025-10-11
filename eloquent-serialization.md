# Eloquent：序列化

- [介紹](#introduction)
- [序列化模型與集合](#serializing-models-and-collections)
    - [序列化為陣列](#serializing-to-arrays)
    - [序列化為 JSON](#serializing-to-json)
- [自 JSON 隱藏屬性](#hiding-attributes-from-json)
- [附加值至 JSON](#appending-values-to-json)
- [日期序列化](#date-serialization)

<a name="introduction"></a>
## 介紹

在使用 Laravel 建立 API 時，您經常需要將您的 models 和 relationships 轉換為陣列或 JSON。Eloquent 包含了方便的方法來進行這些轉換，並控制哪些屬性會包含在 models 的序列化表示中。

> [!NOTE]  
> 若想了解處理 Eloquent model 和 collection JSON 序列化更強大的方式，請查閱 [Eloquent API resources](/docs/{{version}}/eloquent-resources) 的文件。

<a name="serializing-models-and-collections"></a>
## 序列化模型與集合

<a name="serializing-to-arrays"></a>
### 序列化為陣列

要將 model 及其已載入的 [relationships](/docs/{{version}}/eloquent-relationships) 轉換為陣列，您應該使用 `toArray` 方法。此方法是遞迴的，因此所有屬性以及所有關聯（包括關聯的關聯）都將被轉換為陣列：

    use App\Models\User;

    $user = User::with('roles')->first();

    return $user->toArray();

`attributesToArray` 方法可用於將 model 的屬性轉換為陣列，但不會轉換其 relationships：

    $user = User::first();

    return $user->attributesToArray();

您也可以透過呼叫 collection 實例上的 `toArray` 方法，將整個 [collections](/docs/{{version}}/eloquent-collections) 轉換為陣列：

    $users = User::all();

    return $users->toArray();

<a name="serializing-to-json"></a>
### 序列化為 JSON

要將 model 轉換為 JSON，您應該使用 `toJson` 方法。與 `toArray` 類似，`toJson` 方法是遞迴的，因此所有屬性與關聯都將被轉換為 JSON。您也可以指定任何 [PHP 支援](https://secure.php.net/manual/en/function.json-encode.php) 的 JSON 編碼選項：

    use App\Models\User;

    $user = User::find(1);

    return $user->toJson();

    return $user->toJson(JSON_PRETTY_PRINT);

另外，您可以將 model 或 collection 強制轉換為字串，這將會自動呼叫 model 或 collection 上的 `toJson` 方法：

    return (string) User::find(1);

由於 models 和 collections 在強制轉換為字串時會被轉換為 JSON，您可以直接從應用程式的路由或控制器中回傳 Eloquent 物件。當 Laravel 的 Eloquent models 和 collections 從路由或控制器回傳時，Laravel 會自動將它們序列化為 JSON：

    Route::get('/users', function () {
        return User::all();
    });

<a name="relationships"></a>
#### 關聯

當 Eloquent model 被轉換為 JSON 時，其已載入的 relationships 會自動作為屬性包含在 JSON 物件中。此外，儘管 Eloquent relationship 方法是使用「駝峰式 (camel case)」方法名稱定義的，但 relationship 的 JSON 屬性將會是「蛇形 (snake case)」命名。

<a name="hiding-attributes-from-json"></a>
## 自 JSON 隱藏屬性

有時您可能希望限制在 model 的陣列或 JSON 表示中包含的屬性，例如密碼。為此，請將 `$hidden` 屬性加入您的 model 中。列在 `$hidden` 屬性陣列中的屬性將不會包含在 model 的序列化表示中：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 要隱藏起來以進行序列化的屬性。
         *
         * @var array<string>
         */
        protected $hidden = ['password'];
    }

> [!NOTE]  
> 若要隱藏 relationships，請將 relationship 的方法名稱加到您的 Eloquent model 的 `$hidden` 屬性中。

或者，您可以使用 `visible` 屬性來定義一個「允許列表 (allow list)」，其中包含應該包含在 model 的陣列和 JSON 表示中的屬性。所有未出現在 `$visible` 陣列中的屬性，在 model 被轉換為陣列或 JSON 時將會被隱藏：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 在陣列中應可見的屬性。
         *
         * @var array
         */
        protected $visible = ['first_name', 'last_name'];
    }

<a name="temporarily-modifying-attribute-visibility"></a>
#### 暫時修改屬性可見性

如果您想讓某些通常被隱藏的屬性在給定的 model 實例上變得可見，您可以使用 `makeVisible` 方法。`makeVisible` 方法會回傳 model 實例：

    return $user->makeVisible('attribute')->toArray();

同樣地，如果您想隱藏某些通常可見的屬性，您可以使用 `makeHidden` 方法。

    return $user->makeHidden('attribute')->toArray();

如果您希望暫時覆寫所有可見或隱藏的屬性，您可以分別使用 `setVisible` 和 `setHidden` 方法：

    return $user->setVisible(['id', 'name'])->toArray();

    return $user->setHidden(['email', 'password', 'remember_token'])->toArray();

<a name="appending-values-to-json"></a>
## 附加值至 JSON

有時，當將 models 轉換為陣列或 JSON 時，您可能希望添加在資料庫中沒有對應欄位的屬性。為此，請首先為該值定義一個 [accessor](/docs/{{version}}/eloquent-mutators)：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Casts\Attribute;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 判斷使用者是否為管理員。
         */
        protected function isAdmin(): Attribute
        {
            return new Attribute(
                get: fn () => 'yes',
            );
        }
    }

如果您希望 accessor 始終附加到 model 的陣列和 JSON 表示中，您可以將屬性名稱添加到 model 的 `appends` 屬性中。請注意，屬性名稱通常使用其「蛇形 (snake case)」序列化表示來引用，儘管 accessor 的 PHP 方法是使用「駝峰式 (camel case)」定義的：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 要附加到 model 陣列形式的 accessors。
         *
         * @var array
         */
        protected $appends = ['is_admin'];
    }

一旦屬性被添加到 `appends` 列表中，它將會包含在 model 的陣列和 JSON 表示中。`appends` 陣列中的屬性也將遵守在 model 上配置的 `visible` 和 `hidden` 設定。

<a name="appending-at-run-time"></a>
#### 運行時附加

在運行時，您可以指示 model 實例使用 `append` 方法附加額外的屬性。或者，您可以使用 `setAppends` 方法來覆寫給定 model 實例的整個附加屬性陣列：

    return $user->append('is_admin')->toArray();

    return $user->setAppends(['is_admin'])->toArray();

<a name="date-serialization"></a>
## 日期序列化

<a name="customizing-the-default-date-format"></a>
#### 自訂預設日期格式

您可以透過覆寫 `serializeDate` 方法來自訂預設的序列化格式。此方法不影響日期在資料庫中的儲存格式：

    /**
     * 準備日期以進行陣列／JSON 序列化。
     */
    protected function serializeDate(DateTimeInterface $date): string
    {
        return $date->format('Y-m-d');
    }

<a name="customizing-the-date-format-per-attribute"></a>
#### 自訂每個屬性的日期格式

您可以透過在 model 的 [型別轉換宣告 (cast declarations)](/docs/{{version}}/eloquent-mutators#attribute-casting) 中指定日期格式，來自訂個別 Eloquent 日期屬性的序列化格式：

    protected function casts(): array
    {
        return [
            'birthday' => 'date:Y-m-d',
            'joined_at' => 'datetime:Y-m-d H:00',
        ];
    }