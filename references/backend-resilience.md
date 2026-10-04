# 后端系统弹性、参数防御与并发幂等架构指南 (Backend Resilience & System Architecture)

本指南面向资深后端/系统工程师，定义了生产级接口服务、并发写操作、外部通信以及长期运行进程的最高工程标准。

---

## 1. 强类型 DTO 入参白名单防御 (Mass Assignment Defense)

### 核心原则
客户端传入的 Payload 严禁直接穿透至持久层（ORM/SQL）或核心业务逻辑。所有请求必须经过严格的 **DTO（Data Transfer Object）或白名单 Schema** 进行类型铸模与清洗。

### 架构模式 (伪代码示例)
```php
// 1. 定义强类型白名单 DTO
class UpdateUserProfileDTO {
    public int $userId;
    public string $nickname;
    public ?string $bio;

    public static function fromRequest(array $input): self {
        $dto = new self();
        $dto->userId = (int)($input['user_id'] ?? 0);
        $dto->nickname = trim(strip_tags((string)($input['nickname'] ?? '')));
        $dto->bio = isset($input['bio']) ? trim((string)$input['bio']) : null;
        
        // 校验器：过滤恶意注入或超出范围的属性（如 is_admin, role 等被强行阻断）
        if ($dto->userId <= 0 || mb_strlen($dto->nickname) === 0) {
            throw new InvalidArgumentException("非法参数");
        }
        return $dto;
    }
}
```

---

## 2. 服务端绝对幂等保障 (Write Idempotency)

### 核心原则
涉及资产流转、订单创建、积分变更、关键状态机的写接口，**绝对不能仅靠前端按钮禁点防重**。弱网超时重试、用户脚本连击或多端并发均可能产生并发穿透，必须在服务端实现绝对幂等。

### 落地三级防线
1. **第一级：请求在途锁（In-flight Mutex Lock）**：
   - 提取请求头中的 `Idempotency-Key` 或基于 `user_id + action + resource_id` 计算 Hash。
   - 使用 Redis `SET resource_key token NX EX 10` 获取短期运行锁。若获取失败直接返回 `409 Conflict: 请求处理中，请勿重复提交`。
2. **第二级：数据库唯一索引（Unique Constraint）**：
   - 数据表必须具备业务幂等列并建立 `UNIQUE KEY`（如 `order_no` 或 `(user_id, biz_token)`）。
   - 依赖底层数据库引擎的原子唯一性约束，防止任何并发脏数据插入。
3. **第三级：业务状态机单向流转防重入**：
   - 状态更新必须具备严格的前置状态校验：
     ```sql
     UPDATE orders SET status = 'PAID' WHERE id = :id AND status = 'PENDING';
     ```
   - 影响行数为 0 时判定为重复触发或非法状态流转，直接返回前序结果或幂等成功响应。

---

## 3. 全链路 Trace-ID 贯穿与审计追踪 (End-to-End Tracing)

### 核心原则
线上生产排障必须做到“见微知著”。从请求进入系统的第一时刻起，必须分配或透传全局唯一的追踪标识符。

### 实现规范
1. **Header 透传**：网关或入口脚本检查 HTTP 头 `X-Trace-Id`，若不存在则生成 `trace-{UUID}-{timestamp}`，并在响应头中原样回传。
2. **结构化日志绑定**：所有错误日志、审计日志、数据库慢查询日志必须统一注入 `trace_id` 字段：
   ```json
   {
     "trace_id": "trace-8f9a2b-1727960000",
     "level": "ERROR",
     "timestamp": "2026-10-03T21:50:00Z",
     "message": "第三方支付网关超时",
     "context": { "order_id": 10023, "duration_ms": 3002 }
   }
   ```
3. **关键操作审计（Audit Trail）**：用户权限变更、资金结算、敏感数据导出必须向审计表记录不可篡改的操作记录（包含操作人、IP、变更前后 Diff、时间与 Trace-ID）。

---

## 4. 弹性避退重试算法 (Exponential Backoff with Jitter)

