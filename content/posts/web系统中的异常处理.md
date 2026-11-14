+++
date = '2026-11-14T23:52:09+08:00'
draft = false
title = 'Web 系统中的异常处理'
+++

异常处理讨论到最后，常常被简化成"抛异常"和"返回错误码"两个阵营的对立。但在真实的 Web 系统里，这种二选一并不成立：同一个失败，出现在不同的调用位置，合理的表达方式可能完全不同。

决定形式的其实是三件事：这个失败是什么性质，调用方拿到它之后打算做什么，以及这段代码所依赖的"一定成立"到底有多强。下面顺着这个思路展开：先给失败分类，再讨论方法内部如何表达失败，然后看那些语法上可能为空、实际上几乎不会为空的地方要不要判空，最后处理数据库里"该有的没有、该唯一的不唯一"这类问题。例子放在一条电商下单链路上：库存、优惠券、钱包余额、订单查询。

## 一、先给失败分类，再决定怎么表达

### 1.1 四类失败

下单链路里会遇到的失败，大致能分成四类。

**技术异常**：数据库连接超时、连接池耗尽、Redis 抖动、下游 HTTP 503、空指针。这类失败和用户的操作意图无关，重试可能成功，也可能说明代码或环境本身有问题。处理方向只有一个：不要吞，向上抛。在层与层的边界上可以包装成更贴近业务语义的非受检异常，但必须把原异常放进 `cause`。

```java
try {
    return orderMapper.selectByOrderNo(orderNo);
} catch (DataAccessException e) {
    throw new OrderQueryException("查询订单失败, orderNo=" + orderNo, e);
}
```

保留 `cause` 不是为了好看。没有它，日志里只剩一句"查询订单失败"，栈是包装点的栈，真正的 SQL 错误、超时时间、驱动返回的错误码全部丢失，排障只能靠猜。

**预期内、且调用方需要按类型分支的业务失败**：登录时的"用户不存在 / 已锁定 / 密码错误 / 未实名"。这四种情况的后续动作完全不同——引导注册、提示联系客服、提示剩余尝试次数、跳转实名页。这类失败应该用结构化的返回值表达，让调用方在自己的代码里显式分支。

**预期内、且一旦发生就必须终止当前操作的业务失败**：库存不足、余额不足、订单状态不允许取消。这类失败没有"继续往下走"的选项，最适合在 Service 边界抛 `BusinessException`，由全局异常处理器统一转成 4xx，同时让事务回滚。

**正常分支，不是失败**：购物车为空、用户没有设置默认收货地址、当前没有可用的优惠券。这些是业务上完全正常的"没有"，返回空集合、`Optional.empty()` 或一个正常对象即可。用异常表达它，调用方被迫写 try-catch，日志里堆满 error，错误率监控也会失真。

### 1.2 分类看的是调用点，不是错误的名字

同一个"库存不足"，在提交订单时是必须终止的失败：订单不能生成，事务要回滚，用户看到"库存不足，下单失败"。但在商品详情页或购物车试算时，它只是一条需要展示的信息："库存仅剩 2 件"，页面继续渲染，用户可能改成买 1 件。前一种场景抛异常更合适，后一种场景返回一个带可用数量的结果对象更合适。

所以分类不是给错误起个名字然后归档，而是看这段代码拿到失败之后打算做什么。判断只需要两个问题：调用方要不要根据失败类型走不同分支？这次失败是否必须让当前操作停下来？

还有一条容易被忽略的纪律：同一类失败不要在层与层之间反复变形。DAO 层把 `SQLException` 包装成 `DataAccessException` 之后，Service 不应该 catch 住再返回 `null`；Controller 也不应该 catch 住所有异常然后返回 HTTP 200 加一个 `{"code": 500}`。异常向上传递时只应该增加上下文，不应该被换成另一种含义完全不同的表达，否则排障时看到的信息和最初的根因之间会隔着好几层翻译。

