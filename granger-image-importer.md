## Granger Image Importer

Import images and rich metadata into WordPress from a CSV/XLSX file, then optionally transfer those records into Easy Digital Downloads (EDD) products. Includes tools to repair legacy file-path metadata and clean up incomplete imports.

### Key features
- **Custom post type**: `granger_image` with featured image and rich metadata
- **Taxonomies**: `granger_category` (hierarchical), `granger_keywords` (tags)
- **Excel/CSV import**: Reads CSV directly; parses first sheet of XLSX
- **Local folder upload**: Upload an entire image folder from your machine to WordPress uploads
- **Batch import**: Creates posts, sets featured image, maps taxonomies and meta
- **EDD transfer**: Creates/updates `download` posts with price, SKU, thumbnail, terms, and key meta
- **Maintenance tools**: Repair legacy UNC/IP media URLs; delete posts missing a featured image


## Requirements
- WordPress 5.6+ with Administrator access
- PHP 7.4+ (PHP 8.x supported)
- PHP `ZipArchive` extension for parsing XLSX files
- Recommended: Chromium-based browser (Chrome/Edge) for folder upload support
- Optional: Easy Digital Downloads (EDD) plugin for product transfer


## Installation
1. Copy the plugin folder contents into `wp-content/plugins/` (folder name can remain `granger-image-importer`).
2. In WordPress Admin → Plugins, activate “Granger Image Importer”.
3. If you plan to use EDD transfer, install and activate the EDD plugin first.


## Data model
- **Post type**: `granger_image`
- **Taxonomies**:
  - `granger_category`
  - `granger_keywords`
- **Core meta keys** (subset):
  - `granger_id`, `granger_date`, `granger_caption`, `granger_format`, `granger_location`, `granger_source`, `granger_artist`, `granger_notes`, `granger_restrictions`, `granger_short_description`
  - Resolution/URL fields: `granger_thumbnail`, `granger_preview`, `granger_low_res`, `granger_med_res`, `granger_hi_res`, `granger_super_res`, `granger_ultra_res`, `granger_archival_res`, `granger_www`


## End‑to‑end workflow
1. Go to Admin → Granger Importer.
2. Step 1: Upload CSV/XLSX containing records and metadata.
3. Step 2: Select your local images folder and click “Upload Selected Folder”. This copies all files to `wp-content/uploads/granger-images/` on the server and remembers that path.
4. Step 3: Start the import. The importer will:
   - Read your CSV/XLSX and for each row:
   - Locate the corresponding image file on the server by ID (see Image filenames below)
   - Create a `granger_image` post with featured image
   - Set categories and keywords
   - Map metadata from columns to post meta (with UNC/IP cleanup for media URLs)
   - Derive a short description (20 words) from Description and store as excerpt/meta
5. Step 4 (optional): Transfer to EDD. Creates/updates `download` products with price, SKU, thumbnail, categories/tags, and copied metadata.


## CSV/XLSX format
Use CSV for best performance. First row must contain headers. Supported headers are case-insensitive for core fields.

Minimum recommended headers:
- `Id` (or `id`)
- `Title` (or `title`)
- `Description` (or `description`) – optional but recommended
- `Category` – single value
- `Keywords` – pipe or comma separated (e.g. `painting|renaissance|portrait`)

Additional optional headers mapped to meta:
- `Date`, `Caption`, `Fmt`/`Format`, `Loc`/`Location`, `Source`, `Artist Name`/`Artist`, `Notes`, `Restrictions`, `Processed date`, `Fine Art URL`, `API ID`, `Britannica ID`, `FAA Sales`, `First Imported`, `Thumbnail`, `Preview`, `Low-Res`, `Med-Res`, `Hi-Res`, `Super-Res`, `Ultra-Res`, `Archival-Res`, `WWW`

Example CSV snippet:
```csv
Id,Title,Description,Category,Keywords,Date,Caption,Loc,Artist Name
12345,Mona Lisa,"A portrait by Leonardo da Vinci.",Renaissance,"painting|portrait|Italy",1503,"La Gioconda",Florence,Leonardo da Vinci
02346,The Starry Night,"Vincent van Gogh's masterpiece.",Post-Impressionism,"oil,night sky,Netherlands",1889,,Saint-Rémy,Van Gogh
```


## Image filenames and matching
For each row, the importer looks for a matching image in the server folder from Step 2 using these patterns (in order), with extensions: `.jpg`, `.jpeg`, `.png`, `.gif`, `.tiff`.
- `<Id>`
- `0<Id>` (leading zero)
- `<Id>_h`
- `0<Id>_h`

