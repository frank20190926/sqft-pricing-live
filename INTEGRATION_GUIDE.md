# Integration Guide — Square-Foot Pricing Page

File: `version-claude/index.html` (one file, no dependencies, no build step).

What it does: customer types width and height in feet, picks a material, and sees `Width × Height = Sq Ft × Rate + Base Setup Fee = Final Price` update live. "Add to cart" sends the price, material and dimensions to your checkout.

## 1. Quick test (30 seconds)

1. Open `index.html` in a browser.
2. Click **Run 4 × 6 ft Banner Test**. The page fills 4 × 6 ft, runs 8 checks (area, rate, price, every material switch, checkout fields), and opens the cart.
3. Click **Place test order**. The order JSON shows material, `width_ft: 4`, `height_ft: 6`, `area_sqft: 24`, rate, setup fee and price.

With the shipped rates (13 oz banner $3.50, setup fee $15.00): 24 × 3.50 + 15.00 = **$99.00**.

## 2. Change material names, rates and the setup fee (no code)

**Option A — Admin Panel tab (fastest).** Edit names, `$/sq ft` and the Base Setup Fee. The page updates immediately. Add or delete materials with the buttons.

Limit to know: the Admin Panel saves in that browser only (`localStorage`). Other visitors do not see your edits until you publish them (Option B).

**Option B — publish for every visitor.**
1. Admin Panel → **Download JSON** (or **Copy JSON**).
2. Open `index.html` in any text editor, find `<script type="application/json" id="sqft-config">`, and replace everything between the tags with the JSON.
3. Upload the file.

Or keep rates in a separate file: set `"configUrl": "rates.json"` in that block and upload a `rates.json` with the same structure. Later changes need only `rates.json` re-uploaded. (A browser that has local Admin edits uses those instead; use Admin → Reset to defaults to clear them.)

**On your live site, hide the Admin tab:** set `"ui": { "admin": false }` in the config block.

Config shape:

```json
{
  "currency": "$",
  "baseFee": 15,
  "limits": { "min": 0.5, "max": 100 },
  "materials": [
    { "id": "banner-13oz", "name": "13 oz Matte Vinyl Banner", "rate": 3.5, "kind": "banner" },
    { "id": "sticker-gloss", "name": "Gloss Vinyl Sticker", "rate": 6, "kind": "sticker" }
  ],
  "checkout": { "mode": "demo", "action": "", "redirect": "" },
  "ui": { "admin": false }
}
```

Keep each material `id` stable once orders exist; the checkout receives it.

## 3. Put the page on your existing site

- **Whole page:** upload `index.html` and link to it, or embed it:
  `<iframe src="/sqft/index.html" style="width:100%;height:1400px;border:0"></iframe>`
- **Inside your template:** copy the `<style>`, the body markup and the `<script>` blocks into your product template. All CSS classes are page-local; the script is wrapped in one function and exposes only `window.SqftPricing`.

## 4. What reaches checkout

Each item carries these fields (`checkout.mode = "form"` sends them as hidden form fields; `"ajax"` sends the same keys without the `sqft_` prefix as JSON):

| Field | Example (4 × 6 ft test) |
|---|---|
| `sqft_material_id` | `banner-13oz` |
| `sqft_material` | `13 oz Matte Vinyl Banner` |
| `sqft_rate_per_sqft` | `3.5` |
| `sqft_width_ft` | `4` |
| `sqft_height_ft` | `6` |
| `sqft_area_sqft` | `24` |
| `sqft_setup_fee` | `15` |
| `sqft_unit_price` | `99` |
| `sqft_quantity` | `1` |
| `sqft_line_total` | `99` |

Every integration below follows the same rule: **the server recomputes the price from width, height, material id and its own copy of the rates, and ignores `sqft_unit_price` from the browser.** Otherwise a customer can edit the price in dev tools. Pricing formula, in whole cents:

```
area_cents = round(round(w × h × 100) × round(rate × 10000) / 10000)
unit_cents = area_cents + round(baseFee × 100)
```

