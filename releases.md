# 版本發布說明

- [版本控制方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 12](#laravel-12)

<a name="versioning-scheme"></a>
## 版本控制方案

Laravel 及其其他第一方套件遵循 [語義化版本控制](https://semver.org)。主要框架版本每年發布一次（約第一季），而次要版本和修補程式版本則可能每週發布。次要版本和修補程式版本**絕不**應包含破壞性變更。

當您從應用程式或套件中引用 Laravel 框架或其元件時，應始終使用版本約束，例如 `^12.0`，因為 Laravel 的主要版本確實包含破壞性變更。然而，我們始終努力確保您可以在一天或更短的時間內更新到新的主要版本。

<a name="named-arguments"></a>
#### 具名引數

[具名引數](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不受 Laravel 向下相容性準則的涵蓋。我們可能會在必要時選擇重新命名函式引數，以改進 Laravel 程式碼庫。因此，在呼叫 Laravel 方法時使用具名引數應謹慎，並理解參數名稱未來可能會變更。

<a name="support-policy"></a>
## 支援政策

對於所有 Laravel 版本，錯誤修復提供 18 個月，安全修復提供 2 年。對於所有額外的函式庫，只有最新的主要版本會收到錯誤修復。此外，請查閱 [Laravel 支援的資料庫版本](/docs/{{version}}/database#introduction)。

<div class="overflow-auto">

| 版本    | PHP (*)   | 發布日期            | 錯誤修復截止日期    | 安全修復截止日期   |
| ------- | --------- | ------------------- | ------------------- | ------------------ |
| 10      | 8.1 - 8.3 | 2023 年 2 月 14 日 | 2024 年 8 月 6 日   | 2025 年 2 月 4 日  |
| 11      | 8.2 - 8.4 | 2024 年 3 月 12 日 | 2025 年 9 月 3 日   | 2026 年 3 月 12 日 |
| 12      | 8.2 - 8.4 | 2025 年 2 月 24 日 | 2026 年 8 月 13 日  | 2027 年 2 月 24 日 |
| 13      | 8.3 - 8.4 | 2026 年第一季     | 2027 年第三季     | 2028 年第一季    |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>生命週期結束</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅限安全修復</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-12"></a>
## Laravel 12

Laravel 12 透過更新上游依賴項並引入適用於 React、Vue 和 Livewire 的新入門套件，包括使用 [WorkOS AuthKit](https://authkit.com) 進行使用者身份驗證的選項，延續了 Laravel 11.x 所做的改進。我們入門套件的 WorkOS 變體提供社群身份驗證、通行密鑰和 SSO 支援。

<a name="minimal-breaking-changes"></a>
### 最少破壞性變更

在此發布週期中，我們的大部分重點都放在將破壞性變更降至最低。相反，我們致力於全年提供持續的品質改進，這些改進不會破壞現有應用程式。

因此，Laravel 12 版本是一個相對較小的「維護版本」，旨在升級現有依賴項。有鑑於此，大多數 Laravel 應用程式無需更改任何應用程式程式碼即可升級到 Laravel 12。

<a name="new-application-starter-kits"></a>
### 新的應用程式入門套件

Laravel 12 引入了適用於 React、Vue 和 Livewire 的新 [應用程式入門套件](/docs/{{version}}/starter-kits)。React 和 Vue 入門套件使用 Inertia 2、TypeScript、[shadcn/ui](https://ui.shadcn.com) 和 Tailwind，而 Livewire 入門套件則使用基於 Tailwind 的 [Flux UI](https://fluxui.dev) 元件庫和 Laravel Volt。

React、Vue 和 Livewire 入門套件都利用 Laravel 內建的身份驗證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還引入了每個入門套件的 [WorkOS AuthKit 驅動](https://authkit.com) 變體，提供社群身份驗證、通行密鑰和 SSO 支援。WorkOS 為每月活躍使用者高達 100 萬的應用程式提供免費身份驗證。

隨著我們新應用程式入門套件的推出，Laravel Breeze 和 Laravel Jetstream 將不再接收額外更新。

要開始使用我們的新入門套件，請查閱 [入門套件說明文件](/docs/{{version}}/starter-kits)。
