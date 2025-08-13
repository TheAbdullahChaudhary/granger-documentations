## Granger Client Importer

Import clients/users from CSV or XLSX into WordPress with fast batch processing, duplicate prevention, automatic user meta mapping, export tools, and duplicate cleanup utilities.

### Key Features
- **CSV/XLSX import**: Upload a spreadsheet with a header row; columns are auto-detected by smart aliases.
- **Duplicate prevention**: Existing users are skipped based on email and/or username.
- **Automatic field mapping**: Unknown columns are stored as user meta with the `gci_` prefix for easy querying and export.
- **Batch processing**: Tunable batch size with adaptive backoff when the server is busy.
- **Role assignment**: Set a default role for all newly created users.
- **Password handling**: Supports plaintext passwords or “passwords are hashed” mode with optional reset emails.
- **Approval**: Newly created users are marked approved (`user_status = 0`).
- **Progress UI and logs**: Real-time progress bar, log stream, and summary.
- **Users data view + CSV export**: View imported meta fields and export per-page or all users.
- **Duplicate cleanup tools**: Scan and delete duplicates by email or username.
- **Batch cleanup**: Delete all users created in a specific import batch.
- **Contact banner shortcode**: `[gci_contact_banner]` for a simple contact CTA.

---

### Requirements
- WordPress 5.8+ (recommended)
- PHP 7.4+ (PHP 8.x compatible)
- PHP extension: `ZipArchive` (required for `.xlsx` parsing)
- Sufficient PHP memory/time limits for large imports
- Outbound email configured if you plan to send password reset emails

### Installation
1. Copy `granger-client-importer.php` into your WordPress site at `wp-content/plugins/granger-client-importer/`.
2. In the WordPress admin, go to Plugins → “Granger Client Importer” → Activate.
3. On activation, a custom table `wp_gci_user_sheet_data` is created for per-user JSON snapshots.

### Supported File Formats
- **CSV**: Fastest; must include a single header row.
- **XLSX**: Sheet1 is read; must include a single header row.

Headers are matched by aliases. Examples:
- **Email**: `New Email`, `Email`, `Old Email`, `E-mail`, `Email Address`, `user_email`
- **Username**: `Username`, `Login`, `User`, `user_login`, `User Name`
- **Names**: `First Name`/`Last Name` (also recognizes `Firstname`, `Surname`, `cntct fst`, `cntct lst`)
- **Display Name**: `Display Name`, `DisplayName`, `Name`, `Company`
- **Role**: `Role`, `User Role`
- **Password**: `Password`, `Pass`, `Pwd`
- **Website**: `WWW`, `Website`, `URL`, `Web`

Unknown columns are saved as user meta with the `gci_` prefix. For example, a header `Customer Code` becomes meta key `gci_customer_code`.

---

### End-to-End Workflow
1. **Open the importer**: In WP Admin, go to “Client Importer”.
2. **Upload file (Step 1)**: Select a `.csv` or `.xlsx`. The first row must be headers. On success, the file is saved to your uploads folder (e.g., `.../uploads/gci-import.csv`).
3. **Configure & start (Step 2)**:
   - Choose a default role for all new users.
   - Choose password mode:
     - Leave unchecked to use plaintext passwords from the file (or generate random passwords if none provided).
     - Check “Passwords are hashed” if your file contains hashes unknown to WordPress (e.g., legacy system). In this mode, a temporary password is set so users can log in after a reset.
   - Optionally send reset emails to newly created users.
   - Set batch size (start with 100–300; importer adapts automatically if the server is busy).
   - Click “Start Import”. Keep the page open until complete.
4. **Progress & logs**: Watch created/skipped entries and the log stream. On completion you’ll see a summary.
5. **After import**:
   - New users are ready to log in. Existing users were skipped and not modified.
   - All source columns are stored as user meta with `gci_` prefix.
   - The import batch ID is recorded in user meta `gci_import_batch` (format like `YYYYMMDD-HHMMSS`).

---

### Usage Details
#### Duplicate handling
- During import, if an email or username already exists, that row is skipped (no updates are made to existing users).
- Separate tools are provided to scan and clean up duplicates that may have been introduced historically.

#### Username generation
- If the username field is missing, a username is derived from the email’s local part. If taken, a stable hash-based variant is used (no `_1/_2` suffixes).

#### Passwords and reset emails
- Plaintext mode: Uses the provided password or generates a random one.
- Hashed mode: Sets a temporary WordPress password and stores the original hash in user meta `gci_password_hash`. WordPress cannot use arbitrary external hashes, so enabling “send reset email” is recommended.

#### User approval
- Newly created users are set to `user_status = 0` (approved).

---

### Admin Pages
#### Client Importer
- Step 1: Upload CSV/XLSX.
- Step 2: Configure role, passwords, and batch size; start the import and monitor progress.

#### All Users Data
- View a wide table of `ID`, `Username`, `Email`, plus all discovered `gci_` meta fields (derived from import headers).
- Pagination controls; per-page count up to 200.
- Export buttons:
  - “Export This Page (CSV)” respects current pagination.
  - “Export All Users (CSV)” streams all users.
- Certain sensitive or noisy headers are excluded from display/export by default.

#### Duplicate Cleanup
- Scan for duplicate users by email or by username.
- Results are grouped; for each group, you can delete all but the oldest user.

#### Tools
- Delete all users created in a specific import batch. Enter the batch ID (e.g., `20250808-1` or `YYYYMMDD-HHMMSS`) and confirm.
- Batch IDs are stored in user meta `gci_import_batch`. You can retrieve one from any imported user profile or from your logs.

---

### Tips for Large Imports
- Prefer CSV for speed.
- Start with a moderate batch size (100–300). The importer reduces batch size automatically if the server is busy.
- Keep the browser tab open during the import.
- Increase PHP memory/time limits when importing very large files.

---

### Troubleshooting
- “No file uploaded” or “Failed to save file”: Ensure your uploads directory is writable and your file is `.csv` or `.xlsx`.
- “Please upload CSV or XLSX”: Only these two formats are supported.
- XLSX reads nothing: Verify the headers are on the first row of Sheet1; ensure `ZipArchive` is enabled.
- Server busy/retries: The importer will retry with exponential backoff and reduce batch size as needed. Consider lowering batch size or increasing server resources.
- Reset emails not received: Configure SMTP or a transactional email service.
- Existing users not updated: By design, the importer skips existing users entirely.

---

### Developer Notes
- Main plugin file: `granger-client-importer.php`
- Custom table: `{$wpdb->prefix}gci_user_sheet_data`
  - Columns: `id`, `user_id`, `batch_id`, `data_json`, `created_at`, `updated_at`
  - Stores a JSON snapshot of the imported row (with password-like fields removed)
- User meta:
  - `gci_*` keys for all imported columns (excluding password fields)
  - `gci_import_batch` for the batch identifier
  - `gci_password_hash` if “passwords are hashed” mode is used
- AJAX actions (admin):
  - `gci_upload_file`, `gci_start_import`, `gci_bulk_set_role`, `gci_find_duplicates`, `gci_delete_duplicates`
- Shortcode:
  - `gci_contact_banner` with optional attributes `text`, `email`, `name`
    - Example: `[gci_contact_banner text="Need help with a custom plugin?" email="you@example.com" name="Your Name"]`

---

### Changelog
- 1.1.3
  - UI/UX improvements, progress logs, duplicate cleanup, users data export, custom DB storage.

### License
Proprietary or as distributed by the project owner.


