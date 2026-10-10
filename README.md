# Public Ozon product image assets

This branch stores **public product-cover source assets only** for the Ozon Seller workflow. It is intentionally separate from the main source-code branch.

## Asset rules

- Do not place Client-Id, Api-Key, cookies, tokens, customer data, or private Ozon API responses here.
- Use a product-specific cover, not a generic brand logo, for a product flagged as an SPU duplicate.
- Raw SVG asset URL:
  `https://raw.githubusercontent.com/kvantexpert/mcp-ozon/ozon-product-images/ozon-product-covers/<filename>.svg`
- Ozon's product pictures API expects a downloadable image URL. For SVG covers, use the public image proxy to render a JPEG:
  `https://images.weserv.nl/?url=<URL-ENCODED-RAW-GITHUB-URL>&output=jpg&w=600&h=800&fit=cover`
- The proxy is a third-party dependency. Send only public product artwork through it. A first-party HTTPS image host is preferable for production once the server owner grants the required web-root/deployment permissions.

## Verified example (2026-10-10)

Source asset:
`ozon-product-covers/1c-bgu-korp-electronic-delivery.svg`

Rendered Ozon-compatible JPEG:
`https://images.weserv.nl/?url=raw.githubusercontent.com%2Fkvantexpert%2Fmcp-ozon%2Fozon-product-images%2Fozon-product-covers%2F1c-bgu-korp-electronic-delivery.svg&output=jpg&w=600&h=800&fit=cover`

Preflight returned HTTP 200 and `Content-Type: image/jpeg`; the image starts with the JPEG magic bytes `FF D8 FF`.

## Upload checklist

1. Read the current Ozon card with `ozon_product_info_list`.
2. Check the rendered URL from the Ozon VPS: HTTP 200 and `image/jpeg`.
3. Call `ozon_product_pictures_import` with `product_id` and the new cover URL first in `images[]`.
4. Preserve all current photos by appending the old `primary_image` and `images[]` URLs after the new cover. The API replaces the image list.
5. Verify with a fresh `ozon_product_info_list`; if validation is pending or status is updating, wait for moderation instead of repeating the WRITE.