## 二、方法内部的失败：不要用 -1 表示

钱包扣款是一个很典型的场景。方法签名是 `deduct(Long accountId, BigDecimal amount)`，需要返回扣款后的余额。有人会这样写：

```java
@Transactional
public BigDecimal deduct(Long accountId, BigDecimal amount) {
    Account account = accountMapper.selectById(accountId);
    if (account.getBalance().compareTo(amount) < 0) {
        return BigDecimal.valueOf(-1);   // 约定：-1 表示余额不足
    }
    accountMapper.decreaseBalance(accountId, amount);
    flowMapper.insert(new AccountFlow(accountId, amount, "DEDUCT"));
    return account.getBalance().subtract(amount);
}
```

这段代码有几个明显的问题。返回值本来承载的是"扣款后的余额"，现在被塞进了一个哨兵值；如果业务允许透支，余额本身就可能为负数，`-1` 的歧义会更大。调用方拿到 `-1` 只知道失败，不知道是余额不足、账户冻结还是金额非法，错误信息在传递中被压缩成了一个数字。`if (result.compareTo(BigDecimal.valueOf(-1)) == 0)` 这种判断只要漏掉一处，后面的订单状态机就会拿着 `-1` 当成真实余额继续跑。

更隐蔽的是事务问题。Spring 的 `@Transactional` 默认只在遇到 `RuntimeException` 和 `Error` 时回滚，返回 `-1` 属于正常返回，事务照样提交。如果哪天有人调整了语句顺序，把写流水提到校验之前——这种改动在后续维护里并不少见——返回 `-1` 时事务照样提交，结果就是余额没扣、流水多了一条，要等到对账时才会有人发现。

正确的做法取决于调用方需不需要分支。绝大多数情况下，余额不足意味着这次扣款必须终止，那就直接抛异常：

```java
@Transactional
public void deduct(Long accountId, BigDecimal amount) {
    Account account = accountMapper.selectById(accountId);
    if (account.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException(accountId, amount, account.getBalance());
    }
    accountMapper.decreaseBalance(accountId, amount);
    flowMapper.insert(new AccountFlow(accountId, amount, "DEDUCT"));
}
```

异常里带着账户、请求金额和可用余额，全局处理器可以映射成"余额不足，当前可用 xx 元"的提示和对应的错误码，事务也会回滚。`InsufficientBalanceException` 继承自 `BusinessException`，不需要为它单独写处理器，只有在需要特殊文案时才覆盖。

如果调用方确实要根据失败类型走不同路径——比如余额不够时自动改用"余额 + 第三方支付"的组合支付——那就返回结构化结果，而不是 `-1`：

```java
public sealed interface DeductResult {
    record Success(BigDecimal balanceAfter) implements DeductResult {}
    record InsufficientBalance(BigDecimal required, BigDecimal available) implements DeductResult {}
    record AccountFrozen(Long accountId) implements DeductResult {}
}
```

调用方用 `switch` 消费，枚举和 sealed 类型上的 `switch` 本身就会做分支完整性检查：

```java
DeductResult result = accountService.deduct(accountId, amount);
switch (result) {
    case DeductResult.Success s -> orderService.markPaid(order, s.balanceAfter());
    case DeductResult.InsufficientBalance b -> paymentService.switchToCombinedPay(order, b.required(), b.available());
    case DeductResult.AccountFrozen f -> throw new BusinessException(AccountErrorCode.ACCOUNT_FROZEN, f.accountId());
}
```

需要说明的是，返回结果对象意味着"没扣成"是方法的正常返回路径之一，因此方法内部不能在返回之前已经改过余额。要么把所有校验放在写操作之前，要么在发现问题时显式设置回滚。做不到这一点，还是抛异常更安全。

