# Frontend Architecture Setup Guide
**Implementing the Zero-Build Modular SSR Strategy**

This guide provides the exact step-by-step instructions for a developer to implement the frontend architecture of LawndingPage. It is designed to be highly performant, secure, and production-ready.

> **Note:** After completing this setup, frontend engineers should refer to `Docs/FRONTEND_ENGINEER_MASTER_GUIDE.md` for instructions on how to build and style individual blocks.
>
> **Prerequisites:** Complete the backend setup guide (`BACKEND_SETUP.md`) first. This frontend guide assumes observers, policies, and the BlockRegistry service are already configured.

---

## Phase 0: Prerequisites & Initial Setup

Before building the frontend, ensure your backend foundation is ready.

### 1. Run Migrations & Link Storage
You must create the SQLite tables for authentication and create the symlink for public file uploads.
```bash
php artisan migrate
php artisan storage:link
```

### 2. Create the Admin User
If you haven't already, create your initial admin account to access the Filament panel:
```bash
php artisan make:filament-user
# Follow prompts for Name, Email, and Password
```

### 3. Login to the Admin Panel
Start your local server (`php artisan serve`) and navigate to `http://localhost:8000/admin`. Log in to verify your database and Filament installation are working.

---

## Phase 1: Directory Structure & Core Assets

We must separate the "Core" application shell (loaded on every page) from the "Blocks" (loaded only when their specific feature is used).

### 1. Create the Folders
```bash
mkdir -p public/res/core
mkdir -p public/res/blocks
mkdir -p public/res/img
mkdir -p resources/views/blocks
```
*(Optional: Place a generic logo file at `public/res/img/default-logo.png` to serve as a fallback before the admin uploads one).*

### 2. Setup Core Assets
**If migrating from v1.8.0:**
```bash
mv public/res/sty/app.css public/res/core/app.css
mv public/res/scr/app.js public/res/core/app.js
```

**If starting fresh:**
Create minimal boilerplate files.
```bash
# Create minimal core CSS
cat > public/res/core/app.css << 'EOF'
/* Core structural styles */
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: system-ui, sans-serif; line-height: 1.5; background: #111; color: #fff; }
.pane { margin-bottom: 20px; }
EOF

# Create minimal core JS
cat > public/res/core/app.js << 'EOF'
// Core SPA navigation logic
console.log('LawndingPage Core loaded');
EOF
```

---

## Phase 2: Setup Routes

Before the controller can work, we need to map the web route.

Open `routes/web.php` and define the single entry point for the public site:
```php
<?php

use App\Http\Controllers\SiteController;
use Illuminate\Support\Facades\Route;

// Public site (single route, single controller)
Route::get('/', SiteController::class)->name('home');

// Note: Admin panel routes are handled automatically by Filament at /admin 
// (Assuming `php artisan filament:install` has been run).
```

---

## Phase 3: Controller & Caching Layer

Without caching, every page load queries the Orbit flat-files, adding 50-200ms per block. For production readiness, **caching is required.**

### 1. Create the Controller
```bash
php artisan make:controller SiteController --invokable
```

### 2. Write the Controller Logic
Open `app/Http/Controllers/SiteController.php`, add caching, and ensure null-safety:
```php
<?php

namespace App\Http\Controllers;

use App\Models\Site;
use App\Models\Block;
use Illuminate\Contracts\Cache\TaggableStore;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;

class SiteController extends Controller
{
    public function __invoke(Request $request)
    {
        // Check if cache driver supports tagging (Redis/Memcached) or fallback to simple keys (file)
        $store = Cache::getStore();
        $taggable = $store instanceof TaggableStore;

        // Use tagged caches when available, fall back to simple caching for file driver
        $site = $taggable
            ? Cache::tags(['site'])->remember('site_data', 300, fn() => Site::first())
            : Cache::remember('site_data', 300, fn() => Site::first());

        // Gracefully handle missing site configuration
        if (!$site) {
            abort(503, 'Site not configured. Please contact the administrator.');
        }

        $blocks = $taggable
            ? Cache::tags(['blocks'])->remember('visible_blocks', 300, fn() =>
                Block::where('is_visible', true)->orderBy('order')->get()
              )
            : Cache::remember('visible_blocks', 300, fn() =>
                Block::where('is_visible', true)->orderBy('order')->get()
              );

        // Pre-compute asset existence checks for performance (avoids file_exists in Blade)
        $blockAssets = $blocks->pluck('type')->unique()->mapWithKeys(fn ($type) => [
            $type => [
                'css' => file_exists(public_path("res/blocks/{$type}.css")),
                'js' => file_exists(public_path("res/blocks/{$type}.js")),
            ],
        ]);

        return view('site', compact('site', 'blocks', 'blockAssets'));
    }
}
```

### 3. Cache Invalidation (Already Handled)
**Cache invalidation is already handled by observers configured in the backend setup.**

