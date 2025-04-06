// Helper function for formatting file sizes
function formatBytes($bytes, $precision = 2) {
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    $bytes = max($bytes, 0);
    $pow = ($bytes > 0) ? floor(log($bytes) / log(1024)) : 0;
    $pow = min($pow, count($units) - 1); // Moved up to prevent array out-of-bounds
    $bytes /= pow(1024, $pow);
    return round($bytes, $precision) . ' ' . $units[$pow];
}

// Limit default WordPress sizes to thumbnail only
function wpturbo_limit_image_sizes($sizes) {
    return ['thumbnail' => $sizes['thumbnail']];
}
add_filter('intermediate_image_sizes_advanced', 'wpturbo_limit_image_sizes');

// Set thumbnail size to 150x150
function wpturbo_set_thumbnail_size() {
    update_option('thumbnail_size_w', 150);
    update_option('thumbnail_size_h', 150);
    update_option('thumbnail_crop', 1);
}
add_action('admin_init', 'wpturbo_set_thumbnail_size');

// Register custom sizes (up to 3 additional sizes beyond the main one)
add_action('after_setup_theme', 'wpturbo_register_custom_sizes');
function wpturbo_register_custom_sizes() {
    $mode = wpturbo_get_resize_mode();
    if ($mode === 'width') {
        $max_values = wpturbo_get_max_widths();
        $additional_values = array_slice($max_values, 1, 3); // Up to 3 additional sizes
        foreach ($additional_values as $width) {
            add_image_size("custom-$width", $width, 0, false);
        }
    } else {
        $max_values = wpturbo_get_max_heights();
        $additional_values = array_slice($max_values, 1, 3);
        foreach ($additional_values as $height) {
            add_image_size("custom-$height", 0, $height, false);
        }
    }
}

// Get or set max widths (default to mobile-friendly set, limit to 4)
function wpturbo_get_max_widths() {
    $value = get_option('webp_max_widths', '1920,1200,600,300');
    $widths = array_map('absint', array_filter(explode(',', $value)));
    $widths = array_filter($widths, function($w) { return $w > 0 && $w <= 9999; }); // Validate range
    return array_slice($widths, 0, 4); // Up to 4 sizes
}

// Get or set max heights (default to mobile-friendly set, limit to 4)
function wpturbo_get_max_heights() {
    $value = get_option('webp_max_heights', '1080,720,480,360');
    $heights = array_map('absint', array_filter(explode(',', $value)));
    $heights = array_filter($heights, function($h) { return $h > 0 && $h <= 9999; }); // Validate range
    return array_slice($heights, 0, 4);
}

// Get or set resize mode
function wpturbo_get_resize_mode() {
    return get_option('webp_resize_mode', 'width');
}

// Get or set quality (0-100)
function wpturbo_get_quality() {
    return (int) get_option('webp_quality', 80);
}

// Get or set batch size
function wpturbo_get_batch_size() {
    return (int) get_option('webp_batch_size', 5);
}

// Get or set preserve originals
function wpturbo_get_preserve_originals() {
    return (bool) get_option('webp_preserve_originals', false);
}

// Get excluded image IDs
function wpturbo_get_excluded_images() {
    $excluded = get_option('webp_excluded_images', []);
    return is_array($excluded) ? array_map('absint', $excluded) : [];
}