另一种常见的折中是下层返回错误值、上层判断后再抛异常，同样不推荐。失败原因在下层最清楚，压成 `-1` 或 `false` 之后，上层只能写一句 `throw new BusinessException("扣款失败")`，用户看到的提示和运维需要的上下文同时退化；中间只要有一层忘了判断，这个错误值就会被当作正常结果继续使用。要么下层直接抛，要么返回结构化结果让上层显式决策，不要两种各占一半。

至于"这样会不会类太多"，其实是个组织问题。`DeductResult` 完全可以写成 `AccountService` 的嵌套类型，或者和 Service 放在同一个 `.java` 文件里——Java 并不要求一个类一个文件，只要求一个文件最多一个 `public` 顶层类。只有当这个结果契约会被多个包引用时，才值得把它提成独立的 `public` 文件，并按业务分包。为了少写几个类而退回 `-1`、`null` 或 `Map<String, Object>`，省下的是几行代码，丢掉的是编译器能帮忙检查的那部分契约。

## 三、要不要判空，取决于契约有多强

有一类地方，语法上完全可能拿到 `null`，实际运行中却几乎不会：同一个方法里刚 `put` 进去又取出来的 Map、用枚举名做 key 的 `Map<String, Object>`、启动时已经校验过的配置项、从固定模板渲染出的参数表。面对这类代码，每个 `get` 后面都加一个判空是噪音，完全不判又确实会在某次重构后踩坑。

决策可以拆成两个问题。

**这个"一定存在"的保证有多强？** 强契约的例子：同一方法内先 `put` 后 `get`，中间没有别的逻辑；key 来自枚举常量；配置在应用启动时已经用 `requireNonNull` 逐项校验过。弱契约的例子：Controller 组装好的 `Map` 一路传到 Service 再传给工具类；从上游接口反序列化出来的 Map；依赖"先调 `init()` 再调 `use()`"这种隐式顺序的字段。

**出错时后果有多严重？** 如果缺失会立刻抛 NPE，栈正好指在出错那一行，代价其实可以接受。真正难查的是 `null` 被悄悄带走：拼进 SQL 变成 `status = null`（永远不匹配，静默返回空），被 `String.valueOf` 变成字符串 `"null"` 存进库，被当成 0 参与金额汇总，或者在模板里渲染成空白。这些都不会报错，只会让数据一点点变脏。

举个具体的例子。订单导出服务从 `Map<String, Object> row` 里取字段，其中 `amount` 是必须存在的：

```java
// 有问题：把"数据缺失"变成了"金额为 0"
BigDecimal amount = (BigDecimal) row.getOrDefault("amount", BigDecimal.ZERO);
```

报表少了几笔金额，没人会收到告警，只有财务对账时才发现。换成在源头失败：

```java
BigDecimal amount = (BigDecimal) Objects.requireNonNull(
        row.get("amount"),
        () -> "导出行缺少必需字段 amount, orderNo=" + row.get("orderNo"));
```

`Objects.requireNonNull` 的第二个参数是延迟求值的，只有真的为 `null` 时才拼接消息，所以带上订单号、行号、批次号这类上下文几乎没有成本，而这条上下文往往就是定位问题的全部线索。

落到写法上，大致是三种情况：

- 强契约，且失败可以接受为"立刻 NPE"：直接取值可以，但加上 `requireNonNull` 和上下文更好，失败点会更清楚。
- 弱契约，且 `null` 会继续传播：必须显式处理，抛 `IllegalStateException`，或者返回结构化结果让上层决定。
- 缺失本身是合法分支（可选字段、可选配置）：`getOrDefault` 是合理的；但要确认"缺失"确实合法，而不是用它掩盖"本不该缺失"的 bug。

`assert` 不在此列。它默认关闭，只有显式加了 `-ea` 才生效，生产环境基本不会开，写了等于没写，反而让人误以为有检查。

更彻底的办法是从类型上消除这种不确定性：key 用枚举而不是字符串，配一张 `EnumMap`；把配置收进一个不可变的配置类，在启动时一次性校验；给方法参数和返回值标注 `@NonNull` / `@Nullable`，配合 IDE 或静态分析工具在编译期发现漏判。这些做法把"运行时随机炸"提前成了"写代码时就知道"。

