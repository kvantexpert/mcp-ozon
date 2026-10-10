# Public Ozon product image assets

This branch contains only public product-cover images intended to be fetched by Ozon Seller API.

- No credentials, customer data, API responses, or private files belong here.
- Each image must be specific to the product; avoid generic logo-only covers for products that Ozon flags as SPU duplicates.
- Public asset URLs use the direct HTTPS `raw.githubusercontent.com` address for a file in this branch.
- Verify HTTP 200 and the image Content-Type before calling `ozon_product_pictures_import`.
- Preserve the card's existing pictures: read them first, put the new cover at the beginning of `images`, then include the previous image URLs.
- Supported extensions depend on the receiving platform; current Ozon card editor accepts JPEG, JPG, PNG, HEIC and WEBP.
