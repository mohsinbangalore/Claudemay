# MyBake — Application Study & Technical Documentation

**Application:** MyBake (`com.developers.neoline.mybake`)
**Type:** Android van-sales / direct-store-delivery (DSD) order-taking app
**Business:** Bakery products distribution to shops across the UAE, operated by a delivery fleet
**Target device:** Android handheld with built-in receipt printer (POS-style device)
**Backend:** Firebase Realtime Database (project `my-bake`, `https://my-bake.firebaseio.com`)
**Era:** Built December 2017 – January 2018 (author signature "savad" in file headers)
**Version:** 1.0 (versionCode 1)
**Documentation date:** July 2026

---

## 1. Executive Summary

MyBake is a single-purpose order-capture app for a bakery distribution fleet. A salesman on a
delivery van selects his **route**, then the **shop** he is visiting, browses the **product
catalogue** (loaded live from Firebase), builds an **order (cart)** using +/− quantity buttons,
and reviews a **final order list** with line totals and a subtotal, ready to print.

The app is functional as a prototype but was left unfinished in several important ways:

- **The "Print" button does not print.** It shows a placeholder toast ("Testing to be done"),
  wipes the cart, and returns to the home screen. A complete PDF-generation module exists in
  the codebase (iText) but is never called, and no printer SDK is integrated at all.
- **Orders are never saved anywhere.** Nothing is written back to Firebase and the local cart
  is deleted on "Print" — the company has no record of any order taken through the app.
- **The project no longer builds** with today's tooling (dead Fabric/Crashlytics repositories,
  Gradle 4.1 / Android Gradle Plugin 3.0.1, jcenter, compileSdk 26).
- Several genuine bugs exist in cart handling, quantity display, and Firebase data loading
  (detailed in §8).

The rest of this document is a full study: business workflow, screen-by-screen behaviour, data
architecture, code inventory, defects, security posture, and a modernization roadmap.

---

## 2. Business Workflow the App Implements

```
Van salesman starts his day
        │
        ▼
┌─────────────────────┐   Route list is hardcoded in the app:
│ 1. Select ROUTE     │   Sheikh Zayed Road, Emirates Road, Amman Street,
│    (RouteActivity)  │   Umm Ramool Road, Abu Baker Al Siddique Road,
└─────────┬───────────┘   Nadd Al Hammar Road
          ▼
┌─────────────────────┐   Shop list is also hardcoded:
│ 2. Select SHOP +    │   Chateau Blanc, French Bakery, Hummingbird Bakery,
│    browse CATEGORIES│   French Bakery Central Kitchen, …
│  (CategoryActivity) │   Categories load live from Firebase /category
└─────────┬───────────┘
          ▼
┌─────────────────────┐   Products for the tapped category load from
│ 3. Add PRODUCTS     │   Firebase /product/{categoryKey}.
│  (ProductsActivity) │   +/− buttons build the cart in a local SQLite DB.
└─────────┬───────────┘
          ▼
┌─────────────────────┐   Shows No. / Product / Qty / Unit Price / Line Total
│ 4. FINAL ORDER LIST │   and a SubTotal (AED).
│ (FinalListActivity) │   "Print" → placeholder toast, cart wiped, app exits
└─────────────────────┘   to the Android home screen. NO print, NO record.
```

The selected route and shop names are stored in SharedPreferences (`route_name`, `shop_name`)
— clearly intended to appear on the printed invoice, but they are read in
`FinalListActivity` and then **never used**.

---

## 3. Technology Stack

| Layer | Technology | Version | Status today |
|---|---|---|---|
| Language | Java (Android) | — | Fine, but pre-Kotlin era |
| Min / Target SDK | API 14 (Android 4.0) / API 26 (Android 8.0) | — | Far below Play Store requirements |
| Build | Gradle 4.1 + Android Gradle Plugin 3.0.1 | 2017 | Won't run on modern JDK/IDE |
| UI toolkit | android.support libraries (AppCompat, RecyclerView, CardView, Design, ConstraintLayout 1.0.2) | 26.1.0 | Superseded by AndroidX (2018) |
| Remote data | Firebase Realtime Database | 11.6.0 | Very old SDK, still-live service |
| Auth | firebase-auth 11.6.0 dependency present | — | **Never used in code** |
| Local data | Raw SQLite via `SQLiteOpenHelper` | — | Works |
| Serialization | Gson 2.8.1 | — | Used only by dead code |
| PDF | iTextG 5.5.4 (bundled JARs in `app/libs/`) | — | Implemented, never invoked |
| Crash reporting | Fabric Crashlytics 2.8.0 | — | **Service shut down in 2020; blocks the build** |
| Printing | — | — | **Nothing integrated** |