如果非要给这几种做法排个序：最差的是既不判空也不断言，让 `null` 一路流到数据库；其次是 `getOrDefault` 把 bug 吞成默认值；比较合理的是让契约显式化，在离数据源头最近的地方失败。

## 四、数据库里的数据不一致，不是业务失败

支付回调带着 `orderNo` 回来查订单，查不到；按手机号查用户，本该一条却查出两条；订单明细里的 `sku_id` 在商品主表里没有对应记录。这类问题的共同点是：代码依赖的数据完整性契约被破坏了。

它和"用户输错了密码"有本质区别。后者通过重试或改正就能解决，前者无论用户重试多少次，缺失的数据都不会自己长出来。所以它属于系统级异常，不应该混进业务失败的处理路径。

### 4.1 三种典型的错误处理

**静默兜底**。`Optional.ofNullable(order).orElse(new Order())` 或者 `getOrDefault`，把"数据坏了"变成"业务正常"。页面显示一个空订单，用户投诉之后，排查从"为什么订单是空的"开始，而不是从"订单表里为什么少了这条记录"开始。

**多条里取第一条**。`list.get(0)` 看起来无害，但如果本该唯一的手机号真的重复了，取到的是谁完全取决于数据库的返回顺序，轻则展示错人，重则把别人的订单、地址、手机号展示给当前用户。

**伪装成业务异常**。抛 `BusinessException("订单不存在")` 映射成 404 或者 400，用户看到"订单不存在，请重试"，客服收到一堆"订单丢失"的工单，而真正的问题——数据被删了、迁移漏了、分库路由错了——在监控上只表现为业务错误率轻微上升，不会有人去看。

### 4.2 在查询边界 fail-fast，让方法名表达契约

数据异常最适合在查询发生的地方暴露，因为那里还留有查询条件。Repository 层可以用命名把契约写清楚：

```java
public interface OrderRepository {

    Optional<Order> findByOrderNo(String orderNo);   // 查不到是合法结果，由调用方决定

    Order requireByOrderNo(String orderNo);          // 调用方认为一定存在，查不到即数据异常

    User requireUniqueByPhone(String phone);         // 认为唯一，查出多条即数据异常
}
```

```java
@Component
public class OrderRepositoryImpl implements OrderRepository {

    @Override
    public Optional<Order> findByOrderNo(String orderNo) {
        return Optional.ofNullable(orderMapper.selectByOrderNo(orderNo));
    }

    @Override
    public Order requireByOrderNo(String orderNo) {
        return findByOrderNo(orderNo).orElseThrow(() ->
                new DataIntegrityException("订单号有引用但主表无记录, orderNo=" + orderNo));
    }
}
```

这样检查只写在一处，调用方也不会忘。`findByXxx` 和 `requireByXxx` 成对出现，读代码时签名本身就说明了意图：前者要求调用方自己处理"没有"，后者声明"没有"是不该发生的。MyBatis 的 `selectOne` 在多条结果时会抛 `TooManyResultsException`，但消息里没有业务参数，最好在 Repository 里包一层，把手机号之类的条件补进去，或者直接 `selectList` 自己判断条数。

### 4.3 Service 层要区分"用户能改"和"系统坏了"

同样是"订单不存在"，含义可能完全不同。用户在查询框里输入的订单号查不到，是正常的业务失败，返回 4xx 提示"订单号有误"即可，用户改一下就能继续。支付回调里带着的订单号查不到，是数据异常，因为这笔订单在支付侧真实存在，本地查不到说明数据被删、迁移遗漏或者路由错了，必须 5xx 加告警。

判断依据不是异常类型，而是这个值从哪里来、用户能不能改。把两种情况都塞进同一个 `BusinessException`，结果就是数据问题被埋进业务错误里，业务错误率虚高，真正的故障没人发现。

