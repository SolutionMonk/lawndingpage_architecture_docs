# Example: Adding a Third-Party Widget (Elfsight Radio Player)

Sometimes a frontend engineer doesn't need to build a feature from scratch using HTML and CSS. Instead, they just need to embed a third-party widget (like a chat bubble, a mailing list form, or an Elfsight Radio Player) and give the admin the ability to turn it on or off.

This workflow is incredibly fast because it only requires **2 files**: the PHP block definition and the Blade template. No custom CSS or JS is needed since the third party handles it.

---

## The Goal
We want to embed this exact code provided by Elfsight:
```html
<script src="https://elfsightcdn.com/platform.js" async></script>
<div class="elfsight-app-7d0ae2d8-ce0d-4318-baa6-4a4078f443f5" data-elfsight-app-lazy></div>
```
We want the admin to be able to:
1. Paste in their specific Elfsight App ID (so the block can be reused by different users).
2. Toggle the radio player on or off.

---

## Step 1: Create the PHP Class (Admin Form)
**File:** `app/Blocks/ElfsightRadioBlock.php`

The frontend developer creates the Filament schema to ask the admin for their App ID.

```php
<?php
namespace App\Blocks;

use Filament\Forms\Components\TextInput;

class ElfsightRadioBlock
{
    // The name that appears in the Admin Panel dropdown
    public static function getLabel(): string 
    { 
        return 'Elfsight Radio Player'; 
    }

    // The fields the admin must fill out
    public static function getSchema(): array
    {
        return [
            TextInput::make('app_id')
                ->label('Elfsight App ID')
                ->required()
                ->helperText('Example: 7d0ae2d8-ce0d-4318-baa6-4a4078f443f5')
                ->default('7d0ae2d8-ce0d-4318-baa6-4a4078f443f5'),
        ];
    }
}
```
*Note: We don't need to build a "Toggle On/Off" switch here, because every block in our architecture automatically gets a "Visible on Site" toggle built into the core `BlockResource`! If the admin toggles it off, the block simply won't render.*

---

## Step 2: Create the Blade View (Frontend HTML)
**File:** `resources/views/blocks/elfsightradio.blade.php`

The developer takes the App ID provided by Filament and injects it into the Elfsight embed code.

```blade
<!-- The Elfsight Script -->
<!-- Using 'async' ensures it doesn't block the rest of the page from loading -->
<script src="https://elfsightcdn.com/platform.js" async></script>

<!-- The Elfsight Div -->
<!-- We inject the $app_id variable provided by Filament directly into the class name -->
<div class="elfsight-app-{{ $app_id }}" data-elfsight-app-lazy></div>
```

---

## Summary
That's it. It takes less than 3 minutes to build this.

1. **The Admin Experience:** They add a block, select "Elfsight Radio Player", paste their ID, and leave the core "Visible" toggle turned on.
2. **The Frontend Experience:** The server injects the script and the `<div>` with the correct ID. Elfsight's external servers handle all the styling, layout, and interactivity. 
3. **Toggling Off:** If the admin clicks the "Visible" toggle to the off position, the server skips this block entirely. The Elfsight script is never loaded, saving bandwidth and instantly removing the player from the site.