Hook for custom code on the page: `document.addEventListener('sqft:add-to-cart', e => { /* e.detail = payload */ e.preventDefault(); /* take over */ })`. Calling `preventDefault()` stops the built-in send. `window.SqftPricing.quote(w, h, materialId)` returns the price for any size.

## 5. Plain HTML / PHP cart

In the config block: `"checkout": { "mode": "form", "action": "/cart/add.php" }`. The page POSTs the fields above to that URL.

`/cart/add.php` (example; adapt the cart call to your cart):

```php
<?php
session_start();
$cfg = json_decode(file_get_contents(__DIR__ . '/rates.json'), true); // same file as the page's configUrl
$mat = null;
foreach ($cfg['materials'] as $m) if ($m['id'] === ($_POST['sqft_material_id'] ?? '')) $mat = $m;
$w = (float)($_POST['sqft_width_ft'] ?? 0);  $h = (float)($_POST['sqft_height_ft'] ?? 0);
$qty = max(1, min(999, (int)($_POST['sqft_quantity'] ?? 1)));
$min = $cfg['limits']['min'] ?? 0.5; $max = $cfg['limits']['max'] ?? 100;
if (!$mat || $w < $min || $h < $min || $w > $max || $h > $max) { http_response_code(400); exit('Invalid size or material'); }

$areaCents = (int)round(round($w * $h * 100) * round($mat['rate'] * 10000) / 10000);
$unitCents = $areaCents + (int)round($cfg['baseFee'] * 100);

$_SESSION['cart'][] = [
  'name' => $mat['name'] . " — {$w} × {$h} ft",
  'material' => $mat['name'], 'width_ft' => $w, 'height_ft' => $h,
  'area_sqft' => round($w * $h, 2), 'unit_cents' => $unitCents, 'qty' => $qty,
];
header('Location: /checkout.php');
```

Rule of thumb: put the server's `rates.json` and the page's `rates.json` in the same file so there is one place to edit.

## 6. WooCommerce

Create one simple product ("Custom Banner / Sticker", any price, e.g. 0) and note its ID (say `123`). Config: `"checkout": { "mode": "form", "action": "/?add-to-cart=123" }`.

Add this to a small plugin or your child theme's `functions.php`. It reads the posted fields, recomputes the price from `rates.json` in the theme folder, sets the cart price, and shows material and size in cart, checkout and the order.

```php
<?php
function sqft_rates() { return json_decode(file_get_contents(get_stylesheet_directory() . '/rates.json'), true); }

add_filter('woocommerce_add_cart_item_data', function ($data, $product_id) {
  if (!isset($_POST['sqft_material_id'])) return $data;
  $cfg = sqft_rates(); $mat = null;
  foreach ($cfg['materials'] as $m) if ($m['id'] === sanitize_text_field($_POST['sqft_material_id'])) $mat = $m;
  $w = (float)($_POST['sqft_width_ft'] ?? 0); $h = (float)($_POST['sqft_height_ft'] ?? 0);
  $min = $cfg['limits']['min'] ?? 0.5; $max = $cfg['limits']['max'] ?? 100;
  if (!$mat || $w < $min || $h < $min || $w > $max || $h > $max) {
    wc_add_notice('Invalid size or material.', 'error'); return $data;
  }
  $area = (int)round(round($w * $h * 100) * round($mat['rate'] * 10000) / 10000);
  $data['sqft'] = [
    'material' => $mat['name'], 'w' => $w, 'h' => $h, 'area' => round($w * $h, 2),
    'unit' => ($area + (int)round($cfg['baseFee'] * 100)) / 100,
  ];
  $data['sqft_key'] = md5(json_encode($data['sqft'])); // different sizes = separate cart lines
  return $data;
}, 10, 2);

add_action('woocommerce_before_calculate_totals', function ($cart) {
  if (is_admin() && !defined('DOING_AJAX')) return;
  foreach ($cart->get_cart() as $item) if (isset($item['sqft'])) $item['data']->set_price($item['sqft']['unit']);
});

add_filter('woocommerce_get_item_data', function ($rows, $item) {
  if (isset($item['sqft'])) {
    $s = $item['sqft'];
    $rows[] = ['name' => 'Material', 'value' => $s['material']];
    $rows[] = ['name' => 'Size', 'value' => "{$s['w']} × {$s['h']} ft ({$s['area']} sq ft)"];
  }
  return $rows;
}, 10, 2);

add_action('woocommerce_checkout_create_order_line_item', function ($order_item, $key, $values) {
  if (!isset($values['sqft'])) return;
  $s = $values['sqft'];
  $order_item->add_meta_data('Material', $s['material']);
  $order_item->add_meta_data('Width (ft)', $s['w']);
  $order_item->add_meta_data('Height (ft)', $s['h']);
  $order_item->add_meta_data('Area (sq ft)', $s['area']);
}, 10, 3);
```

