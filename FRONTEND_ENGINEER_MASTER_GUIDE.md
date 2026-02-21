# The Frontend Engineer's Master Guide
**Building Blocks for LawndingPage MVP**

Welcome to LawndingPage! This system uses a highly modular, **zero-build-step** Server-Side Rendered (SSR) architecture. 

As a frontend engineer, you will be building UI "Blocks" (e.g., Image Galleries, Pricing Cards, Video Players). You do not need to write database migrations, set up API endpoints, or configure routing. **You only deal with 4 files per block.**

This document provides everything you need to know: the workflow, the gotchas, Filament schema basics, and complete examples.

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
            ])
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
*   **Rule:** You must find all instances of the block using `querySelectorAll`, loop through them, and bind events strictly inside that loop. (See Example 3 below).

---

## Example 1: The Simple UI Block (No JS/CSS Required)
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
            TextInput::make('spotify_id')
                ->label('Spotify Playlist ID')
                ->required()
                ->helperText('Example: 37i9dQZF1DXcBWIGoYBM5M'),
        ];
    }
}
```

**2. `resources/views/blocks/spotify.blade.php`**
```blade
<div class="block-spotify" style="margin-bottom: 20px;">
    <!-- We inject the $spotify_id variable directly into the iframe URL -->
    <iframe 
        src="https://open.spotify.com/embed/playlist/{{ $spotify_id }}" 
        width="100%" 
        height="380" 
        frameBorder="0" 
        allowfullscreen="" 
        allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture">
    </iframe>
</div>
```

---

## Example 2: The Data-Driven Block (HTML & CSS)
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
            TextInput::make('title')->label('Section Title'),
            
            // The Repeater creates an array of items
            Repeater::make('feature_list')
                ->schema([
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
        <!-- Loop through the repeater array -->
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
/* Note how every rule starts with .block-features! */
.block-features { padding: 40px 20px; }
.block-features .features-title { text-align: center; margin-bottom: 30px; }
.block-features .features-grid { display: flex; gap: 20px; flex-wrap: wrap; }
.block-features .feature-card { 
    flex: 1 1 calc(33.333% - 20px); 
    background: #222; 
    padding: 20px; 
    border-radius: 8px; 
}
```

---

## Example 3: The Interactive Block (Vanilla JS Pattern)
*Scenario: An FAQ Accordion. When a user clicks a question, the answer slides open.*

**1. `app/Blocks/FaqBlock.php`**
```php
<?php
namespace App\Blocks;

use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\Repeater;

class FaqBlock
{
    public static function getLabel(): string { return 'FAQ Accordion'; }

    public static function getSchema(): array
    {
        return [
            Repeater::make('questions')
                ->schema([
                    TextInput::make('question')->required(),
                    Textarea::make('answer')->required(),
                ])
        ];
    }
}
```

**2. `resources/views/blocks/faq.blade.php`**
```blade
<div class="block-faq" data-block-type="faq">
    @foreach($questions as $index => $item)
        <div class="faq-item">
            <!-- The Button -->
            <button class="faq-question">
                {{ $item['question'] }}
                <span class="faq-icon">+</span>
            </button>
            
            <!-- The Hidden Content -->
            <div class="faq-answer" style="display: none;">
                <p>{{ $item['answer'] }}</p>
            </div>
        </div>
    @endforeach
</div>
```

**3. `public/res/blocks/faq.css`**
```css
.block-faq .faq-item { border-bottom: 1px solid #444; margin-bottom: 10px; }
.block-faq .faq-question { 
    width: 100%; 
    background: none; 
    color: white; 
    text-align: left; 
    padding: 15px 0; 
    display: flex; 
    justify-content: space-between;
}
.block-faq .faq-answer { padding: 0 0 15px 0; color: #aaa; }
/* State class added by JS */
.block-faq .faq-item.is-open .faq-icon { transform: rotate(45deg); }
```

**4. `public/res/blocks/faq.js`** *(The Critical JS Pattern)*
```javascript
// 1. Wrap in an IIFE to protect the global scope
(function() {
    
    // 2. Wait for DOM to load
    document.addEventListener('DOMContentLoaded', () => {
        
        // 3. Find ALL instances of the FAQ block on the page
        const faqBlocks = document.querySelectorAll('[data-block-type="faq"]');
        
        // 4. Loop through each block independently
        faqBlocks.forEach(block => {
            
            // 5. Query ONLY inside this specific block
            const items = block.querySelectorAll('.faq-item');
            
            items.forEach(item => {
                const questionBtn = item.querySelector('.faq-question');
                const answerDiv = item.querySelector('.faq-answer');
                
                questionBtn.addEventListener('click', () => {
                    // Toggle logic
                    const isOpen = item.classList.contains('is-open');
                    
                    if (isOpen) {
                        item.classList.remove('is-open');
                        answerDiv.style.display = 'none';
                    } else {
                        item.classList.add('is-open');
                        answerDiv.style.display = 'block';
                    }
                });
            });
            
        });
    });

})(); // End IIFE
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