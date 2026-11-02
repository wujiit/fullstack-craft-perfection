# WordPress 工程架构与安全防护精工规范

> [!CAUTION]
> **严格领域边界与隔离休眠铁律 (STRICT DOMAIN BOUNDARY)**
> 本参考文档**仅且仅在**确诊当前为 WordPress 运行环境时方可激活生效（WP 插件、经典/区块主题、Gutenberg 区块及 WP 核心扩展）。
> 
> **三重强制同时命中门禁与主动澄清铁律 (Triple-Lock Gate & Active Clarification Guard)：**
> 1. 单凭对话中出现 `Wordpress`、`wp`、`wp主题`、`wp插件` 等关键词，**绝对 100% 不得激活本规范**！
> 2. 必须同时满足【① 对话明确目标（排除参考/借鉴/对比/迁移） + ② 头部元数据注释（`Plugin Name:`/`Theme Name:`） + ③ 运行时专有底层代码指纹（`ABSPATH`、`WPINC`、`register_activation_hook`、`wp_head()`、官方 `schemas.wp.org` Schema）】多维证据链同时成立！
> 3. **模糊意图强制主动提问**：若用户意图、技术栈或目标运行环境不明确，严禁盲猜或自行套用，**必须先主动向用户提问确认**是否进行 WordPress 相关开发；未确认前绝对保持离线休眠状态！
> 
> 若当前项目**未命中**上述三重门禁，本文件**绝对处于离线休眠状态**。严禁将文中所述的 WordPress 专有模式（如 `esc_html`、`$wpdb`、Nonce 机制或过程式钩子）逆向套用至常规 Web 前端、现代组件框架（React、Vue）或通用后端框架（Laravel、Symfony、FastAPI、Python、Go）。

---

## 1. 安全底线：上下文匹配的晚期转义与双重越权防护

WordPress 运行在特有的过滤体系下，与现代自带自动转义的模板引擎存在本质差异。

### 上下文匹配的晚期转义 (Late Escaping - 强制规约)
转义必须发生在**数据渲染输出至 DOM 的最后一刻**，且必须使用与当前 HTML 物理上下文完全匹配的专用转义函数，严禁提前转义存库。

| 输出渲染上下文 | 强制使用的专有转义函数 | 规范代码示例 |
| :--- | :--- | :--- |
| HTML 标签内文本 | `esc_html()` | `<h1><?php echo esc_html( $title ); ?></h1>` |
| HTML 属性值 | `esc_attr()` | `<input type="text" value="<?php echo esc_attr( $user_name ); ?>">` |
| URL / Href / Src | `esc_url()` | `<a href="<?php echo esc_url( $redirect_url ); ?>">Link</a>` |
| Textarea 文本域内容 | `esc_textarea()` | `<textarea><?php echo esc_textarea( $description ); ?></textarea>` |
| 内联 JS 字符串 / JSON | `esc_js()` / `wp_json_encode()` | `let config = <?php echo wp_json_encode( $settings ); ?>;` |
| 用户提供的富文本 HTML | `wp_kses_post()` 或 `wp_kses()` | `<div><?php echo wp_kses_post( $trusted_content ); ?></div>` |

### 入参数据清洗与 `wp_unslash()` 前提
WordPress 核心历史原因会对全局 `$_GET`、`$_POST`、`$_COOKIE` 自动执行反斜杠转义。
* **铁律**：所有外部入参在进入业务校验前，必须先调用 `wp_unslash()` 解除反斜杠，再通过对应函数进行数据清洗：
```php
// 正确的入参清洗流程
$raw_title = isset( $_POST['item_title'] ) ? wp_unslash( $_POST['item_title'] ) : '';
$clean_title = sanitize_text_field( $raw_title );
```

### 鉴权双重守卫：Nonce 与 Capability 绝不混为一谈
* **Nonce 仅能防范 CSRF 跨站请求伪造**：一个合法的 Nonce 仅证明该请求来自本站的合法会话，**绝不能证明发起者具备该操作的执行权限**；
* **Capability（权限）才能防范越权**：任何引发状态变更的操作、AJAX 接口或表单，必须**同时完成双重核验**：
```php
// 强制双重守卫核验
if ( ! check_admin_referer( 'my_plugin_action_nonce', '_wpnonce' ) ) {
    wp_die( esc_html__( '安全检查未通过。', 'my-plugin' ), 403 );
}
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( '当前用户无权执行此操作。', 'my-plugin' ), 403 );
}
```

---

## 2. 数据库安全与 `$wpdb` 规范

严禁在 SQL 语句中直接拼接变量，严禁硬编码表名，防止注入漏洞及多站点（Multisite）兼容崩溃。

### 参数化预编译与标识符安全
* 表名必须动态获取 `$wpdb->prefix . 'my_table'`，严禁手写 `wp_`；
* 任何带变量的查询必须通过 `$wpdb->prepare()` 预编译绑定参数：
  - `%s` 绑定字符串；
  - `%d` 绑定整数；
  - `%f` 绑定浮点数；
  - `%i` 绑定动态表名或列名（WordPress 6.2+ 原生支持；低版本需走严格白名单校验）。

