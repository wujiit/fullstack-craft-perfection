# WordPress Engineering & Security Craft: Deep-Dive Guardrails

> [!CAUTION]
> **STRICT DOMAIN BOUNDARY — WORDPRESS RUNTIME ONLY**
> This reference document applies **EXCLUSIVELY** to verified WordPress environments (plugins, classic/block themes, Gutenberg blocks, and WP core integrations).
> 
> **MANDATORY DUAL-VERIFICATION GATE & ACTIVE CLARIFICATION GUARD**:
> 1. In existing repositories, this document activates IF AND ONLY IF physical WordPress codebase fingerprints (`ABSPATH`, `WPINC`, `Plugin Name:`, `wp_head()`, `theme.json`) are genuinely verified.
> 2. In clean-slate blank projects, it activates only when an explicit target directive is issued AND negative/comparative context ("reference WP", "like WP", "custom stack", "migrate from WP") is absent.
> 3. A conversational mention of "WordPress" in a generic project MUST NEVER activate this standard.
> 4. **Mandatory Active Clarification on Ambiguity**: When the user's intent, tech stack, or target runtime is ambiguous, guessing is strictly prohibited. The agent MUST actively ask the user whether they are developing for WordPress before applying any WP patterns; until explicitly confirmed, this document remains strictly dormant.
> 
> If the current project does **NOT** meet the WordPress Activation Gate criteria defined in `SKILL.md`, this document is **STRICTLY DORMANT**. Under no circumstances should these WordPress-specific patterns (such as `esc_html`, `$wpdb`, Nonce pairs, or procedural action hooks) be applied to generic web applications, modern frontend frameworks (React, Vue), or standard backend frameworks (Laravel, Symfony, FastAPI).

---

## 1. Security Baseline: Context-Aware Late Escaping & Dual-Layer Access Control

WordPress operates under a specialized security architecture distinct from modern ORM-driven or auto-escaping template frameworks.

### Context-Aware Late Escaping (Mandatory Rule)
Escaping must occur **at the exact moment of output into the DOM**, using the function that strictly matches the rendering context. Never escape early and store escaped strings in databases.

| Output Context | Mandatory Escaping Function | Correct Example |
| :--- | :--- | :--- |
| HTML Body Text | `esc_html()` | `<h1><?php echo esc_html( $title ); ?></h1>` |
| HTML Element Attribute | `esc_attr()` | `<input type="text" value="<?php echo esc_attr( $user_name ); ?>">` |
| URL / Href / Src | `esc_url()` | `<a href="<?php echo esc_url( $redirect_url ); ?>">Link</a>` |
| Textarea Value | `esc_textarea()` | `<textarea><?php echo esc_textarea( $description ); ?></textarea>` |
| Inline JavaScript String | `esc_js()` / `wp_json_encode()` | `let config = <?php echo wp_json_encode( $settings ); ?>;` |
| Rich / User-Provided HTML | `wp_kses_post()` or `wp_kses()` | `<div><?php echo wp_kses_post( $trusted_content ); ?></div>` |

### Input Sanitization: The `wp_unslash()` Precondition
WordPress automatically slashes `$_GET`, `$_POST`, and `$_COOKIE` globally for legacy compatibility.
* **Rule**: Always strip runtime slashes with `wp_unslash()` before sanitizing input fields:
```php
// Correct Input Sanitization
$raw_title = isset( $_POST['item_title'] ) ? wp_unslash( $_POST['item_title'] ) : '';
$clean_title = sanitize_text_field( $raw_title );
```

### Dual-Layer Guard: Nonces vs Capabilities
* **Nonces protect against CSRF ONLY**. A valid Nonce proves that the request came from this site's session; it does **NOT** prove that the user has permission.
* **Capability checks enforce Authorization**. Every state-mutating operation, AJAX handler, or settings form must verify **BOTH**:
```php
// Mandatory Dual-Layer Verification
if ( ! check_admin_referer( 'my_plugin_action_nonce', '_wpnonce' ) ) {
    wp_die( esc_html__( 'Security check failed.', 'my-plugin' ), 403 );
}
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'Unauthorized access.', 'my-plugin' ), 403 );
}
```

---

## 2. Database Persistence via `$wpdb`

Direct raw SQL concatenation or assumptions of default table names cause instant vulnerabilities and breaks multi-site installs.

### Parameterization & Identifiers
* Always access tables via `$wpdb->prefix . 'my_table'`. Never hardcode `wp_`.
* Parameterize all dynamic variables using `$wpdb->prepare()`:
  - `%s` for strings;
  - `%d` for integers;
  - `%f` for floats;
  - `%i` for table/column identifiers (WordPress 6.2+; if targeting older versions, whitelist identifiers against strict arrays).

```php
global $wpdb;
$table_name = $wpdb->prefix . 'custom_records';

// Correct parameterization
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT id, record_title FROM {$table_name} WHERE status = %s AND user_id = %d LIMIT %d",
        $status,
        $user_id,
        $limit
    )
);
```

### Autoloaded Options Size Ceiling
* Store configuration in the Options API (`get_option()`, `update_option()`).
* For large JSON blobs, transients, or infrequently read data, **explicitly pass `autoload = 'no'`**:
  ```php
  update_option( 'my_plugin_large_cache', $data, 'no' );
  ```
  *Failure to do so pollutes `wp_load_alloptions()`, bloating every single page request with hundreds of kilobytes of unused memory.*