---

## 4. Screen-by-Screen Study

### 4.1 RouteActivity — launcher screen
File: `activity/RouteActivity.java`, layout `activity_route.xml`

- Full-screen bakery background image, a dialog-mode `Spinner` of routes, and a **Next** button.
- Routes come from `res/values/strings.xml` → `<string-array name="my_bake_route">`
  (6 real UAE roads + a "Select the Routes" placeholder).
- Validation: if the placeholder is still selected, the spinner text is turned red with an
  error and the spinner is re-opened.
- On success: saves `route_name` to default SharedPreferences and starts `CategoryActivity`.
- Also initializes Fabric Crashlytics here with `.debuggable(true)` (debug logging left on in
  production) and contains a commented-out forced-crash test line.

### 4.2 CategoryActivity — shop + category selection
File: `activity/CategoryActivity.java`, adapter `CategoryListAdapter.java`,
layouts `activity_catgory.xml` (note the typo in the filename), `category_list_single_unit.xml`

- Toolbar + a shop `Spinner` (hardcoded `my_bake_shops` array) + a 3-column
  `RecyclerView` grid of category cards + a loading `ProgressBar`.
- Categories load from Firebase node **`/category`** via `addValueEventListener`
  (a *persistent* listener — see bug B3).
- Tapping a category card validates that a shop is selected, saves `shop_name` to prefs,
  and opens `ProductsActivity` with two extras:
  - `postKey` — the Firebase key of the category (used to build the products path)
  - `posVal` — the *grid position* of the category (used later to synthesize product IDs — see bug B5)

### 4.3 ProductsActivity — order building
File: `activity/ProductsActivity.java`, adapter `ProductListAdapter.java`,
layouts `activity_products.xml`, `product_list_single_unit.xml`

- Toolbar with a "final list"/print-list icon (`btn_final_list`), 3-column grid of product
  cards. Each card: product name, − button, quantity counter, + button.
- Products load from Firebase node **`/product/{postKey}`**.
- **+ button:** increments the ViewHolder's `qty` counter (max 20, though the error toast says
  "Maximum quantity is four"), then inserts or updates a row in the local SQLite cart
  (`DBHandler`). The cart row's `pro_id` is the string concatenation
  `posVal + position` parsed as an integer.
- **− button:** decrements; at zero it deletes the cart row. Pressing − before ever
  pressing + crashes (bug B7).
- **Final-list icon:** saves the (always empty) static `Utility.cartsList` to SharedPreferences
  via the dead `SharedPreference` favourites mechanism, then opens `FinalListActivity`.
  The real cart travels through SQLite, not through this call.

### 4.4 FinalListActivity — order review / "invoice"
File: `activity/FinalListActivity.java`, adapter `FinalListAdapter.java`,
layouts `activity_final_list.xml`, `final_list_single_unit.xml`

- A header row (No. / Product Name / Qty / UnitPrice / line Total), a linear `RecyclerView`
  of cart lines read from SQLite (`DBHandler.getAllCarts()`), a **SubTotal** computed by
  summing line totals in Java, and a **Print** button.
