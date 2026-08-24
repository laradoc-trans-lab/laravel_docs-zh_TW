# 速率限制

- [簡介](#introduction)
    - [快取設定](#cache-configuration)
- [基本用法](#basic-usage)
    - [手動增加嘗試次數](#manually-incrementing-attempts)
    - [清除嘗試次數](#clearing-attempts)

<a name="introduction"></a>
## 簡介

Laravel 包含了一個易於使用的速率限制（Rate Limiting）抽象層，搭配應用程式的 [快取](cache)，能輕鬆限制在指定時間範圍內的任何操作。

> [!NOTE]
> 如果您想對傳入的 HTTP 請求進行速率限制，請參考 [速率限制器中介層文件](/docs/{{version}}/routing#rate-limiting)。

<a name="cache-configuration"></a>
### 快取設定

通常，速率限制器會使用應用程式 `cache` 設定檔中 `default` 鍵值所定義的預設應用程式快取。不過，您可以透過在應用程式的 `cache` 設定檔中定義 `limiter` 鍵值，來指定速率限制器應該使用的快取驅動器：

```php
'default' => env('CACHE_STORE', 'database'),

'limiter' => 'redis', // [tl! add]
```

<a name="basic-usage"></a>
## 基本用法

可以使用 `Illuminate\Support\Facades\RateLimiter` Facade 來與速率限制器進行互動。速率限制器提供的最簡單方法是 `attempt` 方法，該方法會在指定的秒數內限制給定回呼的執行速率。

當回呼沒有剩餘可用嘗試次數時，`attempt` 方法會傳回 `false`；否則，`attempt` 方法將傳回回呼的結果或 `true`。`attempt` 方法接收的第一個引數是速率限制器的「鍵值（key）」，可以是您選擇的任何代表被限制操作的字串：

```php
use Illuminate\Support\Facades\RateLimiter;

$executed = RateLimiter::attempt(
    'send-message:'.$user->id,
    $perMinute = 5,
    function() {
        // Send message...
    }
);

if (! $executed) {
    return 'Too many messages sent!';
}
```

如有需要，您可以為 `attempt` 方法提供第四個引數，即「衰減率（decay rate）」，或者說是可用嘗試次數重設前的秒數。例如，我們可以修改上述範例，允許每兩分鐘進行五次嘗試：

```php
$executed = RateLimiter::attempt(
    'send-message:'.$user->id,
    $perTwoMinutes = 5,
    function() {
        // Send message...
    },
    $decayRate = 120,
);
```

<a name="manually-incrementing-attempts"></a>
### 手動增加嘗試次數

若您想手動與速率限制器互動，還有其他多種方法可用。例如，您可以呼叫 `tooManyAttempts` 方法來判斷指定的速率限制鍵值是否已超過每分鐘允許的最大嘗試次數：

```php
use Illuminate\Support\Facades\RateLimiter;

if (RateLimiter::tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
    return 'Too many attempts!';
}

RateLimiter::increment('send-message:'.$user->id);

// Send message...
```

當對可能同時接收許多請求的端點進行速率限制時，您可能希望檢查 `increment` 方法傳回的值，而不是將 `tooManyAttempts` 與 `increment` 分開作為獨立操作。當使用 `redis`、`memcached` 或 `database` 快取儲存時，此值會以原子化方式遞增，確保每個同時發送的請求都能獲得唯一的計數值：

```php
use Illuminate\Support\Facades\RateLimiter;

$perMinute = 5;

if (RateLimiter::increment('send-message:'.$user->id) > $perMinute) {
    return 'Too many attempts!';
}

// Send message...
```

或者，您可以使用 `remaining` 方法取得指定鍵值剩餘的嘗試次數。若給定的鍵值仍有剩餘重試次數，您可以呼叫 `increment` 方法來增加總嘗試次數：

```php
use Illuminate\Support\Facades\RateLimiter;

if (RateLimiter::remaining('send-message:'.$user->id, $perMinute = 5)) {
    RateLimiter::increment('send-message:'.$user->id);

    // Send message...
}
```

如果您想將指定速率限制鍵值的值一次增加超過 1，可以向 `increment` 方法提供所需的數量：

```php
RateLimiter::increment('send-message:'.$user->id, amount: 5);
```

<a name="determining-limiter-availability"></a>
#### 判斷限制器的可用性

當某個鍵值沒有剩餘嘗試次數時，`availableIn` 方法會傳回距離下次可以使用嘗試次數所剩餘的秒數：

```php
use Illuminate\Support\Facades\RateLimiter;

if (RateLimiter::tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
    $seconds = RateLimiter::availableIn('send-message:'.$user->id);

    return 'You may try again in '.$seconds.' seconds.';
}

RateLimiter::increment('send-message:'.$user->id);

// Send message...
```

<a name="clearing-attempts"></a>
### 清除嘗試次數

您可以使用 `clear` 方法來重設指定速率限制鍵值的嘗試次數。例如，可以在接收者讀取特定訊息時重設嘗試次數：

```php
use App\Models\Message;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Mark the message as read.
 */
public function read(Message $message): Message
{
    $message->markAsRead();

    RateLimiter::clear('send-message:'.$message->user_id);

    return $message;
}
```