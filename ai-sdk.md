# Laravel AI SDK

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [自訂基底 URL](#custom-base-urls)
    - [相容 OpenAI 的提供者](#openai-compatible-providers)
    - [提供者支援](#provider-support)
- [AI 代理](#agents)
    - [發送提示](#prompting)
    - [對話上下文](#conversation-context)
    - [結構化輸出](#structured-output)
    - [附件](#attachments)
    - [串流](#streaming)
    - [廣播](#broadcasting)
    - [佇列](#queueing)
    - [工具](#tools)
    - [檔案儲存工具](#file-storage-tools)
    - [MCP 工具](#mcp-tools)
    - [提供者工具](#provider-tools)
    - [子代理程式](#sub-agents)
    - [中介層](#middleware)
    - [匿名 AI 代理](#anonymous-agents)
    - [AI 代理設定](#agent-configuration)
    - [提供者選項](#provider-options)
- [人工工具核准](#human-tool-approval)
    - [完整的核准流程](#complete-approval-flow)
- [圖片](#images)
- [語音 (TTS)](#audio)
- [語音轉寫 (STT)](#transcription)
- [文字摘要](#text-summarization)
- [向量嵌入](#embeddings)
    - [多模態向量嵌入](#multimodal-embeddings)
    - [查詢向量嵌入](#querying-embeddings)
    - [快取向量嵌入](#caching-embeddings)
- [重新排序](#reranking)
- [檔案](#files)
- [向量儲存庫](#vector-stores)
    - [新增檔案至儲存庫](#adding-files-to-stores)
- [故障轉移](#failover)
- [測試](#testing)
    - [AI 代理](#testing-agents)
    - [圖片](#testing-images)
    - [語音](#testing-audio)
    - [語音轉寫](#testing-transcriptions)
    - [向量嵌入](#testing-embeddings)
    - [重新排序](#testing-reranking)
    - [檔案](#testing-files)
    - [向量儲存庫](#testing-vector-stores)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel AI SDK](https://github.com/laravel/ai) 提供了一個統一且表達力豐富的 API，用於與 OpenAI、Anthropic、Gemini 等 AI 提供者進行互動。透過 AI SDK，您可以建構帶有工具與結構化輸出的智慧 AI 代理、生成圖片、合成與轉寫語音、建立向量嵌入等等——這一切都使用一致且對 Laravel 友好的介面。


<a name="installation"></a>
## 安裝

您可以透過 Composer 安裝 Laravel AI SDK：

```shell
composer require laravel/ai
```

接下來，您應該使用 `vendor:publish` Artisan 指令發布 AI SDK 的設定檔與遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將會建立 `agent_conversations` 與 `agent_conversation_messages` 資料表，AI SDK 會用它們來儲存對話紀錄：

```shell
php artisan migrate
```


<a name="configuration"></a>
### 設定

您可以在應用程式的 `config/ai.php` 設定檔中定義 AI 提供者的憑證，或是定義為應用程式 `.env` 檔案中的環境變數：

```ini
ANTHROPIC_API_KEY=
AZURE_OPENAI_API_KEY=
COHERE_API_KEY=
DEEPSEEK_API_KEY=
ELEVENLABS_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
MISTRAL_API_KEY=
OLLAMA_API_KEY=
OPENAI_API_KEY=
OPENAI_COMPATIBLE_API_KEY=
OPENAI_COMPATIBLE_URL=
OPENROUTER_API_KEY=
JINA_API_KEY=
VOYAGEAI_API_KEY=
XAI_API_KEY=
```

用於文字、圖片、語音、轉寫與向量嵌入的預設模型也可以在應用程式的 `config/ai.php` 設定檔中進行設定。


<a name="custom-base-urls"></a>
### 自訂基底 URL

預設情況下，Laravel AI SDK 會直接連線至各個提供者的公開 API 端點。然而，您可能需要透過不同的端點來路由請求——例如使用代理服務來集中管理 API 金鑰、實作速率限制，或是透過企業網關來路由流量。

您可以在提供者設定中新增 `url` 參數來設定自訂基底 URL：

```php
'providers' => [
    'openai' => [
        'driver' => 'openai',
        'key' => env('OPENAI_API_KEY'),
        'url' => env('OPENAI_URL'),
    ],

    'anthropic' => [
        'driver' => 'anthropic',
        'key' => env('ANTHROPIC_API_KEY'),
        'url' => env('ANTHROPIC_BASE_URL'),
    ],
],
```

這在透過代理服務（例如 LiteLLM 或 Azure OpenAI Gateway）路由請求或使用替代端點時非常有用。

以下提供者支援自訂基底 URL：OpenAI、Anthropic、Gemini、Groq、Cohere、DeepSeek、xAI 和 OpenRouter。


<a name="openai-compatible-providers"></a>
### 相容 OpenAI 的提供者

如果您使用的是相容 OpenAI 的 API（例如 LM Studio、vLLM、Together、Fireworks 或本地網關），您可以設定一個 `openai-compatible` 提供者。`url` 選項是必填的，而 `key` 選項則是可選的，當存在時會作為 Bearer Token 發送：

```php
'providers' => [
    'local' => [
        'driver' => 'openai-compatible',
        'url' => env('LOCAL_AI_URL'),
        'key' => env('LOCAL_AI_API_KEY'),
    ],
],
```

設定完成後，您可以像使用任何其他提供者一樣使用這個具名提供者：

```php
agent()->prompt('What is Laravel?', provider: 'local', model: 'local-model');
```

您也可以為該提供者設定預設的文字模型，這樣您就無需顯式傳入模型：

```php
'local' => [
    'driver' => 'openai-compatible',
    'url' => env('LOCAL_AI_URL'),
    'key' => env('LOCAL_AI_API_KEY'),
    'models' => [
        'text' => [
            'default' => env('LOCAL_AI_MODEL'),
        ],
    ],
],
```

您可以透過在其設定中定義 `headers` 陣列，為該提供者的每個發出請求加入自訂 HTTP 標頭。當端點需要除了 Bearer Token 之外的額外識別或認證標頭時，這非常有用：

```php
'local' => [
    'driver' => 'openai-compatible',
    'url' => env('LOCAL_AI_URL'),
    'key' => env('LOCAL_AI_API_KEY'),
    'headers' => [
        'X-Tenant-Id' => env('LOCAL_AI_TENANT_ID'),
    ],
],
```

相容 OpenAI 的提供者支援文字生成、串流、工具、結構化輸出、圖片附件和向量嵌入。如果您的端點需要額外的請求內文欄位，請使用[提供者選項](#provider-options)來提供它們。


<a name="openai-compatible-embeddings"></a>
#### 相容 OpenAI 的向量嵌入

由於任意端點都沒有預知的模型，您必須設定預設的向量嵌入模型，才能在相容 OpenAI 的提供者上使用 `embeddings()`。您也可以設定固定的維度數值；若省略，發送請求時將不包含 `dimensions` 參數，並使用該模型的原生維度。

```php
'local' => [
    'driver' => 'openai-compatible',
    'url' => env('LOCAL_AI_URL'),
    'key' => env('LOCAL_AI_API_KEY'),
    'models' => [
        'embeddings' => [
            'default' => 'text-embedding-qwen3-embedding-0.6b',
            'dimensions' => 1024, // optional
        ],
    ],
],
```


<a name="provider-support"></a>
### 提供者支援

AI SDK 的各項功能支援多種提供者。下表總結了每個功能可用的提供者：

<div class="overflow-auto">

| 功能 | 提供者 |
|---|---|
| 文字 | OpenAI, OpenAI Compatible, Anthropic, Gemini, Azure, Bedrock, Groq, xAI, DeepSeek, Mistral, Ollama, OpenRouter |
| 圖片 | OpenAI, Gemini, xAI, Azure, Bedrock, OpenRouter |
| 語音 (TTS) | OpenAI, ElevenLabs, Gemini |
| 語音轉寫 (STT) | OpenAI, ElevenLabs, Mistral, Gemini |
| 向量嵌入 | OpenAI, OpenAI-Compatible, Gemini, Azure, Bedrock, Cohere, Mistral, Jina, VoyageAI, Ollama, OpenRouter |
| 重新排序 | Cohere, Jina, VoyageAI |
| 檔案 | OpenAI, Anthropic, Gemini, Azure |

</div>

您可以使用 `Laravel\Ai\Enums\Lab` 列舉在整個程式碼中引用提供者，而非使用純字串：

```php
use Laravel\Ai\Enums\Lab;

Lab::Anthropic;
Lab::OpenAI;
Lab::OpenAiCompatible;
Lab::Gemini;
// ...
```

<a name="agents"></a>
## AI 代理

AI 代理是在 Laravel AI SDK 中與 AI 提供者互動的基本建構區塊。每個 AI 代理都是一個專用的 PHP 類別，封裝了與大型語言模型互動所需的指示、對話上下文、工具與輸出 Schema。您可以將 AI 代理想像成一個專門的助手——銷售教練、文件分析師、客服機器人——您只需要設定一次，就能在整個應用程式中依需求向其發送提示。

您可以使用 `make:agent` Artisan 指令建立 AI 代理：

```shell
php artisan make:agent SalesCoach

php artisan make:agent SalesCoach --structured
```

在產生的 AI 代理類別中，您可以定義系統提示詞 / 指示、訊息上下文、可用工具以及輸出 Schema（若適用）：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Tools\RetrievePreviousTranscripts;
use App\Models\History;
use App\Models\User;
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Messages\Message;
use Laravel\Ai\Promptable;
use Stringable;

class SalesCoach implements Agent, Conversational, HasTools, HasStructuredOutput
{
    use Promptable;

    public function __construct(public User $user) {}

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): Stringable|string
    {
        return 'You are a sales coach, analyzing transcripts and providing feedback and an overall sales strength score.';
    }

    /**
     * Get the list of messages comprising the conversation so far.
     */
    public function messages(): iterable
    {
        return History::where('user_id', $this->user->id)
            ->latest()
            ->limit(50)
            ->get()
            ->reverse()
            ->map(function ($message) {
                return new Message($message->role, $message->content);
            })->all();
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new RetrievePreviousTranscripts,
        ];
    }

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'feedback' => $schema->string()->required(),
            'score' => $schema->integer()->min(1)->max(10)->required(),
        ];
    }
}
```

<a name="prompting"></a>
### 發送提示

要向 AI 代理發送提示，請先使用 `make` 方法或標準的實例化方式建立實例，然後呼叫 `prompt`：

```php
$response = (new SalesCoach)
    ->prompt('Analyze this sales transcript...');

return (string) $response;
```

`make` 方法會從容器解析您的 AI 代理，實現自動依賴注入。您也可以將引數傳遞給 AI 代理的建構子：

```php
$agent = SalesCoach::make(user: $user);
```

透過傳遞額外的引數給 `prompt` 方法，您可以在發送提示時覆寫預設的提供者、模型或 HTTP 逾時時間：

```php
$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: Lab::Anthropic,
    model: 'claude-sonnet-5',
    timeout: 120,
);
```

<a name="raw-http-responses"></a>
#### 原始 HTTP 回應

從文字生成 AI 代理回傳的每個回應，都會透過 `raw` 屬性公開來自底層提供者 API 呼叫的原始 HTTP 回應。這讓您可以存取不屬於 AI SDK 通用回應中的特定提供者資訊——速率限制標頭 (rate-limit headers)、請求 ID 或其他確切的 Payload 欄位：

```php
$response = (new SalesCoach)->prompt('Analyze this sales transcript...');

$response->raw; // Illuminate\Http\Client\Response|null

$response->raw->header('X-RateLimit-Remaining-Requests');
$response->raw->json('id');
```

在工具呼叫的迴圈中，每個步驟都會保留其自身請求的原始回應：

```php
foreach ($response->steps as $step) {
    $step->raw?->header('X-RateLimit-Remaining-Requests');
}
```

> **Note：** 當串流回應、使用 Bedrock 提供者（該提供者是透過 AWS SDK 而非 HTTP 客戶端執行 API 呼叫），或是使用模擬 (Faked) 回應時，`raw` 屬性皆為 `null`，除非透過 `withRawResponse` 明確提供。

<a name="conversation-context"></a>
### 對話上下文

若你的 AI 代理實作了 `Conversational` 介面，你可以使用 `messages` 方法來回傳先前的對話上下文（若適用）：

```php
use App\Models\History;
use Laravel\Ai\Messages\Message;

/**
 * Get the list of messages comprising the conversation so far.
 */
public function messages(): iterable
{
    return History::where('user_id', $this->user->id)
        ->latest()
        ->limit(50)
        ->get()
        ->reverse()
        ->map(function ($message) {
            return new Message($message->role, $message->content);
        })->all();
}
```

<a name="remembering-conversations"></a>
#### 記憶對話

> **警告：**在使用 `RemembersConversations` Trait 之前，你應該使用 `vendor:publish` Artisan 指令發布並執行 AI SDK 的資料庫遷移。這些遷移將會建立儲存對話所需的資料庫資料表。

若你希望 Laravel 自動為你的 AI 代理儲存與檢索對話紀錄，可以使用 `RemembersConversations` Trait。此 Trait 提供了一種簡單的方式來將對話訊息持久化儲存至資料庫，而無需手動實作 `Conversational` 介面：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Concerns\RemembersConversations;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, Conversational
{
    use Promptable, RemembersConversations;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You are a sales coach...';
    }
}
```

使用 `RemembersConversations` Trait 時，請勿在你的 AI 代理類別中手動定義 `messages` 方法。如果存在 `messages` 方法，它的優先權會高於 Trait 的實作，導致對話紀錄無法從資料庫載入。

要為使用者開始一個新的對話，請在發送提示詞前呼叫 `forUser` 方法：

```php
$response = (new SalesCoach)->forUser($user)->prompt('Hello!');

$conversationId = $response->conversationId;
```

對話 ID 會包含在回應中回傳，你可以將其儲存起來以供日後參考。若你想使用 Eloquent 取得使用者的所有對話，可以將 `HasConversations` Trait 新增至你的 User Model：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Ai\Concerns\HasConversations;

class User extends Authenticatable
{
    use HasConversations;
}
```

將該 Trait 新增至 Model 後，你就可以透過 `conversations` 關聯來取得與查詢該使用者的對話：

```php
$conversations = $user->conversations()
    ->latest('updated_at')
    ->paginate(20);
```

若要繼續現有的對話，請使用 `continue` 方法：

```php
$response = (new SalesCoach)
    ->continue($conversationId, as: $user)
    ->prompt('Tell me more about that.');
```

使用 `RemembersConversations` Trait 時，發送提示詞時會自動載入先前的訊息並包含在對話上下文之中。每次互動後，新訊息（包含使用者與助理的訊息）都會自動儲存。

<a name="conversation-participants"></a>
#### 對話參與者

雖然使用者是最常見的對話參與者，但對話也可以屬於任何 Eloquent Model。請使用 `forParticipant` 方法為其他型態的 Model 開始對話：

```php
$response = (new SalesCoach)
    ->forParticipant($team)
    ->prompt('Review our latest sales results.');
```

參與者的多型類別 (morph class) 和主鍵 (primary key) 會與對話一同儲存。因此，具有相同主鍵的不同型態 Model（例如 `User` ID 為 `1` 與 `Team` ID 為 `1`）將擁有獨立的對話紀錄。`forUser` 方法是 `forParticipant` 的別名。

你可以使用 `continueLastConversation` 方法來繼續參與者最新的對話：

```php
$response = (new SalesCoach)
    ->continueLastConversation($team)
    ->prompt('Tell me more about that.');
```

繼續特定對話時，請將參與者傳遞給 `continue` 方法：

```php
$response = (new SalesCoach)
    ->continue($conversationId, as: $team)
    ->prompt('Tell me more about that.');
```

`HasConversations` Trait 可以新增至任何參與對話的 Eloquent Model。所產生的 `conversations` 關聯是一個多型關聯 (Polymorphic relationship)，作用域限定於該 Model 的型態與主鍵。你也可以透過反向關聯存取擁有該對話的參與者：

```php
$conversations = $team->conversations;

$participant = $conversation->participant;
```

如果你的應用程式使用多種參與者 Model 型態，你應該考慮定義 [Eloquent 多型對映 (Morph Map)](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，如此一來儲存的參與者型態就不會與你的 Model 類別名稱直接耦合。

> [!WARNING]
> `continue` 方法不會驗證給定的參與者是否擁有該對話。你的應用程式應在繼續該對話之前先對存取權限進行授權。

<a name="structured-output"></a>
### 結構化輸出

如果您希望 AI 代理回傳結構化輸出，請實作 `HasStructuredOutput` 介面，這需要您的 AI 代理定義一個 `schema` 方法：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
        ];
    }
}
```

當對回傳結構化輸出的 AI 代理發送提示詞時，您可以像使用陣列一樣存取回傳的 `StructuredAgentResponse`：

```php
$response = (new SalesCoach)->prompt('Analyze this sales transcript...');

return $response['score'];
```

<a name="structured-output-nested-objects"></a>
#### 巢狀物件

若要定義巢狀的結構化輸出，請將 `object` 方法搭配閉包使用：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
            'metadata' => $schema->object(fn ($schema) => [
                'confidence' => $schema->string()->enum(['low', 'medium', 'high'])->required(),
                'language' => $schema->string()->required(),
            ])->required(),
        ];
    }
}
```

<a name="structured-output-arrays-of-objects"></a>
#### 物件陣列

如果您的 AI 代理應該回傳結構化項目的清單，請結合使用 `array` 與 `object` 方法：

```php
public function schema(JsonSchema $schema): array
{
    return [
        'feedback' => $schema->array()
            ->items(
                $schema->object(fn ($schema) => [
                    'comment' => $schema->string()->required(),
                    'score' => $schema->integer()->required(),
                ])
            )
            ->required(),
    ];
}
```

若數值可能符合多個結構定義的其中一個，請使用 `anyOf` 方法：

```php
public function schema(JsonSchema $schema): array
{
    return [
        'content' => $schema->anyOf([
            $schema->object(fn ($schema) => [
                'type' => $schema->string()->enum(['article'])->required(),
                'title' => $schema->string()->required(),
            ]),
            $schema->object(fn ($schema) => [
                'type' => $schema->string()->enum(['image'])->required(),
                'url' => $schema->string()->required(),
            ]),
        ])->required(),
    ];
}
```

<a name="attachments"></a>
### 附件

在發送提示時，您也可以隨提示詞傳遞附件，讓模型檢查圖片與文件：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...',
    attachments: [
        Files\Document::fromStorage('transcript.pdf'), // Attach a document from a filesystem disk...
        Files\Document::fromPath('/home/laravel/transcript.md'), // Attach a document from a local path...
        $request->file('transcript'), // Attach an uploaded file...
    ]
);
```

同樣地，`Laravel\Ai\Files\Image` 類別可用於將圖片附加至提示詞中：

```php
use App\Ai\Agents\ImageAnalyzer;
use Laravel\Ai\Files;

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Files\Image::fromStorage('photo.jpg'), // Attach an image from a filesystem disk...
        Files\Image::fromPath('/home/laravel/photo.jpg'), // Attach an image from a local path...
        $request->file('photo'), // Attach an uploaded file...
    ]
);
```

<a name="streaming"></a>
### 串流

您可以透過呼叫 `stream` 方法來串流 AI 代理的回應。傳回的 `StreamableAgentResponse` 可以直接從路由回傳，以自動傳送串流回應 (SSE) 給用戶端：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)->stream('Analyze this sales transcript...');
});
```

`then` 方法可用於提供一個閉包，該閉包將在整個回應完整串流傳送至用戶端後被呼叫：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Responses\StreamedAgentResponse;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->then(function (StreamedAgentResponse $response) {
            // $response->text, $response->events, $response->usage...
        });
});
```

或者，您也可以手動巡覽串流事件：

```php
$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    // ...
}
```

<a name="streaming-using-the-vercel-ai-sdk-protocol"></a>
#### 使用 Vercel AI SDK 協定進行串流

您可以在可串流回應上呼叫 `usingVercelDataProtocol` 方法，以使用 [Vercel AI SDK 串流協定](https://ai-sdk.dev/docs/ai-sdk-ui/stream-protocol) 串流傳送事件：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->usingVercelDataProtocol();
});
```

<a name="broadcasting"></a>
### 廣播

您可以透過幾種不同的方式廣播串流事件。首先，您可以直接在串流事件上呼叫 `broadcast` 或 `broadcastNow` 方法：

```php
use App\Ai\Agents\SalesCoach;
use Illuminate\Broadcasting\Channel;

$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    $event->broadcast(new Channel('channel-name'));
}
```

或者，您可以呼叫 AI 代理的 `broadcastOnQueue` 方法，將 AI 代理操作放入佇列，並在串流事件可用時將其廣播出去：

```php
(new SalesCoach)->broadcastOnQueue(
    'Analyze this sales transcript...'
    new Channel('channel-name'),
);
```

<a name="skipping-oversized-events"></a>
#### 略過過大的事件

某些廣播平台將 WebSocket 訊息限制在大約 10KB 左右。資料量較大的串流事件（例如大型工具的執行結果）可能會超出此限制並導致廣播失敗。您可以使用 `WithoutBroadcasting` 屬性將特定的事件類型從廣播中排除：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Attributes\WithoutBroadcasting;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Promptable;
use Laravel\Ai\Streaming\Events\ToolCall;
use Laravel\Ai\Streaming\Events\ToolResult;

#[WithoutBroadcasting(ToolCall::class, ToolResult::class)]
class SearchAgent implements Agent, HasTools
{
    use Promptable;

    // ...
}
```

被排除的事件永遠不會進行廣播，但它們仍會被持久化儲存至 `agent_conversation_messages` 資料表中，因此您的前端可以在串流完成後載入完整的工具資料。這對於佇列廣播 (`broadcastOnQueue`) 和同步廣播 (`broadcast` / `broadcastNow`) 皆能正常運作。

<a name="queueing"></a>
### 佇列

使用 AI 代理的 `queue` 方法，您可以向代理發送提示詞，但允許它在背景處理回應，從而保持您的應用程式運作迅速且流暢。`then` 和 `catch` 方法可用於註冊閉包 (Closure)，這些閉包會在收到回應或發生例外狀況時被呼叫：

```php
use Illuminate\Http\Request;
use Laravel\Ai\Responses\AgentResponse;
use Throwable;

Route::post('/coach', function (Request $request) {
    (new SalesCoach)
        ->queue($request->input('transcript'))
        ->then(function (AgentResponse $response) {
            // ...
        })
        ->catch(function (Throwable $e) {
            // ...
        });

    return back();
});
```

<a name="tools"></a>
### 工具

工具可用於賦予 AI 代理額外功能，使其在回應提示詞時可以使用。可以使用 `make:tool` Artisan 指令建立工具：

```shell
php artisan make:tool RandomNumberGenerator
```

產生的工具將會放置在您應用程式的 `app/Ai/Tools` 目錄中。每個工具都包含一個 `handle` 方法，當 AI 代理需要使用該工具時就會呼叫此方法：

```php
<?php

namespace App\Ai\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Tools\Request;
use Stringable;

class RandomNumberGenerator implements Tool
{
    /**
     * Get the description of the tool's purpose.
     */
    public function description(): Stringable|string
    {
        return 'This tool may be used to generate cryptographically secure random numbers.';
    }

    /**
     * Execute the tool.
     */
    public function handle(Request $request): Stringable|string
    {
        return (string) random_int($request['min'], $request['max']);
    }

    /**
     * Get the tool's schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'min' => $schema->integer()->min(0)->required(),
            'max' => $schema->integer()->required(),
        ];
    }
}
```

當您定義好工具後，便可以從任何 AI 代理的 `tools` 方法中傳回它：

```php
use App\Ai\Tools\RandomNumberGenerator;

/**
 * Get the tools available to the agent.
 *
 * @return Tool[]
 */
public function tools(): iterable
{
    return [
        new RandomNumberGenerator,
    ];
}
```

<a name="similarity-search"></a>
#### 相似度搜尋

`SimilaritySearch` 工具允許 AI 代理使用儲存在資料庫中的向量嵌入，搜尋與指定查詢相似的文件。當您想要讓 AI 代理能夠搜尋您應用程式的資料時，這對於檢索增強生成 (Retrieval-Augmented Generation，RAG) 非常有用。

建立相似度搜尋工具最簡單的方式，就是搭配具有向量嵌入的 Eloquent 模型使用 `usingModel` 方法：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        SimilaritySearch::usingModel(Document::class, 'embedding'),
    ];
}
```

第一個引數是 Eloquent 模型類別，第二個引數則是包含向量嵌入的欄位。

您也可以提供介於 `0.0` 和 `1.0` 之間的最低相似度門檻，以及一個閉包來自訂查詢：

```php
SimilaritySearch::usingModel(
    model: Document::class,
    column: 'embedding',
    minSimilarity: 0.7,
    limit: 10,
    query: fn ($query) => $query->where('published', true),
),
```

若需要更多控制，您可以使用傳回搜尋結果的自訂閉包來建立相似度搜尋工具：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        new SimilaritySearch(using: function (string $query) {
            return Document::query()
                ->where('user_id', $this->user->id)
                ->whereVectorSimilarTo('embedding', $query)
                ->limit(10)
                ->get();
        }),
    ];
}
```

您可以使用 `withDescription` 方法自訂工具的描述：

```php
SimilaritySearch::usingModel(Document::class, 'embedding')
    ->withDescription('Search the knowledge base for relevant articles.'),
```

<a name="file-storage-tools"></a>
### 檔案儲存工具

`FileStorage` 工具工廠允許您賦予 AI 代理存取 Laravel [檔案系統磁碟](/docs/{{version}}/filesystem) 的權限。`all` 方法會傳回一系列工具，允許 AI 代理在指定的磁碟上進行列出、讀取、檢視、產生 URL、寫入、刪除以及複製檔案等操作：

```php
use Laravel\Ai\Tools\FileStorage;

public function tools(): iterable
{
    return FileStorage::all('local');
}
```

如果您的 AI 代理應該只能檢視檔案，請使用 `readOnly` 方法：

```php
return FileStorage::readOnly('local');
```

這些方法會傳回一個 `Illuminate\Support\Collection`，允許您進一步過濾提供給 AI 代理的工具：

```php
use Laravel\Ai\Tools\Filesystem\DeleteFile;

return FileStorage::all('s3')
    ->reject(fn ($tool) => $tool instanceof DeleteFile);
```

<a name="mcp-tools"></a>
### MCP 工具

如果您的應用程式使用 [Laravel MCP](/docs/{{version}}/mcp)，您可以向 AI 代理提供由 [模型上下文協議 (Model Context Protocol)](https://modelcontextprotocol.io) 伺服器所暴露的工具。使用 [Laravel MCP 用戶端](/docs/{{version}}/mcp#client)，您可以連線至遠端或本機 MCP 伺服器，並將其工具直接傳遞給您的 AI 代理。

> [!NOTE]
> MCP 工具需要您的應用程式中安裝 [Laravel MCP](/docs/{{version}}/mcp) 套件。

由於 MCP 用戶端的 `tools` 方法會傳回一個集合 (Collection)，請使用 `...` 運算子將其展開到 AI 代理的 `tools` 陣列中：

```php
use App\Ai\Tools\RandomNumberGenerator;
use Laravel\Mcp\Client;

/**
 * Get the tools available to the agent.
 *
 * @return Tool[]
 */
public function tools(): iterable
{
    return [
        ...Client::web('https://mcp.example.com')
            ->withToken($token)
            ->tools(),

        new RandomNumberGenerator,
    ];
}
```

AI SDK 會自動包裝每個 MCP 工具，因此 AI 代理可以像呼叫其他任何工具一樣呼叫它。您也可以使用 [具名 MCP 用戶端](/docs/{{version}}/mcp#named-clients)：

```php
use Laravel\Mcp\Facades\Mcp;

public function tools(): iterable
{
    return [
        ...Mcp::client('github')->tools(),
    ];
}
```

或連線至 [本機 MCP 伺服器](/docs/{{version}}/mcp#client-connecting)：

```php
use Laravel\Mcp\Client;

public function tools(): iterable
{
    return [
        ...Client::local('php', ['artisan', 'mcp:start'])->tools(),
    ];
}
```

關於建立和驗證 MCP 用戶端（包含 Bearer 令牌與 OAuth）的更多資訊，請參考 [MCP 用戶端文件](/docs/{{version}}/mcp#client)。

<a name="provider-tools"></a>
### 提供者工具

提供者工具是由 AI 提供者原生實作的特殊工具，提供網頁搜尋、URL 擷取和檔案搜尋等功能。與一般工具不同的是，提供者工具是由提供者本身執行，而非您的應用程式。

提供者工具可以由 AI 代理的 `tools` 方法回傳。


<a name="web-search"></a>
#### 網路搜尋

`WebSearch` 提供者工具允許 AI 代理在網路上搜尋即時資訊。這對於回答有關時事、近期資料或自模型訓練截止以來可能已變更的主題等問題非常有用。

**支援的提供者：** Anthropic, OpenAI, Azure, Gemini, OpenRouter

```php
use Laravel\Ai\Providers\Tools\WebSearch;

public function tools(): iterable
{
    return [
        new WebSearch,
    ];
}
```

您可以設定網路搜尋工具來限制搜尋次數，或是將結果限制在特定網域：

```php
(new WebSearch)->max(5)->allow(['laravel.com', 'php.net']),
```

若要根據使用者位置精確化搜尋結果，請使用 `location` 方法：

```php
(new WebSearch)->location(
    city: 'New York',
    region: 'NY',
    country: 'US'
);
```


<a name="web-fetch"></a>
#### 網路擷取

`WebFetch` 提供者工具允許 AI 代理擷取並讀取網頁內容。當您需要 AI 代理分析特定的 URL 或從已知網頁檢索詳細資訊時，這非常有用。

**支援的提供者：** Anthropic, Gemini

```php
use Laravel\Ai\Providers\Tools\WebFetch;

public function tools(): iterable
{
    return [
        new WebFetch,
    ];
}
```

您可以設定網路擷取工具來限制擷取次數或限制特定網域：

```php
(new WebFetch)->max(3)->allow(['docs.laravel.com']),
```


<a name="file-search"></a>
#### 檔案搜尋

`FileSearch` 提供者工具允許 AI 代理搜尋儲存在[向量儲存庫](#vector-stores)中的[檔案](#files)。這允許 AI 代理在您上傳的文件中搜尋相關資訊，從而實現檢索增強生成 (RAG)。

**支援的提供者：** OpenAI, Gemini

```php
use Laravel\Ai\Providers\Tools\FileSearch;

public function tools(): iterable
{
    return [
        new FileSearch(stores: ['store_id']),
    ];
}
```

您可以提供多個向量儲存庫 ID，以跨多個儲存庫進行搜尋：

```php
new FileSearch(stores: ['store_1', 'store_2']);
```

如果您的檔案帶有[元資料 (metadata)](#adding-files-to-stores)，您可以透過提供 `where` 引數來過濾搜尋結果。對於簡單的相等性過濾，請傳入一個陣列：

```php
new FileSearch(stores: ['store_id'], where: [
    'author' => 'Taylor Otwell',
    'year' => 2026,
]);
```

對於更複雜的過濾，您可以傳入接收 `FileSearchQuery` 實例的閉包：

```php
use Laravel\Ai\Providers\Tools\FileSearchQuery;

new FileSearch(stores: ['store_id'], where: fn (FileSearchQuery $query) =>
    $query->where('author', 'Taylor Otwell')
        ->whereNot('status', 'draft')
        ->whereIn('category', ['news', 'updates'])
);
```


<a name="sub-agents"></a>
### 子代理程式

AI 代理也可以從另一個 AI 代理的 `tools` 方法回傳。當 AI 代理作為工具回傳時，父 AI 代理可以將特定任務委派給子代理程式，並在回答原始提示詞時使用子代理程式的回應。當通用 AI 代理需要存取擁有專屬指令、工具、模型設定或提供者偏好的專門 AI 代理時，這非常有用。

例如，客戶支援 AI 代理可以將退款資格問題委派給專門的退款 AI 代理：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Promptable;

class CustomerSupportAgent implements Agent, HasTools
{
    use Promptable;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You help customers with account, order, and billing questions. Delegate refund policy questions to the refunds specialist.';
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new RefundsAgent,
        ];
    }
}
```

若要自訂子代理程式如何公開給父 AI 代理，請在子代理程式上實作 `CanActAsTool` 介面，並定義面向工具的名稱與說明：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Tools\LookupOrder;
use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\CanActAsTool;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)]
class RefundsAgent implements Agent, CanActAsTool, HasTools
{
    use Promptable;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You are a refunds specialist. Use order details and the refund policy to give concise eligibility guidance.';
    }

    /**
     * Get the agent's tool name.
     */
    public function name(): string
    {
        return 'refunds_specialist';
    }

    /**
     * Get the agent's tool description.
     */
    public function description(): string
    {
        return 'Determine whether an order is eligible for a refund and explain the next step.';
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new LookupOrder,
        ];
    }
}
```

如果子代理程式未實作 `CanActAsTool`，Laravel 將使用該 AI 代理的類別基礎名稱 (basename) 作為工具名稱，並使用通用說明要求父 AI 代理傳遞清晰且獨立完整的任務說明。每次對子代理程式的呼叫都是獨立執行的，並且不會收到父 AI 代理的對話歷史紀錄。


<a name="middleware"></a>
### 中介層

AI 代理支援中介層，允許您在提示詞傳送到提供者之前進行攔截與修改。中介層可以使用 `make:agent-middleware` Artisan 指令建立：

```shell
php artisan make:agent-middleware LogPrompts
```

產生的中介層將放置在您應用程式的 `app/Ai/Middleware` 目錄中。若要將中介層新增至 AI 代理，請實作 `HasMiddleware` 介面並定義回傳中介層類別陣列的 `middleware` 方法：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Middleware\LogPrompts;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasMiddleware;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasMiddleware
{
    use Promptable;

    // ...

    /**
     * Get the agent's middleware.
     */
    public function middleware(): array
    {
        return [
            new LogPrompts,
        ];
    }
}
```

每個中介層類別都應定義一個 `handle` 方法，該方法接收 `AgentPrompt` 和一個將提示詞傳遞給下一個中介層的 `Closure`：

```php
<?php

namespace App\Ai\Middleware;

use Closure;
use Laravel\Ai\Prompts\AgentPrompt;

class LogPrompts
{
    /**
     * Handle the incoming prompt.
     */
    public function handle(AgentPrompt $prompt, Closure $next)
    {
        Log::info('Prompting agent', ['prompt' => $prompt->prompt]);

        return $next($prompt);
    }
}
```

您可以在回應上使用 `then` 方法，以便在 AI 代理處理完成後執行程式碼。這適用於同步與串流回應：

```php
public function handle(AgentPrompt $prompt, Closure $next)
{
    return $next($prompt)->then(function (AgentResponse $response) {
        Log::info('Agent responded', ['text' => $response->text]);
    });
}
```

<a name="anonymous-agents"></a>
### 匿名 AI 代理

有時您可能想快速與模型進行互動，而不需要建立專屬的 AI 代理類別。您可以使用 `agent` 函式建立一個臨時的匿名 AI 代理：

```php
use function Laravel\Ai\{agent};

$response = agent(
    instructions: 'You are an expert at software development.',
    messages: [],
    tools: [],
)->prompt('Tell me about Laravel')
```

匿名 AI 代理也可以產生結構化輸出：

```php
use Illuminate\Contracts\JsonSchema\JsonSchema;

use function Laravel\Ai\{agent};

$response = agent(
    schema: fn (JsonSchema $schema) => [
        'number' => $schema->integer()->required(),
    ],
)->prompt('Generate a random number less than 100')
```


<a name="agent-configuration"></a>
### AI 代理設定

您可以使用 PHP 屬性（Attributes）來設定 AI 代理的文字產生選項。有以下屬性可供使用：

- `MaxSteps`：AI 代理在使用工具時可以執行的最大步驟數。
- `MaxTokens`：模型可以產生的最大 token 數量。
- `Model`：AI 代理應該使用的模型。
- `Provider`：AI 代理所使用的 AI 提供者（或用於故障轉移的提供者群）。
- `Temperature`：用於產生的取樣溫度（0.0 至 1.0）。
- `Timeout`：AI 代理請求的 HTTP 逾時時間（以秒為單位，預設值：60）。
- `TopP`：用於產生的核取樣（Nucleus sampling）機率（0.0 至 1.0）。
- `UseCheapestModel`：使用提供者最便宜的文字模型，以進行成本最佳化。
- `UseSmartestModel`：使用提供者能力最強的文字模型，以處理複雜任務。

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Attributes\MaxSteps;
use Laravel\Ai\Attributes\MaxTokens;
use Laravel\Ai\Attributes\Model;
use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Attributes\Temperature;
use Laravel\Ai\Attributes\Timeout;
use Laravel\Ai\Attributes\TopP;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)]
#[Model('claude-sonnet-5')]
#[MaxSteps(10)]
#[MaxTokens(4096)]
#[Temperature(0.7)]
#[Timeout(120)]
#[TopP(0.9)]
class SalesCoach implements Agent
{
    use Promptable;

    // ...
}
```

`UseCheapestModel` 與 `UseSmartestModel` 屬性讓您不必指定模型名稱，就能自動選擇特定提供者最具成本效益或能力最強的模型。當您想跨不同提供者最佳化成本或能力時，這非常實用：

```php
use Laravel\Ai\Attributes\UseCheapestModel;
use Laravel\Ai\Attributes\UseSmartestModel;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Promptable;

#[UseCheapestModel]
class SimpleSummarizer implements Agent
{
    use Promptable;

    // Will use the cheapest model (e.g., Haiku)...
}

#[UseSmartestModel]
class ComplexReasoner implements Agent
{
    use Promptable;

    // Will use the most capable model (e.g., Opus)...
}
```

> [!NOTE]
> 當提供者推出新模型時，由 `UseCheapestModel` 與 `UseSmartestModel` 所選擇的底層模型可能會在 Laravel AI SDK 的不同版本間發生變動。切換模型可能會引進行為改變、已棄用的參數以及顯著的價格差異。如果您需要穩定、可預測的模型與價格，請使用 `Model` 屬性明確指定模型。


<a name="provider-options"></a>
### 提供者選項

如果您的 AI 代理需要傳遞特定提供者的選項（例如 OpenAI 的推理努力程度或懲罰設定），請實作 `HasProviderOptions` 契約（Contract）並定義 `providerOptions` 方法：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasProviderOptions;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasProviderOptions
{
    use Promptable;

    // ...

    /**
     * Get provider-specific generation options.
     */
    public function providerOptions(Lab|string $provider): array
    {
        return match ($provider) {
            Lab::OpenAI => [
                'reasoning' => ['effort' => 'low'],
                'frequency_penalty' => 0.5,
                'presence_penalty' => 0.3,
            ],
            Lab::Anthropic => [
                'thinking' => ['budget_tokens' => 1024],
                'cache_control' => ['type' => 'ephemeral'],
            ],
            default => [],
        };
    }
}
```

`providerOptions` 方法會接收當前正在使用的提供者（`Lab` 列舉或字串），讓您能為每個提供者回傳不同的選項。這在使用[故障轉移](#failover)時特別有用，因為每個備用提供者都可以接收自己的設定。

上述的 Anthropic 範例還透過 `cache_control` 啟用了[提示詞快取 (Prompt Caching)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)。

<a name="human-tool-approval"></a>
## 人工工具核准

> [!WARNING]
> 工具核准需要實作 `Conversational` 的 AI 代理，並且其對話歷史必須被持久化儲存，以便暫停的呼叫可以被恢復。`RemembersConversations` Trait 提供了所需的持久化功能。

執行敏感或不可逆動作的工具可能需要在執行前獲得人工核准。若要使工具支援核准，請實作 `Approvable` 契約(Contracts)，並使用 `InteractsWithApprovals` Trait。預設情況下，可核准的工具都需要經過核准：

```php
<?php

namespace App\Ai\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Illuminate\Support\Facades\Storage;
use Laravel\Ai\Concerns\InteractsWithApprovals;
use Laravel\Ai\Contracts\Approvable;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Tools\Request;
use Stringable;

class DeleteFile implements Approvable, Tool
{
    use InteractsWithApprovals;

    /**
     * Get the description of the tool's purpose.
     */
    public function description(): Stringable|string
    {
        return 'Delete a file from storage.';
    }

    /**
     * Execute the tool.
     */
    public function handle(Request $request): Stringable|string
    {
        Storage::delete($request['path']);

        return "Deleted [{$request['path']}].";
    }

    /**
     * Get the tool's schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'path' => $schema->string()->required(),
        ];
    }
}
```

若要根據工具呼叫的引數來判斷是否需要核准，可以在工具上定義 `needsApproval` 方法。此方法可以傳回布林值，或者傳回包含核准請求原因的 `Approval` 實例：

```php
use Laravel\Ai\Approvals\Approval;

/**
 * Determine whether the tool needs approval for the given request.
 */
protected function needsApproval(Request $request): Approval|bool
{
    return str_starts_with($request['path'], 'temporary/')
        ? false
        : Approval::required('This will permanently delete a file.');
}
```

當從 AI 代理的 `tools` 方法傳回工具時，您可以覆寫工具的核准需求：

```php
public function tools(): iterable
{
    return [
        (new SendNotification)->withoutApproval(),
        (new DeleteFile)->requireApproval('Deletion review required.'),
    ];
}
```

當呼叫需要核准的工具時，AI 代理會在執行它之前暫停。您可以檢查回應中的待核准項目（pending approvals），其中包含每個工具呼叫的 ID、工具名稱、引數以及核准原因：

```php
$response = (new FileAssistant)
    ->forUser($user)
    ->prompt('Delete the old invoice.');

if ($response->hasPendingApprovals()) {
    foreach ($response->pendingApprovals as $approval) {
        // $approval->id
        // $approval->tool
        // $approval->arguments
        // $approval->reason
    }
}
```

若要恢復 AI 代理的運作，請繼續對話並提供包含每個待核准工具呼叫之決定的 `Decisions` 實例。決定可以核准該呼叫、拒絕該呼叫，或在執行前編輯其引數：

```php
use Laravel\Ai\Approvals\Decision;
use Laravel\Ai\Approvals\Decisions;

$response = (new FileAssistant)
    ->continue($conversationId, as: $user)
    ->prompt(Decisions::from([
        'call_abc' => Decision::approve(),
        'call_ghi' => Decision::reject('The invoice must be retained.'),
    ]));
```

布林值 `true` 與 `false` 可以作為核准與拒絕的簡寫。每個待處理的工具呼叫都必須收到一個決定。未知、遺失或先前已處置的工具呼叫 ID 將會導致拋出 `ApprovalMismatchException`。您可以使用 `approveRemaining` 或 `rejectRemaining` 方法，為沒有明確決定的呼叫提供預設值：

```php
$decisions = Decisions::from([
    'call_abc' => true,
])->rejectRemaining('Not approved.');

$response = (new FileAssistant)
    ->continue($conversationId, as: $user)
    ->prompt($decisions);
```

帶有結果的拒絕（例如 `Decision::reject('Not approved.')`）會傳回給模型，使其可以繼續回應。不帶結果的拒絕則會在記錄拒絕後停止生成循環。

工具核准支援 `prompt`、`stream`、`queue`、`broadcast`、`broadcastNow` 以及 `broadcastOnQueue` 方法。

在串流與廣播期間，暫停會以 `tool_approval_request` 事件表示。當使用 [Vercel AI SDK 串流協定](#streaming-using-the-vercel-ai-sdk-protocol) 時，核准請求與結果會使用該協定原生的工具核准部分發送。

對於排入佇列的 AI 代理，產生的回應會傳遞給 `then` 回呼，且 Laravel 還會發送 `ToolApprovalRequested` 事件。

Laravel 會在要求模型繼續之前儲存已核准工具的結果。如果後續生成失敗，該核准已經完成處置。請使用一般的文字提示詞繼續對話，而非再次送出相同的核准決定。


<a name="complete-approval-flow"></a>
### 完整的核准流程

以下路由展示了完整的核准流程。`GET` 路由會傳回聊天畫面，而 `POST` 路由則接收新的文字提示詞或來自聊天畫面的核准決定。此範例假設應用程式的 `User` 模型使用了 `HasConversations` Trait：

```php
use App\Ai\Agents\FileAssistant;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Route;
use Illuminate\Validation\Rule;
use Laravel\Ai\Approvals\Decision;
use Laravel\Ai\Approvals\Decisions;
use Laravel\Ai\Models\Conversation;

Route::get('/chat/{conversation}', function (Request $request, Conversation $conversation) {
    Gate::authorize('view', $conversation);

    return view('chat', [
        'conversation' => $conversation,
    ]);
})->middleware('auth');

Route::post('/chat/{conversation}', function (Request $request, Conversation $conversation) {
    Gate::authorize('view', $conversation);

    $validated = $request->validate([
        'message' => ['nullable', 'string', 'required_without:decisions', 'prohibited_with:decisions'],
        'decisions' => ['nullable', 'array', 'required_without:message', 'prohibited_with:message'],
        'decisions.*.action' => ['required_with:decisions', Rule::in(['approve', 'reject'])],
        'decisions.*.result' => ['nullable', 'string'],
    ]);

    $prompt = isset($validated['decisions'])
        ? Decisions::from($validated->collect('decisions')->map(
            fn (array $decision) => match ($decision['action']) {
                'approve' => Decision::approve(),
                'reject' => Decision::reject($decision['result'] ?? null),
            }
        )->all())
        : $validated['message'];

    $response = (new FileAssistant)
        ->continue($conversation->id, as: $request->user())
        ->prompt($prompt);

    return [
        'conversation_id' => $response->conversationId,
        'status' => $response->hasPendingApprovals() ? 'awaiting_approval' : 'complete',
        'message' => $response->text,
        'approvals' => $response->pendingApprovals,
    ];
})->middleware('auth');
```

當回應狀態為 `awaiting_approval` 時，聊天畫面應該渲染待核准的項目，並使用工具呼叫 ID 作為每個決定的鍵值（Key），將使用者的選擇提交至相同的端點：

```json
{
    "decisions": {
        "call_abc": {
            "action": "approve"
        },
        "call_def": {
            "action": "reject",
            "result": "The invoice must be retained."
        }
    }
}
```

對於一般的聊天訊息，畫面則可以改為提交 `message` 值：

```json
{
    "message": "Delete the old invoice."
}
```

<a name="images"></a>
## 圖片

`Laravel\Ai\Image` 類別可用於使用 `openai`、`gemini` 或 `xai` 提供者來生成圖片：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')->generate();

$rawContent = (string) $image;
```

`square`、`portrait` 與 `landscape` 方法可以用來控制圖片的長寬比，而 `quality` 方法則可用於引導模型最終的圖片品質（`high`、`medium`、`low`）。`timeout` 方法可用於指定 HTTP 超時時間（秒）：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')
    ->quality('high')
    ->landscape()
    ->timeout(120)
    ->generate();
```

您可以使用 `attachments` 方法附加參考圖片：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Image;

$image = Image::of('Update this photo of me to be in the style of an impressionist painting.')
    ->attachments([
        Files\Image::fromStorage('photo.jpg'),
        // Files\Image::fromPath('/home/laravel/photo.jpg'),
        // Files\Image::fromUrl('https://example.com/photo.jpg'),
        // $request->file('photo'),
    ])
    ->landscape()
    ->generate();
```

生成的圖片可以輕鬆儲存到您應用程式的 `config/filesystems.php` 設定檔中所設定的預設磁碟上：

```php
$image = Image::of('A donut sitting on the kitchen counter');

$path = $image->store();
$path = $image->storeAs('image.jpg');
$path = $image->storePublicly();
$path = $image->storePubliclyAs('image.jpg');
```

圖片生成也可以加入佇列：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Responses\ImageResponse;

Image::of('A donut sitting on the kitchen counter')
    ->portrait()
    ->queue()
    ->then(function (ImageResponse $image) {
        $path = $image->store();

        // ...
    });
```


<a name="audio"></a>
## 語音 (TTS)

`Laravel\Ai\Audio` 類別可用於從給定的文字生成語音：

```php
use Laravel\Ai\Audio;

$audio = Audio::of('I love coding with Laravel.')->generate();

$rawContent = (string) $audio;
```

您也可以透過 Laravel `Stringable` 類別提供的 `toAudio` 方法從字串生成語音：

```php
use Illuminate\Support\Str;

$audio = Str::of('I love coding with Laravel.')->toAudio();
```

`male`、`female` 和 `voice` 方法可用於決定生成語音的聲音：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->generate();

$audio = Audio::of('I love coding with Laravel.')
    ->voice('voice-id-or-name')
    ->generate();
```

同樣地，`instructions` 方法可用於動態指導模型生成的語音聽起來應該如何：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->instructions('Said like a pirate')
    ->generate();
```

生成的語音可以輕鬆儲存到您應用程式的 `config/filesystems.php` 設定檔中所設定的預設磁碟上：

```php
$audio = Audio::of('I love coding with Laravel.')->generate();

$path = $audio->store();
$path = $audio->storeAs('audio.mp3');
$path = $audio->storePublicly();
$path = $audio->storePubliclyAs('audio.mp3');
```

語音生成也可以加入佇列：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Responses\AudioResponse;

Audio::of('I love coding with Laravel.')
    ->queue()
    ->then(function (AudioResponse $audio) {
        $path = $audio->store();

        // ...
    });
```


<a name="transcription"></a>
## 語音轉寫 (STT)

`Laravel\Ai\Transcription` 類別可用於為給定的語音生成逐字稿：

```php
use Laravel\Ai\Transcription;

$transcript = Transcription::fromPath('/home/laravel/audio.mp3')->generate();
$transcript = Transcription::fromStorage('audio.mp3')->generate();
$transcript = Transcription::fromUpload($request->file('audio'))->generate();

return (string) $transcript;
```

`diarize` 方法可以用來表示您希望回應除了原始文字逐字稿之外，還包含分句的逐字稿，讓您可以依說話者存取區段逐字稿：

```php
$transcript = Transcription::fromStorage('audio.mp3')
    ->diarize()
    ->generate();
```

語音轉寫生成也可以加入佇列：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Responses\TranscriptionResponse;

Transcription::fromStorage('audio.mp3')
    ->queue()
    ->then(function (TranscriptionResponse $transcript) {
        // ...
    });
```


<a name="text-summarization"></a>
## 文字摘要

您可以使用 Laravel 的 `Stringable` 類別提供的 `summarize` 方法來摘要文字。預設情況下，摘要將包含不超過三句話，並使用所設定提供者的最便宜文字模型來生成：

```php
use Illuminate\Support\Str;

$summary = Str::of($article)->summarize();
```

您可以指定用於生成摘要的最大句子數、提供者、模型和超時時間。`Str` 類別也提供了該方法的靜態版本：

```php
use Laravel\Ai\Enums\Lab;

$summary = Str::of($article)->summarize(
    sentences: 4,
    provider: Lab::Anthropic,
    model: 'claude-sonnet-5',
    timeout: 30,
);

$summary = Str::summarize($article, sentences: 4);
```

<a name="embeddings"></a>
## 向量嵌入

您可以使用 Laravel 的 `Stringable` 類別所提供的全新 `toEmbeddings` 方法，輕鬆為任何給定的字串生成向量嵌入：

```php
use Illuminate\Support\Str;

$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings();
```

或者，您可以使用 `Embeddings` 類別一次為多個輸入生成向量嵌入：

```php
use Laravel\Ai\Embeddings;

$response = Embeddings::for([
    'Napa Valley has great wine.',
    'Laravel is a PHP framework.',
])->generate();

$response->embeddings; // [[0.123, 0.456, ...], [0.789, 0.012, ...]]
```

您可以為向量嵌入指定維度與提供者：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->dimensions(1536)
    ->generate(Lab::OpenAI, 'text-embedding-3-small');
```

<a name="multimodal-embeddings"></a>
### 多模態向量嵌入

除了字串以外，`Embeddings::for` 方法還接受圖片、語音、文件與影片等輸入，讓您可以為非文字內容生成向量嵌入。Gemini 支援圖片、語音、文件與影片向量嵌入，而 VoyageAI 則支援圖片與影片向量嵌入：

```php
use Laravel\Ai\Embeddings;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Files\Image;
use Laravel\Ai\Files\Video;

$response = Embeddings::for([
    'A vineyard at sunset.',
    Image::fromStorage('vineyard.jpg'),
    Video::fromPath('/home/laravel/tour.mp4'),
])->generate(Lab::Gemini);
```

多模態輸入使用與[附件](#attachments)相同的檔案類別。這些檔案可以從本機路徑、檔案系統磁碟、遠端 URL 或 Base64 編碼的內容建立。圖片、文件與影片也可以從上傳的檔案建立，而文件則可以從原始字串內容建立：

```php
use Laravel\Ai\Files\Audio;
use Laravel\Ai\Files\Document;
use Laravel\Ai\Files\Image;
use Laravel\Ai\Files\Video;

Image::fromPath('/home/laravel/photo.jpg');
Image::fromStorage('photo.jpg');
Image::fromUpload($request->file('photo'));

Audio::fromPath('/home/laravel/clip.mp3');
Audio::fromStorage('clip.mp3');
Audio::fromUpload($request->file('clip.mp3'));

Video::fromPath('/home/laravel/video.mp4');
Video::fromStorage('video.mp4');
Video::fromUpload($request->file('video'));

Document::fromUrl('https://example.com/report.pdf');
Document::fromString('Laravel is a PHP framework.', 'text/plain');
Document::fromUpload($request->file('report'));
```

> [!NOTE]
> VoyageAI 不允許在單一請求中混合遠端 URL 媒體與 Base64 編碼的媒體。本機、已儲存與已上傳的檔案會以 Base64 編碼內容傳送，且文字輸入可以與任一媒體來源組合。請參閱您的提供者文件，以確認可用的多模態模型與輸入種類。

<a name="querying-embeddings"></a>
### 查詢向量嵌入

生成向量嵌入後，您通常會將它們儲存在資料庫的 `vector` 欄位中，以便後續查詢。Laravel 透過 `pgvector` 擴充功能，為 PostgreSQL 上的向量欄位提供原生支援。首先，在您的遷移（Migration）中定義一個 `vector` 欄位，並指定維度數量：

```php
Schema::ensureVectorExtensionExists();

Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vector('embedding', dimensions: 1536);
    $table->timestamps();
});
```

您也可以新增向量索引來加速相似度搜尋。當在向量欄位上呼叫 `index` 時，Laravel 會自動建立包含餘弦距離（Cosine distance）的 HNSW 索引：

```php
$table->vector('embedding', dimensions: 1536)->index();
```

在您的 Eloquent 模型上，您應該將向量欄位型別轉換為 `array`：

```php
protected function casts(): array
{
    return [
        'embedding' => 'array',
    ];
}
```

若要查詢相似的記錄，請使用 `whereVectorSimilarTo` 方法。此方法會透過最小餘弦相似度（介於 `0.0` 與 `1.0` 之間，其中 `1.0` 代表完全相同）來篩選結果，並依相似度進行排序：

```php
use App\Models\Document;

$documents = Document::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

`$queryEmbedding` 可以是浮點數陣列或純字串。當傳入字串時，Laravel 會自動為其生成向量嵌入：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', 'best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

如果您需要更深入的控制，可以獨立使用較低階的 `whereVectorDistanceLessThan`、`selectVectorDistance` 與 `orderByVectorDistance` 方法：

```php
$documents = Document::query()
    ->select('*')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding)
    ->limit(10)
    ->get();
```

如果您希望賦予 AI 代理作為工具執行相似度搜尋的能力，請參考 [相似度搜尋](#similarity-search) 工具文件。

> [!NOTE]
> 向量查詢目前僅支援使用 `pgvector` 擴充功能的 PostgreSQL 連線。

<a name="caching-embeddings"></a>
### 快取向量嵌入

向量嵌入的生成過程可以被快取，以避免對相同輸入進行重複的 API 呼叫。若要啟用快取，請將 `ai.caching.embeddings.cache` 設定選項設定為 `true`：

```php
'caching' => [
    'embeddings' => [
        'cache' => true,
        'store' => env('CACHE_STORE', 'database'),
        // ...
    ],
],
```

當快取啟用時，向量嵌入將被快取 30 天。快取金鑰是基於提供者、模型、維度與輸入內容，確保相同的請求能回傳快取結果，而不同的設定則生成最新的向量嵌入。

即使全域快取已被停用，您也可以使用 `cache` 方法為特定請求啟用快取：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache()
    ->generate();
```

您可以以秒為單位指定自訂快取時間：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache(seconds: 3600) // Cache for 1 hour
    ->generate();
```

Stringable 的 `toEmbeddings` 方法也接受 `cache` 引數：

```php
// Cache with default duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: true);

// Cache for a specific duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: 3600);
```

<a name="reranking"></a>
## 重新排序

重新排序 (Reranking) 允許您根據文件與給定查詢的相關性重新調整文件列表的順序。這對於透過語意理解來改進搜尋結果非常有幫助：

可以使用 `Laravel\Ai\Reranking` 類別來對文件進行重新排序：

```php
use Laravel\Ai\Reranking;

$response = Reranking::of([
    'Django is a Python web framework.',
    'Laravel is a PHP web application framework.',
    'React is a JavaScript library for building user interfaces.',
])->rerank('PHP frameworks');

// Access the top result...
$response->first()->document; // "Laravel is a PHP web application framework."
$response->first()->score;    // 0.95
$response->first()->index;    // 1 (original position)
```

可以使用 `limit` 方法來限制傳回的結果數量：

```php
$response = Reranking::of($documents)
    ->limit(5)
    ->rerank('search query');
```


<a name="reranking-collections"></a>
### 重新排序集合

為了方便起見，可以使用 `rerank` 巨集對 Laravel 集合進行重新排序。第一個引數指定用於重新排序的欄位，第二個引數則為查詢：

```php
// Rerank by a single field...
$posts = Post::all()
    ->rerank('body', 'Laravel tutorials');

// Rerank by multiple fields (sent as JSON)...
$reranked = $posts->rerank(['title', 'body'], 'Laravel tutorials');

// Rerank using a closure to build the document...
$reranked = $posts->rerank(
    fn ($post) => $post->title.': '.$post->body,
    'Laravel tutorials'
);
```

您也可以限制結果數量並指定提供者：

```php
$reranked = $posts->rerank(
    by: 'content',
    query: 'Laravel tutorials',
    limit: 10,
    provider: Lab::Cohere
);
```


<a name="files"></a>
## 檔案

`Laravel\Ai\Files` 類別或各個獨立的檔案類別可用於將檔案儲存在您的 AI 提供者端，以便日後在對話中使用。這對於大型文件或您想要多次引用而不需重複上傳的檔案特別有用：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Files\Image;

// Store a file from a local path...
$response = Document::fromPath('/home/laravel/document.pdf')->put();
$response = Image::fromPath('/home/laravel/photo.jpg')->put();

// Store a file that is stored on a filesystem disk...
$response = Document::fromStorage('document.pdf', disk: 'local')->put();
$response = Image::fromStorage('photo.jpg', disk: 'local')->put();

// Store a file that is stored on a remote URL...
$response = Document::fromUrl('https://example.com/document.pdf')->put();
$response = Image::fromUrl('https://example.com/photo.jpg')->put();

return $response->id;
```

您也可以儲存原始內容或上傳的檔案：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

// Store raw content...
$stored = Document::fromString('Hello, World!', 'text/plain')->put();

// Store an uploaded file...
$stored = Document::fromUpload($request->file('document'))->put();
```

當檔案儲存後，您可以在透過 AI 代理生成文字時引用該檔案，而不需要重新上傳檔案：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...'
    attachments: [
        Files\Document::fromId('file-id') // Attach a stored document...
    ]
);
```

若要取得先前儲存的檔案，請在檔案實例上使用 `get` 方法：

```php
use Laravel\Ai\Files\Document;

$file = Document::fromId('file-id')->get();

$file->id;
$file->mimeType();
```

若要從提供者端刪除檔案，請使用 `delete` 方法：

```php
Document::fromId('file-id')->delete();
```

預設情況下，`Files` 類別會使用您應用程式 `config/ai.php` 設定檔中設定的預設 AI 提供者。對於大多數操作，您可以使用 `provider` 引數指定不同的提供者：

```php
$response = Document::fromPath(
    '/home/laravel/document.pdf'
)->put(provider: Lab::Anthropic);
```

您可以使用 `withProviderOptions` 方法傳遞特定提供者的上傳選項。例如，您可以設定 OpenAI 檔案的 `purpose`：

```php
use Laravel\Ai\Files\Document;

$response = Document::fromPath('/home/laravel/knowledge.txt')
    ->withProviderOptions(['purpose' => 'assistants'])
    ->put();
```

若要針對每個提供者單獨設定選項，請傳入一個接收當前提供者的閉包：

```php
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Files\Document;

$response = Document::fromPath('/home/laravel/training.jsonl')
    ->withProviderOptions(fn (Lab|string $provider) => match ($provider) {
        Lab::OpenAI => ['purpose' => 'fine-tune'],
        default => [],
    })
    ->put();
```


<a name="using-stored-files-in-conversations"></a>
### 在對話中使用已儲存的檔案

當檔案儲存在提供者端後，您可以使用 `Document` 或 `Image` 類別上的 `fromId` 方法在 AI 代理對話中引用它：

```php
use App\Ai\Agents\DocumentAnalyzer;
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

$stored = Document::fromPath('/path/to/report.pdf')->put();

$response = (new DocumentAnalyzer)->prompt(
    'Summarize this document.',
    attachments: [
        Document::fromId($stored->id),
    ],
);
```

同樣地，已儲存的圖片也可以使用 `Image` 類別來引用：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Image;

$stored = Image::fromPath('/path/to/photo.jpg')->put();

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Image::fromId($stored->id),
    ],
);
```

<a name="vector-stores"></a>
## 向量儲存庫

向量儲存庫允許您建立可搜尋的檔案集合，用於檢索增強生成 (Retrieval-Augmented Generation, RAG)。`Laravel\Ai\Stores` 類別提供了建立、取得與刪除向量儲存庫的方法：

```php
use Laravel\Ai\Stores;

// Create a new vector store...
$store = Stores::create('Knowledge Base');

// Create a store with additional options...
$store = Stores::create(
    name: 'Knowledge Base',
    description: 'Documentation and reference materials.',
    expiresWhenIdleFor: days(30),
);

return $store->id;
```

若要透過 ID 取得現有的向量儲存庫，請使用 `get` 方法：

```php
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

$store->id;
$store->name;
$store->fileCounts;
$store->ready;
```

若要刪除向量儲存庫，可以使用 `Stores` 類別或儲存庫實例上的 `delete` 方法：

```php
use Laravel\Ai\Stores;

// Delete by ID...
Stores::delete('store_id');

// Or delete via a store instance...
$store = Stores::get('store_id');

$store->delete();
```

<a name="adding-files-to-stores"></a>
### 新增檔案至儲存庫

擁有向量儲存庫後，您可以透過 `add` 方法將[檔案](#files)新增至其中。新增到儲存庫的檔案會透過[檔案搜尋提供者工具](#file-search)自動建立索引以進行語意搜尋：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

// Add a file that has already been stored with the provider...
$document = $store->add('file_id');
$document = $store->add(Document::fromId('file_id'));

// Or, store and add a file in one step...
$document = $store->add(Document::fromPath('/path/to/document.pdf'));
$document = $store->add(Document::fromStorage('manual.pdf'));
$document = $store->add($request->file('document'));

$document->id;
$document->fileId;
```

> **Note:** 通常將先前儲存的檔案新增至向量儲存庫時，回傳的文檔 ID 會與該檔案先前被賦予的 ID 相同；然而，某些向量儲存提供者可能會回傳一個全新且不同的 "document ID"。因此，建議您將這兩個 ID 都儲存於資料庫中以供日後參考。

當您將檔案新增至儲存庫時，可以附加中繼資料 (Metadata)。使用[檔案搜尋提供者工具](#file-search)時，這些中繼資料後續可用於篩選搜尋結果：

```php
$store->add(Document::fromPath('/path/to/document.pdf'), metadata: [
    'author' => 'Taylor Otwell',
    'department' => 'Engineering',
    'year' => 2026,
]);
```

若要從儲存庫中移除檔案，請使用 `remove` 方法：

```php
$store->remove('file_id');
```

從向量儲存庫移除檔案並不會將其從提供者的[檔案儲存庫](#files)中刪除。若要從向量儲存庫移除檔案並同時將其從檔案儲存庫中永久刪除，請使用 `deleteFile` 引數：

```php
$store->remove('file_abc123', deleteFile: true);
```

<a name="failover"></a>
## 故障轉移

發送提示詞或生成其他媒體時，您可以提供一個提供者 / 模型的陣列，以便在主要提供者遇到服務中斷或速率限制 (Rate limit) 時，自動故障轉移至備用提供者 / 模型：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Image;

$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: [Lab::OpenAI, Lab::Anthropic],
);

$image = Image::of('A donut sitting on the kitchen counter')
    ->generate(provider: [Lab::Gemini, Lab::xAI]);
```

故障轉移僅會在拋出 `FailoverableException` 時觸發——例如速率限制 (`RateLimitedException`)、提供者過載或無法使用 (`ProviderOverloadedException`)，或點數不足 (`InsufficientCreditsException`)。一般的錯誤，例如驗證失敗或錯誤的請求 (Bad request) 錯誤，將不會觸發故障轉移。

當您傳入平鋪的提供者列表（如 `[Lab::OpenAI, Lab::Anthropic]`）時，每個提供者都會使用其預設模型。若要為故障轉移鏈中的每個提供者指定特定模型，請傳入以提供者為鍵 (Key) 的關聯陣列，並使用 `Lab` Enum 的 `value` 作為鍵（因為 Enum Case 無法直接作為 PHP 陣列的鍵）：

```php
use Laravel\Ai\Enums\Lab;

$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: [
        Lab::Gemini->value => 'gemini-3-flash-preview',
        Lab::DeepSeek->value => 'deepseek-v4-pro',
    ],
);
```

<a name="testing"></a>
## 測試


<a name="testing-agents"></a>
### AI 代理

若要在測試期間模擬 AI 代理的回應，請在 AI 代理類別上呼叫 `fake` 方法。您可以選擇提供回應陣列或 Closure：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Prompts\AgentPrompt;

// Automatically generate a fixed response for every prompt...
SalesCoach::fake();

// Provide a list of prompt responses...
SalesCoach::fake([
    'First response',
    'Second response',
]);

// Dynamically handle prompt responses based on the incoming prompt...
SalesCoach::fake(function (AgentPrompt $prompt) {
    return 'Response for: '.$prompt->prompt;
});
```

當模擬會傳回結構化輸出的 AI 代理時，您可以提供陣列作為回應。該 AI 代理將傳回包含給定資料的結構化回應：

```php
SalesCoach::fake([
    ['score' => 87],
]);
```

您也可以模擬正在等待工具核准的回應：

```php
use Laravel\Ai\Approvals\PendingApproval;
use Laravel\Ai\Responses\AgentResponse;

FileAssistant::fake([
    AgentResponse::fakeWithPendingApprovals([
        new PendingApproval(
            id: 'call_abc',
            tool: 'DeleteFile',
            arguments: ['path' => 'invoice.pdf'],
            reason: 'This will permanently delete a file.',
        ),
    ]),
]);

$response = (new FileAssistant)->prompt('Delete the invoice.');

$response->hasPendingApprovals(); // true
```

> **註記：**當在會傳回結構化輸出的 AI 代理上呼叫 `Agent::fake()`，且未明確提供模擬輸出時，Laravel 將自動產生符合該 AI 代理所定義之輸出結構的假資料。

向 AI 代理發送提示後，您可以針對收到的提示進行斷言：

```php
use Laravel\Ai\Prompts\AgentPrompt;

SalesCoach::assertPrompted('Analyze this...');

SalesCoach::assertPrompted(function (AgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotPrompted('Missing prompt');

SalesCoach::assertNeverPrompted();
```

當斷言核准延續流程時，您可以檢查提示的核准決策：

```php
use Laravel\Ai\Approvals\Decisions;
use Laravel\Ai\Prompts\AgentPrompt;

FileAssistant::fake();

(new FileAssistant)->prompt(Decisions::from([
    'call_abc' => true,
]));

FileAssistant::assertPrompted(function (AgentPrompt $prompt) {
    return $prompt->hasApprovalDecisions()
        && $prompt->approvalDecisions->get('call_abc')->isApproved();
});
```

對於佇列中的 AI 代理呼叫，請使用佇列相關的斷言方法：

```php
use Laravel\Ai\QueuedAgentPrompt;

SalesCoach::assertQueued('Analyze this...');

SalesCoach::assertQueued(function (QueuedAgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotQueued('Missing prompt');

SalesCoach::assertNeverQueued();
```

若要確保所有的 AI 代理呼叫都有對應的模擬回應，您可以使用 `preventStrayPrompts`。如果在未定義模擬回應的情況下呼叫了 AI 代理，將會拋出例外：

```php
SalesCoach::fake()->preventStrayPrompts();
```


<a name="testing-images"></a>
### 圖片

可以透過在 `Image` 類別上呼叫 `fake` 方法來模擬圖片生成。圖片被模擬後，即可針對記錄的圖片生成提示執行各種斷言：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Prompts\ImagePrompt;
use Laravel\Ai\Prompts\QueuedImagePrompt;

// Automatically generate a fixed response for every prompt...
Image::fake();

// Provide a list of prompt responses...
Image::fake([
    base64_encode($firstImage),
    base64_encode($secondImage),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Image::fake(function (ImagePrompt $prompt) {
    return base64_encode('...');
});
```

產生圖片後，您可以針對收到的提示進行斷言：

```php
Image::assertGenerated(function (ImagePrompt $prompt) {
    return $prompt->contains('sunset') && $prompt->isLandscape();
});

Image::assertNotGenerated('Missing prompt');

Image::assertNothingGenerated();
```

對於佇列中的圖片生成，請使用佇列相關的斷言方法：

```php
Image::assertQueued(
    fn (QueuedImagePrompt $prompt) => $prompt->contains('sunset')
);

Image::assertNotQueued('Missing prompt');

Image::assertNothingQueued();
```

若要確保所有的圖片生成都有對應的模擬回應，您可以使用 `preventStrayImages`。如果在未定義模擬回應的情況下生成了圖片，將會拋出例外：

```php
Image::fake()->preventStrayImages();
```


<a name="testing-audio"></a>
### 語音

可以透過在 `Audio` 類別上呼叫 `fake` 方法來模擬語音生成。語音被模擬後，即可針對記錄的語音生成提示執行各種斷言：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Prompts\AudioPrompt;
use Laravel\Ai\Prompts\QueuedAudioPrompt;

// Automatically generate a fixed response for every prompt...
Audio::fake();

// Provide a list of prompt responses...
Audio::fake([
    base64_encode($firstAudio),
    base64_encode($secondAudio),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Audio::fake(function (AudioPrompt $prompt) {
    return base64_encode('...');
});
```

產生語音後，您可以針對收到的提示進行斷言：

```php
Audio::assertGenerated(function (AudioPrompt $prompt) {
    return $prompt->contains('Hello') && $prompt->isFemale();
});

Audio::assertNotGenerated('Missing prompt');

Audio::assertNothingGenerated();
```

對於佇列中的語音生成，請使用佇列相關的斷言方法：

```php
Audio::assertQueued(
    fn (QueuedAudioPrompt $prompt) => $prompt->contains('Hello')
);

Audio::assertNotQueued('Missing prompt');

Audio::assertNothingQueued();
```

若要確保所有的語音生成都有對應的模擬回應，您可以使用 `preventStrayAudio`。如果在未定義模擬回應的情況下生成了語音，將會拋出例外：

```php
Audio::fake()->preventStrayAudio();
```


<a name="testing-transcriptions"></a>
### 語音轉寫

可以透過在 `Transcription` 類別上呼叫 `fake` 方法來模擬語音轉寫生成。語音轉寫被模擬後，即可針對記錄的轉寫生成提示執行各種斷言：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Prompts\TranscriptionPrompt;
use Laravel\Ai\Prompts\QueuedTranscriptionPrompt;

// Automatically generate a fixed response for every prompt...
Transcription::fake();

// Provide a list of prompt responses...
Transcription::fake([
    'First transcription text.',
    'Second transcription text.',
]);

// Dynamically handle prompt responses based on the incoming prompt...
Transcription::fake(function (TranscriptionPrompt $prompt) {
    return 'Transcribed text...';
});
```

產生語音轉寫後，您可以針對收到的提示進行斷言：

```php
Transcription::assertGenerated(function (TranscriptionPrompt $prompt) {
    return $prompt->language === 'en' && $prompt->isDiarized();
});

Transcription::assertNotGenerated(
    fn (TranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingGenerated();
```

對於佇列中的語音轉寫生成，請使用佇列相關的斷言方法：

```php
Transcription::assertQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->isDiarized()
);

Transcription::assertNotQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingQueued();
```

若要確保所有的語音轉寫生成都有對應的模擬回應，您可以使用 `preventStrayTranscriptions`。如果在未定義模擬回應的情況下生成了語音轉寫，將會拋出例外：

```php
Transcription::fake()->preventStrayTranscriptions();
```

<a name="testing-embeddings"></a>
### 向量嵌入

可透過對 `Embeddings` 類別呼叫 `fake` 方法來模擬向量嵌入的生成。一旦模擬了向量嵌入，即可針對記錄的向量嵌入生成提示詞進行各種斷言：

```php
use Laravel\Ai\Embeddings;
use Laravel\Ai\Prompts\EmbeddingsPrompt;
use Laravel\Ai\Prompts\QueuedEmbeddingsPrompt;

// Automatically generate fake embeddings of the proper dimensions for every prompt...
Embeddings::fake();

// Provide a list of prompt responses...
Embeddings::fake([
    [$firstEmbeddingVector],
    [$secondEmbeddingVector],
]);

// Dynamically handle prompt responses based on the incoming prompt...
Embeddings::fake(function (EmbeddingsPrompt $prompt) {
    return array_map(
        fn () => Embeddings::fakeEmbedding($prompt->dimensions),
        $prompt->inputs
    );
});
```

生成向量嵌入後，您可以針對收到的提示詞進行斷言：

```php
Embeddings::assertGenerated(function (EmbeddingsPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->dimensions === 1536;
});

Embeddings::assertNotGenerated(
    fn (EmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingGenerated();
```

針對進入佇列的向量嵌入生成，請使用佇列相關的斷言方法：

```php
Embeddings::assertQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Laravel')
);

Embeddings::assertNotQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingQueued();
```

若要確保所有向量嵌入生成都有對應的模擬回應，您可以使用 `preventStrayEmbeddings`。如果在沒有定義模擬回應的情況下生成向量嵌入，系統將會拋出例外：

```php
Embeddings::fake()->preventStrayEmbeddings();
```


<a name="testing-reranking"></a>
### 重新排序

可透過對 `Reranking` 類別呼叫 `fake` 方法來模擬重新排序操作：

```php
use Laravel\Ai\Reranking;
use Laravel\Ai\Prompts\RerankingPrompt;
use Laravel\Ai\Responses\Data\RankedDocument;

// Automatically generate a fake reranked responses...
Reranking::fake();

// Provide custom responses...
Reranking::fake([
    [
        new RankedDocument(index: 0, document: 'First', score: 0.95),
        new RankedDocument(index: 1, document: 'Second', score: 0.80),
    ],
]);
```

進行重新排序後，您可以針對執行的操作進行斷言：

```php
Reranking::assertReranked(function (RerankingPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->limit === 5;
});

Reranking::assertNotReranked(
    fn (RerankingPrompt $prompt) => $prompt->contains('Django')
);

Reranking::assertNothingReranked();
```


<a name="testing-files"></a>
### 檔案

可透過對 `Files` 類別呼叫 `fake` 方法來模擬檔案操作：

```php
use Laravel\Ai\Files;

Files::fake();
```

檔案操作模擬完成後，您可以針對上傳與刪除行為進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

// Store files...
Document::fromString('Hello, Laravel!', mimeType: 'text/plain')
    ->as('hello.txt')
    ->put();

// Make assertions...
Files::assertStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, Laravel!' &&
        $file->mimeType() === 'text/plain';
);

Files::assertNotStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, World!'
);

Files::assertNothingStored();
```

若要斷言檔案刪除，您可以傳入檔案 ID：

```php
Files::assertDeleted('file-id');
Files::assertNotDeleted('file-id');
Files::assertNothingDeleted();
```


<a name="testing-vector-stores"></a>
### 向量儲存庫

可透過對 `Stores` 類別呼叫 `fake` 方法來模擬向量儲存庫操作。模擬儲存庫也會自動模擬[檔案操作](#files)：

```php
use Laravel\Ai\Stores;

Stores::fake();
```

儲存庫操作模擬完成後，您可以針對建立或刪除的儲存庫進行斷言：

```php
use Laravel\Ai\Stores;

// Create store...
$store = Stores::create('Knowledge Base');

// Make assertions...
Stores::assertCreated('Knowledge Base');

Stores::assertCreated(fn (string $name, ?string $description) =>
    $name === 'Knowledge Base'
);

Stores::assertNotCreated('Other Store');

Stores::assertNothingCreated();
```

若要斷言儲存庫刪除，您可以提供儲存庫 ID：

```php
Stores::assertDeleted('store_id');
Stores::assertNotDeleted('other_store_id');
Stores::assertNothingDeleted();
```

若要斷言檔案是否已新增至儲存庫或從儲存庫中移除，請使用指定 `Store` 實例上的斷言方法：

```php
Stores::fake();

$store = Stores::get('store_id');

// Add / remove files...
$store->add('added_id');
$store->remove('removed_id');

// Make assertions...
$store->assertAdded('added_id');
$store->assertRemoved('removed_id');

$store->assertNotAdded('other_file_id');
$store->assertNotRemoved('other_file_id');
```

若檔案是在同一個請求中先儲存在提供者的[檔案儲存](#files)中，然後新增至向量儲存庫，您可能無法得知該檔案的提供者 ID。在此情況下，您可以傳入閉包至 `assertAdded` 方法，來針對新增檔案的內容進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

$store->add(Document::fromString('Hello, World!', 'text/plain')->as('hello.txt'));

$store->assertAdded(fn (StorableFile $file) => $file->name() === 'hello.txt');
$store->assertAdded(fn (StorableFile $file) => $file->content() === 'Hello, World!');
```

<a name="events"></a>
## 事件

Laravel AI SDK 會發送多種[事件](/docs/{{version}}/events)，包含：

- `AddingFileToStore`
- `AgentPrompted`
- `AgentStreamed`
- `AudioGenerated`
- `CreatingStore`
- `EmbeddingsGenerated`
- `FileAddedToStore`
- `FileDeleted`
- `FileRemovedFromStore`
- `FileStored`
- `GeneratingAudio`
- `GeneratingEmbeddings`
- `GeneratingImage`
- `GeneratingTranscription`
- `ImageGenerated`
- `InvokingTool`
- `PromptingAgent`
- `RemovingFileFromStore`
- `Reranked`
- `Reranking`
- `StoreCreated`
- `StoringFile`
- `StreamingAgent`
- `ToolApprovalRequested`
- `ToolApprovalResolved`
- `ToolInvoked`
- `TranscriptionGenerated`

您可以監聽這些事件中的任何一個，以記錄或儲存 AI SDK 的使用資訊。