// Add an image to the excluded list
function wpturbo_add_excluded_image($attachment_id) {
    $attachment_id = absint($attachment_id);
    $excluded = wpturbo_get_excluded_images();
    if (!in_array($attachment_id, $excluded)) {
        $excluded[] = $attachment_id;
        update_option('webp_excluded_images', array_unique($excluded));
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Excluded image added: Attachment ID %d', 'wpturbo'), $attachment_id);
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Remove an image from the excluded list
function wpturbo_remove_excluded_image($attachment_id) {
    $attachment_id = absint($attachment_id);
    $excluded = wpturbo_get_excluded_images();
    $index = array_search($attachment_id, $excluded);
    if ($index !== false) {
        unset($excluded[$index]);
        update_option('webp_excluded_images', array_values($excluded));
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Excluded image removed: Attachment ID %d', 'wpturbo'), $attachment_id);
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Core WebP conversion function
function wpturbo_convert_to_webp($file_path, $dimension, &$log = null, $attachment_id = null, $suffix = '') {
    $path_info = pathinfo($file_path);
    $new_file_path = $path_info['dirname'] . '/' . $path_info['filename'] . $suffix . '.webp';
    $quality = wpturbo_get_quality();
    $mode = wpturbo_get_resize_mode();

    if (!(extension_loaded('imagick') || extension_loaded('gd'))) {
        if ($log !== null) $log[] = sprintf(__('Error: No image library (Imagick/GD) available for %s', 'wpturbo'), basename($file_path));
        return false;
    }

    $editor = wp_get_image_editor($file_path);
    if (is_wp_error($editor)) {
        if ($log !== null) $log[] = sprintf(__('Error: Image editor failed for %s - %s', 'wpturbo'), basename($file_path), $editor->get_error_message());
        return false;
    }

    $dimensions = $editor->get_size();
    $resized = false;
    if ($mode === 'width' && $dimensions['width'] > $dimension) {
        $editor->resize($dimension, null, false);
        $resized = true;
    } elseif ($mode === 'height' && $dimensions['height'] > $dimension) {
        $editor->resize(null, $dimension, false);
        $resized = true;
    }

    $result = $editor->save($new_file_path, 'image/webp', ['quality' => $quality]);
    if (is_wp_error($result)) {
        if ($log !== null) $log[] = sprintf(__('Error: Conversion failed for %s - %s', 'wpturbo'), basename($file_path), $result->get_error_message());
        return false;
    }

    if ($log !== null) {
        $log[] = sprintf(
            __('Converted: %s → %s %s', 'wpturbo'),
            basename($file_path),
            basename($new_file_path),
            $resized ? sprintf(__('(resized to %dpx %s, quality %d)', 'wpturbo'), $dimension, $mode, $quality) : sprintf(__('(quality %d)', 'wpturbo'), $quality)
        );
    }

    return $new_file_path;
}

// Handle new uploads: Convert to WebP, generate sizes, update metadata
add_filter('wp_handle_upload', 'wpturbo_handle_upload_convert_to_webp', 10, 1);
function wpturbo_handle_upload_convert_to_webp($upload) {
    $file_extension = strtolower(pathinfo($upload['file'], PATHINFO_EXTENSION));
    $allowed_extensions = ['jpg', 'jpeg', 'png', 'webp'];

    if (!in_array($file_extension, $allowed_extensions)) {
        return $upload;
    }

    $file_path = $upload['file'];
    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();
    $log = get_option('webp_conversion_log', []);
    $attachment_id = attachment_url_to_postid($upload['url']);
    $new_files = [];

    // Convert to all specified sizes
    foreach ($max_values as $index => $dimension) {
        $suffix = ($index === 0) ? '' : "-{$dimension}";
        $new_file_path = wpturbo_convert_to_webp($file_path, $dimension, $log, $attachment_id, $suffix);
        if ($new_file_path) {
            if ($index === 0) {
                $upload['file'] = $new_file_path;
                $upload['url'] = str_replace(basename($upload['url']), basename($new_file_path), $upload['url']);
                $upload['type'] = 'image/webp';
            }
            $new_files[] = $new_file_path;
        }
    }

    // Generate 150x150 thumbnail
    if ($attachment_id) {
        $editor = wp_get_image_editor($file_path);
        if (!is_wp_error($editor)) {
            $editor->resize(150, 150, true);
            $thumbnail_path = dirname($file_path) . '/' . pathinfo($file_path, PATHINFO_FILENAME) . '-150x150.webp';
            $saved = $editor->save($thumbnail_path, 'image/webp', ['quality' => wpturbo_get_quality()]);
            if (!is_wp_error($saved)) {
                $log[] = sprintf(__('Generated thumbnail: %s', 'wpturbo'), basename($thumbnail_path));
                $new_files[] = $thumbnail_path;
            }
        }

        // Update metadata with all sizes
        $metadata = wp_generate_attachment_metadata($attachment_id, $upload['file']);
        if (!is_wp_error($metadata)) {
            $base_name = pathinfo($file_path, PATHINFO_FILENAME);
            $dirname = dirname($file_path);
            foreach ($max_values as $index => $dimension) {
                if ($index === 0) continue;
                $size_file = "$dirname/$base_name-$dimension.webp";
                if (file_exists($size_file)) {
                    $metadata['sizes']["custom-$dimension"] = [
                        'file' => "$base_name-$dimension.webp",
                        'width' => ($mode === 'width') ? $dimension : 0,
                        'height' => ($mode === 'height') ? $dimension : 0,
                        'mime-type' => 'image/webp'
                    ];
                }
            }
            $thumbnail_file = "$dirname/$base_name-150x150.webp";
            if (file_exists($thumbnail_file)) {
                $metadata['sizes']['thumbnail'] = [
                    'file' => "$base_name-150x150.webp",
                    'width' => 150,
                    'height' => 150,
                    'mime-type' => 'image/webp'
                ];
            }
            $metadata['webp_quality'] = wpturbo_get_quality();
            update_attached_file($attachment_id, $upload['file']);
            wp_update_post(['ID' => $attachment_id, 'post_mime_type' => 'image/webp']);
            wp_update_attachment_metadata($attachment_id, $metadata);
        } else {
            $log[] = sprintf(__('Error: Metadata regeneration failed for %s - %s', 'wpturbo'), basename($file_path), $metadata->get_error_message());
        }
    }

// Clean up original if not preserving
if ($file_extension !== 'webp' && file_exists($file_path) && !wpturbo_get_preserve_originals()) {
    $attempts = 0;
    while ($attempts < 5 && file_exists($file_path)) {
        if (!is_writable($file_path)) {
            @chmod($file_path, 0644); // Use @ to suppress warnings
            if (!is_writable($file_path)) {
                $log[] = sprintf(__('Error: Cannot make %s writable - skipping deletion', 'wpturbo'), basename($file_path));
                break;
            }
        }
        if (@unlink($file_path)) {
            $log[] = sprintf(__('Deleted original: %s', 'wpturbo'), basename($file_path));
            break;
        }
        $attempts++;
        sleep(1);
    }
    if (file_exists($file_path)) {
        $log[] = sprintf(__('Error: Failed to delete original %s after 5 retries', 'wpturbo'), basename($file_path));
        // Removed error_log to avoid server overload
    }
}

    update_option('webp_conversion_log', array_slice($log, -100));
    return $upload;
}

// Fix metadata for WebP images
add_filter('wp_generate_attachment_metadata', 'wpturbo_fix_webp_metadata', 10, 2);
function wpturbo_fix_webp_metadata($metadata, $attachment_id) {
    $file = get_attached_file($attachment_id);
    if (pathinfo($file, PATHINFO_EXTENSION) !== 'webp') {
        return $metadata;
    }

    $uploads = wp_upload_dir();
    $file_path = $file;
    $file_name = basename($file_path);
    $dirname = dirname($file_path);
    $base_name = pathinfo($file_name, PATHINFO_FILENAME);
    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();

    $metadata['file'] = str_replace($uploads['basedir'] . '/', '', $file_path);
    $metadata['mime_type'] = 'image/webp';

    // Ensure all sizes are in metadata
    foreach ($max_values as $index => $dimension) {
        if ($index === 0) continue;
        $size_file = "$dirname/$base_name-$dimension.webp";
        if (file_exists($size_file)) {
            $metadata['sizes']["custom-$dimension"] = [
                'file' => "$base_name-$dimension.webp",
                'width' => ($mode === 'width') ? $dimension : 0,
                'height' => ($mode === 'height') ? $dimension : 0,
                'mime-type' => 'image/webp'
            ];
        }
    }
    $thumbnail_file = "$dirname/$base_name-150x150.webp";
    if (file_exists($thumbnail_file)) {
        $metadata['sizes']['thumbnail'] = [
            'file' => "$base_name-150x150.webp",
            'width' => 150,
            'height' => 150,
            'mime-type' => 'image/webp'
        ];
    }

    return $metadata;
}

// Batch convert existing images
function wpturbo_convert_single_image() {
    check_ajax_referer('webp_converter_nonce', 'nonce');
    if (!current_user_can('manage_options') || !isset($_POST['offset'])) {
        wp_send_json_error(__('Permission denied or invalid offset', 'wpturbo'));
    }

    $offset = absint($_POST['offset']);
    $batch_size = wpturbo_get_batch_size();
    wp_raise_memory_limit('image');
    set_time_limit(max(30, 10 * $batch_size)); // Dynamic timeout

    $args = [
        'post_type' => 'attachment',
        'post_mime_type' => ['image/jpeg', 'image/png', 'image/webp'],
        'posts_per_page' => $batch_size,
        'offset' => $offset,
        'fields' => 'ids',
        'post__not_in' => wpturbo_get_excluded_images(),
    ];

    $attachments = get_posts($args);
    $log = get_option('webp_conversion_log', []);
    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();
    $current_quality = wpturbo_get_quality();

    if (empty($attachments)) {
        update_option('webp_conversion_complete', true);
        $log[] = "<span style='font-weight: bold; color: #281E5D;'>" . __('Conversion Complete', 'wpturbo') . "</span>: " . __('No more images to process', 'wpturbo');
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => true]);
    }

    foreach ($attachments as $attachment_id) {
        $file_path = get_attached_file($attachment_id);
        if (!file_exists($file_path)) {
            $log[] = sprintf(__('Skipped: File not found for Attachment ID %d', 'wpturbo'), $attachment_id);
            continue;
        }

        $metadata = wp_get_attachment_metadata($attachment_id);
        $existing_quality = isset($metadata['webp_quality']) ? (int) $metadata['webp_quality'] : null;
        $is_webp = strtolower(pathinfo($file_path, PATHINFO_EXTENSION)) === 'webp';

        $reprocess = !$is_webp || $existing_quality !== $current_quality;
        if ($is_webp && !$reprocess) {
            $editor = wp_get_image_editor($file_path);
            if (!is_wp_error($editor)) {
                $current_size = $editor->get_size();
                $current_dimension = ($mode === 'width') ? $current_size['width'] : $current_size['height'];
                $reprocess = !in_array($current_dimension, $max_values);
            }
        }

        if (!$reprocess) {
            continue;
        }

        $new_files = [];
        foreach ($max_values as $index => $dimension) {
            $suffix = ($index === 0) ? '' : "-{$dimension}";
            $new_file_path = wpturbo_convert_to_webp($file_path, $dimension, $log, $attachment_id, $suffix);
            if ($new_file_path) {
                if ($index === 0) {
                    update_attached_file($attachment_id, $new_file_path);
                    wp_update_post(['ID' => $attachment_id, 'post_mime_type' => 'image/webp']);
                }
                $new_files[] = $new_file_path;
            }
        }

        $editor = wp_get_image_editor($file_path);
        if (!is_wp_error($editor)) {
            $editor->resize(150, 150, true);
            $thumbnail_path = dirname($file_path) . '/' . pathinfo($file_path, PATHINFO_FILENAME) . '-150x150.webp';
            $saved = $editor->save($thumbnail_path, 'image/webp', ['quality' => $current_quality]);
            if (!is_wp_error($saved)) {
                $log[] = sprintf(__('Generated thumbnail: %s', 'wpturbo'), basename($thumbnail_path));
                $new_files[] = $thumbnail_path;
            }
        }

        if ($attachment_id && !empty($new_files)) {
            $metadata = wp_generate_attachment_metadata($attachment_id, $new_files[0]);
            if (!is_wp_error($metadata)) {
                $base_name = pathinfo($file_path, PATHINFO_FILENAME);
                $dirname = dirname($file_path);
                foreach ($max_values as $index => $dimension) {
                    if ($index === 0) continue;
                    $size_file = "$dirname/$base_name-$dimension.webp";
                    if (file_exists($size_file)) {
                        $metadata['sizes']["custom-$dimension"] = [
                            'file' => "$base_name-$dimension.webp",
                            'width' => ($mode === 'width') ? $dimension : 0,
                            'height' => ($mode === 'height') ? $dimension : 0,
                            'mime-type' => 'image/webp'
                        ];
                    }
                }
                $thumbnail_file = "$dirname/$base_name-150x150.webp";
                if (file_exists($thumbnail_file)) {
                    $metadata['sizes']['thumbnail'] = [
                        'file' => "$base_name-150x150.webp",
                        'width' => 150,
                        'height' => 150,
                        'mime-type' => 'image/webp'
                    ];
                }
                $metadata['webp_quality'] = $current_quality;
                wp_update_attachment_metadata($attachment_id, $metadata);
            } else {
                $log[] = sprintf(__('Error: Metadata regeneration failed for %s', 'wpturbo'), basename($file_path));
            }
        }

if (!$is_webp && file_exists($file_path) && !wpturbo_get_preserve_originals()) {
    $attempts = 0;
    while ($attempts < 5 && file_exists($file_path)) {
        if (!is_writable($file_path)) {
            @chmod($file_path, 0644);
            if (!is_writable($file_path)) {
                $log[] = sprintf(__('Error: Cannot make %s writable - skipping deletion', 'wpturbo'), basename($file_path));
                break;
            }
        }
        if (@unlink($file_path)) {
            $log[] = sprintf(__('Deleted original: %s', 'wpturbo'), basename($file_path));
            break;
        }
        $attempts++;
        sleep(1);
    }
    if (file_exists($file_path)) {
        $log[] = sprintf(__('Error: Failed to delete original %s after 5 retries', 'wpturbo'), basename($file_path));
        // Removed error_log to avoid server overload
    }
}
    }

    update_option('webp_conversion_log', array_slice($log, -100));
    wp_send_json_success(['complete' => false, 'offset' => $offset + $batch_size]);
}

// Progress tracking via AJAX
function wpturbo_webp_conversion_status() {
    check_ajax_referer('webp_converter_nonce', 'nonce');
    if (!current_user_can('manage_options')) {
        wp_send_json_error(__('Permission denied', 'wpturbo'));
    }

    $total = wp_count_posts('attachment')->inherit;
    $converted = count(get_posts([
        'post_type' => 'attachment',
        'posts_per_page' => -1,
        'fields' => 'ids',
        'post_mime_type' => 'image/webp'
    ]));
    $skipped = count(get_posts([
        'post_type' => 'attachment',
        'posts_per_page' => -1,
        'fields' => 'ids',
        'post_mime_type' => ['image/jpeg', 'image/png']
    ]));
    $remaining = $total - $converted - $skipped;
    $excluded_images = wpturbo_get_excluded_images();
    $excluded_data = [];
    foreach ($excluded_images as $id) {
        $thumbnail = wp_get_attachment_image_src($id, 'thumbnail');
        $excluded_data[] = [
            'id' => $id,
            'title' => get_the_title($id),
            'thumbnail' => $thumbnail ? $thumbnail[0] : ''
        ];
    }

    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();

    wp_send_json([
        'total' => $total,
        'converted' => $converted,
        'skipped' => $skipped,
        'remaining' => $remaining,
        'excluded' => count($excluded_images),
        'excluded_images' => $excluded_data,
        'log' => get_option('webp_conversion_log', []),
        'complete' => get_option('webp_conversion_complete', false),
        'resize_mode' => $mode,
        'max_values' => implode(', ', $max_values),
        'max_widths' => implode(', ', wpturbo_get_max_widths()), // Added
        'max_heights' => implode(', ', wpturbo_get_max_heights()), // Added
        'quality' => wpturbo_get_quality(),
        'preserve_originals' => wpturbo_get_preserve_originals()
    ]);
}

// Clear log
function wpturbo_clear_log() {
    if (!isset($_GET['clear_log']) || !current_user_can('manage_options')) {
        return false;
    }
    update_option('webp_conversion_log', [__('Log cleared', 'wpturbo')]);
    return true;
}

// Reset to defaults
function wpturbo_reset_defaults() {
    if (!isset($_GET['reset_defaults']) || !current_user_can('manage_options')) {
        return false;
    }
    update_option('webp_max_widths', '1920,1200,600,300');
    update_option('webp_max_heights', '1080,720,480,360');
    update_option('webp_resize_mode', 'width');
    update_option('webp_quality', 80);
    update_option('webp_batch_size', 5);
    update_option('webp_preserve_originals', false);
    $log = get_option('webp_conversion_log', []);
    $log[] = __('Settings reset to defaults', 'wpturbo');
    update_option('webp_conversion_log', array_slice($log, -100));
    return true;
}

// Set max widths
function wpturbo_set_max_widths() {
    if (!isset($_GET['set_max_width']) || !current_user_can('manage_options') || !isset($_GET['max_width'])) {
        return false;
    }
    $max_widths = sanitize_text_field($_GET['max_width']);
    $width_array = array_filter(array_map('absint', explode(',', $max_widths)));
    $width_array = array_filter($width_array, function($w) { return $w > 0 && $w <= 9999; });
    $width_array = array_slice($width_array, 0, 4);
    if (!empty($width_array)) {
        update_option('webp_max_widths', implode(',', $width_array));
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Max widths set to: %spx', 'wpturbo'), implode(', ', $width_array));
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Set max heights
function wpturbo_set_max_heights() {
    if (!isset($_GET['set_max_height']) || !current_user_can('manage_options') || !isset($_GET['max_height'])) {
        return false;
    }
    $max_heights = sanitize_text_field($_GET['max_height']);
    $height_array = array_filter(array_map('absint', explode(',', $max_heights)));
    $height_array = array_filter($height_array, function($h) { return $h > 0 && $h <= 9999; });
    $height_array = array_slice($height_array, 0, 4);
    if (!empty($height_array)) {
        update_option('webp_max_heights', implode(',', $height_array));
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Max heights set to: %spx', 'wpturbo'), implode(', ', $height_array));
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Set resize mode
function wpturbo_set_resize_mode() {
    if (!isset($_GET['set_resize_mode']) || !current_user_can('manage_options') || !isset($_GET['resize_mode'])) {
        return false;
    }
    $mode = sanitize_text_field($_GET['resize_mode']);
    if (in_array($mode, ['width', 'height'])) {
        $current_mode = get_option('webp_resize_mode', 'width');
        if ($current_mode !== $mode) {
            update_option('webp_resize_mode', $mode);
            $log = get_option('webp_conversion_log', []);
            $log[] = sprintf(__('Resize mode set to: %s', 'wpturbo'), $mode);
            update_option('webp_conversion_log', array_slice($log, -100));
        }
        return true;
    }
    return false;
}

// Set quality
function wpturbo_set_quality() {
    if (!isset($_GET['set_quality']) || !current_user_can('manage_options') || !isset($_GET['quality'])) {
        return false;
    }
    $quality = absint($_GET['quality']);
    if ($quality >= 0 && $quality <= 100) {
        $current_quality = (int) get_option('webp_quality', 80);
        if ($current_quality !== $quality) {
            update_option('webp_quality', $quality);
            $log = get_option('webp_conversion_log', []);
            $log[] = sprintf(__('Quality set to: %d', 'wpturbo'), $quality);
            update_option('webp_conversion_log', array_slice($log, -100));
        }
        return true;
    }
    return false;
}

// Set batch size
function wpturbo_set_batch_size() {
    if (!isset($_GET['set_batch_size']) || !current_user_can('manage_options') || !isset($_GET['batch_size'])) {
        return false;
    }
    $batch_size = absint($_GET['batch_size']);
    if ($batch_size > 0 && $batch_size <= 50) { // Reasonable upper limit
        update_option('webp_batch_size', $batch_size);
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Batch size set to: %d', 'wpturbo'), $batch_size);
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Set preserve originals
function wpturbo_set_preserve_originals() {
    if (!isset($_GET['set_preserve_originals']) || !current_user_can('manage_options') || !isset($_GET['preserve_originals'])) {
        return false;
    }
    $preserve = rest_sanitize_boolean($_GET['preserve_originals']);
    $current_preserve = wpturbo_get_preserve_originals();
    if ($current_preserve !== $preserve) {
        update_option('webp_preserve_originals', $preserve);
        $log = get_option('webp_conversion_log', []);
        $log[] = sprintf(__('Preserve originals set to: %s', 'wpturbo'), $preserve ? __('Yes', 'wpturbo') : __('No', 'wpturbo'));
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Cleanup leftover originals, preserving all WebP sizes
function wpturbo_cleanup_leftover_originals() {
    if (!isset($_GET['cleanup_leftover_originals']) || !current_user_can('manage_options')) {
        return false;
    }

    $log = get_option('webp_conversion_log', []);
    $uploads_dir = wp_upload_dir()['basedir'];
    $files = new RecursiveIteratorIterator(new RecursiveDirectoryIterator($uploads_dir));
    $deleted = 0;
    $failed = 0;
    $preserve_originals = wpturbo_get_preserve_originals();

    $attachments = get_posts([
        'post_type' => 'attachment',
        'posts_per_page' => -1,
        'fields' => 'ids',
        'post_mime_type' => ['image/jpeg', 'image/png', 'image/webp'],
    ]);
    $active_files = [];
    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();
    $excluded_images = wpturbo_get_excluded_images();

    foreach ($attachments as $attachment_id) {
        $file = get_attached_file($attachment_id);
        $metadata = wp_get_attachment_metadata($attachment_id);
        $dirname = dirname($file);
        $base_name = pathinfo($file, PATHINFO_FILENAME);

        if (in_array($attachment_id, $excluded_images)) {
            if ($file && file_exists($file)) $active_files[$file] = true;
            $possible_extensions = ['jpg', 'jpeg', 'png', 'webp'];
            foreach ($possible_extensions as $ext) {
                $potential_file = "$dirname/$base_name.$ext";
                if (file_exists($potential_file)) $active_files[$potential_file] = true;
            }
            foreach ($max_values as $index => $dimension) {
                $suffix = ($index === 0) ? '' : "-{$dimension}";
                $webp_file = "$dirname/$base_name$suffix.webp";
                if (file_exists($webp_file)) $active_files[$webp_file] = true;
            }
            $thumbnail_file = "$dirname/$base_name-150x150.webp";
            if (file_exists($thumbnail_file)) $active_files[$thumbnail_file] = true;
            if ($metadata && isset($metadata['sizes'])) {
                foreach ($metadata['sizes'] as $size_data) {
                    $size_file = "$dirname/" . $size_data['file'];
                    if (file_exists($size_file)) $active_files[$size_file] = true;
                }
            }
            continue;
        }

        // Protect all WebP sizes
        if ($file && file_exists($file)) {
            $active_files[$file] = true;
            foreach ($max_values as $index => $dimension) {
                $suffix = ($index === 0) ? '' : "-{$dimension}";
                $webp_file = "$dirname/$base_name$suffix.webp";
                if (file_exists($webp_file)) $active_files[$webp_file] = true;
            }
            $thumbnail_file = "$dirname/$base_name-150x150.webp";
            if (file_exists($thumbnail_file)) $active_files[$thumbnail_file] = true;
        }
    }

    if (!$preserve_originals) {
    foreach ($files as $file) {
        if ($file->isDir()) continue;

        $file_path = $file->getPathname();
        $extension = strtolower(pathinfo($file_path, PATHINFO_EXTENSION));
        if (!in_array($extension, ['webp', 'jpg', 'jpeg', 'png'])) continue;

        $relative_path = str_replace($uploads_dir . '/', '', $file_path);
        $path_parts = explode('/', $relative_path);
        $is_valid_path = (count($path_parts) === 1) || (count($path_parts) === 3 && is_numeric($path_parts[0]) && is_numeric($path_parts[1]));

        if (!$is_valid_path || isset($active_files[$file_path])) continue;

        $attempts = 0;
        while ($attempts < 5 && file_exists($file_path)) {
            if (!is_writable($file_path)) {
                @chmod($file_path, 0644);
                if (!is_writable($file_path)) {
                    $log[] = sprintf(__('Error: Cannot make %s writable - skipping deletion', 'wpturbo'), basename($file_path));
                    $failed++;
                    break;
                }
            }
            if (@unlink($file_path)) {
                $log[] = sprintf(__('Cleanup: Deleted %s', 'wpturbo'), basename($file_path));
                $deleted++;
                break;
            }
            $attempts++;
            sleep(1);
        }
        if (file_exists($file_path)) {
            $log[] = sprintf(__('Cleanup: Failed to delete %s', 'wpturbo'), basename($file_path));
            $failed++;
            // Removed error_log to avoid server overload
        }
    }
}

    $log[] = "<span style='font-weight: bold; color: #281E5D;'>" . __('Cleanup Complete', 'wpturbo') . "</span>: " . sprintf(__('Deleted %d files, %d failed', 'wpturbo'), $deleted, $failed);
    update_option('webp_conversion_log', array_slice($log, -100));

    foreach ($attachments as $attachment_id) {
        if (in_array($attachment_id, $excluded_images)) continue;

        $file_path = get_attached_file($attachment_id);
        if (file_exists($file_path) && strtolower(pathinfo($file_path, PATHINFO_EXTENSION)) === 'webp') {
            $metadata = wp_get_attachment_metadata($attachment_id);
            $thumbnail_file = $uploads_dir . '/' . dirname($metadata['file']) . '/' . pathinfo($file_path, PATHINFO_FILENAME) . '-150x150.webp';
            if (!file_exists($thumbnail_file)) {
                $metadata = wp_generate_attachment_metadata($attachment_id, $file_path);
                if (!is_wp_error($metadata)) {
                    wp_update_attachment_metadata($attachment_id, $metadata);
                    $log[] = sprintf(__('Regenerated thumbnail for %s', 'wpturbo'), basename($file_path));
                }
            }
        }
    }

    $log[] = "<span style='font-weight: bold; color: #281E5D;'>" . __('Thumbnail Regeneration Complete', 'wpturbo') . "</span>";
    update_option('webp_conversion_log', array_slice($log, -100));
    return true;
}

// AJAX handlers for exclusion
add_action('wp_ajax_webp_add_excluded_image', 'wpturbo_add_excluded_image_ajax');
function wpturbo_add_excluded_image_ajax() {
    check_ajax_referer('webp_converter_nonce', 'nonce');
    if (!current_user_can('manage_options') || !isset($_POST['attachment_id'])) {
        wp_send_json_error(__('Permission denied or invalid attachment ID', 'wpturbo'));
    }
    $attachment_id = absint($_POST['attachment_id']);
    if (wpturbo_add_excluded_image($attachment_id)) {
        wp_send_json_success(['message' => __('Image excluded successfully', 'wpturbo')]);
    } else {
        wp_send_json_error(__('Image already excluded or invalid ID', 'wpturbo'));
    }
}

add_action('wp_ajax_webp_remove_excluded_image', 'wpturbo_remove_excluded_image_ajax');
function wpturbo_remove_excluded_image_ajax() {
    check_ajax_referer('webp_converter_nonce', 'nonce');
    if (!current_user_can('manage_options') || !isset($_POST['attachment_id'])) {
        wp_send_json_error(__('Permission denied or invalid attachment ID', 'wpturbo'));
    }
    $attachment_id = absint($_POST['attachment_id']);
    if (wpturbo_remove_excluded_image($attachment_id)) {
        wp_send_json_success(['message' => __('Image removed from exclusion list', 'wpturbo')]);
    } else {
        wp_send_json_error(__('Image not in exclusion list', 'wpturbo'));
    }
}

// Convert post content image URLs to WebP
add_action('wp_ajax_convert_post_images_to_webp', 'wpturbo_convert_post_images_to_webp');
function wpturbo_convert_post_images_to_webp() {
    check_ajax_referer('webp_converter_nonce', 'nonce');
    if (!current_user_can('manage_options')) {
        wp_send_json_error(__('Permission denied', 'wpturbo'));
    }

    $log = get_option('webp_conversion_log', []);
    function add_log_entry($message) {
        global $log;
        $log[] = "[" . date("Y-m-d H:i:s") . "] " . $message;
        update_option('webp_conversion_log', array_slice($log, -100));
    }

    add_log_entry(__('Starting post image conversion to WebP...', 'wpturbo'));
    $post_types = get_post_types(['public' => true], 'names');
    $args = [
        'post_type' => $post_types,
        'posts_per_page' => -1,
        'fields' => 'ids'
    ];
    $posts = get_posts($args);

    if (!$posts) {
        add_log_entry(__('No posts found', 'wpturbo'));
        wp_send_json_success(['message' => __('No posts found', 'wpturbo')]);
    }

    $updated_count = 0;
    $checked_images = 0;

    foreach ($posts as $post_id) {
        $content = get_post_field('post_content', $post_id);
        $original_content = $content;

        $content = preg_replace_callback('/<img[^>]+src=["\']([^"\']+\.(?:jpg|jpeg|png))["\'][^>]*>/i', function ($matches) use (&$checked_images) {
            $original_url = $matches[1];
            $checked_images++;
            $webp_url = preg_replace('/\.(jpg|jpeg|png)$/i', '.webp', $original_url);
            $webp_path = str_replace(site_url(), ABSPATH, $webp_url);
            if (file_exists($webp_path)) {
                add_log_entry(sprintf(__('Replacing: %s → %s', 'wpturbo'), $original_url, $webp_url));
                return str_replace($original_url, $webp_url, $matches[0]);
            }
            return $matches[0];
        }, $content);

        if ($content !== $original_content) {
            wp_update_post(['ID' => $post_id, 'post_content' => $content]);
            $updated_count++;
        }

        $thumbnail_id = get_post_thumbnail_id($post_id);
        if ($thumbnail_id && !in_array($thumbnail_id, wpturbo_get_excluded_images())) {
            $thumbnail_path = get_attached_file($thumbnail_id);
            if ($thumbnail_path && !str_ends_with($thumbnail_path, '.webp')) {
                $webp_path = preg_replace('/\.(jpg|jpeg|png)$/i', '.webp', $thumbnail_path);
                if (file_exists($webp_path)) {
                    update_attached_file($thumbnail_id, $webp_path);
                    wp_update_post(['ID' => $thumbnail_id, 'post_mime_type' => 'image/webp']);
                    $metadata = wp_generate_attachment_metadata($thumbnail_id, $webp_path);
                    wp_update_attachment_metadata($thumbnail_id, $metadata);
                    add_log_entry(sprintf(__('Updated thumbnail: %s → %s', 'wpturbo'), basename($thumbnail_path), basename($webp_path)));
                }
            }
        }
    }

    add_log_entry(sprintf(__('Checked %d images, updated %d posts', 'wpturbo'), $checked_images, $updated_count));
    wp_send_json_success(['message' => sprintf(__('Checked %d images, updated %d posts', 'wpturbo'), $checked_images, $updated_count)]);
}

// Custom srcset to include all WebP sizes
add_filter('wp_calculate_image_srcset', 'wpturbo_custom_srcset', 10, 5);
function wpturbo_custom_srcset($sources, $size_array, $image_src, $image_meta, $attachment_id) {
    if (in_array($attachment_id, wpturbo_get_excluded_images())) {
        return $sources;
    }

    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();
    $upload_dir = wp_upload_dir();
    $base_path = $upload_dir['basedir'] . '/' . dirname($image_meta['file']);
    $base_name = pathinfo($image_meta['file'], PATHINFO_FILENAME);
    $base_url = $upload_dir['baseurl'] . '/' . dirname($image_meta['file']);

    foreach ($max_values as $index => $dimension) {
        if ($index === 0) continue;
        $file = "$base_path/$base_name-$dimension.webp";
        if (file_exists($file)) {
            $size_key = "custom-$dimension";
            $width = ($mode === 'width') ? $dimension : (isset($image_meta['sizes'][$size_key]['width']) ? $image_meta['sizes'][$size_key]['width'] : 0);
            $sources[$width] = [
                'url' => "$base_url/$base_name-$dimension.webp",
                'descriptor' => 'w',
                'value' => $width
            ];
        }
    }

    $thumbnail_file = "$base_path/$base_name-150x150.webp";
    if (file_exists($thumbnail_file)) {
        $sources[150] = [
            'url' => "$base_url/$base_name-150x150.webp",
            'descriptor' => 'w',
            'value' => 150
        ];
    }

    return $sources;
}

// Admin interface
add_action('admin_menu', function() {
    add_media_page(
        __('WebP Converter', 'wpturbo'),
        __('WebP Converter', 'wpturbo'),
        'manage_options',
        'webp-converter',
        'wpturbo_webp_converter_page'
    );
});

function wpturbo_webp_converter_page() {
    wp_enqueue_media();
    wp_enqueue_script('media-upload');
    wp_enqueue_style('media');

    if (isset($_GET['set_max_width'])) wpturbo_set_max_widths();
    if (isset($_GET['set_max_height'])) wpturbo_set_max_heights();
    if (isset($_GET['set_resize_mode'])) wpturbo_set_resize_mode();
    if (isset($_GET['set_quality'])) wpturbo_set_quality();
    if (isset($_GET['set_batch_size'])) wpturbo_set_batch_size();
    if (isset($_GET['set_preserve_originals'])) wpturbo_set_preserve_originals();
    if (isset($_GET['cleanup_leftover_originals'])) wpturbo_cleanup_leftover_originals();
    if (isset($_GET['clear_log'])) wpturbo_clear_log();
    if (isset($_GET['reset_defaults'])) wpturbo_reset_defaults();

    $has_image_library = extension_loaded('imagick') || extension_loaded('gd');
    ?>
    <div class="wrap" style="padding: 0; font-size: 14px;">
        <div style="display: flex; gap: 1%; align-items: flex-start;">
            <!-- Main Controls -->
            <div style="width: 48%; background: #FFFFFF; padding: 20px; border-radius: 10px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);">
                <h1 style="font-size: 20px; font-weight: bold; color: #333; margin: -5px 0 15px 0;"><?php _e('WebP Converter - v2.1', 'wpturbo'); ?></h1>

                <?php if (!$has_image_library): ?>
                    <div class="notice notice-error" style="margin-bottom: 20px;">
                        <p><?php _e('Warning: No image processing libraries (Imagick or GD) available. Conversion requires one of these.', 'wpturbo'); ?></p>
                    </div>
                <?php endif; ?>

                <?php if (current_user_can('manage_options')): ?>
                    <div style="margin-bottom: 20px;">
                        <label for="resize-mode" style="font-weight: bold;"><?php _e('Resize Mode:', 'wpturbo'); ?></label><br>
                        <select id="resize-mode" style="width: 150px; margin-right: 10px; padding: 5px;">
                            <option value="width" <?php echo wpturbo_get_resize_mode() === 'width' ? 'selected' : ''; ?>><?php _e('Width', 'wpturbo'); ?></option>
                            <option value="height" <?php echo wpturbo_get_resize_mode() === 'height' ? 'selected' : ''; ?>><?php _e('Height', 'wpturbo'); ?></option>
                        </select>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <label for="max-width-input" style="font-weight: bold;"><?php _e('Max Widths (up to 4, e.g., 1920,1200,600,300):', 'wpturbo'); ?></label><br>
                        <input type="text" id="max-width-input" value="<?php echo esc_attr(implode(', ', wpturbo_get_max_widths())); ?>" style="width: 200px; margin-right: 10px; padding: 5px;" placeholder="1920,1200,600,300">
                        <button id="set-max-width" class="button"><?php _e('Set Widths', 'wpturbo'); ?></button>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <label for="max-height-input" style="font-weight: bold;"><?php _e('Max Heights (up to 4, e.g., 1080,720,480,360):', 'wpturbo'); ?></label><br>
                        <input type="text" id="max-height-input" value="<?php echo esc_attr(implode(', ', wpturbo_get_max_heights())); ?>" style="width: 200px; margin-right: 10px; padding: 5px;" placeholder="1080,720,480,360">
                        <button id="set-max-height" class="button"><?php _e('Set Heights', 'wpturbo'); ?></button>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <label for="quality-slider" style="font-weight: bold;"><?php _e('Quality (0-100):', 'wpturbo'); ?></label><br>
                        <input type="range" id="quality-slider" value="<?php echo esc_attr(wpturbo_get_quality()); ?>" min="0" max="100" style="width: 200px; margin-right: 10px;">
                        <span id="quality-value"><?php echo wpturbo_get_quality(); ?></span>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <label><input type="checkbox" id="preserve-originals" <?php echo wpturbo_get_preserve_originals() ? 'checked' : ''; ?>> <?php _e('Preserve Original Files', 'wpturbo'); ?></label>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <button id="start-conversion" class="button"><?php _e('Convert/Scale', 'wpturbo'); ?></button>
                        <button id="cleanup-originals" class="button"><?php _e('Cleanup Images', 'wpturbo'); ?></button>
                        <button id="convert-post-images" class="button"><?php _e('Fix URLs', 'wpturbo'); ?></button>
                        <button id="run-all" class="button button-primary"><?php _e('Run All', 'wpturbo'); ?></button>
                        <button id="stop-conversion" class="button" style="display: none;"><?php _e('Stop', 'wpturbo'); ?></button>
                        <button id="reset-defaults" class="button" style="margin-left: 10px;"><?php _e('Reset Defaults', 'wpturbo'); ?></button>
                    </div>
                    <div style="margin-bottom: 20px;">
                        <button id="clear-log" class="button"><?php _e('Clear Log', 'wpturbo'); ?></button>
                    </div>
                    <h3 style="font-size: 16px; margin: 0 0 10px 0;"><?php _e('Log (Last 100 Entries)', 'wpturbo'); ?></h3>
                    <pre id="log" style="background: #f9f9f9; padding: 15px; max-height: 400px; overflow-y: auto; border: 1px solid #ddd; border-radius: 5px;"></pre>
                <?php else: ?>
                    <p><?php _e('You need manage_options permission to use this tool.', 'wpturbo'); ?></p>
                <?php endif; ?>
            </div>

            <!-- Exclude Images -->
            <div style="width: 22%; background: #FFFFFF; padding: 20px; border-radius: 10px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);">
                <h2 style="font-size: 16px; margin: 0 0 15px 0;"><?php _e('Exclude Images', 'wpturbo'); ?></h2>
                <button id="open-media-library" class="button" style="margin-bottom: 20px;"><?php _e('Add from Media Library', 'wpturbo'); ?></button>
                <div id="excluded-images">
                    <h3 style="font-size: 14px; margin: 0 0 10px 0;"><?php _e('Excluded Images', 'wpturbo'); ?></h3>
                    <ul id="excluded-images-list" style="list-style: none; padding: 0; max-height: 600px; overflow-y: auto;"></ul>
                </div>
            </div>

            <!-- Instructions -->
            <div style="width: 28%; background: #FFFFFF; padding: 20px; border-radius: 10px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);">
                <h2 style="font-size: 16px; margin: 0 0 15px 0;"><?php _e('How It Works', 'wpturbo'); ?></h2>
                <p style="line-height: 1.5;">
                    <?php _e('Convert images to WebP with responsive sizes (e.g: 1920, 1200, 600, 300) and a 150x150 thumbnail to save on File storage. The original file will be deleted unless "Preserve" is selected. The log will inform you when conversions are complete. ', 'wpturbo'); ?><br><br><br>
					<b>Apply for New Uploads:</b> <?php _e('', 'wpturbo'); ?><br>
                    <b>1. Resize Mode:</b> <?php _e('Select scaling is set on Width or Height.', 'wpturbo'); ?><br>
					<b>2. Set Sizes:</b> <?php _e('Add preferred sizes (up to 4). No need to add 150.', 'wpturbo'); ?><br>
					<b>3. Set Quality:</b> <?php _e('Slide for conversion levels. (Default is 80).', 'wpturbo'); ?><br>
					<b>4. Keep Original:</b> <?php _e('Original image database files are not deleted.', 'wpturbo'); ?><br>
					<b>5. Upload:</b> <?php _e('Upload within Media Library or via a widget/element.', 'wpturbo'); ?><br><br>

					<b>Apply for Existing Images:</b> <?php _e('', 'wpturbo'); ?><br>
					<b>1. Repeat:</b> <?php _e('Follow Steps 1 to 4 from New Uploads', 'wpturbo'); ?><br>
					<b>2. Run All:</b> <?php _e('Click to Execute the Full Sequence.', 'wpturbo'); ?><br><br>

					<b>Apply Manually for Existing Images:</b> <?php _e('', 'wpturbo'); ?><br>
					<b>1. Repeat:</b> <?php _e('Follow Steps 1 to 4 from New Uploads', 'wpturbo'); ?><br>
                    <b>2. Convert/Scale:</b> <?php _e('Process existing images to WebP.', 'wpturbo'); ?><br>
                    <b>3. Cleanup Images:</b> <?php _e('Remove non-WebP files (unless preserved).', 'wpturbo'); ?><br>
                    <b>4. Fix URLs:</b> <?php _e('Update Media URLs to use WebP.', 'wpturbo'); ?><br><br>

					<b>NOTE:</b> <?php _e('', 'wpturbo'); ?><br>
					<b>a) Apply No Conversion:</b> <?php _e('Deactivate Snippet before uploading images.', 'wpturbo'); ?><br>
					<b>b) Backups:</b> <?php _e('Use a robust backup system before running any code/tool.', 'wpturbo'); ?><br>
					<b>c) Processing Speed:</b> <?php _e('This depends on server, number of images.'); ?><br>
					<b>d) Log Delays:</b> <?php _e('Updates appear in batches of 50.'); ?><br>
					<b>e) Option to Stop:</b> <?php _e('Click Stop to stop the process.'); ?>
                </p>
            </div>
        </div>
    </div>

    <style>
        :root {
            --primary-color: #FF0050;
            --hover-color: #444444;
        }
        #quality-slider {
            -webkit-appearance: none;
            height: 6px;
            border-radius: 3px;
            background: #ddd;
        }
        #quality-slider::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 16px;
            height: 16px;
            background: var(--primary-color);
            border-radius: 50%;
            cursor: pointer;
        }
        .button:not(.button-primary) {
            background: #f7f7f7;
            border: 1px solid #ccc;
            transition: all 0.2s;
        }
        .button:not(.button-primary):hover {
            background: #e1e1e1;
            border-color: var(--primary-color);
        }
        .button.button-primary {
            background: var(--primary-color);
            border-color: var(--primary-color);
            color: #fff;
        }
        .button.button-primary:hover {
            background: var(--hover-color);
            border-color: var(--hover-color);
        }
        #excluded-images-list li {
            display: flex;
            align-items: center;
            margin-bottom: 10px;
        }
        #excluded-images-list img {
            max-width: 50px;
            margin-right: 10px;
        }
    </style>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            let isConverting = false;

            function updateStatus() {
                fetch('<?php echo admin_url('admin-ajax.php?action=webp_status&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>')
                    .then(response => response.json())
                    .then(data => {
                        document.getElementById('log').innerHTML = data.log.reverse().join('<br>');
                        document.getElementById('quality-slider').value = data.quality;
                        document.getElementById('quality-value').textContent = data.quality;
                        document.getElementById('resize-mode').value = data.resize_mode;
                        document.getElementById('max-width-input').value = data.max_widths; // Updated
                        document.getElementById('max-height-input').value = data.max_heights; // Updated
                        document.getElementById('preserve-originals').checked = data.preserve_originals;
                        updateExcludedImages(data.excluded_images);
                        updateSliderBackground(data.quality);
                    });
            }

            function updateSliderBackground(value) {
                const slider = document.getElementById('quality-slider');
                const percentage = (value - slider.min) / (slider.max - slider.min) * 100;
                slider.style.background = `linear-gradient(to right, var(--primary-color) ${percentage}%, #ddd ${percentage}%)`;
            }

            function updateExcludedImages(excludedImages) {
                const list = document.getElementById('excluded-images-list');
                list.innerHTML = '';
                excludedImages.forEach(image => {
                    const li = document.createElement('li');
                    li.innerHTML = `<img decoding="async" src="${image.thumbnail}" alt="${image.title}"><span>${image.title} (ID: ${image.id})</span><button class="remove-excluded button" data-id="${image.id}"><?php echo esc_html__('Remove', 'wpturbo'); ?></button>`;
                    list.appendChild(li);
                });
                document.querySelectorAll('.remove-excluded').forEach(button => {
                    button.addEventListener('click', () => {
                        fetch('<?php echo admin_url('admin-ajax.php?action=webp_remove_excluded_image&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>', {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                            body: 'attachment_id=' + encodeURIComponent(button.getAttribute('data-id'))
                        }).then(response => response.json()).then(data => {
                            if (data.success) updateStatus();
                            else alert('Error: ' + data.data);
                        });
                    });
                });
            }

            function convertNextImage(offset) {
                if (!isConverting) return;
                fetch('<?php echo admin_url('admin-ajax.php?action=webp_convert_single&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'offset=' + encodeURIComponent(offset)
                }).then(response => response.json()).then(data => {
                    if (data.success) {
                        updateStatus();
                        if (!data.data.complete && isConverting) convertNextImage(data.data.offset);
                        else document.getElementById('stop-conversion').style.display = 'none';
                    }
                });
            }

            <?php if (current_user_can('manage_options')): ?>
            const mediaFrame = wp.media({
                title: '<?php echo esc_js(__('Select Images to Exclude', 'wpturbo')); ?>',
                button: { text: '<?php echo esc_js(__('Add to Excluded List', 'wpturbo')); ?>' },
                multiple: true,
                library: { type: 'image' }
            });
            document.getElementById('open-media-library').addEventListener('click', () => mediaFrame.open());
            mediaFrame.on('select', () => {
                const selection = mediaFrame.state().get('selection');
                selection.each(attachment => {
                    fetch('<?php echo admin_url('admin-ajax.php?action=webp_add_excluded_image&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>', {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                        body: 'attachment_id=' + encodeURIComponent(attachment.id)
                    }).then(response => response.json()).then(data => {
                        if (data.success) updateStatus();
                    });
                });
            });

            document.getElementById('set-max-width').addEventListener('click', () => {
                const maxWidths = document.getElementById('max-width-input').value;
                fetch('<?php echo admin_url('admin.php?page=webp-converter&set_max_width=1&max_width='); ?>' + encodeURIComponent(maxWidths))
                    .then(() => updateStatus());
            });

            document.getElementById('set-max-height').addEventListener('click', () => {
                const maxHeights = document.getElementById('max-height-input').value;
                fetch('<?php echo admin_url('admin.php?page=webp-converter&set_max_height=1&max_height='); ?>' + encodeURIComponent(maxHeights))
                    .then(() => updateStatus());
            });

            document.getElementById('resize-mode').addEventListener('change', () => {
                const mode = document.getElementById('resize-mode').value;
                fetch('<?php echo admin_url('admin.php?page=webp-converter&set_resize_mode=1&resize_mode='); ?>' + encodeURIComponent(mode))
                    .then(() => updateStatus());
            });

            document.getElementById('quality-slider').addEventListener('input', () => {
                const quality = document.getElementById('quality-slider').value;
                document.getElementById('quality-value').textContent = quality;
                updateSliderBackground(quality);
            });
            document.getElementById('quality-slider').addEventListener('change', () => {
                const quality = document.getElementById('quality-slider').value;
                fetch('<?php echo admin_url('admin.php?page=webp-converter&set_quality=1&quality='); ?>' + encodeURIComponent(quality))
                    .then(() => updateStatus());
            });

            document.getElementById('preserve-originals').addEventListener('change', () => {
                const preserve = document.getElementById('preserve-originals').checked;
                fetch('<?php echo admin_url('admin.php?page=webp-converter&set_preserve_originals=1&preserve_originals='); ?>' + encodeURIComponent(preserve ? 1 : 0))
                    .then(() => updateStatus());
            });

            document.getElementById('start-conversion').addEventListener('click', () => {
                isConverting = true;
                document.getElementById('stop-conversion').style.display = 'inline-block';
                fetch('<?php echo admin_url('admin.php?page=webp-converter&convert_existing_images_to_webp=1'); ?>')
                    .then(() => { updateStatus(); convertNextImage(0); });
            });

            document.getElementById('cleanup-originals').addEventListener('click', () => {
                fetch('<?php echo admin_url('admin.php?page=webp-converter&cleanup_leftover_originals=1'); ?>')
                    .then(() => updateStatus());
            });

            document.getElementById('convert-post-images').addEventListener('click', () => {
                if (confirm('<?php echo esc_js(__('Update all post images to WebP?', 'wpturbo')); ?>')) {
                    fetch('<?php echo admin_url('admin-ajax.php?action=convert_post_images_to_webp&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>', {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
                    }).then(response => response.json()).then(data => {
                        alert(data.success ? data.data.message : 'Error: ' + data.data);
                        updateStatus();
                    });
                }
            });

            document.getElementById('run-all').addEventListener('click', () => {
                if (confirm('<?php echo esc_js(__('Run all steps?', 'wpturbo')); ?>')) {
                    isConverting = true;
                    document.getElementById('stop-conversion').style.display = 'inline-block';
                    fetch('<?php echo admin_url('admin.php?page=webp-converter&convert_existing_images_to_webp=1'); ?>')
                        .then(() => {
                            convertNextImage(0);
                            return new Promise(resolve => {
                                const checkComplete = setInterval(() => {
                                    fetch('<?php echo admin_url('admin-ajax.php?action=webp_status&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>')
                                        .then(response => response.json())
                                        .then(data => {
                                            if (data.complete) {
                                                clearInterval(checkComplete);
                                                resolve();
                                            }
                                        });
                                }, 1000);
                            });
                        })
                        .then(() => fetch('<?php echo admin_url('admin-ajax.php?action=convert_post_images_to_webp&nonce=' . wp_create_nonce('webp_converter_nonce')); ?>', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }))
                        .then(() => fetch('<?php echo admin_url('admin.php?page=webp-converter&cleanup_leftover_originals=1'); ?>'))
                        .then(() => {
                            isConverting = false;
                            document.getElementById('stop-conversion').style.display = 'none';
                            updateStatus();
                            alert('<?php echo esc_js(__('All steps completed!', 'wpturbo')); ?>');
                        });
                }
            });

            document.getElementById('stop-conversion').addEventListener('click', () => {
                isConverting = false;
                document.getElementById('stop-conversion').style.display = 'none';
            });

            document.getElementById('clear-log').addEventListener('click', () => {
                fetch('<?php echo admin_url('admin.php?page=webp-converter&clear_log=1'); ?>')
                    .then(() => updateStatus());
            });

            document.getElementById('reset-defaults').addEventListener('click', () => {
                if (confirm('<?php echo esc_js(__('Reset all settings to defaults?', 'wpturbo')); ?>')) {
                    fetch('<?php echo admin_url('admin.php?page=webp-converter&reset_defaults=1'); ?>')
                        .then(() => updateStatus());
                }
            });
            <?php endif; ?>

            updateStatus();
        });
    </script>
    <?php
}

