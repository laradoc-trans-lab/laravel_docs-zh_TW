# 速率限制

- [簡介](#introduction)
    - [快取設定](#cache-configuration)
- [基本用法](#basic-usage)
    - [手動增加嘗試次數](#manually-incrementing-attempts)
    - [清除嘗試次數](#clearing-attempts)

<a name="introduction"></a>
## 簡介

Laravel 內建了一個簡單易用的速率限制抽象化功能，它與應用程式的 [快取](cache) 結合，提供了一種在指定時間範圍內限制任何操作的簡便方式。

> [!NOTE]
> 如果您對限制傳入的 HTTP 請求速率感興趣，請查閱 [速率限制器中介層文件](/docs/{{version}}/routing#rate-limiting)。

<a name="cache-configuration"></a>
### 快取設定

通常，速率限制器會使用您的預設應用程式快取，此快取由應用程式 `cache` 設定檔中的 `default` 鍵定義。然而，您可以透過在應用程式的 `cache` 設定檔中定義一個 `limiter` 鍵來指定速率限制器應使用的快取驅動：

```php
'default' => env('CACHE_STORE', 'database'),

'limiter' => 'redis', // [tl! add]
```

<a name="basic-usage"></a>
## 基本用法

您可以使用 `Illuminate\Support\Facades\RateLimiter` Facade 來與速率限制器互動。速率限制器提供的最簡單方法是 `attempt` 方法，它會在指定秒數內限制給定回呼的速率。

當回呼沒有剩餘嘗試次數時，`attempt` 方法會返回 `false`；否則，`attempt` 方法將返回回呼的結果或 `true`。`attempt` 方法接受的第一個引數是一個速率限制器「鍵」，它可以是您選擇的任何字串，代表正在被速率限制的操作：

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

如有需要，您可以為 `attempt` 方法提供第四個引數，即「衰減速率」，或直到可用嘗試次數重置的秒數。例如，我們可以修改上述範例，允許每兩分鐘五次嘗試：

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

如果您想手動與速率限制器互動，還有許多其他方法可用。例如，您可以呼叫 `tooManyAttempts` 方法來判斷給定的速率限制器鍵是否已超出每分鐘允許的最大嘗試次數：

```php
use Illuminate\Support\Facades\RateLimiter;

if (RateLimiter::tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
    return 'Too many attempts!';
}

RateLimiter::increment('send-message:'.$user->id);

// Send message...
```

或者，您可以使用 `remaining` 方法來檢索給定鍵剩餘的嘗試次數。如果給定鍵仍有剩餘重試次數，您可以呼叫 `increment` 方法來增加總嘗試次數：

```php
use Illuminate\Support\Facades\RateLimiter;

if (RateLimiter::remaining('send-message:'.$user->id, $perMinute = 5)) {
    RateLimiter::increment('send-message:'.$user->id);

    // Send message...
}
```

如果您想將給定速率限制器鍵的值增加超過一，您可以將所需的數量提供給 `increment` 方法：

```php
RateLimiter::increment('send-message:'.$user->id, amount: 5);
```

<a name="determining-limiter-availability"></a>
#### 判斷限制器可用性

當一個鍵不再有剩餘嘗試次數時，`availableIn` 方法會返回剩餘的秒數，直到有更多嘗試次數可用：

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

您可以使用 `clear` 方法來重設給定速率限制器鍵的嘗試次數。例如，當接收者讀取給定訊息時，您可以重設嘗試次數：

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