### 核心原则
调用不可靠的外部第三方接口或分布式服务时，严禁固定间隔连续盲目重试，必须采用**带随机抖动的指数退避重试**，杜绝重试风暴引发下游服务雪崩。

### 算法实现 (毫秒级)
$$\text{Delay} = \min\left(\text{MaxDelay}, \text{BaseDelay} \times 2^{\text{Attempt}}\right) + \text{RandomJitter}(0, \text{BaseDelay})$$

```javascript
async function fetchWithRetry(url, options = {}, maxAttempts = 3) {
  let baseDelay = 300; // 300ms 基准
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fetchWithTimeout(url, options);
    } catch (err) {
      if (attempt === maxAttempts) throw err;
      // 指数退避 + 随机抖动
      const jitter = Math.random() * baseDelay;
      const delay = Math.min(3000, baseDelay * Math.pow(2, attempt - 1)) + jitter;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## 5. 常驻进程优雅停机 (Graceful Shutdown)

### 核心原则
消费队列 Worker、WebSocket 服务或长时批处理脚本在发布更新或容器缩容重启时，严禁直接 `kill -9` 强杀，必须监听操作系统退出信号。

### 处理规范
1. 监听信号：捕获 `SIGTERM`（容器编排停止信号）与 `SIGINT`（Ctrl+C 中断信号）；
2. 状态切换：将服务状态置为 `SHUTTING_DOWN`，立即拒绝接入新的入站请求/不再拉取新消息；
3. 排空与释放：允许当前正在处理的在途任务事务完成（最多等待优雅超时阈值，如 15 秒）；
4. 资源释放：在 `finally` 块显式关闭数据库连接池、Redis 句柄与文件锁，安全调用 `exit(0)` 退出。

---

## 6. 行级数据归属鉴权与防水平越权 (IDOR / BOLA Defense)

### 核心原则
Insecure Direct Object References (IDOR) 是业务系统最常被利用的高危漏洞。对于所有非超管用户的私有资源读取、编辑、删除与状态流转，**严禁仅凭客户端传入的资源主键 ID 作为唯一判定条件**，必须强制在持久层绑定当前已鉴权用户的身份上下文。

### 架构模式 (持久层所有者绑定)
```php
// ❌ 存在致命水平越权风险：任意用户修改 URL 参数即可取消他人订单或删除他人文件
$stmt = $pdo->prepare("UPDATE orders SET status = 'CANCELLED' WHERE id = :id");
$stmt->execute([':id' => $input['order_id']]);

// ✅ 强制行级所有者绑定：通过当前 Session/Token 中的鉴权 user_id 进行物理隔离
$stmt = $pdo->prepare("UPDATE orders SET status = 'CANCELLED' WHERE id = :id AND user_id = :userId");
$stmt->execute([
    ':id'     => (int)$input['order_id'],
    ':userId' => (int)$currentUser->id
]);
if ($stmt->rowCount() === 0) {
    // 资源不存在或无权操作（对外统一返回 404 或友好无权提示，防资源 ID 探测）
    throw new ResourceNotFoundOrForbiddenException("资源不存在或无权操作");
}
```

---

## 7. 核心业务软删除与唯一性设计 (Soft Delete Architecture)

### 核心原则
核心资产数据（订单、用户、资产流水、配置项、审计痕迹）在生产环境**严禁执行物理 `DELETE`**。必须通过时间戳软删除标记（`deleted_at DATETIME/TIMESTAMP NULL`）保留完整历史，用于业务防误删恢复、历史追溯与合规审计。

### 落地规范与唯一索引设计
1. **统一时间戳定义**：软删除列命名统一为 `deleted_at`，默认值为 `NULL`（代表活跃/未删除），被删除时记录当前时间戳 `NOW()`。
2. **全局查询过滤**：
   ```sql
   SELECT id, title, updated_at FROM posts WHERE user_id = :userId AND deleted_at IS NULL;
   ```
3. **软删除与业务唯一索引冲突解决**：
   - 当业务要求某字段全局唯一（如 `account` 用户名），但已软删除的数据可能释放该值允许重用时：
   - **方案 A (联合唯一索引，MySQL 兼容)**：建立 `UNIQUE KEY uk_account_del (account, deleted_at)`。因为在 MySQL/InnoDB 中，多个 `NULL` 值在唯一索引中互不冲突，而删除后 `deleted_at` 具有固定时间戳，既保证未删除记录的唯一性，又保留软删除历史。
   - **方案 B (归档表迁移)**：在物理删除前，通过事务原子写入 `xxx_archive` 归档表，再执行主表物理清理。

---

## 8. PII 敏感隐私与日志脱敏规约 (PII Masking & Sanitization)

### 核心原则
个人敏感信息 (PII, Personally Identifiable Information) 包括手机号、身份证、邮箱、银行卡号等，在对外 API 响应、前端页面渲染以及服务端运行日志中**绝对禁止明文裸露**。

### 脱敏规则与实现
```php
class PiiMasker {
    // 手机号：前 3 后 4，中间 4 位星号（138****1234）
    public static function maskMobile(string $mobile): string {
        return preg_replace('/(\d{3})\d{4}(\d{4})/', '$1****$2', $mobile) ?? '';
    }

