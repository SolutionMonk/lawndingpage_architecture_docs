# Frontend Developer Workflow: Adding a New Feature

This guide explains the exact steps a frontend developer takes to add a brand new feature (a "Block") to the LawndingPage CMS. 

Because of our modular architecture, you do not need to touch the core backend routing, database migrations, or the master layout file. You work entirely in isolation.

---

## Scenario: Adding an "Image Carousel" Feature
Your goal is to build an Image Carousel where an admin can upload multiple images, and the public site displays them with "Next" and "Previous" buttons.

Here is the exact 4-step workflow.

### Step 1: Define the Admin Controls (The PHP File)
*This tells the Admin Panel what data you need to build the feature.*

1. **Create the file:** `app/Blocks/CarouselBlock.php`
2. **Write the Class:** Copy the standard template and define your fields using Filament's schema.

**Code:**
```php
<?php
namespace App\Blocks;

use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\FileUpload;

class CarouselBlock
{
    // The name that appears in the Admin Panel dropdown
    public static function getLabel(): string 
    { 
        return 'Image Carousel'; 
    }

    // The fields the admin must fill out
    public static function getSchema(): array
    {
        return [
            // A Repeater lets the admin add multiple items (images)
            Repeater::make('slides')
                ->label('Carousel Images')
                ->schema([
                    FileUpload::make('image')
                        ->image()
                        ->directory('carousel')
                        ->required(),
                ])
                ->minItems(1) // Force them to upload at least one image
        ];
    }
}
```
**Result:** The admin panel now magically has an "Image Carousel" option in the dropdown, allowing the admin to upload photos. The data will be saved as an array called `$slides`.

---

### Step 2: Build the HTML Structure (The Blade File)
*This takes the data from the Admin Panel and turns it into HTML.*

1. **Create the file:** `resources/views/blocks/carousel.blade.php`
    * *Rule: The filename must match your PHP class name (lowercased, without "Block").*
2. **Write the HTML:** Use a standard `div` wrapper, loop through the `$slides` array, and build your buttons. 

**Code:**
```blade
<!-- Rule: Always wrap your feature in a unique CSS class -->
<div class="block-carousel" data-block-type="carousel">
    
    <div class="carousel-track">
        <!-- Loop through the data provided by Filament -->
        @foreach($slides as $slide)
            <div class="carousel-slide" style="display: none;">
                <!-- Use Storage::url() to turn the filename into a real web link -->
                <img src="{{ Storage::url($slide['image']) }}" alt="Carousel Image">
            </div>
        @endforeach
    </div>

    <!-- The controls -->
    <button class="carousel-prev">Previous</button>
    <button class="carousel-next">Next</button>
</div>
```

---

### Step 3: Style the Feature (The CSS File)
*This makes it look good. It is loaded automatically only when the Carousel is on the page.*

1. **Create the file:** `public/res/blocks/carousel.css`
    * *Rule: The filename must exactly match your Blade filename.*
2. **Write the CSS:** Scope every single rule starting with `.block-carousel` so you don't accidentally style the rest of the website.

**Code:**
```css
.block-carousel {
    position: relative;
    max-width: 800px;
    margin: 0 auto;
    overflow: hidden;
}

.block-carousel .carousel-slide img {
    width: 100%;
    border-radius: 12px;
}

/* Make the first slide visible by default */
.block-carousel .carousel-slide:first-child {
    display: block;
}

.block-carousel .carousel-prev,
.block-carousel .carousel-next {
    position: absolute;
    top: 50%;
    background: rgba(0,0,0,0.5);
    color: white;
    padding: 10px;
    border: none;
    cursor: pointer;
}

.block-carousel .carousel-prev { left: 10px; }
.block-carousel .carousel-next { right: 10px; }
```

---

### Step 4: Add the Interactivity (The JavaScript File)
*This makes the buttons work. It is loaded automatically only when the Carousel is on the page.*

1. **Create the file:** `public/res/blocks/carousel.js`
    * *Rule: The filename must exactly match your Blade filename.*
2. **Write the Logic:** You must use an IIFE (Immediately Invoked Function Expression) to protect your variables, and you must loop through all Carousels on the page (in case the admin added more than one).

**Code:**
```javascript
// 1. Wrap everything in an IIFE so your variables don't break other features
(function() {
    
    // 2. Wait for the HTML to load
    document.addEventListener('DOMContentLoaded', () => {
        
        // 3. Find ALL carousels on the page
        const carousels = document.querySelectorAll('[data-block-type="carousel"]');
        
        // 4. Loop through them and attach logic individually
        carousels.forEach(carousel => {
            
            // Query elements ONLY inside this specific carousel
            const slides = carousel.querySelectorAll('.carousel-slide');
            const prevBtn = carousel.querySelector('.carousel-prev');
            const nextBtn = carousel.querySelector('.carousel-next');
            
            let currentIndex = 0;

            // Helper function to show a slide
            function showSlide(index) {
                // Hide all slides
                slides.forEach(slide => slide.style.display = 'none');
                
                // Show the active slide
                slides[index].style.display = 'block';
            }

            // Click Events
            nextBtn.addEventListener('click', () => {
                currentIndex++;
                if (currentIndex >= slides.length) currentIndex = 0; // Loop back to start
                showSlide(currentIndex);
            });

            prevBtn.addEventListener('click', () => {
                currentIndex--;
                if (currentIndex < 0) currentIndex = slides.length - 1; // Loop to end
                showSlide(currentIndex);
            });
            
        });
    });

})(); // End of IIFE
```

---

## Workflow Summary

1.  **Ask the Admin for Data:** Write `CarouselBlock.php`
2.  **Structure the HTML:** Write `carousel.blade.php`
3.  **Make it Pretty:** Write `carousel.css`
4.  **Make it Work:** Write `carousel.js`

**You do not have to register these files anywhere.** The system will automatically find your PHP class, show it in the admin panel, and dynamically load your CSS and JS files on the frontend whenever the admin uses your feature.