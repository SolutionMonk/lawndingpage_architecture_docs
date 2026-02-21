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

## 4. Basic Core Examples

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

## 5. Master UI Component Examples

### Example 3: Tabbed Content Interface
*Scenario: An interactive Tab component where clicking a header swaps out the visible content panel.*

**PHP (`app/Blocks/TabsBlock.php`):**
```php
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\MarkdownEditor;

// Inside getSchema():
Repeater::make('tabs')->schema([
    TextInput::make('tab_title')->required(),
    MarkdownEditor::make('tab_content')->required(),
])->minItems(1)
```

**Blade (`resources/views/blocks/tabs.blade.php`):**
```blade
<div class="block-tabs" data-block-type="tabs">
    <!-- Tab Headers -->
    <div class="tab-headers">
        @foreach($tabs as $index => $tab)
            <button class="tab-btn {{ $index === 0 ? 'active' : '' }}" data-index="{{ $index }}">
                {{ $tab['tab_title'] }}
            </button>
        @endforeach
    </div>
    
    <!-- Tab Panels -->
    <div class="tab-panels">
        @foreach($tabs as $index => $tab)
            <div class="tab-panel {{ $index === 0 ? 'active' : '' }}" data-index="{{ $index }}">
                {!! Str::markdown($tab['tab_content']) !!}
            </div>
        @endforeach
    </div>
</div>
```

**JavaScript (`public/res/blocks/tabs.js`):**
```javascript
(function() {
    document.addEventListener('DOMContentLoaded', () => {
        const tabBlocks = document.querySelectorAll('[data-block-type="tabs"]');
        
        tabBlocks.forEach(block => {
            const buttons = block.querySelectorAll('.tab-btn');
            const panels = block.querySelectorAll('.tab-panel');
            
            buttons.forEach(btn => {
                btn.addEventListener('click', () => {
                    const targetIndex = btn.getAttribute('data-index');
                    
                    // Remove active class from all
                    buttons.forEach(b => b.classList.remove('active'));
                    panels.forEach(p => p.classList.remove('active'));
                    
                    // Add active class to clicked
                    btn.classList.add('active');
                    block.querySelector(`.tab-panel[data-index="${targetIndex}"]`).classList.add('active');
                });
            });
        });
    });
})();
```

**CSS (`public/res/blocks/tabs.css`):**
```css
.block-tabs .tab-headers { display: flex; border-bottom: 2px solid #333; }
.block-tabs .tab-btn { padding: 10px 20px; background: none; color: #aaa; cursor: pointer; }
.block-tabs .tab-btn.active { color: #fff; border-bottom: 2px solid #3b82f6; margin-bottom: -2px; }
.block-tabs .tab-panel { display: none; padding: 20px 0; }
.block-tabs .tab-panel.active { display: block; }
```

---

### Example 4: Animated Progress / Skill Bars
*Scenario: Showing a list of skills with dynamic width percentages and admin-controlled colors.*

**PHP (`app/Blocks/SkillsBlock.php`):**
```php
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\ColorPicker;

// Inside getSchema():
Repeater::make('skills')->schema([
    TextInput::make('skill_name')->required(),
    TextInput::make('percentage')->numeric()->minValue(0)->maxValue(100)->required(),
    ColorPicker::make('bar_color')->default('#3b82f6'),
])
```

**Blade (`resources/views/blocks/skills.blade.php`):**
```blade
<div class="block-skills">
    @foreach($skills as $skill)
        <div class="skill-row">
            <div class="skill-labels">
                <span>{{ $skill['skill_name'] }}</span>
                <span>{{ $skill['percentage'] }}%</span>
            </div>
            <div class="skill-track">
                <!-- Inject percentage into width, and color into background -->
                <div class="skill-fill" 
                     style="width: {{ $skill['percentage'] }}%; background-color: {{ $skill['bar_color'] }};">
                </div>
            </div>
        </div>
    @endforeach
</div>
```

**CSS (`public/res/blocks/skills.css`):**
```css
.block-skills .skill-row { margin-bottom: 15px; }
.block-skills .skill-labels { display: flex; justify-content: space-between; margin-bottom: 5px; font-weight: bold; }
.block-skills .skill-track { width: 100%; height: 12px; background: #222; border-radius: 6px; overflow: hidden; }
.block-skills .skill-fill { height: 100%; border-radius: 6px; transition: width 1s ease-in-out; }
```

