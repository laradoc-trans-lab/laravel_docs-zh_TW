# 貢獻指南

- [錯誤報告](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [使用哪個分支？](#which-branch)
- [編譯後的資源](#compiled-assets)
- [安全漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤報告

為鼓勵積極協作，Laravel 強烈建議提交 Pull Request，而不僅僅是錯誤報告。Pull Request 只有在標記為「ready for review」（非「draft」狀態）且所有新功能的測試均通過時才會進行審閱。那些處於「draft」狀態、懸而未決且不活躍的 Pull Request 將在幾天後被關閉。

但是，如果您提交錯誤報告，您的 Issue 應包含標題和對問題的清晰描述。您還應盡可能多地包含相關資訊以及演示問題的程式碼範例。錯誤報告的目標是讓您自己以及其他人能夠輕鬆重現該錯誤並開發出修復程式。

請記住，建立錯誤報告是希望其他有相同問題的人能夠與您協作解決問題。不要期望錯誤報告會自動收到任何活動，或其他人會立即著手修復。建立錯誤報告有助於您自己和他人開始解決問題。如果您想貢獻，可以透過修復[我們問題追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)來提供幫助。您必須透過 GitHub 驗證才能查看所有 Laravel 的 Issue。

如果您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請勿建立 GitHub Issue。相反，請提交 Pull Request 來修復問題。

Laravel 原始碼在 GitHub 上管理，每個 Laravel 專案都有其獨立的儲存庫：

<div class="content-list" markdown="1">

- [Laravel Application](https://github.com/laravel/laravel)
- [Laravel Art](https://github.com/laravel/art)
- [Laravel Breeze](https://github.com/laravel/breeze)
- [Laravel Documentation](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravel Framework](https://github.com/laravel/framework)
- [Laravel Homestead](https://github.com/laravel/homestead) ([Build Scripts](https://github.com/laravel/settler))
- [Laravel Horizon](https://github.com/laravel/horizon)
- [Laravel Jetstream](https://github.com/laravel/jetstream)
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
- [Laravel Website](https://github.com/laravel/laravel.com)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub Issue 追蹤器不旨在提供 Laravel 協助或支援。請改用以下任一管道：

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

您可以在 Laravel framework 儲存庫的 [GitHub discussion board](https://github.com/laravel/framework/discussions) 中提出新功能或改進現有 Laravel 行為的建議。如果您提出新功能，請願意實作完成該功能所需的部分程式碼。

有關錯誤、新功能和現有功能實作的非正式討論會在 [Laravel Discord server](https://discord.gg/laravel) 的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常在工作日（美國/芝加哥時間，UTC-06:00）上午 8 點至下午 5 點之間出現在頻道中，其他時間則會不定時出現。


<a name="which-branch"></a>
## 使用哪個分支？

**所有**錯誤修正都應發送至支援錯誤修正的最新版本（目前為 `11.x`）。錯誤修正**絕不**應發送至 `master` 分支，除非它們修正僅存在於即將發布版本中的功能。

與當前版本**完全向下相容**的**次要**功能可以發送至最新的穩定分支（目前為 `11.x`）。

**主要**新功能或包含破壞性變更的功能應始終發送至 `master` 分支，該分支包含即將發布的版本。


<a name="compiled-assets"></a>
## 編譯後的資源

如果您提交的變更會影響編譯檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 的大多數檔案，請勿提交編譯後的檔案。由於它們體積龐大，維護者無法實際審閱。這可能被利用作為向 Laravel 注入惡意程式碼的方式。為防禦性地防止這種情況，所有編譯後的檔案將由 Laravel 維護者生成並提交。


<a name="security-vulnerabilities"></a>
## 安全漏洞

如果您在 Laravel 中發現安全漏洞，請發送電子郵件至 Taylor Otwell (taylor@laravel.com)。所有安全漏洞都將及時處理。


<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 程式碼標準和 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 說明區塊範例。請注意，`@param` 屬性後跟兩個空格、參數型別、再兩個空格，最後是變數名稱：

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

當 `@param` 或 `@return` 屬性因使用原生型別而冗餘時，可以將其刪除：

    /**
     * Execute the job.
     */
    public function handle(AudioProcessor $processor): void
    {
        //
    }

但是，當原生型別是泛型時，請透過使用 `@param` 或 `@return` 屬性來指定泛型型別：

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


<a name="styleci"></a>
### StyleCI

如果您的程式碼風格不完美，請不必擔心！Pull Request 合併後，[StyleCI](https://styleci.io/) 會自動將任何風格修正合併到 Laravel 儲存庫中。這使我們能夠專注於貢獻的內容，而非程式碼風格。

<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則源自 Ruby 的行為準則。任何違反此行為準則的行為都可以向 Taylor Otwell (taylor@laravel.com) 舉報：

<div class="content-list" markdown="1">

- 參與者應容忍不同的觀點。
- 參與者必須確保他們的言行沒有人身攻擊和貶低他人的言論。
- 在詮釋他人的言行時，參與者應始終秉持善意。
- 任何可合理認定為騷擾的行為將不被容忍。

</div>