// Setup AJAX hooks
add_action('admin_init', function() {
    add_action('wp_ajax_webp_status', 'wpturbo_webp_conversion_status');
    add_action('wp_ajax_webp_convert_single', 'wpturbo_convert_single_image');
    if (isset($_GET['convert_existing_images_to_webp']) && current_user_can('manage_options')) {
        delete_option('webp_conversion_complete');
    }
});

// Admin notices
add_action('admin_notices', function() {
    if (isset($_GET['convert_existing_images_to_webp'])) {
        echo '<div class="notice notice-success"><p>' . __('WebP conversion started. Monitor progress in Media > WebP Converter.', 'wpturbo') . '</p></div>';
    }
    if (isset($_GET['set_max_width']) && wpturbo_set_max_widths()) {
        echo '<div class="notice notice-success"><p>' . __('Max widths updated.', 'wpturbo') . '</p></div>';
    }
    if (isset($_GET['set_max_height']) && wpturbo_set_max_heights()) {
        echo '<div class="notice notice-success"><p>' . __('Max heights updated.', 'wpturbo') . '</p></div>';
    }
    if (isset($_GET['reset_defaults']) && wpturbo_reset_defaults()) {
        echo '<div class="notice notice-success"><p>' . __('Settings reset to defaults.', 'wpturbo') . '</p></div>';
    }
});