---

### Example 5: Before / After Image Slider
*Scenario: An interactive block comparing two images. Uses an invisible range slider to update a CSS variable controlling the clip-path of the top image.*

**PHP (`app/Blocks/BeforeAfterBlock.php`):**
```php
use Filament\Forms\Components\FileUpload;
// Inside getSchema():
FileUpload::make('image_before')->image()->required(),
FileUpload::make('image_after')->image()->required(),
```

**Blade (`resources/views/blocks/beforeafter.blade.php`):**
```blade
<div class="block-before-after" data-block-type="beforeafter" style="--position: 50%;">
    <div class="image-container">
        <!-- Bottom Image (After) -->
        <img class="img-background" src="{{ Storage::url($image_after) }}" alt="After">
        
        <!-- Top Image (Before) clipped by CSS -->
        <img class="img-foreground" src="{{ Storage::url($image_before) }}" alt="Before">
        
        <!-- The invisible slider control -->
        <input type="range" min="0" max="100" value="50" class="slider-control" aria-label="Percentage of before photo shown">
        
        <div class="slider-line"></div>
    </div>
</div>
```

**CSS (`public/res/blocks/beforeafter.css`):**
```css
.block-before-after {
    width: 100%; max-width: 800px; margin: 0 auto;
}
.block-before-after .image-container {
    position: relative; aspect-ratio: 16/9; overflow: hidden; border-radius: 8px;
}
.block-before-after img {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; pointer-events: none;
}
/* The magic CSS variable --position controls the clipping */
.block-before-after .img-foreground {
    clip-path: polygon(0 0, var(--position) 0, var(--position) 100%, 0 100%);
}
.block-before-after .slider-line {
    position: absolute; top: 0; bottom: 0; left: var(--position); width: 4px; background: white; pointer-events: none; transform: translateX(-50%);
}
.block-before-after .slider-control {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%; opacity: 0; cursor: ew-resize; margin: 0;
}
```

**JavaScript (`public/res/blocks/beforeafter.js`):**
```javascript
(function() {
    document.addEventListener('DOMContentLoaded', () => {
        const sliders = document.querySelectorAll('[data-block-type="beforeafter"]');
        
        sliders.forEach(block => {
            const input = block.querySelector('.slider-control');
            
            // Listen for dragging the range slider
            input.addEventListener('input', (e) => {
                // Update the CSS variable on the parent block
                block.style.setProperty('--position', `${e.target.value}%`);
            });
        });
    });
})();
```

---

## 6. Advanced Technical Patterns

### Pattern A: The Admin-Themed Block (Inline Styles)
*Scenario: A Hero Banner where the Admin gets to pick the background color and text alignment.*

**The Concept:** Pass Filament's `ColorPicker` and `Select` values directly into the HTML `style=""` attribute.

**Blade (`resources/views/blocks/hero.blade.php`):**
```blade
<!-- Inject the variables directly into the style attribute -->
<div class="block-hero" style="background-color: {{ $bg_color }}; color: {{ $text_color }}; text-align: {{ $alignment }}; padding: 60px 20px; border-radius: 12px;">
    <h1>{{ $heading }}</h1>
    <p>{{ $subheading }}</p>
</div>
```

---

### Pattern B: Loading External JS Libraries (The `@once` Directive)
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
    ...
</div>
```

---

### Pattern C: Safe Markdown Output
*Scenario: The admin uses `MarkdownEditor` to write bold text, lists, and links. You need to output it to the page without breaking HTML or exposing XSS vulnerabilities.*

**PHP (`app/Blocks/RichTextBlock.php`):**
```php
use Filament\Forms\Components\MarkdownEditor;
MarkdownEditor::make('body_text')->required(),
```

**Blade (`resources/views/blocks/richtext.blade.php`):**
```blade
<div class="block-richtext">
    <!-- Use {!! !!} to render unescaped HTML. -->
    <!-- Use Str::markdown() to convert the markdown to HTML safely. -->
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
- [ ] Did I check if variables exist (`@if(!empty($var))`) before printing HTML tags?
- [ ] Did I wrap my HTML in a specific, unique CSS class?
- [ ] Did I scope all my CSS rules to that unique class?
- [ ] Did I use the `(function(){})()` wrapper for my JavaScript?
- [ ] Did my JavaScript loop through all instances of the block using `querySelectorAll`?
- [ ] Did I use `@once` if I needed to include a CDN script?