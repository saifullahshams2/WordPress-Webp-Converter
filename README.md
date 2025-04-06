🖼️ WordPress Image Converter to WebP – Code Snippet

This repository contains a single PHP file designed as a **code snippet** for WordPress to **convert images to WebP**, **optimize them**, **replace image URLs**, and optionally **delete the original file types**.

Created by [**WebSquadron**](https://www.youtube.com/@websquadron/), this is **version 2.1** of the script.

---

## 🚀 Features

- ✅ Converts JPEG, PNG, and GIF images to WebP format
- ✅ Replaces image URLs in WordPress posts and pages
- ✅ Deletes original image files (optional)
- ✅ Optimizes existing images for performance
- ✅ Lightweight single-file implementation (no plugin required)


---

## 📂 Installation

1. **Copy** the code from `wp-webp-converter.php` file.
2. Upload it to your theme's `functions.php` (Not Recommended)
OR 
4. Use a code snippet plugin (Recommended)
5. You Will find Image Menu inside Media Menu.


## **How It Works** From Original Creator

Convert images to WebP with responsive sizes (e.g: 1920, 1200, 600, 300) and a 150x150 thumbnail to save on File storage. The original file will be deleted unless "Preserve" is selected. The log will inform you when conversions are complete.

Apply for New Uploads:
1. Resize Mode: Select scaling is set on Width or Height.
2. Set Sizes: Add preferred sizes (up to 4). No need to add 150.
3. Set Quality: Slide for conversion levels. (Default is 80).
4. Keep Original: Original image database files are not deleted.
5. Upload: Upload within Media Library or via a widget/element.

Apply for Existing Images:
1. Repeat: Follow Steps 1 to 4 from New Uploads
2. Run All: Click to Execute the Full Sequence.

Apply Manually for Existing Images:
1. Repeat: Follow Steps 1 to 4 from New Uploads
2. Convert/Scale: Process existing images to WebP.
3. Cleanup Images: Remove non-WebP files (unless preserved).
4. Fix URLs: Update Media URLs to use WebP.

NOTE:
a) Apply No Conversion: Deactivate Snippet before uploading images.
b) Backups: Use a robust backup system before running any code/tool.
c) Processing Speed: This depends on server, number of images.
d) Log Delays: Updates appear in batches of 50.
e) Option to Stop: Click Stop to stop the process.
