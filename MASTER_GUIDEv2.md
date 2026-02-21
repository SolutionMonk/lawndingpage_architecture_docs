# The Frontend Engineer's Master Guide
**Building Blocks for LawndingPage MVP**

Welcome to LawndingPage! This system uses a highly modular, **zero-build-step** Server-Side Rendered (SSR) architecture. 

As a frontend engineer, you will be building UI "Blocks" (e.g., Image Galleries, Pricing Cards, Video Players). You do not need to write database migrations, set up API endpoints, or configure routing. **You only deal with 4 files per block.**

This document provides everything you need to know: the workflow, the gotchas, Filament schema basics, and an extensive library of examples and advanced patterns.

---

## 1. The Core Workflow (The 4 Files)

Every UI block you build requires 2 mandatory files and 2 optional files. The names must match exactly.

Let's assume you are building a block called `Profile`.

1.  **The Data Schema (Required):** `app/Blocks/ProfileBlock.php`
    *   *Defines the form the admin fills out in the Filament backend.*
2.  **The HTML Template (Required):** `resources/views/blocks/profile.blade.php`
    *   *Receives the data from the schema and renders it on the server.*
3.  **The Styling (Optional):** `public/res/blocks/profile.css`
    *   *Auto-loaded by the server only if the block is on the page. Must be scoped.*
4.  **The Interactivity (Optional):** `public/res/blocks/profile.js`
    *   *Auto-loaded by the server. Must use vanilla JS and IIFE wrappers.*

---

## 2. Filament Schema Basics (For Frontend Devs)

You must tell the backend what data your HTML needs. You do this using Filament Form Components inside your PHP class. 

Here are the most common fields you will copy and paste:

```php
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\MarkdownEditor;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Toggle;
use Filament\Forms\Components\ColorPicker;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\DateTimePicker;

public static function getSchema(): array
{
    return [
        // 1. A simple text string
        TextInput::make('heading')->required(),
        
        // 2. A multi-line text string
        Textarea::make('description')->rows(3),
        
        // 3. Rich text (outputs HTML string)
        MarkdownEditor::make('content'),
        
        // 4. An Image Upload (Returns a string path like 'images/photo.jpg')
        FileUpload::make('avatar')->image()->directory('images'),
        
        // 5. A True/False switch
        Toggle::make('show_button')->default(true),
        
        // 6. A Hex Color code (e.g. '#ff0000')
        ColorPicker::make('bg_color')->default('#ffffff'),
        
        // 7. A Dropdown choice
        Select::make('alignment')
            ->options(['left' => 'Left', 'center' => 'Center'])
            ->default('left'),
            
        // 8. An Array of repeatable items (e.g. a list of features)
        Repeater::make('features')
            ->schema([
                TextInput::make('feature_name'),
                TextInput::make('icon'),
            ]),
            
        // 9. A Date/Time selector
        DateTimePicker::make('target_date')
    ];
}
```

---

## 3. Crucial "Gotchas" & Rules

Before looking at the examples, memorize these 4 rules. If your block breaks, it is likely because you violated one of these.

### 🚨 Gotcha 1: `Storage::url()` for Images
When Filament uploads an image, it passes you a relative path (e.g., `images/my-photo.jpg`). If you put that directly into an `<img>` tag, it will break. **You must wrap it in `Storage::url()`**.
*   **BAD:** `<img src="{{ $avatar }}">`
*   **GOOD:** `<img src="{{ Storage::url($avatar) }}">`

### 🚨 Gotcha 2: Handling Empty Data
Admins will leave fields blank. If you don't check for empty data, your HTML will have empty tags, breaking your layout.
*   **BAD:** `<p class="subtitle">{{ $subtitle }}</p>` *(Renders an empty `<p>` tag if null)*
*   **GOOD:** `@if(!empty($subtitle)) <p class="subtitle">{{ $subtitle }}</p> @endif`

### 🚨 Gotcha 3: Scoping your CSS
Because there is no build step (no Vite/Tailwind processing), your CSS is loaded globally. If you write `.title { font-size: 50px; }`, you will change every title on the entire website.
*   **Rule:** Wrap your block in a specific class (e.g., `.block-profile`) and prefix all your CSS rules with it.

### 🚨 Gotcha 4: Isolating your JavaScript
The admin might add **three** Profile blocks to the exact same page. If your JS says `document.querySelector('.profile-btn')`, it will only find the button in the *first* block and the other two will break.
*   **Rule:** You must find all instances of the block using `querySelectorAll`, loop through them, and bind events strictly inside that loop. 

