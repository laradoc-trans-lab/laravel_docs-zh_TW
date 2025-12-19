# 發行說明

- [版本控制方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 12](#laravel-12)

<a name="versioning-scheme"></a>
## 版本控制方案

Laravel 及其其他第一方套件遵循 [語意化版本控制](https://semver.org)。主要框架版本每年發布一次 (約第一季度)，而次要版本和補丁版本則可能每週發布一次。次要版本和補丁版本**絕不**應包含破壞性變更。

當從您的應用程式或套件中引用 Laravel 框架或其元件時，您應該始終使用版本約束，例如 `^12.0`，因為 Laravel 的主要版本確實包含破壞性變更。然而，我們力求始終確保您可以在一天或更短的時間內更新到新的主要版本。

<a name="named-arguments"></a>
#### 具名引數

[具名引數](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不屬於 Laravel 的向後相容性準則範圍。我們可能會在必要時重新命名函式引數，以改進 Laravel 程式碼庫。因此，在使用具名引數呼叫 Laravel 方法時應謹慎，並理解參數名稱未來可能會變更。

<a name="support-policy"></a>
## 支援政策

對於所有 Laravel 版本，錯誤修復提供 18 個月，安全修復提供 2 年。對於所有額外的函式庫，只有最新的主要版本會收到錯誤修復。此外，請查閱 [Laravel 支援的資料庫版本](/docs/{{version}}/database#introduction)。

<div class="overflow-auto">

| 版本 | PHP (*)   | 發行                | 錯誤修復至        | 安全修復至         |
| ---- |-----------| ------------------- | ------------------- | -------------------- |
| 10   | 8.1 - 8.3 | 2023 年 2 月 14 日  | 2024 年 8 月 6 日   | 2025 年 2 月 4 日    |
| 11   | 8.2 - 8.4 | 2024 年 3 月 12 日  | 2025 年 9 月 3 日   | 2026 年 3 月 12 日   |
| 12   | 8.2 - 8.5 | 2025 年 2 月 24 日  | 2026 年 8 月 13 日  | 2027 年 2 月 24 日   |
| 13   | 8.3 - 8.5 | 2026 年第一季度     | 2027 年第三季度     | 2028 年第一季度      |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>生命週期結束</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅提供安全修復</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-12"></a>
## Laravel 12

Laravel 12 透過更新上游依賴項並為 React、Vue 和 Livewire 引入新的應用程式入門套件，延續了 Laravel 11.x 所做的改進，其中包含使用 [WorkOS AuthKit](https://authkit.com) 進行使用者身份驗證的選項。我們入門套件的 WorkOS 變體提供了社群身份驗證、通行密鑰 (passkeys) 和 SSO 支援。

<a name="minimal-breaking-changes"></a>
### 最少破壞性變更

本次發行週期的重點是將破壞性變更降至最低。相反地，我們致力於全年提供持續性的品質改進，這些改進不會破壞現有應用程式。

因此，Laravel 12 版本是一個相對較小的「維護版本」，旨在升級現有依賴項。鑑於此，大多數 Laravel 應用程式無需更改任何應用程式程式碼即可升級到 Laravel 12。

<a name="new-application-starter-kits"></a>
### 新的應用程式入門套件

Laravel 12 為 React、Vue 和 Livewire 引入了新的 [應用程式入門套件](/docs/{{version}}/starter-kits)。React 和 Vue 入門套件利用了 Inertia 2、TypeScript、[shadcn/ui](https://ui.shadcn.com) 和 Tailwind，而 Livewire 入門套件則利用了基於 Tailwind 的 [Flux UI](https://fluxui.dev) 元件庫和 Laravel Volt。

React、Vue 和 Livewire 入門套件都利用了 Laravel 內建的身份驗證系統，提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們正在引入每個入門套件的 [WorkOS AuthKit 驅動](https://authkit.com) 變體，提供社群身份驗證、通行密鑰 (passkeys) 和 SSO 支援。WorkOS 為每月活躍使用者數達 100 萬的應用程式提供免費身份驗證。

隨著新的應用程式入門套件的推出，Laravel Breeze 和 Laravel Jetstream 將不再接收額外的更新。

若要開始使用我們新的入門套件，請查閱 [入門套件文件](/docs/{{version}}/starter-kits)。