- Reads `route_name` and `shop_name` from prefs but never displays or prints them.
- **Print button (the app's dead end):**
  1. `Toast: "Testing to be done"`
  2. `db.deleteAll()` — the entire cart is erased
  3. Launches the Android home screen and finishes.
  No print, no PDF, no upload, no order history.

---

## 5. Data Architecture

### 5.1 Firebase Realtime Database (remote, read-only)

Project: **`my-bake`** · Database: `https://my-bake.firebaseio.com` ·
Storage bucket: `my-bake.appspot.com` (unused) · Config committed in `app/google-services.json`.

Inferred schema from the model classes and query paths:

```
/category
    {pushId}: { key: "<categoryKey>", name: "<Category name>" }      → Category.java
/product
    /{categoryKey}
        {pushId}: { name: "<Product name>", price: "<integer as string>" }   → Product.java
```

Notes:
- The app only **reads**. There is no write path to Firebase anywhere.
- `firebase-auth` is a declared dependency but no sign-in code exists, so the database rules
  must currently allow **unauthenticated public reads** for the app to work — and, unless the
  rules were configured carefully, possibly public writes too (see §9).
- Prices are stored as **strings** and the app does `Integer.parseInt(price)` — a decimal
  price like `"12.50"` or a stray space crashes the app (bug B6). All prices must be whole AED.

### 5.2 Local SQLite cart (`DBHandler.java`)

Database `cartinfo`, version 1, single table:

```sql
CREATE TABLE items (
    id       INTEGER PRIMARY KEY,   -- autoincrement row id
    pro_id   INTEGER,               -- synthesized: int(posVal ++ gridPosition)  ⚠ collision-prone
    pro_name TEXT,
    qty      INTEGER,
    price    TEXT,                  -- unit price as string
    line_tot INTEGER                -- qty * price, precomputed
);
```

Operations: `addShop` (insert), `updateCart` (update by `pro_id`), `dropCart` (delete by
`pro_id`), `Exists` (full-table scan in Java rather than a WHERE clause), `getAllCarts`,
`getShopsCount`, `deleteAll`. The naming ("shop") is leftover from copied tutorial code —
rows are actually cart *line items*.

The cart survives app restarts; it is only cleared by the Print button.

### 5.3 SharedPreferences

| Store | Keys | Used for |
|---|---|---|
| Default prefs (`Utility.setPrefs/getPrefs`) | `route_name`, `shop_name` | Selected route/shop. Read in FinalListActivity but never displayed/printed. |
| `PRODUCT_APP` / key `Product_Favorite` (`SharedPreference.java`, Gson) | JSON list of `Carts` | **Dead code** — an earlier cart implementation superseded by SQLite. Still called once with an always-empty list. |

### 5.4 The unused PDF module (`ProductListAllPDF.java`)

A complete iText report generator: creates `/sdcard/MYBAKE/REPORT_PRODUCT/<name>.pdf` with a
title page, a 4-column table, and page-number footers. It is **never called** and is still
full of copy-paste template artifacts: title "PLUS Electronics Pvt. Ltd.", footer
"Powered By SIAS ERP", hardcoded date "17.12.2015", table headers (Product/Brand/Category/Unit)
that don't match the data it would be fed, and an empty `generateTableData()`. It would also
fail at runtime because the manifest declares no `WRITE_EXTERNAL_STORAGE` permission.
This was clearly the abandoned starting point for the printing feature.

---

## 6. Code Inventory

```
MyBake/
├── build.gradle                     Root build (AGP 3.0.1, google-services 3.1.1, Fabric, jcenter)
├── settings.gradle                  Single module ':app'
├── gradle/wrapper/                  Gradle 4.1
└── app/
    ├── build.gradle                 SDK 26, deps (support libs, Firebase 11.6.0, Gson, iText, Crashlytics)
    ├── google-services.json         ⚠ Firebase config committed to source control
    ├── fabric.properties            ⚠ Fabric secret committed to source control
    ├── libs/itextg-5.5.4*.jar       Bundled iText for Android (+ javadoc jar wrongly added as a dependency)
    └── src/main/
        ├── AndroidManifest.xml      INTERNET permission; 4 activities; hardcoded Fabric API key
        ├── java/…/mybake/
        │   ├── activity/            RouteActivity, CategoryActivity, ProductsActivity, FinalListActivity
        │   ├── adapter/             CategoryListAdapter, ProductListAdapter, FinalListAdapter
        │   ├── models/              Category, Product, Carts (all simple POJOs)
        │   └── util/                DBHandler (SQLite), Utility (prefs + constants),
        │                            SharedPreference (dead), ProductListAllPDF (dead)
        └── res/
            ├── layout/              4 screens + 3 RecyclerView item layouts
            ├── values/strings.xml   ⚠ Route & shop master data hardcoded here
            └── drawable*/…          Icons (+/−, print, search, arrow), bakery photo
```

Roughly 1,600 lines of Java, of which an estimated 30–35 % is commented-out experiments
and debug logging — consistent with a learning-on-the-job build.

---

## 7. What Works Today (functional inventory)

| Feature | Status |
|---|---|
| Route selection + validation | ✅ Works |
| Shop selection + validation | ✅ Works |
| Live category list from Firebase | ✅ Works (with duplication bug B3) |
| Live product list per category | ✅ Works (same bug) |
| Add/remove quantities, cart persisted locally | ⚠ Works with significant bugs (B4, B5, B7) |
| Final list with line totals + subtotal | ✅ Works |
| Route/shop on the final invoice | ❌ Captured but never shown |
| Printing (the device's whole purpose) | ❌ Not implemented — placeholder toast |
| Order history / upload to Firebase | ❌ Not implemented — data deleted on "Print" |
| Multi-shop trip (cart per shop) | ❌ One global cart; changing shop mid-order mixes items |
| Building the project in 2026 | ❌ Broken (dead Fabric repo, jcenter, ancient AGP) |

---

## 8. Defect Register

Ordered by impact on the business.

**B1 — Print button is a stub (blocker).**
`FinalListActivity` "Print" shows *"Testing to be done"*, deletes the cart, and exits. No
printer SDK (no Bluetooth/ESC-POS/vendor SDK) exists anywhere in the project, and the iText
PDF module that was meant to feed printing was never wired up (§5.4).

**B2 — No order record (blocker).**
Orders are never uploaded to Firebase or kept locally after "printing". The company gets no
sales data, no shop order history, no reconciliation trail. `deleteAll()` on print destroys
the only copy.

**B3 — Firebase list duplication.**
Both `CategoryActivity.prepareCategoryData()` and `ProductsActivity.prepareProductData()`
attach `addValueEventListener` (fires on *every* database change, forever) and append to the
backing list without clearing it first. Any edit to the catalogue while the app is open
duplicates every card on screen. Listeners are also never removed, leaking them across the
activity lifecycle. Fix: clear the list at the top of `onDataChange`, or use
`addListenerForSingleValueEvent`.

**B4 — RecyclerView recycling corrupts displayed quantities.**
The quantity counter lives in the **ViewHolder** (`MyViewHolder.qty`), not in the data model.
With a 3-column grid, scrolling recycles holders, so a product can show another product's
quantity, and revisiting a category always shows 0 even when the SQLite cart has items.
Worse, pressing + on such a product then **overwrites** the real DB quantity with the stale
holder value (`updateCart`). Quantity must live in the `Product`/cart model and be re-read
from the DB in `onBindViewHolder`.

**B5 — Colliding synthetic product IDs.**
Cart keys are built as `Integer.parseInt(String.valueOf(posVal) + String.valueOf(position))` —
category position 1 + product 12 → `112`, and category 11 + product 2 → `112` as well. Two
different products can overwrite each other's cart rows. IDs are also *positional*, so any
reordering of the Firebase data shifts every ID. The Firebase push key should be the cart key.

**B6 — Crash on non-integer prices.**
`Carts` constructors do `Integer.parseInt(price) * quantity`. Any price entered in Firebase as
`"12.50"`, `"12 "`, or empty crashes the app the moment + is tapped. Given prices are
maintained by hand in the Firebase console, this is a live operational risk.

**B7 — Crash pressing − before +.**
`ProductListAdapter.db` is only initialized inside the + handler. A − press that reaches
`db.dropCart(...)`/`db.updateCart(...)` first throws a `NullPointerException`.
(Reaching it requires the recycled-holder state of B4, which makes it a real path.)

**B8 — `Category.setName()` self-assignment.**
`this.categoryName = categoryName;` assigns the field to itself instead of the parameter.
Currently harmless (Firebase populates the public field directly) but a trap.

**B9 — One global cart across shops.**
The cart has no route/shop dimension. If the salesman backs out and picks a different shop
without "printing", the previous shop's items silently merge into the new order.

**B10 — UX / cosmetic defects.**
Max-quantity toast says "four" while the limit is 20; "Logical Erro" typo; layout filename
`activity_catgory.xml`; duplicate "French Bakery" entries in the shop list; unused search
and print icons shipped in drawables; `strings.xml` contains stray text (`my_bake_shop_prompt`
and comment lines) that will be rendered by some tooling.

**B11 — Master data hardcoded in the APK.**
Routes and shops live in `strings.xml`. Adding a route or a new customer shop requires
rebuilding and redistributing the app, while categories/products are already dynamic in
Firebase. Routes and shops (and a shop→route mapping) belong in Firebase too.

---

## 9. Security & Privacy Review

| Finding | Severity | Detail |
|---|---|---|
| Firebase RTDB likely world-readable (and possibly writable) | **High** | No authentication code exists, so the DB rules must permit anonymous access for the app to function. Anyone with the URL (`https://my-bake.firebaseio.com/.json`) could read — and if writes were never locked down, alter — the product catalogue and prices. Verify the rules in the Firebase console immediately. |
| `google-services.json` committed | Medium | API key + project identifiers in source control. These keys are not full secrets, but combined with open DB rules they hand out everything needed. |
| Fabric API key hardcoded in manifest + `fabric.properties` committed | Medium | Moot now that Fabric is dead, but indicates secret-handling habits; rotate anything reused. |
| `android:allowBackup="true"` | Low | Cart DB and prefs are extractable via ADB backup on these OS versions. |
| Crashlytics `.debuggable(true)` in production | Low | Verbose crash-kit logging left enabled. |
| No ProGuard/R8 (`minifyEnabled false`) | Low | APK trivially decompilable (which is also how this analysis was easy). |

---

## 10. Build & Toolchain Status (why it won't compile in 2026)

1. **Fabric is gone.** `maven.fabric.io` was decommissioned when Google shut Fabric down
   (2020). Both the root and app `build.gradle` pull the `io.fabric.tools:gradle:1.+` plugin
   from it — dependency resolution fails before compilation starts. Fabric/Crashlytics must
   be removed or replaced with Firebase Crashlytics.
2. **jcenter()** is sunset (read-only since 2021, unreliable) — must move to `mavenCentral()`.
3. **Gradle 4.1 / AGP 3.0.1** cannot run on any modern JDK or be opened by a current Android
   Studio; requires roughly JDK 8 and a ~2017 Android Studio 3.0 to build as-is.
4. **compileSdk/targetSdk 26** — Google Play today requires targetSdk 34+; the support
   libraries used were frozen in 2018 in favour of AndroidX.
5. Legacy `compile` dependency configuration (removed in later Gradle versions).
6. The iText **javadoc** jar is declared as a code dependency (`implementation files(...)`) —
   harmless but wrong.

**Practical implication:** any bug fix, however small, first requires a toolchain migration
(AndroidX, AGP 8.x, Fabric removal). Plan for that as the first work package.

---

## 11. Recommendations & Modernization Roadmap

### Phase 0 — Protect the business today (no code)
- **Audit Firebase rules** for `my-bake` right now; lock writes (and ideally reads) behind auth.
- Export a snapshot of `/category` and `/product` as a backup of your master data.

### Phase 1 — Make it buildable (1–2 days of work)
- Remove Fabric/Crashlytics entirely (or swap to Firebase Crashlytics).
- Migrate to AndroidX, AGP 8.x, Gradle 8.x, compile/targetSdk 34+, `mavenCentral()`.
- Update Firebase SDKs (BoM) and replace the bundled iText jars with a maintained dependency
  — or drop iText if printing goes straight to ESC/POS (below).

### Phase 2 — Finish the core product
1. **Printing** — the reason the hardware exists. Identify the device's printer interface
   (integrated POS devices ship a vendor SDK — e.g. Sunmi/iMin/PAX — or accept generic
   Bluetooth ESC/POS commands) and print a real receipt: company header, route, shop,
   date/time, line items, subtotal. Route/shop are already captured; they just need to be used.
2. **Order upload** — on print, write the order to Firebase
   (`/orders/{date}/{route}/{shop}/{orderId}`) *before* clearing the cart. This single change
   gives the company order history, per-shop statements, and end-of-day reconciliation.
3. **Fix defects B3–B9** (listed with fixes in §8) — about a week of careful work.
4. **Move routes & shops into Firebase** with a shop→route mapping, so the office can add a
   customer without shipping a new APK.

### Phase 3 — Worth considering after that
- Simple salesman login (Firebase Auth) so orders carry who sold them.
- Offline mode (Firebase disk persistence + queued order upload) for coverage gaps on the road.
- Cart-per-shop, order editing/void, day-summary screen, VAT field on the receipt (UAE VAT
  arrived in Jan 2018 — the same month this app was finished — and the app has no tax concept).
- Longer term: this codebase is small enough (~1,600 lines) that a clean rewrite
  (Kotlin + Jetpack Compose, or even keeping the same screens) is cheaper than deep surgery,
  while reusing the existing Firebase data as-is.

---

## Appendix A — Key Identifiers

| Item | Value |
|---|---|
| Package / applicationId | `com.developers.neoline.mybake` |
| Firebase project / DB | `my-bake` / `https://my-bake.firebaseio.com` |
| Firebase nodes read | `/category`, `/product/{categoryKey}` |
| Local DB | `cartinfo`, table `items` |
| SharedPreferences keys | `route_name`, `shop_name` (+ dead `PRODUCT_APP`/`Product_Favorite`) |
| Intended PDF output path | `<external storage>/MYBAKE/REPORT_PRODUCT/*.pdf` (never used) |
| Launcher activity | `.activity.RouteActivity` |

## Appendix B — Screen → Class → Layout Map

| Step | Activity | Adapter | Layouts |
|---|---|---|---|
| Route selection | `RouteActivity` | — | `activity_route` |
| Shop + categories | `CategoryActivity` | `CategoryListAdapter` | `activity_catgory`, `category_list_single_unit` |
| Products / cart | `ProductsActivity` | `ProductListAdapter` | `activity_products`, `product_list_single_unit` |
| Final order list | `FinalListActivity` | `FinalListAdapter` | `activity_final_list`, `final_list_single_unit` |
