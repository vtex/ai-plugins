# VTEX Catalog — Platform Reference

## Entity Hierarchy

```
Department
  └─► Category
        └─► Subcategory (optional, up to 2 extra levels)
              └─► Product
                    └─► SKU
```

Every product must belong to exactly one category (or subcategory). A
category belongs to exactly one department. The hierarchy is fixed: you
cannot move a product between categories without recreating it or using
the Catalog API.

---

## Departments and Categories

**Departments** are the top-level grouping — typically used as the main
navigation menu (e.g., "Men", "Women", "Electronics").

**Categories** are the second level within a department — typically
product families (e.g., "T-shirts", "Laptops").

Categories can have up to two additional levels of subcategories. Beyond
four total levels (department + category + 2 subcategories), the tree
is not supported.

### Category configuration fields

| Field              | Description                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| Name               | Display name                                                                |
| Title              | SEO title tag (defaults to name if blank)                                   |
| Description        | SEO meta description                                                        |
| Active             | Whether the category is visible in the storefront                           |
| Show in storefront | Controls whether the category appears in navigation menus                   |
| Store front URL    | The slug used in the URL (auto-generated from name, can be overridden)      |
| Global category    | Google product taxonomy mapping for Merchant Center / Shopping feeds        |

**Known behavior:** Deactivating a category does not deactivate its
products — products remain individually active. Deactivating a category
only hides the category page and navigation entry.

---

## Products

A **Product** is the commercial entity — what the customer searches for and
adds to their wishlist. It holds shared attributes (name, brand, description,
specifications) that apply to all its SKUs.

### Key product fields

| Field                 | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| Product name          | Display name (appears in the storefront and order)                                   |
| Reference code        | Optional internal identifier (ERP code, supplier code)                               |
| Brand                 | Associated brand entity (must exist before product creation)                         |
| Category              | Exactly one category from the catalog tree                                           |
| Description           | Long-form text shown on the product page                                             |
| Meta tag description  | SEO description                                                                      |
| Title                 | SEO title tag                                                                        |
| Is active             | Whether the product is visible and sellable                                          |
| Trade policy          | Which trade policies (sales channels) this product is associated with                |
| Show without stock    | Whether to display the product page when all SKUs are out of stock                   |

### Product specifications

Product specifications hold attributes shared by all SKUs (e.g., "Material",
"Care instructions"). They are defined at the category level — a specification
group is associated with a category and all products in that category can use
those specifications.

- **Specification groups** organize related specifications.
- **Specification fields** define the attribute (name, type, allowed values).
- **Field types**: text, number, combo (dropdown), checkbox, radio.
- Specifications can be used as facets in Intelligent Search if indexed.

---

## SKUs

A **SKU** (Stock Keeping Unit) is the purchasable variant — the actual item
the customer buys, the item that has a price, and the unit tracked in
inventory.

### Key SKU fields

| Field              | Description                                                                     |
| ------------------ | ------------------------------------------------------------------------------- |
| SKU name           | Variant label (e.g., "Blue / M") — shown in the variant selector                |
| Reference code     | Internal or supplier code for this variant                                      |
| EAN / GTIN         | Barcode identifier (used for inventory and shipping integrations)               |
| Weight             | Physical weight (used in freight calculation)                                   |
| Dimensions         | Height, width, length (used in cubic weight and freight)                        |
| Is active          | Whether this SKU is available for purchase                                      |
| Is kit             | Whether the SKU is a bundle of other SKUs                                       |
| Images             | One or more images; the first image is the primary display image                |

### SKU specifications

SKU specifications define variant-level attributes (e.g., "Color", "Size").
These are what drive the variant selector on the product page.

- SKU specifications are also defined in specification groups at the category
  level, but configured per-SKU.
- **Important:** specification fields used for variant selection must be of
  type combo or radio so the storefront can render the selector correctly.

### SKU vs Product — what lives where

| Attribute           | Product | SKU |
| ------------------- | ------- | --- |
| Name                | ✅      |     |
| Brand               | ✅      |     |
| Description         | ✅      |     |
| Category            | ✅      |     |
| Price               |         | ✅  |
| Stock               |         | ✅  |
| Weight / dimensions |         | ✅  |
| EAN / barcode       |         | ✅  |
| Color / Size        |         | ✅ (SKU spec) |
| Material            | ✅ (Product spec) | |