### 4.4 全局处理器单独收口

数据异常需要一个和业务异常分开的出口：日志打 error 并带上 traceId 和查询条件，触发告警，返回 5xx 和固定的系统错误码，但不把表名、SQL、堆栈返回给客户端。

```java
@ExceptionHandler(DataIntegrityException.class)
public ResponseEntity<ApiResponse<Void>> handleDataIntegrity(DataIntegrityException e,
                                                             HttpServletRequest request) {
    log.error("数据完整性校验失败, uri={}, reason={}", request.getRequestURI(), e.getMessage(), e);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.fail(SystemErrorCode.DATA_INTEGRITY_VIOLATION));
}
```

告警规则可以直接挂在这个错误码或者日志关键字上。这样"数据坏了"和"用户操作不对"在监控上是两条独立曲线，前者应该长期接近零，一旦出现就值得有人去看。

### 4.5 区分暂时性不一致和永久性脏数据

主从延迟导致的"刚写入就查不到"是暂时性的，读取自己刚写入的数据时可以走主库，或者做有限次重试。历史数据迁移遗漏、手工改库、程序 bug 写坏的数据是永久性的，重试多少次结果都一样，只能修数据、补迁移脚本。把两者混在一起处理，要么对永久性问题做了无意义的重试，要么把暂时性问题当成数据损坏去告警。

不管哪一种，最有效的一层防线都在数据库本身：唯一索引、非空约束、外键。应用层的"先查后插"在并发下并不可靠，两个请求同时查到"不存在"，然后同时插入，唯一索引是最后那个能兜住的约束。

```sql
ALTER TABLE `user` ADD UNIQUE KEY uk_user_phone (phone);
```

加了约束之后，重复数据会在写入的那一刻失败，而不是在三个月后某个用户打开订单列表时随机炸掉。代价是要处理 `DuplicateKeyException`，但这总比处理一条来源不明的脏数据便宜。

最后，有一类情况看起来像"多条取一条"，其实不属于本文讨论的问题：业务上明确允许并期望从多条里取一条。比如取用户最近一次登录记录，或者取某商品最新的一条价格，用 `ORDER BY created_at DESC LIMIT 1` 表达即可，语义是清晰的，不需要报错。非关键配置缺失也类似，允许降级到默认值，但要打一条 warn 日志，让"这次走了默认值"在日志里可查。

## 五、把判断串起来

一个失败进来时，判断顺序大致是这样：

1. 是代码或环境的错吗（连接超时、空指针、下游不可用）？是技术异常，向上抛，边界包装时保留 `cause`。
2. 是数据本身坏了吗（该有的没有、该唯一的不唯一）？是数据异常，在查询边界 fail-fast，走 5xx、系统错误码和告警。
3. 是业务规则拦住了吗？看调用方要不要分支：需要按类型走不同逻辑就返回结构化结果，必须终止当前操作就在 Service 边界抛 `BusinessException`。
4. 本来就没有吗（空列表、未设置、无可用项）？正常返回空集合、`Optional` 或普通对象。
5. 这段代码依赖的"一定存在"有多强？强契约可以信任，但仍建议在源头用 `requireNonNull`、`requireByXxx` 把不变量写出来；弱契约必须显式处理。

分层上的分工也随之确定：DAO 层表达查询契约，抛出技术异常和数据异常；Service 层把底层失败翻译成有业务含义的异常并补充上下文；全局异常处理器统一转成响应，给系统异常单独的错误码和告警通道。

这些做法背后是同一个目标：**把代码依赖的不变量显式化**。类型和数据库约束能保证的，就不要留到运行时判空；能 fail-fast 的，就不要让 `null` 或 `-1` 继续往下流；该由调用方做决策的，就不要替他决定；该进告警的，就不要混进业务错误率。异常处理写得好不好，最终体现在故障发生的那一个小时里，日志和错误码能不能直接指向根因。
