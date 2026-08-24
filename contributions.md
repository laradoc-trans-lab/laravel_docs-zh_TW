# 貢獻指南

- [Bug 回報](#bug-reports)
- [技術支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [選擇哪個分支？](#which-branch)
- [編譯後的靜態資源](#compiled-assets)
- [AI 生成的貢獻](#ai-generated-contributions)
- [安全性漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## Bug 回報

為了鼓勵積極協作，Laravel 強烈建議提交 Pull Request，而不只是 Bug 回報。Pull Request 只有在標記為「ready for review」（非「draft」狀態）且所有新功能的測試均通過時才會進行審核。處於「draft」狀態且長時間無活動的 Pull Request 將在幾天後被關閉。

然而，若您提交 Bug 回報，您的 Issue 應包含標題與問題的清楚描述。您還應該盡可能附上所有相關資訊以及能夠重現該問題的程式碼範例。Bug 回報的目標是讓您自己以及其他人都能輕鬆重現該 Bug 並著手修復。

請記住，建立 Bug 回報是希望能與遇到相同問題的人一起協作解決。請不要預期 Bug 回報會自動受到關注或其他人會立刻著手修復。建立 Bug 回報的作用是幫助您自己和他人開啟解決問題的第一步。如果您想出一份力，可以透過修復[我們 Issue 追蹤器中列出的任何 Bug](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)來提供協助。您必須通過 GitHub 認證才能查看 Laravel 的所有 Issue。

若您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請勿建立 GitHub Issue。相反地，請提交 Pull Request 來修復該問題。

Laravel 原始碼託管於 GitHub，並且每個 Laravel 專案都有各自的儲存庫：

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
## 技術支援問題

Laravel 的 GitHub Issue 追蹤器並非用於提供 Laravel 的求助或技術支援。相反地，請使用以下管道之一：

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

您可以在 Laravel Framework 儲存庫的 [GitHub 討論區](https://github.com/laravel/framework/discussions)中提出新功能或針對既有 Laravel 行為的改進建議。如果您提出了一項新功能，請願意至少實作完成該功能所需的部分程式碼。

關於 Bug、新功能以及現有功能實作的非正式討論會在 [Laravel Discord 伺服器](https://discord.gg/laravel)的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常會在工作日的上午 8 點至下午 5 點（UTC-06:00 或美加中部時間 America/Chicago）出現在該頻道中，其他時間也會不定時出現。


<a name="which-branch"></a>
## 選擇哪個分支？

**所有** Bug 修復都應發送至支援 Bug 修復的最新版本（目前為 `13.x`）。除非修復的是僅存在於即將發布版本中的功能，否則**絕對不要**將 Bug 修復發送至 `master` 分支。

與目前版本**完全向下相容**的**次要**功能可以發送至最新的穩定分支（目前為 `13.x`）。

**主要**新功能或包含破壞性變更的功能應始終發送至 `master` 分支，該分支包含即將發布的版本。


<a name="compiled-assets"></a>
## 編譯後的靜態資源

如果您提交的變更會影響編譯後的檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 內的大多數檔案，請不要提交編譯後的檔案。由於檔案體積龐大，維護者實際上無法對其進行有效審查。這可能會被利用來向 Laravel 注入惡意程式碼。為了防禦此類問題，所有編譯後的檔案都將由 Laravel 維護者生成並提交。


<a name="ai-generated-contributions"></a>
## AI 生成的貢獻

我們感謝提交給 Laravel 的每一個 Pull Request。然而，未經深思熟慮的人工審查與考量、主要由 AI 生成的貢獻是不可接受的。

若您選擇使用 AI 工具來協助您的貢獻，在提交之前，您**必須**徹底審查、測試並理解所產生的程式碼。

**大量開啟完全由 AI 生成的 Issue 或 Pull Request 將不被允許。** 此類 Pull Request 將在未經審核的情況下直接關閉，且該貢獻者可能會被此儲存庫封鎖。

我們鼓勵貢獻者熟悉現有的程式碼庫、參與社群互動，並提交能體現自己對所解決問題的理解與審慎考量的 Pull Request。


<a name="security-vulnerabilities"></a>
## 安全性漏洞

如果您在 Laravel 中發現安全性漏洞，請發送電子郵件給我們的安全團隊：<a href="mailto:security@laravel.com">security@laravel.com</a>。所有安全性漏洞都將會被迅速處理。

<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 編碼標準與 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後面會接著兩個空格、引數型別、再兩個空格，最後則是變數名稱：

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

當原生型別已明確宣告使得 `@param` 或 `@return` 屬性顯得多餘時，可以將其移除：

```php
/**
 * Execute the job.
 * [tl! remove]
 * @return void [tl! remove]
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生型別為泛型時，請使用 `@param` 或 `@return` 屬性來明確指定泛型型別：

```php
/**
 * Get the attachments for the message.
 * [tl! add]
 * @return array<int, \Illuminate\Mail\Mailables\Attachment> [tl! add]
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

如果您的程式碼風格不夠完美，請別擔心！[StyleCI](https://styleci.io/) 會在 Pull Request 合併後，自動將任何風格修正合併至 Laravel 儲存庫中。這使我們能夠專注於貢獻的內容本身，而不是程式碼風格。


<a name="code-of-conduct"></a>
## 行為準則

Laravel 行為準則衍生自 Ruby 行為準則。任何違反行為準則的情況都可以向 Taylor Otwell（taylor@laravel.com）檢舉：

<div class="content-list" markdown="1">

- 參與者將包容不同的觀點。
- 參與者必須確保自己的言行不含有人身攻擊和貶損個人的言論。
- 在解讀他人的言語和行為時，參與者應始終抱持善意的出發點。
- 任何可被合理視為騷擾的行為都絕不容許。

</div>