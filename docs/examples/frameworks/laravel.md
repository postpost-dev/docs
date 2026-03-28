---
title: "Laravel Integration"
description: "Build social media features into your Laravel app with PostPost API."
---

Build social media features into your Laravel app with PostPost API.

## Installation

```bash
composer require guzzlehttp/guzzle
```

## Configuration

```php
// config/services.php
return [
    // ...
    'postpost' => [
        'api_key' => env('PUBLORA_API_KEY'),
        'base_url' => env('PUBLORA_BASE_URL', 'https://api.postpost.dev/api/v1'),
    ],
];
```

```bash
# .env
PUBLORA_API_KEY=sk_your_api_key_here
```

## PostPost Service Class

```php
<?php
// app/Services/PostPostService.php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\RequestException;

class PostPostException extends \Exception
{
    public int $statusCode;
    public array $body;

    public function __construct(int $statusCode, string $message, array $body = [])
    {
        parent::__construct($message);
        $this->statusCode = $statusCode;
        $this->body = $body;
    }
}

class PostPostService
{
    protected string $apiKey;
    protected string $baseUrl;
    protected ?string $userId = null;

    public function __construct()
    {
        $this->apiKey = config('services.postpost.api_key');
        $this->baseUrl = config('services.postpost.base_url');
    }

    public function forUser(string $userId): self
    {
        $this->userId = $userId;
        return $this;
    }

    protected function request(string $method, string $endpoint, array $data = [])
    {
        $headers = [
            'x-api-key' => $this->apiKey,
        ];

        if ($this->userId) {
            $headers['x-postpost-user-id'] = $this->userId;
        }

        $response = Http::withHeaders($headers)
            ->$method($this->baseUrl . $endpoint, $data);

        if ($response->failed()) {
            $body = $response->json() ?? [];
            $message = $body['error'] ?? $body['message'] ?? 'API request failed';
            throw new PostPostException($response->status(), $message, $body);
        }

        return $response->json();
    }

    public function getConnections(): array
    {
        $result = $this->request('get', '/platform-connections');
        return $result['connections'] ?? [];
    }

    public function createPost(array $data): array
    {
        return $this->request('post', '/create-post', $data);
    }

    public function getPost(string $postGroupId): array
    {
        return $this->request('get', "/get-post/{$postGroupId}");
    }

    public function updatePost(string $postGroupId, array $updates): array
    {
        return $this->request('put', "/update-post/{$postGroupId}", $updates);
    }

    public function deletePost(string $postGroupId): array
    {
        return $this->request('delete', "/delete-post/{$postGroupId}");
    }

    public function getUploadUrl(string $fileName, string $contentType, string $postGroupId): array
    {
        return $this->request('post', '/get-upload-url', [
            'fileName' => $fileName,
            'contentType' => $contentType,
            'postGroupId' => $postGroupId,
        ]);
    }

    public function getLinkedInStats(string $platformId, string $postedId): array
    {
        return $this->request('post', '/linkedin-post-statistics', [
            'platformId' => $platformId,
            'postedId' => $postedId,
            'queryTypes' => 'ALL',
        ]);
    }
}
```

## Service Provider

```php
<?php
// app/Providers/PostPostServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\PostPostService;

class PostPostServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(PostPostService::class, function ($app) {
            return new PostPostService();
        });
    }

    public function boot()
    {
        //
    }
}
```

## Models

```php
<?php
// app/Models/SocialPost.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use App\Services\PostPostService;

class SocialPost extends Model
{
    protected $fillable = [
        'user_id',
        'post_group_id',
        'content',
        'platforms',
        'scheduled_time',
        'status',
    ];

    protected $casts = [
        'platforms' => 'array',
        'scheduled_time' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function refreshStatus(): void
    {
        try {
            $postpost = app(PostPostService::class);
            $data = $postpost->getPost($this->post_group_id);
            $this->update(['status' => $data['status'] ?? $this->status]);
        } catch (\Exception $e) {
            \Log::error("Failed to refresh post status: {$e->getMessage()}");
        }
    }
}
```

```php
<?php
// database/migrations/create_social_posts_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('social_posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('post_group_id')->unique();
            $table->text('content');
            $table->json('platforms');
            $table->timestamp('scheduled_time')->nullable();
            $table->string('status')->default('draft');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('social_posts');
    }
};
```

## Controllers