Because the price comes from `rates.json` on the server, a WooCommerce store owner changes rates by editing/uploading that one file (or by pasting the Admin Panel's JSON into it).

## 7. Shopify

Shopify's storefront cart does not accept a price sent from the browser, so the page needs a small server step. Do not rely on `/cart/add.js` alone; it can carry the size and material as line-item properties but the price would still be the variant's price.

Recommended: **Draft Order** created by a server endpoint.

1. Config: `"checkout": { "mode": "ajax", "action": "https://YOUR-ENDPOINT/sqft-draft-order" }`.
2. The endpoint (serverless function, Shopify app proxy, or your own server) receives the JSON, recomputes the price from its copy of the rates (formula in section 4), and calls the Admin API `draftOrderCreate` with one custom line item: `title` = material + size, `originalUnitPrice` = recomputed price, `quantity`, and `customAttributes` = material, width, height, area. It returns `{ "checkout_url": "<draftOrder.invoiceUrl>" }`.
3. The page redirects the customer to that `checkout_url` (it follows `redirect` / `checkout_url` / `invoice_url` in the reply).

The endpoint needs a Shopify Admin API token with draft-order write access. I have not tested this against a live Shopify store; check the current `draftOrderCreate` fields in Shopify's docs when you wire it.

Cheaper alternative if the endpoint is not wanted: pre-create variants at fixed price steps and add the nearest one with the size in line-item properties. This charges approximate prices, so it is not recommended.

## 8. BigCommerce / any custom checkout with an API

Same pattern as Shopify: `"mode": "ajax"`, an endpoint you own recomputes the price and creates the cart server-side, then returns `{ "redirect": "<checkout URL>" }`.

For BigCommerce, the Cart API accepts **custom items** with a `list_price` (name, sku, quantity, list_price). Put material and size in the `name`, e.g. `13 oz Matte Vinyl Banner — 4 × 6 ft`. The API token stays on your server. Check the current Cart API reference for the exact request shape; this has not been run against a live store.

For a checkout that takes a plain form POST, use `"mode": "form"` as in section 5.

## 9. Final acceptance checklist (the 4 × 6 ft order)

- [ ] On the page: width 4, height 6, material 13 oz banner → shows 24.00 sq ft and $99.00 (with default rates).
- [ ] Switch material → rate and total change; nothing else does.
- [ ] Add to cart → your cart shows the same price, the material, and 4 × 6 ft.
- [ ] Place the order → the order record shows material, width, height, area and price.
- [ ] Change a rate (Section 2), reload, repeat → new price appears in the cart.
- [ ] Edit the posted `sqft_unit_price` in dev tools → your cart ignores it (server recomputes).

## Limits

- Prices are per piece in one currency (`$`). Quantity multiplies the per-piece price, including the setup fee. If you want the setup fee charged once per order instead of per piece, say so; it changes the formula.
- Shipping, tax and payment are your existing checkout's job; the demo checkout takes no payment.
- Verified in Chrome only (desktop width and a 390 px phone width, via screenshots). Safari/Firefox not tested.