---

## 3. Plugin Architecture, Lifecycle & Clean Uninstallation

### Top-Level Lifecycle Hook Registration
Activation and deactivation hooks must be registered in the **main plugin file at root scope**. Never nest them inside `init`, `plugins_loaded`, or other hooks:
```php
// Root scope of main plugin file
register_activation_hook( __FILE__, 'my_plugin_activate' );
register_deactivation_hook( __FILE__, 'my_plugin_deactivate' );

function my_plugin_activate() {
    // 1. Register post types / rewrite rules first
    my_plugin_register_post_types();
    // 2. Then flush rewrite rules cleanly
    flush_rewrite_rules( false );
}
```

### Destruction Quarantine: `uninstall.php`
When a user deletes a plugin from the admin interface, WordPress executes `uninstall.php`.
* **Guardrail**: Never perform destructive table drops or option deletions in deactivation hooks (deactivation may be temporary during updates).
* Place cleanup exclusively in `uninstall.php` guarded by `WP_UNINSTALL_PLUGIN`:
```php
// uninstall.php
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

// Clean up custom options and transients
delete_option( 'my_plugin_settings' );
delete_site_option( 'my_plugin_network_settings' );

// Clean custom database tables if requested by user setting
global $wpdb;
$wpdb->query( "DROP TABLE IF EXISTS " . $wpdb->prefix . "custom_records" );
```

### Admin Scope Boundaries
Enclose admin-only menus, settings pages, and metabox registrations with `if ( is_admin() )` or attach them exclusively to admin hooks (`admin_menu`, `admin_init`, `admin_enqueue_scripts`) to avoid adding execution weight to public frontend requests.

---

## 4. REST API & AJAX Integration Standards

### REST API Route Registration
* Register all REST endpoints on the `rest_api_init` hook.
* **Mandatory `permission_callback`**: Never omit `permission_callback`. For deliberately public endpoints, declare `__return_true`:
```php
add_action( 'rest_api_init', function () {
    register_rest_route( 'my-plugin/v1', '/records', array(
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'my_plugin_get_records',
        'permission_callback' => function () {
            return current_user_can( 'read' );
        },
        'args'                => array(
            'limit' => array(
                'type'              => 'integer',
                'default'           => 10,
                'sanitize_callback' => 'absint',
                'validate_callback' => function( $param ) {
                    return is_numeric( $param ) && $param > 0;
                },
            ),
        ),
    ) );
} );
```

* **Standard Responses**:
  - Success: return `rest_ensure_response( $data )` or instantiate `new WP_REST_Response( $data, 200 )`;
  - Failure: return `new WP_Error( 'code_name', 'Human readable error', array( 'status' => 400 ) )`.
  *(Do not wrap WP REST responses into external alien envelopes like `{code: 200, msg: "success"}` which break standard WordPress REST client adapters).*

### AJAX Handlers
* Enforce Nonce check (`check_ajax_referer()`) and capability check on `wp_ajax_*`.
* Treat `wp_ajax_nopriv_*` as completely untrusted, public, unauthenticated traffic.
* Always conclude AJAX execution with `wp_send_json_success( $payload )` or `wp_send_json_error( $error_message, $status_code )`.

---

## 5. Themes & Gutenberg Modern Block Development

### Classic Themes: The Holy Trinity
Any functional WordPress classic theme must implement:
1. `wp_head()` immediately before `</head>` in `header.php`;
2. `wp_footer()` immediately before `</body>` in `footer.php`;
3. Enqueue frontend scripts and styles exclusively on the `wp_enqueue_scripts` action hook via `wp_enqueue_style()` and `wp_enqueue_script()`.

### Block Themes & `block.json` (apiVersion 3)
* Blocks must be registered via `block.json` using `register_block_type_from_metadata( __DIR__ )`.
* **API Version 3**: In WordPress 6.9+ and 7.0+, declare `"apiVersion": 3`. This guarantees proper styling isolation within the iframed block editor.
* **Interactivity API**: Interactive frontend components should leverage the WordPress Interactivity API (`data-wp-interactive`, `data-wp-bind`, `data-wp-on`) rather than injecting arbitrary unmanaged vanilla script tags.

---

## 6. WordPress.org Directory Compliance & Global Prefixing

When writing code intended for distribution or production themes/plugins:
1. **Universal Unique Prefixing**: Every single global entity must be prefixed with a unique identifier derived from the plugin/theme slug:
   - Functions: `myplugin_get_data()`
   - Classes / Interfaces: `MyPlugin_Data_Manager` or namespace `MyPlugin\...`
   - Constants: `MYPLUGIN_VERSION`
   - Hooks / Filters: `do_action( 'myplugin_after_save', ... )`
   - Options & Transients: `myplugin_api_key`
   - Script / Style Handles: `myplugin-admin-css`
2. **Strict GPLv2+ Licensing**: All bundled third-party libraries, CSS, icons, and PHP files must be compatible with the GNU General Public License v2 or later.
3. **Zero Remote Scripts / No Obfuscation**: Never execute remote JS/PHP code via CDN script tags that execute unchecked; never use `eval()`, `base64_decode()`, or binary obfuscators.