```php
<?php
// app/Http/Controllers/Api/SocialController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\SocialPost;
use App\Services\PostPostService;
use App\Services\PostPostException;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class SocialController extends Controller
{
    protected PostPostService $postpost;

    public function __construct(PostPostService $postpost)
    {
        $this->postpost = $postpost;
    }

    public function connections(): JsonResponse
    {
        try {
            $connections = $this->postpost->getConnections();
            return response()->json(['connections' => $connections]);
        } catch (PostPostException $e) {
            return response()->json(['error' => $e->getMessage()], $e->statusCode);
        }
    }

    public function createPost(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'content' => 'required|string|max:10000',
            'platforms' => 'required|array|min:1',
            'platforms.*' => 'string',
            'scheduled_time' => 'nullable|date|after:now',
        ]);

        try {
            $postData = [
                'content' => $validated['content'],
                'platforms' => $validated['platforms'],
            ];

            if (!empty($validated['scheduled_time'])) {
                $postData['scheduledTime'] = $validated['scheduled_time'];
            }

            $result = $this->postpost->createPost($postData);

            // Save to local database
            $post = SocialPost::create([
                'user_id' => auth()->id(),
                'post_group_id' => $result['postGroupId'],
                'content' => $validated['content'],
                'platforms' => $validated['platforms'],
                'scheduled_time' => $validated['scheduled_time'] ?? null,
                'status' => 'scheduled',
            ]);

            return response()->json([
                'success' => true,
                'post_group_id' => $result['postGroupId'],
                'post' => $post,
            ], 201);
        } catch (PostPostException $e) {
            return response()->json(['error' => $e->getMessage()], $e->statusCode);
        }
    }

    public function getPost(string $postGroupId): JsonResponse
    {
        try {
            $result = $this->postpost->getPost($postGroupId);
            return response()->json($result);
        } catch (PostPostException $e) {
            return response()->json(['error' => $e->getMessage()], $e->statusCode);
        }
    }

    public function deletePost(string $postGroupId): JsonResponse
    {
        try {
            $result = $this->postpost->deletePost($postGroupId);

            // Delete from local database
            SocialPost::where('user_id', auth()->id())
                ->where('post_group_id', $postGroupId)
                ->delete();

            return response()->json($result);
        } catch (PostPostException $e) {
            return response()->json(['error' => $e->getMessage()], $e->statusCode);
        }
    }

    public function userPosts(): JsonResponse
    {
        $posts = SocialPost::where('user_id', auth()->id())
            ->orderBy('created_at', 'desc')
            ->paginate(20);

        return response()->json($posts);
    }
}
```

## Routes

```php
<?php
// routes/api.php

use App\Http\Controllers\Api\SocialController;

Route::middleware('auth:sanctum')->group(function () {
    Route::prefix('social')->group(function () {
        Route::get('connections', [SocialController::class, 'connections']);
        Route::get('posts', [SocialController::class, 'userPosts']);
        Route::post('posts', [SocialController::class, 'createPost']);
        Route::get('posts/{postGroupId}', [SocialController::class, 'getPost']);
        Route::delete('posts/{postGroupId}', [SocialController::class, 'deletePost']);
    });
});
```

## Form Request Validation

```php
<?php
// app/Http/Requests/CreateSocialPostRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CreateSocialPostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'content' => 'required|string|max:10000',
            'platforms' => 'required|array|min:1',
            'platforms.*' => 'string|regex:/^[a-z]+-[A-Za-z0-9]+$/',
            'scheduled_time' => 'nullable|date|after:now',
            'platform_settings' => 'nullable|array',
        ];
    }

    public function messages(): array
    {
        return [
            'platforms.*.regex' => 'Platform ID must be in format: platform-id (e.g., twitter-123456)',
            'scheduled_time.after' => 'Scheduled time must be in the future',
        ];
    }
}
```

## Jobs for Background Processing

```php
<?php
// app/Jobs/ScheduleSocialPost.php

namespace App\Jobs;

use App\Models\SocialPost;
use App\Services\PostPostService;
use App\Services\PostPostException;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ScheduleSocialPost implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        protected SocialPost $post
    ) {}

    public function handle(PostPostService $postpost): void
    {
        try {
            $result = $postpost->createPost([
                'content' => $this->post->content,
                'platforms' => $this->post->platforms,
                'scheduledTime' => $this->post->scheduled_time?->toIso8601String(),
            ]);

            $this->post->update([
                'post_group_id' => $result['postGroupId'],
                'status' => 'scheduled',
            ]);
        } catch (PostPostException $e) {
            $this->post->update(['status' => 'failed']);
            throw $e;
        }
    }
}
```

```php
<?php
// app/Jobs/RefreshPostStatuses.php

namespace App\Jobs;

use App\Models\SocialPost;
use App\Services\PostPostService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class RefreshPostStatuses implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle(PostPostService $postpost): void
    {
        $pendingPosts = SocialPost::whereIn('status', ['scheduled', 'processing'])->get();

        foreach ($pendingPosts as $post) {
            try {
                $data = $postpost->getPost($post->post_group_id);
                $post->update(['status' => $data['status'] ?? $post->status]);
            } catch (\Exception $e) {
                \Log::warning("Failed to refresh post {$post->post_group_id}: {$e->getMessage()}");
            }

            usleep(200000); // Rate limiting: 200ms between requests
        }
    }
}
```

## Scheduled Task

```php
<?php
// app/Console/Kernel.php

protected function schedule(Schedule $schedule)
{
    $schedule->job(new \App\Jobs\RefreshPostStatuses)
        ->everyFiveMinutes()
        ->withoutOverlapping();
}
```

## Livewire Component

