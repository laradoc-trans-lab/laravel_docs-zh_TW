# 貢獻指南

- [錯誤回報](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [該使用哪個分支？](#which-branch)
- [編譯資產](#compiled-assets)
- [安全性漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤回報

為了鼓勵積極協作，Laravel 強烈建議提交 Pull request，而不僅僅是錯誤回報。Pull request 只有在標記為 "ready for review"（非 "draft" 狀態）且所有新功能的測試皆通過時才會進行審核。長期處於 "draft" 狀態且無活動的 Pull request 將在幾天後關閉。

然而，如果您提交錯誤回報，您的 Issue 應包含標題和清楚的問題描述。您還應盡可能包含所有相關資訊以及示範該問題的程式碼範例。錯誤回報的目標是讓您自己以及其他人能夠輕鬆地重現該錯誤並開發修復程式。

請記住，建立錯誤回報是希望遇到相同問題的其他人能夠與您協作解決。不要指望錯誤回報會自動引起關注或其他人會立即著手修復。建立錯誤回報是為了幫助您自己和他人開始修復問題。如果您想出一份力，可以透過修復[我們的問題追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)來提供協助。您必須通過 GitHub 身份驗證才能查看所有 Laravel 的 Issue。

如果您在使用 Laravel 時注意到不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub Issue。相反地，請提交 Pull request 來修復該問題。

Laravel 原始碼託管在 GitHub 上，且每個 Laravel 專案都有各自的儲存庫：

<div class="content-list" markdown="1">

- [Laravel Application](https://github.com/laravel/laravel)
- [Laravel Art](https://github.com/laravel/art)
- [Laravel Boost](https://github.com/laravel/boost)
- [Laravel Documentation](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravel Framework](https://github.com/laravel/framework)
- [Laravel Horizon](https://github.com/laravel/horizon)
- [Laravel Passport](https://github.com/laravel/passport)
- [Laravel Pennant](https://github.com/laravel/pennant)
- [Laravel Pint](https://github.com/laravel/pint)
- [Laravel Prompts](https://github.com/laravel/prompts)
- [Laravel Reverb](https://github.com/laravel/reverb)
- [Laravel Sail](https://github.com/laravel/sail)
- [Laravel Sanctum](https://github.com/laravel/sanctum)
- [Laravel Scout](https://github.com/laravel/scout)
- [Laravel Socialite](https://github.com/laravel/socialite)
- [Laravel Telescope](https://github.com/laravel/telescope)
- [Laravel Livewire Starter Kit](https://github.com/laravel/livewire-starter-kit)
- [Laravel React Starter Kit](https://github.com/laravel/react-starter-kit)
- [Laravel Svelte Starter Kit](https://github.com/laravel/svelte-starter-kit)
- [Laravel Vue Starter Kit](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub 問題追蹤器不提供 Laravel 的協助或支援。相反地，請使用以下管道：

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracasts Forums](https://laracasts.com/discuss)
- [Laravel.io Forums](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>


<a name="core-development-discussion"></a>
## 核心開發討論

您可以在 Laravel 框架儲存庫的 [GitHub 討論板](https://github.com/laravel/framework/discussions) 中提議新功能或對現有 Laravel 行為的改進。如果您提議新功能，請願意實作完成該功能所需的至少部分程式碼。

關於錯誤、新功能以及現有功能實作的非正式討論會在 [Laravel Discord 伺服器](https://discord.gg/laravel) 的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常會在工作日的早上 8 點至下午 5 點（UTC-06:00 或美國/芝加哥時間）出現在頻道中，其他時間則偶爾出現。


<a name="which-branch"></a>
## 該使用哪個分支？

**所有** 錯誤修復都應發送到支援錯誤修復的最新版本（目前為 `12.x`）。錯誤修復 **絕不** 應發送到 `master` 分支，除非它們修復的是僅存在於即將發布版本中的功能。

與當前版本 **完全向下相容** 的 **次要** 功能可以發送到最新的穩定分支（目前為 `12.x`）。

**重大** 的新功能或具有破壞性變更的功能應始終發送到 `master` 分支，其中包含即將發布的版本。


<a name="compiled-assets"></a>
## 編譯資產

如果您提交的變更會影響編譯後的檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 中的大部分檔案，請不要提交編譯後的檔案。由於這些檔案體積龐大，維護者實際上無法對其進行審核。這可能會被利用來將惡意程式碼注入到 Laravel 中。為了防禦性地防止這種情況，所有編譯後的檔案都將由 Laravel 維護者生成並提交。


<a name="security-vulnerabilities"></a>
## 安全性漏洞

如果您在 Laravel 中發現安全性漏洞，請發送電子郵件給 Taylor Otwell：<a href="mailto:taylor@laravel.com">taylor@laravel.com</a>。所有安全性漏洞都將得到及時處理。


<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 編碼標準和 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

下面是一個有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後跟著兩個空格、參數類型、另外兩個空格，最後是變數名稱：

```php
/**
 * Register a binding with the container.
 *
 * @param  string|array  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 *
 * @throws \Exception
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    // ...
}
```

當 `@param` 或 `@return` 屬性因使用原生類型而顯得冗餘時，可以將其刪除：

```php
/**
 * Execute the job.
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生類型是泛型時，請透過使用 `@param` 或 `@return` 屬性來指定泛型類型：

```php
/**
 * Get the attachments for the message.
 *
 * @return array<int, \Illuminate\Mail\Mailables\Attachment>
 */
public function attachments(): array
{
    return [
        Attachment::fromStorage('/path/to/file'),
    ];
}
```


<a name="styleci"></a>
### StyleCI

如果您的程式碼風格不夠完美，請不要擔心！[StyleCI](https://styleci.io/) 會在 Pull request 合併後自動將任何風格修復合併到 Laravel 儲存庫中。這讓我們能夠專注於貢獻的內容而非程式碼風格。

<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則衍生自 Ruby 的行為準則。任何違反行為準則的情況都可以回報給 Taylor Otwell (taylor@laravel.com)：

<div class="content-list" markdown="1">

- 參與者應包容反對意見。
- 參與者必須確保其言論和行為不含人身攻擊及貶低他人的評論。
- 在解讀他人的言行時，參與者應始終假設對方的出發點是良善的。
- 任何可被合理視為騷擾的行為都將不被容忍。

</div>