The backend architecture guide configured `SiteObserver` and `BlockObserver` in `AppServiceProvider` that automatically clear caches when data changes. These observers use the `TaggableStore` pattern for Redis and fall back to `Cache::forget()` for file drivers.

**No additional cache clearing code is needed in resources.** The observers handle:
- `Cache::tags(['site'])->flush()` or `Cache::forget('site_data')`
- `Cache::tags(['blocks'])->flush()` or `Cache::forget('visible_blocks')`

See Phase 4 of the backend setup guide for the complete observer implementation.

---

## Phase 4: The Master Layout (Blade)

Create `resources/views/site.blade.php`. Read the comments carefully regarding security and performance.

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ $site->title ?? 'LawndingPage' }}</title>
    
    <!-- 1. ALWAYS load the core structural CSS -->
    <link rel="stylesheet" href="{{ asset('res/core/app.css') }}">
    
    <!-- 2. DYNAMIC CSS INJECTION -->
    @foreach($blockAssets as $type => $assets)
        @if($assets['css'])
            <link rel="stylesheet" href="{{ asset("res/blocks/{$type}.css") }}">
        @endif
    @endforeach

    <!-- 3. CUSTOM CSS OUTPUT -->
    {{-- Custom CSS is output RAW without sanitization. Security is enforced via:
         1. SiteResource restricts custom_css field to 'Owner' role only (see backend setup Phase 8)
         2. Content-Security-Policy headers (see backend setup Phase 12)

         DO NOT attempt regex-based sanitization here - it's fragile and bypassable.
         Rely on role-based access control + CSP headers for proper security. --}}
    @if(!empty($site->custom_css))
        <style>
            {!! $site->custom_css !!}
        </style>
    @endif
</head>