    // 电子邮箱：保留前缀首字母与域名后缀（z***y@domain.com）
    public static function maskEmail(string $email): string {
        return preg_replace('/(?<=.).(?=.*@)/u', '*', $email) ?? '';
    }

    // 身份证号：保留前 6 后 4，中间掩码
    public static function maskIdCard(string $idCard): string {
        return preg_replace('/(\d{6})\d+(\d{4})/', '$1********$2', $idCard) ?? '';
    }
}
```

### 日志脱敏过滤器
在写入日志文件或推送至日志收集器前，日志 Handler 必须配置白名单或通配脱敏规则，自动对包含 `password`, `token`, `secret`, `access_key`, `card_no` 等键的值替换为 `******`。

---

## 9. 数据库平滑演进与幂等迁移 (Expand-Contract & Idempotent DDL)

### 核心原则
生产系统部署绝非单机停机发布，多实例滚动更新期间新旧代码必定短时间并存。数据库变更脚本（DDL）必须具备**重入幂等性**与**向前向后双向兼容性**。

### 幂等 DDL 书写准则
所有迁移脚本必须支持安全重复执行而无害：
```sql
-- 表创建幂等
CREATE TABLE IF NOT EXISTS sys_configs (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cfg_key VARCHAR(64) NOT NULL,
    cfg_value TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_key (cfg_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 增列幂等 (存储过程或框架迁移机制)
-- 严禁无保护重复 ADD COLUMN 导致执行中断
```

### Expand-Contract 零停机平滑演进四步法
严禁在生产库中直接执行瞬间破坏性 DDL（如 `DROP COLUMN old_col` 或重命名列）：
1. **阶段一 (Expand 扩充)**：执行 DDL 新增目标字段 `new_col`（允许为空），此时旧代码继续读写 `old_col`；
2. **阶段二 (Dual-Write 双写 & 回填)**：发布新版本应用，写操作同时写入 `old_col` 与 `new_col`；后台异步批量脚本回填历史存量数据；
3. **阶段三 (Contract 读切换)**：将所有读取操作安全平移至 `new_col`，停止写入 `old_col`，观察运行平稳；
4. **阶段四 (Cleanup 清理)**：在低峰期下线旧列 `DROP COLUMN old_col`，完成无损平滑蜕变。

---

## 10. 全局架构自洽、彻底重构与依赖验证规范 (Architecture Cohesion, Clean Refactor & API Verification)

### 10.1 拒绝过度工程与代码膨胀 (Anti-Overengineering & KISS)
- **代码经济原则**：优先寻找最自然、扁平、短路径的工程实现。如果 100 行清晰紧凑的代码可以解决问题，绝对禁止引入 500 行的多层工厂、包装类与虚假策略模式；
- **防虚假扩展性 (Anti-YAGNI)**：严禁脱离当下需求凭空预设未来可能用到的配置项、中间层与抽象接口；每一行代码必须有确定的当下业务归宿。

### 10.2 彻底斩断重构法则 (Clean-Cut Refactoring)
- **重构闭环**：当方案从 A 升级为 B（例如从轮询重构为事件驱动、从同步数组重构为生成器流），必须对方案 A 进行连根拔除：
  1. 删除旧方案的专有 Helper、类与废弃参数；
  2. 删除对旧方案的一切引用与配置调用；
  3. 严禁留下一半 A、一半 B 的互扯后腿代码，更严禁保留大段注释掉的僵尸代码（Zombie Code）。

### 10.3 依赖版本精准对齐、第三方接口实证与零 API 幻觉 (Version Alignment & Real API Grounding)
- **依赖实证**：在编写任何调用前，必须查验 `composer.json` / `package.json` 中的实际包版本及当前运行环境的语言版本；
- **第三方接口端点与字段实证铁律**：
  - 对接第三方 API 时，绝对禁止凭直觉捏造请求端点路径（URL / Method / Protocol）；
  - 请求参数、Header 标头、加密/签名算法及响应数据结构必须 100% 对齐官方最新权威文档或真实报文；
  - 严禁盲目臆测深层返回字段（如 `res.data.list` / `res.result.items`），在缺乏官方文档时必须主动查证或要求提供真实报文样例，坚决杜绝“闭门造车”；
- **杜绝废弃与越级 API**：低版本禁止使用高版本语法糖，高版本严禁调用已废弃/移除的方法；
- **零 API 幻觉**：绝对禁止凭感觉编造不存在的函数、参数签名或第三方包，未完全掌握的接口必须以官方文档或项目既有调用为准。

### 10.4 宏观全局架构观与自洽性 (Holistic Architecture & Cohesion)
- **单点与全局的因果推演**：编写任何局部函数前，先通盘审视整个系统的分层架构、数据流向、状态机契约与异常处理规范；
- **统一公共资产复用**：系统已有现成的公共 Service、Helper 或网络层时，严禁在局部随意自创私有轮子，保持全局统一单一事实来源（Single Source of Truth）。

---

## 11. 变更爆炸半径评估、资产前置查重与极值边界防御 (Blast Radius, Asset Audit & Edge-Case Defense)

### 11.1 变更影响面与爆炸半径控制 (Blast Radius Control)
- **反向依赖排查**：在修改底层公共类、公共中间件、基础数据库字段时，强制先反向检索所有调用点（References / Call Hierarchy）；
- **向后兼容性优先**：底层公共接口必须保持向前向下兼容。新增入参必须设置合理的默认值，严禁突变破坏性修改已有参数签名导致下游多模块连环破裂。

### 11.2 编码前置查重与单一真实来源 (Pre-Coding Deduplication & SSOT)
- **拒绝碎片化重造轮子**：编码前必须在代码库中检索相似命名与关键词。项目内已具备类似实现时，严禁盲目复制或另立门户，必须优先复用或提炼为通用方法；
- **业务版本收敛**：同一种核心业务计算或流程判定（如折扣算法、身份状态判定），坚决杜绝 A/B/C 多套版本并存，强制收敛至单一标准实现。

### 11.3 极值边界推演与根因排查协议 (Edge-Case Simulation & Root-Cause Debugging)
- **五大极值防御清单**：
  1. 空值：针对 `null`、`undefined`、空数组、空字符串配置防御回退；
  2. 极值数字：处理 `0`、负数、边界浮点数（金额必须使用整型分或高精度数学扩展）；
  3. 并发安全：防连击互斥锁（In-flight lock）与数据库行锁/版本号；
  4. 权限与状态机防逆流：拦截非法逆向状态流转与越权访问；
  5. 超时与重试：显式配置超时阈值与熔断兜底。
- **根因直击**：面对跨模块复杂 Bug，严禁靠盲目增加 `try-catch` 或打补丁掩盖报错；必须通过日志和链路追踪找到引发异常的根因数据源头并从根本上解决。