---

## 4. Basic Examples

### Example 1: The Simple UI Block (No JS/CSS Required)
*Scenario: You want to add a 3rd party widget, like a Spotify Embed.*

**1. `app/Blocks/SpotifyBlock.php`**
```php
<?php
namespace App\Blocks;
use Filament\Forms\Components\TextInput;

class SpotifyBlock
{
    public static function getLabel(): string { return 'Spotify Embed'; }
    public static function getSchema(): array
    {
        return [
            TextInput::make('spotify_id')->required()->helperText('Example: 37i9dQZF1DXcBWIGoYBM5M'),
        ];
    }
}
```

**2. `resources/views/blocks/spotify.blade.php`**
```blade
<div class="block-spotify" style="margin-bottom: 20px;">
    <iframe src="https://open.spotify.com/embed/playlist/{{ $spotify_id }}" width="100%" height="380" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture"></iframe>
</div>
```

---

### Example 2: The Data-Driven Block (HTML & CSS)
*Scenario: You want to build a Feature Grid. The admin can add as many features as they want using a Repeater.*

**1. `app/Blocks/FeaturesBlock.php`**
```php
<?php
namespace App\Blocks;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\Repeater;

class FeaturesBlock
{
    public static function getLabel(): string { return 'Feature Grid'; }
    public static function getSchema(): array
    {
        return [
            TextInput::make('title'),
            Repeater::make('feature_list')->schema([
                TextInput::make('heading')->required(),
                Textarea::make('description'),
                TextInput::make('emoji_icon')->maxLength(2)->default('✨'),
            ])
        ];
    }
}
```

**2. `resources/views/blocks/features.blade.php`**
```blade
<div class="block-features" data-block-type="features">
    @if(!empty($title))
        <h2 class="features-title">{{ $title }}</h2>
    @endif
    <div class="features-grid">
        @foreach($feature_list as $feature)
            <div class="feature-card">
                <div class="feature-icon">{{ $feature['emoji_icon'] ?? '✨' }}</div>
                <h3>{{ $feature['heading'] }}</h3>
                <p>{{ $feature['description'] ?? '' }}</p>
            </div>
        @endforeach
    </div>
</div>
```

**3. `public/res/blocks/features.css`**
```css
.block-features { padding: 40px 20px; }
.block-features .features-title { text-align: center; margin-bottom: 30px; }
.block-features .features-grid { display: flex; gap: 20px; flex-wrap: wrap; }
.block-features .feature-card { flex: 1 1 calc(33.333% - 20px); background: #222; padding: 20px; border-radius: 8px; }
```

---

## 5. Advanced Patterns

### Pattern A: The Admin-Themed Block (Inline Styles)
*Scenario: A Hero Banner where the Admin gets to pick the background color and text alignment.*

**The Concept:** Pass Filament's `ColorPicker` and `Select` values directly into the HTML `style=""` attribute.

**PHP (`app/Blocks/HeroBlock.php`):**
```php
use Filament\Forms\Components\ColorPicker;
use Filament\Forms\Components\Select;

// Inside getSchema():
ColorPicker::make('bg_color')->default('#3b82f6')->label('Background Color'),
ColorPicker::make('text_color')->default('#ffffff')->label('Text Color'),
Select::make('alignment')
    ->options(['left' => 'Left', 'center' => 'Center', 'right' => 'Right'])
    ->default('center'),
```

**Blade (`resources/views/blocks/hero.blade.php`):**
```blade
<!-- Inject the variables directly into the style attribute -->
<div class="block-hero" style="background-color: {{ $bg_color }}; color: {{ $text_color }}; text-align: {{ $alignment }}; padding: 60px 20px; border-radius: 12px;">
    <h1>{{ $heading }}</h1>
    <p>{{ $subheading }}</p>
</div>
```

---

### Pattern B: Time-Based Interactivity (Countdown Timer)
*Scenario: A block that counts down to a specific date. Requires native JS `setInterval` and DOM manipulation.*

**PHP (`app/Blocks/CountdownBlock.php`):**
```php
use Filament\Forms\Components\DateTimePicker;

// Inside getSchema():
DateTimePicker::make('target_date')->required()->label('Event Date'),
```

**Blade (`resources/views/blocks/countdown.blade.php`):**
```blade
<div class="block-countdown" data-block-type="countdown" data-target="{{ $target_date }}">
    <h3>Countdown to Event</h3>
    <div class="timer-display">
        <span class="days">00</span>d :
        <span class="hours">00</span>h :
        <span class="minutes">00</span>m :
        <span class="seconds">00</span>s
    </div>
</div>
```