```php
<?php
// app/Livewire/SocialPostForm.php

namespace App\Livewire;

use Livewire\Component;
use App\Services\PostPostService;
use App\Services\PostPostException;
use App\Models\SocialPost;

class SocialPostForm extends Component
{
    public string $content = '';
    public array $selectedPlatforms = [];
    public ?string $scheduledTime = null;
    public array $connections = [];
    public ?string $successMessage = null;
    public ?string $errorMessage = null;

    public function mount(PostPostService $postpost)
    {
        try {
            $this->connections = $postpost->getConnections();
        } catch (PostPostException $e) {
            $this->errorMessage = 'Failed to load connections';
        }
    }

    public function togglePlatform(string $platformId)
    {
        if (in_array($platformId, $this->selectedPlatforms)) {
            $this->selectedPlatforms = array_diff($this->selectedPlatforms, [$platformId]);
        } else {
            $this->selectedPlatforms[] = $platformId;
        }
    }

    public function submit(PostPostService $postpost)
    {
        $this->validate([
            'content' => 'required|string|max:10000',
            'selectedPlatforms' => 'required|array|min:1',
        ]);

        try {
            $postData = [
                'content' => $this->content,
                'platforms' => array_values($this->selectedPlatforms),
            ];

            if ($this->scheduledTime) {
                $postData['scheduledTime'] = $this->scheduledTime;
            }

            $result = $postpost->createPost($postData);

            SocialPost::create([
                'user_id' => auth()->id(),
                'post_group_id' => $result['postGroupId'],
                'content' => $this->content,
                'platforms' => $this->selectedPlatforms,
                'scheduled_time' => $this->scheduledTime,
                'status' => 'scheduled',
            ]);

            $this->successMessage = "Post created: {$result['postGroupId']}";
            $this->reset(['content', 'selectedPlatforms', 'scheduledTime']);
            $this->errorMessage = null;

        } catch (PostPostException $e) {
            $this->errorMessage = $e->getMessage();
            $this->successMessage = null;
        }
    }

    public function render()
    {
        return view('livewire.social-post-form');
    }
}
```

```blade
<!-- resources/views/livewire/social-post-form.blade.php -->
<div class="max-w-2xl mx-auto p-4">
    <h2 class="text-xl font-bold mb-4">Create Social Post</h2>

    @if($successMessage)
        <div class="bg-green-100 text-green-800 p-3 rounded mb-4">
            {{ $successMessage }}
        </div>
    @endif

    @if($errorMessage)
        <div class="bg-red-100 text-red-800 p-3 rounded mb-4">
            {{ $errorMessage }}
        </div>
    @endif

    <form wire:submit="submit">
        <div class="mb-4">
            <label class="block text-sm font-medium mb-1">Content</label>
            <textarea
                wire:model="content"
                class="w-full p-2 border rounded"
                rows="4"
            ></textarea>
            <p class="text-sm text-gray-500 mt-1">{{ strlen($content) }} characters</p>
            @error('content') <p class="text-red-500 text-sm">{{ $message }}</p> @enderror
        </div>

        <div class="mb-4">
            <label class="block text-sm font-medium mb-1">Platforms</label>
            <div class="flex flex-wrap gap-2">
                @foreach($connections as $conn)
                    <button
                        type="button"
                        wire:click="togglePlatform('{{ $conn['platformId'] }}')"
                        class="px-3 py-1 rounded {{ in_array($conn['platformId'], $selectedPlatforms) ? 'bg-blue-500 text-white' : 'bg-gray-200' }}"
                    >
                        {{ $conn['platform'] }}: {{ $conn['username'] }}
                    </button>
                @endforeach
            </div>
            @error('selectedPlatforms') <p class="text-red-500 text-sm">{{ $message }}</p> @enderror
        </div>

        <div class="mb-4">
            <label class="block text-sm font-medium mb-1">Schedule (optional)</label>
            <input
                type="datetime-local"
                wire:model="scheduledTime"
                class="p-2 border rounded"
            >
        </div>

        <button
            type="submit"
            class="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
            wire:loading.attr="disabled"
        >
            <span wire:loading.remove>Schedule Post</span>
            <span wire:loading>Posting...</span>
        </button>
    </form>
</div>
```

## Artisan Command

```php
<?php
// app/Console/Commands/SyncConnections.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Services\PostPostService;
use App\Services\PostPostException;

class SyncConnections extends Command
{
    protected $signature = 'postpost:sync-connections';
    protected $description = 'Sync platform connections from PostPost';

    public function handle(PostPostService $postpost): int
    {
        try {
            $connections = $postpost->getConnections();

            $this->info("Found " . count($connections) . " connections:");

            foreach ($connections as $conn) {
                $this->line("  - {$conn['platform']}: {$conn['username']} ({$conn['platformId']})");
            }

            return Command::SUCCESS;
        } catch (PostPostException $e) {
            $this->error("Failed: {$e->getMessage()}");
            return Command::FAILURE;
        }
    }
}
```

---

*[PostPost](https://postpost.dev) — Social media API with free tier, paid plans from $2.99/account*