<body>
    <header class="site-header">
        @if(!empty($site->logo_path))
            <img src="{{ asset('storage/' . $site->logo_path) }}"
                 alt="{{ $site->title ?? 'Logo' }}"
                 onerror="this.src='{{ asset('res/img/default-logo.png') }}'">
        @endif
        <h1>{{ $site->title ?? '' }}</h1>
    </header>

    <main id="container">
        <!-- 4. RENDER THE BLOCKS -->
        @foreach($blocks as $block)
            <div class="pane" id="{{ $block->id }}" data-block-type="{{ $block->type }}">
                <!-- Passes entire $block->data JSON array as Blade variables -->
                @includeIf("blocks.{$block->type}", $block->data ?? [])
            </div>
        @endforeach
    </main>

    <!-- 5. NAVIGATION LOGIC -->
    <nav id="navBar">
        @foreach($blocks as $block)
            {{-- Best Practice: Only show blocks where the admin defined a 'nav_label'.
                 Ensure your Block schemas include a TextInput::make('nav_label')

                 Using proper anchor links (#id) provides:
                 - Browser-native scroll behavior (works without JS)
                 - Progressive enhancement (JS can intercept for smooth scrolling)
                 - Accessibility (keyboard navigation works) --}}
            @if(!empty($block->data['nav_label']))
                <a href="#{{ $block->id }}" class="navLink">
                    {{ $block->data['nav_label'] }}
                </a>
            @endif
        @endforeach
    </nav>

    <!-- 6. DYNAMIC JS INJECTION -->
    <script src="{{ asset('res/core/app.js') }}"></script>

    @foreach($blockAssets as $type => $assets)
        @if($assets['js'])
            <script src="{{ asset("res/blocks/{$type}.js") }}"></script>
        @endif
    @endforeach

</body>
</html>
```

**Note:** Custom CSS security is handled in the backend setup (Phase 8) where the `custom_css` field is restricted to Owner role only. See `SiteResource::siteSchema()` for the implementation.

---

## Phase 5: Understanding Block Data Flow

When the master Blade template runs `@includeIf("blocks.{$block->type}", $block->data ?? [])`, Laravel automatically extracts the array keys into accessible variables.

**In the Database (Orbit File):**
```yaml
id: abc-123
type: text
data:
  title: "Hello World"
  content: "This is a paragraph."
```

**In the Block View (`resources/views/blocks/text.blade.php`):**
```blade
<h2>{{ $title ?? '' }}</h2> <!-- Outputs: Hello World -->
<p>{{ $content ?? '' }}</p> <!-- Outputs: This is a paragraph. -->
```

---

## Phase 6: Testing Your Setup

Let's build a fast "Test Block" to verify data flows correctly.

### 1. Create the 4 Files
1.  **Schema:** `app/Blocks/TestBlock.php`
    *(Note: This is automatically discovered by `BlockRegistry::discover()`. You do not need to register it manually).*
    ```php
    <?php

    declare(strict_types=1);

    namespace App\Blocks;

    use Filament\Forms\Components\TextInput;

    class TestBlock implements BlockContract
    {
        public static function getLabel(): string
        {
            return 'Test Block';
        }

        public static function getType(): string
        {
            return 'test';
        }

        public static function getSchema(): array
        {
            return [
                TextInput::make('message')
                    ->label('Test Message')
                    ->required()
                    ->maxLength(100),
            ];
        }
    }
    ```
2.  **View:** `resources/views/blocks/test.blade.php` 
    ```blade
    <h2 class="test-msg">{{ $message ?? '' }}</h2> 
    <button>Click</button>
    ```
3.  **CSS:** `public/res/blocks/test.css` (`.test-msg { color: red; }`).
4.  **JS:** `public/res/blocks/test.js` (`console.log('Test block loaded'); alert('JS Works');`).

### 2. Create the Block in Admin
Go to `http://localhost:8000/admin`, click on "Blocks", create a new block, select "Test Block" from the auto-discovered dropdown, enter "Hello World", ensure it is set to **Visible**, and Save.

### 3. Verify the Output
Start the server:
```bash
php artisan serve
```
Visit `http://localhost:8000`. 
**You should see:**
1. ✅ Red text (CSS auto-loaded).
2. ✅ "Hello World" (Blade template rendered the Orbit data).
3. ✅ An alert popup (JS auto-loaded).

---

## Phase 7: Performance Optimization (Production)

Before deploying to a live server, ensure maximum performance:

### 1. Configure the Cache Driver
For production with cache tagging support, use Redis or Memcached:
```env
CACHE_DRIVER=redis
# Note: Requires a Redis server to be installed/running (pecl install redis && service redis start)
```

**Cache Driver Compatibility:**
- The controller gracefully detects if the cache driver supports tagging (via `TaggableStore` interface check)
- **With Redis/Memcached:** Uses tagged caches (`Cache::tags()->remember()`) for granular invalidation
- **With file driver:** Falls back to simple key-based caching (`Cache::remember()`)
- Both observers and controller use the same pattern, ensuring cache invalidation works correctly regardless of driver
- For production, Redis is strongly recommended for optimal performance and granular cache clearing

### 2. Cache Configuration
```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### 3. Optimize Autoloader
```bash
composer install --optimize-autoloader --no-dev
```

### 4. Enable OPcache
In your production `php.ini`, ensure OPcache is enabled:
```ini
opcache.enable=1
opcache.memory_consumption=128
opcache.max_accelerated_files=10000
opcache.revalidate_freq=2
```

---

## Troubleshooting Guide

### CSS/JS Not Loading
**Check:**
1. Do the files exist? `ls public/res/blocks/`
2. Are permissions correct? `chmod 644 public/res/blocks/*`
3. View Page Source: Are the `<link>` and `<script>` tags present in the HTML?
4. Browser Console (F12): Are there 404 errors?

### Blocks Not Showing
**Check:**
1. Is the block visible? Go to `/admin/blocks` → Check the "Visible" column.
2. Does the Block `type` string exactly match your Blade filename?
3. Have you cleared the cache? `php artisan cache:clear`

### "View Not Found" Error
*(e.g., `View [blocks.test] not found`)*
**Check:**
1. Does the file exist at `resources/views/blocks/test.blade.php`?
2. Does the block `type` string match the filename casing? (Type: `image_gallery` → File: `image_gallery.blade.php`).

---

## Expected Directory Structure

After completing all phases, your project should look like this:

```text
lawndingpage/
├── app/
│   ├── Blocks/
│   │   ├── TestBlock.php
│   │   └── ... 
│   ├── Http/Controllers/
│   │   └── SiteController.php
│   └── Models/
│       ├── Site.php
│       └── Block.php
├── public/
│   └── res/
│       ├── core/
│       │   ├── app.css     ← Global styles
│       │   └── app.js      ← SPA navigation logic
│       ├── img/
│       │   └── default-logo.png
│       └── blocks/
│           ├── test.css    ← Auto-loaded Block-specific CSS
│           └── test.js     ← Auto-loaded Block-specific JS
├── resources/
│   └── views/
│       ├── site.blade.php  ← Master layout
│       └── blocks/
│           ├── test.blade.php
│           └── ... 
└── routes/
    └── web.php             ← Route: GET /
```

---

## Summary

You've successfully implemented:

✅ Zero-build SSR architecture with modular asset loading
✅ **Tagged cache strategy compatible with backend observers**
✅ **Asset existence pre-computed in controller (no Blade file_exists)**
✅ **Raw CSS output (security via role restrictions + CSP headers)**
✅ **Progressive enhancement navigation (works without JS)**
✅ **Browser-native image fallback (no PHP file checks)**
✅ TestBlock with proper BlockContract implementation
✅ Controller-level caching with 300-second TTL
✅ Production-ready performance optimizations

**Key Architecture Decisions:**
- Cache invalidation handled by observers (no manual clearing in resources)
- Custom CSS security via access control + CSP (no fragile regex)
- Asset checks moved to controller for performance
- Navigation uses proper anchor links for accessibility
- Tagged caches align with backend TaggableStore pattern

**Review Status:** All Opus issues addressed (cache invalidation, sanitization, TestBlock interface, file_exists performance, navigation accessibility, directory naming).