// Custom image size names
add_filter('image_size_names_choose', 'wpturbo_disable_default_sizes', 999);
function wpturbo_disable_default_sizes($sizes) {
    $mode = wpturbo_get_resize_mode();
    $max_values = ($mode === 'width') ? wpturbo_get_max_widths() : wpturbo_get_max_heights();
    $custom_sizes = ['thumbnail' => __('Thumbnail (150x150)', 'wpturbo')];
    $additional_values = array_slice($max_values, 1, 3);
    foreach ($additional_values as $value) {
        $custom_sizes["custom-$value"] = ($mode === 'width') ? sprintf(__('Custom %dpx Width', 'wpturbo'), $value) : sprintf(__('Custom %dpx Height', 'wpturbo'), $value);
    }
    return $custom_sizes;
}

// Disable scaling
add_filter('big_image_size_threshold', '__return_false', 999);

// Clean up attachment files on deletion
add_action('wp_delete_attachment', 'wpturbo_delete_attachment_files', 10, 1);
function wpturbo_delete_attachment_files($attachment_id) {
    if (in_array($attachment_id, wpturbo_get_excluded_images())) return;

    $file = get_attached_file($attachment_id);
    if ($file && file_exists($file)) @unlink($file);

    $metadata = wp_get_attachment_metadata($attachment_id);
    if ($metadata && isset($metadata['sizes'])) {
        $upload_dir = wp_upload_dir()['basedir'];
        foreach ($metadata['sizes'] as $size) {
            $size_file = $upload_dir . '/' . dirname($metadata['file']) . '/' . $size['file'];
            if (file_exists($size_file)) @unlink($size_file);
        }
    }
}
