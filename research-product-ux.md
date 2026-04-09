# Product Page UX Research — Tex-Hub
> Saved: April 2026 | Based on analysis of 3 reference sites

---

## 1. Reference Sites Analysed

| Site | What to steal |
|---|---|
| textilwaren24.eu | Overall product layout, step-by-step flow (Step 1: Color → Step 2: Qty per size → Step 3: Add to cart), clean breadcrumb, image gallery |
| shirt-guenstig-kaufen.de | BEST color swatches with thumbnail images + labels, per-color stock table with 2 warehouses, qty input per size |
| textil-grosshandel.eu | Good breadcrumb, color grid labeled with names, "ab X.XX€" pricing, related products by color |

---

## 2. Core UX Pattern Decided

**The B2B Wholesale Matrix Flow:**

```
Page Load → First color pre-selected → Matrix visible immediately
       ↓
User clicks color swatch
       ↓
Main image fades → new color image loads
       ↓
Size-qty matrix updates (stock per warehouse per size)
       ↓
User fills qty per size (e.g. 5×M, 10×L, 3×XL)
       ↓
Running total updates live (qty + price)
       ↓
Single "In den Warenkorb" button adds ALL sizes at once
```

**Why this beat standard WooCommerce UX:**
- B2B buyers order multiple sizes per color in one transaction
- Seeing stock levels gives confidence
- Matrix is faster than clicking size → set qty → click size → set qty

---

## 3. Design Decisions

- **Color swatches**: Grid layout, colored circle + name below, small border-radius card, active = red ring
- **Matrix table columns**: Größe | Lager 1 (fast, 1-3 days) | Lager 2 (slow, 3-8 days) | Anzahl (qty input)
- **Image swap**: Fade out/in (opacity 0→1, 200ms)
- **Matrix visibility**: Show immediately with first color pre-selected (not hidden behind "select color" gate)
- **Bulk price table**: Stays above matrix, highlights based on total qty across all sizes
- **Order bar**: Sticky summary at bottom of form showing total Stk + total €

---

## 4. HTML/JS Architecture (product.html)

### Color Data Object
```js
const COLORS = [
  { name: 'Schwarz', hex: '#1a1a1a', img: 'URL' },
  { name: 'Weiß',   hex: '#f0f0f0', border: true, img: 'URL' },
  // ... 12 colors
];
```

### Size/Price Data
```js
const SIZES = ['XS','S','M','L','XL','2XL','3XL','4XL','5XL'];
const PRICE_MAP = { XS:2.68, ..., '3XL':3.44, '4XL':3.44 };
```

### Stock Data
- In mockup: deterministic pseudo-random from color+size hash
- In production: from WooCommerce variation stock via REST API or `wp_localize_script`

### Key DOM IDs
- `#swatchGrid` — color swatches rendered here
- `#selectedColorLabel` — shows active color name
- `#mainImg` — main product image (fade-swapped on color change)
- `#matrixPlaceholder` — hidden after color selected
- `#matrixTable` — matrix rendered here by JS
- `#stockLegend` — warehouse legend, shown after color selected
- `mqty-{SIZE}` — qty input per size (e.g. `#mqty-L`)
- `#totalQty`, `#totalPrice` — order bar
- `#btnAddCart` — disabled until qty > 0

---

## 5. WooCommerce Implementation Guide

### WooCommerce Product Type
Use **Variable Product** with two attributes:
- `pa_color` (Color) — set to "Visible on product page"
- `pa_size` (Size) — set to "Visible on product page"

Create all `Color × Size` combinations as variations. Each variation gets:
- Its own price
- Its own stock quantity
- Its own image (color-specific product photo)

### Color → Image Swap (Native WC)
WooCommerce does this **out of the box** — when you assign an image to a variation (e.g., "Schwarz × any size"), clicking the Black swatch will auto-swap the gallery image.
No extra code needed for basic swap.

For the swatch circles (instead of dropdowns), use one of:
- **YITH WooCommerce Color and Label Variations** (free/premium)
- **Variation Swatches for WooCommerce** by Emran Ahmed (free, WordPress.org)
- **WooSwatches** plugin

### Size-Qty Matrix (NOT Native — Needs Custom Code)

**Option A: Plugin (quickest)**
- **WooCommerce Product Table** by Barn2 (paid, ~$89/yr) — shows all variations in table form with qty inputs
- **WC Variations Radio Buttons** + custom JS — shows radio or checkboxes with qty fields

**Option B: Custom Theme Override (recommended for full control)**

1. Copy `wp-content/plugins/woocommerce/templates/single-product/add-to-cart/variable.php` to `wp-content/themes/YOUR-THEME/woocommerce/single-product/add-to-cart/variable.php`

2. Replace WC's default variation dropdowns with your custom matrix HTML. In the template, after getting `$attributes`, output the matrix table.

3. Use `wp_localize_script` to pass variation data to JS:
```php
wp_localize_script('theme-product', 'wcVariations', [
  'variations' => $product->get_available_variations(),
  'attributes'  => $product->get_variation_attributes(),
]);
```

4. JS reads `wcVariations.variations`, filters by selected color, renders matrix.

**Option C: WooCommerce REST API Approach**
```js
// Get variations on page load:
fetch('/wp-json/wc/v3/products/{ID}/variations?per_page=100')

// Add single item to cart:
fetch('/wp-json/wc/v2/cart/add-item', {
  method: 'POST',
  body: JSON.stringify({ id: variationId, quantity: qty })
})

// To add multiple sizes at once: loop and POST each
```

**Option D: AJAX (classic WC way)**
```js
// Add to cart multiple variations:
// Loop through selected qty per size, for each call:
jQuery.post(wc_add_to_cart_params.ajax_url, {
  action: 'woocommerce_add_to_cart',
  product_id: productId,
  variation_id: variationId,  // color×size variation ID
  quantity: qty,
  variation: { attribute_pa_color: colorSlug, attribute_pa_size: sizeSlug }
});
```

### Showing Matrix Immediately (No "Select Color" Gate)

Two options:
1. **Pre-select first variation**: In your template, use JS to trigger `$('select[name=attribute_pa_color]').val('schwarz').trigger('change')` on page load
2. **Show all sizes for all colors at once**: Use a flat table that groups by color — more complex but best for bulk ordering

### URL Pattern (like textil-grosshandel.eu `?color=FFFFFF`)

Add this to `functions.php`:
```php
add_action('wp_loaded', function() {
  if (is_product() && isset($_GET['color'])) {
    // JS picks this up and pre-selects the swatch
    wp_add_inline_script('wc-add-to-cart-variation',
      'window._preselectedColor = "' . sanitize_text_field($_GET['color']) . '";',
      'before'
    );
  }
});
```
Then in JS: `if (window._preselectedColor) selectColorVariant(window._preselectedColor);`

---

## 6. Recommended Plugin Stack (if going plugin route)

| Need | Plugin |
|---|---|
| Color swatches (not dropdowns) | Variation Swatches for WooCommerce (free) |
| Size-qty matrix table | WooCommerce Product Table by Barn2 OR custom |
| Bulk pricing by qty | WooCommerce Wholesale Prices (free) |
| Stock per warehouse display | WooCommerce Multi-Location Inventory (or custom meta) |

---

## 7. Priority Decision
For the client mockup: **Custom JS matrix approach** (what's in product.html now) is the best reference.
For the live WordPress build: **Option B (custom template override)** gives most control.
The JS logic in product.html (`COLORS`, `SIZES`, `PRICE_MAP`, `renderMatrix`, `updateOrderBar`, `addToCartMatrix`) maps 1:1 to what the WC template would need.
