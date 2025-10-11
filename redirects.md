# HTTP 重新導向

- [建立重新導向](#creating-redirects)
- [重新導向至具名路由](#redirecting-named-routes)
- [重新導向至控制器動作](#redirecting-controller-actions)
- [重新導向並附帶閃存 Session 資料](#redirecting-with-flashed-session-data)

<a name="creating-redirects"></a>
## 建立重新導向

重新導向回應是 `Illuminate\Http\RedirectResponse` 類別的實例，並包含將使用者重新導向到另一個 URL 所需的正確標頭。有幾種方法可以產生 `RedirectResponse` 實例。最簡單的方法是使用全域 `redirect` 輔助函式：

    Route::get('/dashboard', function () {
        return redirect('/home/dashboard');
    });

有時您可能希望將使用者重新導向到他們之前的位置，例如當提交的表單無效時。您可以使用全域 `back` 輔助函式來達成此目的。由於此功能利用了 [session](/docs/{{version}}/session)，請確保呼叫 `back` 函式的路由使用了 `web` 中介層群組，或已套用了所有 session 中介層：

    Route::post('/user/profile', function () {
        // Validate the request...

        return back()->withInput();
    });

<a name="redirecting-named-routes"></a>
## 重新導向至具名路由

當您呼叫不帶參數的 `redirect` 輔助函式時，會回傳 `Illuminate\Routing\Redirector` 的實例，讓您可以在 `Redirector` 實例上呼叫任何方法。例如，要產生指向具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

    return redirect()->route('login');

如果您的路由有參數，您可以將它們作為第二個引數傳遞給 `route` 方法：

    // For a route with the following URI: profile/{id}

    return redirect()->route('profile', ['id' => 1]);

為方便起見，Laravel 也提供了全域 `to_route` 函式：

    return to_route('profile', ['id' => 1]);

<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent 模型填入參數

如果您要重新導向到帶有「ID」參數的路由，且該參數是從 Eloquent 模型填充的，您可以傳遞模型本身。ID 將會自動被提取：

    // For a route with the following URI: profile/{id}

    return redirect()->route('profile', [$user]);

如果您想自訂放置在路由參數中的值，您應該覆寫您的 Eloquent 模型上的 `getRouteKey` 方法：

    /**
     * Get the value of the model's route key.
     */
    public function getRouteKey(): mixed
    {
        return $this->slug;
    }

<a name="redirecting-controller-actions"></a>
## 重新導向至控制器動作

您也可以產生重新導向至 [控制器動作](/docs/{{version}}/controllers)。為此，請將控制器和動作名稱傳遞給 `action` 方法：

    use App\Http\Controllers\HomeController;

    return redirect()->action([HomeController::class, 'index']);

如果您的控制器路由需要參數，您可以將它們作為第二個引數傳遞給 `action` 方法：

    return redirect()->action(
        [UserController::class, 'profile'], ['id' => 1]
    );

<a name="redirecting-with-flashed-session-data"></a>
## 重新導向並附帶閃存 Session 資料

重新導向到新 URL 並 [將資料閃存到 session](/docs/{{version}}/session#flash-data) 通常是同時進行的。通常，這是在成功執行某個動作後，將成功訊息閃存到 session 時完成的。為方便起見，您可以透過單一、流暢的方法鏈來建立 `RedirectResponse` 實例並將資料閃存到 session：

    Route::post('/user/profile', function () {
        // Update the user's profile...

        return redirect('/dashboard')->with('status', 'Profile updated!');
    });

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重新導向到新位置之前，將當前請求的輸入資料閃存到 session。一旦輸入資料已閃存到 session，您可以在下一個請求期間輕鬆地 [擷取它](/docs/{{version}}/requests#retrieving-old-input)：

    return back()->withInput();

在使用者被重新導向後，您可以顯示從 [session](/docs/{{version}}/session) 中閃存的訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

    @if (session('status'))
        <div class="alert alert-success">
            {{ session('status') }}
        </div>
    @endif