---

## Brands

A **Brand** is a catalog entity that groups products by their commercial brand.
Brands have their own pages and can be used as facets in search.

| Field        | Description                                      |
| ------------ | ------------------------------------------------ |
| Name         | Brand name                                       |
| Text         | Description shown on the brand page              |
| Image        | Brand logo                                       |
| Is active    | Whether the brand appears in the storefront      |

Brands must be created before products that reference them.

---

## Collections

**Collections** are curated groupings of SKUs used for merchandising — landing
pages, promotional campaigns, editorial shelves.

Collections are independent of the category hierarchy. An SKU can belong
to multiple collections simultaneously.

### Collection types

| Type            | Description                                                                   |
| --------------- | ----------------------------------------------------------------------------- |
| Manual          | SKUs explicitly added to the collection one by one                            |
| Automatic       | SKUs matched by rules (brand, category, specification value, price range)     |
| Mixed           | Automatic base with manual additions or exclusions                            |

**Known behavior:** Collections are eventually consistent — a SKU added to
an automatic collection may take a few minutes to appear, depending on
catalog index state.

---

## Trade Policies (Sales Channels)

A **Trade Policy** defines a commercial context: which products are visible,
under which prices, to which customers, and under which payment conditions.

A product is associated with one or more trade policies. A storefront or
channel (marketplace, B2B portal) is bound to exactly one trade policy.

Key uses:
- Hide products not sold in a given country or channel.
- Apply different price tables per channel.
- Restrict payment methods per channel.

---

## Catalog API Reference

Key endpoints (verify contracts with `vtex-developer` MCP):

| Operation                       | Method | Path                                                              |
| ------------------------------- | ------ | ----------------------------------------------------------------- |
| Get product by ID               | GET    | `/api/catalog/pvt/product/{productId}`                            |
| Create product                  | POST   | `/api/catalog/pvt/product`                                        |
| Update product                  | PUT    | `/api/catalog/pvt/product/{productId}`                            |
| Get SKU by ID                   | GET    | `/api/catalog/pvt/stockkeepingunit/{skuId}`                       |
| Create SKU                      | POST   | `/api/catalog/pvt/stockkeepingunit`                               |
| Update SKU                      | PUT    | `/api/catalog/pvt/stockkeepingunit/{skuId}`                       |
| Get category tree               | GET    | `/api/catalog_system/pub/category/tree/{treeLevels}`              |
| Get specification groups        | GET    | `/api/catalog_system/pvt/specification/groupbycategory/{categoryId}` |
| List SKUs by product            | GET    | `/api/catalog/pvt/product/{productId}/stockkeepingunit`           |
| Associate SKU to trade policy   | POST   | `/api/catalog/pvt/stockkeepingunit/{skuId}/tradepolicy`           |

### Bulk operations

The Catalog API does not have a native batch upsert endpoint. For bulk
catalog operations:

- Use individual POST/PUT calls with retry logic from a VTEX IO background
  service.
- Rate limit applies: the API is throttled per account; heavy bulk loads
  should be paced (typically 30–50 req/s max, verify current limits with
  `vtex-developer` MCP).
- For initial catalog import, the **VTEX Catalog Spreadsheet Import** via
  Admin is available for smaller catalogs; for large catalogs, use the API.

---

## Common Catalog Mistakes

| Mistake                                              | Consequence                                                                 |
| ---------------------------------------------------- | --------------------------------------------------------------------------- |
| Creating specifications without a group              | Specifications become orphaned and cannot be assigned to products           |
| Using product specs for variant selection            | Variant selector will not render — must use SKU specs for variant fields    |
| Not setting EAN on SKUs                              | Carrier integrations and WMS sync may fail; IS search by barcode won't work |
| Activating products with no active SKUs              | Product page loads but shows no purchasable variant; bad UX                 |
| Moving a product between categories via API          | Specifications not defined in the new category become orphaned              |
| Setting weight/dimensions to 0 on a physical SKU    | Freight calculation returns 0 or no shipping options                        |