```php
global $wpdb;
$table_name = $wpdb->prefix . 'custom_records';

// 规范的参数化查询
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT id, record_title FROM {$table_name} WHERE status = %s AND user_id = %d LIMIT %d",
        $status,
        $user_id,
        $limit
    )
);
```

### Autoload 自动加载选项内存防爆
* 业务配置使用 Options API（`get_option()`、`update_option()`）；
* 对于大体积的 JSON 缓存、长文本或非每次必读的配置，**显式传入 `autoload = 'no'`**：
  ```php
  update_option( 'my_plugin_large_cache', $data, 'no' );
  ```
  *若遗漏此参数，该大对象会被默认塞进 `wp_load_alloptions()`，导致前台每个普通页面访问时瞬间暴增数百 KB 甚至数 MB 无用内存。*

---

## 3. 插件生命周期与卸载隔离

### 激活与停用钩子顶层注册
`register_activation_hook` 与 `register_deactivation_hook` 必须注册在**插件主入口文件的最外层全局作用域**，严禁嵌套在 `init`、`plugins_loaded` 等钩子内：
```php
// 必须在插件主文件顶层注册
register_activation_hook( __FILE__, 'my_plugin_activate' );
register_deactivation_hook( __FILE__, 'my_plugin_deactivate' );

function my_plugin_activate() {
    // 1. 先注册自定义文章类型与重定向路由
    my_plugin_register_post_types();
    // 2. 然后干净刷新重定向规则
    flush_rewrite_rules( false );
}
```

### 彻底清理：`uninstall.php` 隔离
用户在后台彻底“删除”插件时，WordPress 会触发 `uninstall.php`。
* **安全规约**：绝对禁止在停用钩子（Deactivation）中删表或删配置（停用往往只是临时更新操作）；
* 破坏性清理逻辑必须独立写入 `uninstall.php`，并在首行校验常量：
```php
// uninstall.php
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

// 清理自定义配置
delete_option( 'my_plugin_settings' );
delete_site_option( 'my_plugin_network_settings' );

// 若用户配置允许清空数据，删除自定义表
global $wpdb;
$wpdb->query( "DROP TABLE IF EXISTS " . $wpdb->prefix . "custom_records" );
```

### 后台逻辑作用域守卫
后台专属的菜单、设置页与 Metabox，必须使用 `if ( is_admin() )` 守卫包裹，或仅挂载在后台专属钩子（`admin_menu`、`admin_init`）上，杜绝增加前台访客请求负担。

---

## 4. REST API 与 AJAX 架构规范

### REST API 路由注册
* 统一挂载在 `rest_api_init` 钩子下；
* **强制声明 `permission_callback`**：绝不能漏写该参数；对显式公开接口，明确传递 `__return_true`：
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

* **原生响应格式**：
  - 成功：返回 `rest_ensure_response( $data )` 或 `new WP_REST_Response( $data, 200 )`；
  - 失败：返回 `new WP_Error( 'code_name', '可读的错误说明', array( 'status' => 400 ) )`。
  *(严禁强行给 WP REST 接口套用外层 `{code: 200, msg: "success"}` 外壳，防止破坏标准 WP REST 客户端解析)*。

### AJAX 处理规范
* `wp_ajax_*` 必须同时核验 Nonce（`check_ajax_referer()`）与权限；
* `wp_ajax_nopriv_*` 必须视作未登录黑客流量严防死守；
* 统一使用 `wp_send_json_success( $payload )` 或 `wp_send_json_error( $error_message, $status_code )` 终结输出。

---

## 5. 主题与现代 Gutenberg 区块开发

### 经典主题三大件
任何能正常运行的 WordPress 经典主题，必须包含：
1. `header.php` 中 `</head>` 闭合前必须调用 `wp_head()`；
2. `footer.php` 中 `</body>` 闭合前必须调用 `wp_footer()`；
3. 样式表与 JS 脚本必须挂载在 `wp_enqueue_scripts` 钩子上加载。

### 区块主题与 `block.json` (apiVersion 3)
* 区块必须通过 `block.json` 元数据进行注册（`register_block_type_from_metadata( __DIR__ )`）；
* **API Version 3**：在 WordPress 6.9+ / 7.0+ 中强制声明 `"apiVersion": 3`，确保区块在 iframe 编辑器沙箱中样式完美隔离；
* **Interactivity API**：前台交互优先使用 WordPress 官方交互 API（`data-wp-interactive`、`data-wp-bind`、`data-wp-on`）。

---

## 6. WP.org 官方目录合规与唯一命名前缀

所有面向分发或生产级交付的插件与主题，必须遵守：
1. **全局实体唯一前缀 (Prefixing)**：所有全局函数名、类名、常量、选项名、Transient、Hook 及脚本样式 Handle，必须全部带上专属唯一前缀（如 `myplugin_`）；
2. **严格 GPLv2+ 开源兼容**：引入的第三方库与资源必须符合 GPLv2 或更高兼容协议；
3. **零远程脚本 / 零代码混淆**：绝对禁止引入未经审核的外部第三方远程 JS 脚本执行；严禁使用 `eval()`、`base64_decode()` 或二进制混淆。