**JavaScript (`public/res/blocks/countdown.js`):**
```javascript
(function() {
    document.addEventListener('DOMContentLoaded', () => {
        // Find all countdown blocks
        const timers = document.querySelectorAll('[data-block-type="countdown"]');
        
        timers.forEach(timer => {
            // Read the date from the Blade data attribute
            const targetDate = new Date(timer.getAttribute('data-target')).getTime();
            
            // Scope DOM queries to THIS specific timer
            const daysEl = timer.querySelector('.days');
            const hoursEl = timer.querySelector('.hours');
            const minsEl = timer.querySelector('.minutes');
            const secsEl = timer.querySelector('.seconds');

            const interval = setInterval(() => {
                const now = new Date().getTime();
                const distance = targetDate - now;

                if (distance < 0) {
                    clearInterval(interval);
                    timer.querySelector('.timer-display').innerHTML = "EVENT STARTED!";
                    return;
                }

                daysEl.innerText = String(Math.floor(distance / (1000 * 60 * 60 * 24))).padStart(2, '0');
                hoursEl.innerText = String(Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60))).padStart(2, '0');
                minsEl.innerText = String(Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60))).padStart(2, '0');
                secsEl.innerText = String(Math.floor((distance % (1000 * 60)) / 1000)).padStart(2, '0');
            }, 1000);
        });
    });
})();
```

---

### Pattern C: Loading External JS Libraries (The `@once` Directive)
*Scenario: You want to use a lightweight 3rd-party library (like Swiper.js or Chart.js) via CDN, but you don't want to load the `<script>` tag 5 times if the admin adds 5 carousels to the page.*

**The Concept:** Use Laravel Blade's `@once` directive. Anything inside `@once` is guaranteed to only be rendered a single time per page load, no matter how many times the block is looped.

**Blade (`resources/views/blocks/carousel.blade.php`):**
```blade
<!-- This will only print to the HTML once, even if there are 10 carousels! -->
@once
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swiper@11/swiper-bundle.min.css" />
    <script src="https://cdn.jsdelivr.net/npm/swiper@11/swiper-bundle.min.js"></script>
@endonce

<!-- This HTML prints for every carousel instance -->
<div class="block-carousel swiper" data-block-type="carousel">
    <div class="swiper-wrapper">
        @foreach($slides as $slide)
            <div class="swiper-slide">
                <img src="{{ Storage::url($slide['image']) }}">
            </div>
        @endforeach
    </div>
</div>
```

**JavaScript (`public/res/blocks/carousel.js`):**
```javascript
(function() {
    document.addEventListener('DOMContentLoaded', () => {
        // Wait briefly to ensure the CDN script has parsed
        setTimeout(() => {
            if (typeof Swiper !== 'undefined') {
                const carousels = document.querySelectorAll('[data-block-type="carousel"]');
                carousels.forEach(carousel => {
                    // Initialize Swiper on this specific instance
                    new Swiper(carousel, {
                        loop: true,
                        autoplay: { delay: 3000 },
                    });
                });
            }
        }, 100);
    });
})();
```

---

### Pattern D: Safe Markdown Output
*Scenario: The admin uses `MarkdownEditor` to write bold text, lists, and links. You need to output it to the page without breaking HTML or exposing XSS vulnerabilities.*

**PHP (`app/Blocks/RichTextBlock.php`):**
```php
use Filament\Forms\Components\MarkdownEditor;
// Inside getSchema():
MarkdownEditor::make('body_text')->required(),
```

**Blade (`resources/views/blocks/richtext.blade.php`):**
```blade
<div class="block-richtext">
    <!-- 
        Use {!! !!} to render unescaped HTML. 
        Use Str::markdown() to convert the markdown to HTML safely. 
    -->
    <div class="markdown-container">
        {!! Str::markdown($body_text ?? '') !!}
    </div>
</div>
```

---

## Summary Checklist for Building a Block
- [ ] Did I create the PHP schema in `app/Blocks/`?
- [ ] Does my Blade filename in `resources/views/blocks/` match my class name?
- [ ] Did I use `Storage::url()` for any Image/FileUpload fields?
- [ ] Did I wrap my HTML in a specific, unique CSS class?
- [ ] Did I scope all my CSS rules to that unique class?
- [ ] Did I use the `(function(){})()` wrapper for my JavaScript?
- [ ] Did my JavaScript loop through all instances of the block using `querySelectorAll`?
- [ ] Did I use `@once` if I needed to include a CDN script?