Examples for ID `12345`:
- `12345.jpg`, `012345.jpg`, `12345_h.jpeg`, `012345_h.tiff`

If the image file is not found in the server folder, that row is skipped.


## Using the admin screens
### Step 1: Upload CSV/XLSX
- Choose your file and upload. The path is stored server-side as `granger-excel-import.<ext>` in the uploads directory.

### Step 2: Upload images folder
- Click “Select Folder” and pick the parent folder containing your images.
- Click “Upload Selected Folder”. Files upload sequentially. The target directory becomes `wp-content/uploads/granger-images/` and will pre-fill the path for Step 3.

### Step 3: Run import
- Confirm the “Images Folder Path (on server)” is correct.
- Choose a **Batch Size** (50–100 recommended to start). The process runs server-side; progress shows in the UI.

### Step 4: Transfer to Easy Digital Downloads (optional)
- Requires the EDD plugin (the page shows status). Configure:
  - **Default Price**: single price stored on the Download
  - **Batch Size**: items per request
  - **Update Existing**: if checked, re-syncs already-linked Downloads; otherwise only creates missing ones
- The plugin avoids duplicates by checking, in order: linked meta, SKU (uses formatted `granger_id`), slug/title, and matching thumbnail.


## How metadata is mapped
- Title/Content: from CSV `Title`/`Description`
- Excerpt: derived short description (20 words) or `granger_short_description` if provided
- Taxonomies:
  - `Category` → `granger_category`
  - `Keywords` (comma or pipe separated) → `granger_keywords`
- Meta fields: the plugin tries multiple header variants. For example:
  - `granger_format` ⇐ `Fmt`, `fmt`, `Format`
  - `granger_location` ⇐ `Loc`, `loc`, `Location`
  - ...and others listed above
- UNC/IP cleanup: if a meta value looks like a UNC path (`\\server\share`), a single backslash path (`\path`), an IPv4 path (e.g. `172.x.x.x`), or includes `gchires`, it is replaced with the actual featured image URL.


## EDD transfer details
- Creates or reuses `download` posts.
- Sets:
  - Featured image (from `granger_image` post thumbnail)
  - Single price meta (`edd_price`)
  - SKU (`edd_sku`) from formatted `granger_id` (also stored as `image_id`)
  - Files (`edd_download_files`) with the image URL for simple setups
  - Copies key metadata for template use (e.g., `image_short_description`)
  - Maps `granger_category` → `download_category`, `granger_keywords` → `download_tag` (creates terms if missing)
- Update mode respects existing content: it fills empty fields on the EDD side and syncs excerpt/price/SKU as needed.


## Tools page
Admin → Granger Importer → Tools
- **Repair Media URLs**: Replaces legacy UNC/IP file paths in image URL meta keys with the actual featured image URL.
- **Delete Posts Without Featured Image**: Removes any `granger_image` posts missing a thumbnail (useful after a failed run before images were uploaded).


## Permissions and security
- All admin actions require `manage_options` capability (Administrators only).
- All AJAX actions are nonce-protected.


## Performance tips
- Prefer CSV over XLSX for large imports.
- Start with a batch size of 50–100 and increase if your server can handle it.
- Ensure PHP max execution time and memory are sufficient for your dataset and image sizes.


## Troubleshooting
- **Upload fails**: Verify file size limits (`upload_max_filesize`, `post_max_size`) and use CSV.
- **XLSX reads empty**: Ensure the first row has headers and the file has a first worksheet named `sheet1` (the importer reads the first sheet). Confirm `ZipArchive` is installed.
- **Images not found**: Confirm the server folder path is correct and filenames follow the matching patterns.
- **Timeouts**: Reduce batch size; increase PHP time/memory limits.
- **EDD transfer button disabled**: Install and activate Easy Digital Downloads.


## FAQ
- **Can I use .xls files?** No. Save as CSV or XLSX.
- **Can I re-run the import safely?** Yes. Existing `granger_image` posts with the same `granger_id` are skipped. Use Tools to clean up if needed.
- **How is `granger_id` formatted?** Non-digits are removed and a leading zero is added if not present (e.g. `12345` → `012345`).
- **Which browsers support folder upload?** Chrome/Edge support selecting a folder. On other browsers, compress and upload via another method or use a supported browser.


## Developer notes
- Admin assets are enqueued only on plugin pages.
- AJAX actions:
  - `upload_excel_file`, `upload_granger_images_batch`, `import_granger_images`, `transfer_granger_to_edd`
- Storage locations:
  - Excel/CSV: `wp-content/uploads/granger-excel-import.<ext>`
  - Images: `wp-content/uploads/granger-images/`


## License
This project is provided as-is without warranty. See plugin header for author attribution.


