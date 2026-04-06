---
title: Activity 幂等性实践指南
date: 2026-04-04
tags:
  - temporal
  - activity
  - idempotency
  - best-practices
aliases:
  - Activity 幂等性设计
  - Temporal 幂等性实现
---

# Activity 幂等性实践指南

## 核心概念

> [!important] 核心定义
> **幂等性（Idempotency）**：同一个操作执行一次和执行多次，产生的结果是一样的。

**为什么重要？** Temporal 会在 Activity 失败时自动重试，因此 Activity 必须保证重试安全。

## 常见非幂等操作

### ❌ 问题代码示例

```java
// ❌ 问题 1：累加操作
@Override
public void addBonusPoints(String userId, int points) {
    User user = database.findUser(userId);
    user.points += points;  // 每次重试都会增加
    database.save(user);
    // 第 1 次：100 + 10 = 110
    // 第 2 次：110 + 10 = 120  // ❌ 重复增加了
}

// ❌ 问题 2：发送消息
@Override
public void sendEmail(String to, String subject, String body) {
    emailService.send(to, subject, body);
    // 第 1 次：发送成功
    // 第 2 次：重试导致重复发送  // ❌
}

// ❌ 问题 3：创建订单
@Override
public void createOrder(OrderData data) {
    Order order = new Order(data);
    database.insert(order);  // 重复插入会报错或重复创建
}

// ❌ 问题 4：外部 API 调用
@Override
public void callPaymentAPI(PaymentRequest request) {
    paymentService.charge(request);
    // 网络超时但实际扣款成功
    // 重试导致重复扣款  // ❌
}
```

## 幂等性实现模式

### 模式 1：唯一 ID 模式

**适用场景**：创建资源、执行一次性操作

```java
// ✅ 正确实现：使用唯一事务 ID
@Override
public void addBonusPoints(String transactionId, String userId, int points) {
    // 1. 检查事务是否已处理
    Transaction existing = database.findTransaction(transactionId);
    if (existing != null) {
        log.info("事务 {} 已处理，跳过", transactionId);
        return;  // 已处理，直接返回
    }

    // 2. 执行业务逻辑
    User user = database.findUser(userId);
    user.points += points;
    database.save(user);

    // 3. 记录事务
    Transaction transaction = new Transaction(transactionId, userId, points);
    database.saveTransaction(transaction);

    // 结果：无论重试多少次，积分只增加一次 ✅
}

// 调用方式
@WorkflowMethod
public void processBonus(String userId, int points) {
    String transactionId = "bonus-" + userId + "-" + System.currentTimeMillis();
    activity.addBonusPoints(transactionId, userId, points);
}
```

**数据库设计：**

```sql
CREATE TABLE transactions (
    transaction_id VARCHAR(255) PRIMARY KEY,  -- 唯一约束
    user_id VARCHAR(255),
    amount INT,
    created_at TIMESTAMP
);
```

### 模式 2：检查-执行模式

**适用场景**：更新状态、创建资源

```java
// ✅ 创建订单（幂等）
@Override
public void createOrder(String orderId, OrderData data) {
    // 1. 检查订单是否已存在
    Order existing = database.findOrderById(orderId);
    if (existing != null) {
        log.info("订单 {} 已存在，跳过创建", orderId);
        return existing;  // 返回已存在的订单
    }

    // 2. 创建新订单
    Order order = new Order(orderId, data);
    database.insertOrder(order);

    return order;
    // 结果：无论重试多少次，只会创建一个订单 ✅
}

// ✅ 发送邮件（幂等）
@Override
public void sendEmail(String emailId, String to, String subject, String body) {
    // 检查是否已发送
    if (emailService.isSent(emailId)) {
        log.info("邮件 {} 已发送，跳过", emailId);
        return;
    }

    // 发送邮件
    emailService.send(emailId, to, subject, body);
    emailService.markAsSent(emailId);
}
```

### 模式 3：条件更新模式

**适用场景**：状态更新、数据修改

```java
// ✅ 更新订单状态（幂等）
@Override
public void updateOrderStatus(String orderId, String newStatus) {
    // 使用 SQL WHERE 条件确保幂等性
    String sql = "UPDATE orders SET status = ? WHERE id = ? AND status != ?";
    int updated = jdbcTemplate.update(sql, newStatus, orderId, newStatus);

    if (updated == 0) {
        log.info("订单 {} 状态已是 {}，无需更新", orderId, newStatus);
    }

    // 结果：无论执行多少次，最终状态都是 newStatus ✅
}

// ✅ 扣减库存（幂等）
@Override
public void deductStock(String productId, int quantity, String transactionId) {
    // 检查是否已扣减
    InventoryTransaction existing = database.findTransaction(transactionId);
    if (existing != null) {
        return;  // 已扣减
    }

    // 使用乐观锁扣减库存
    String sql = "UPDATE inventory SET stock = stock - ? " +
                 "WHERE product_id = ? AND stock >= ?";
    int updated = jdbcTemplate.update(sql, quantity, productId, quantity);

    if (updated == 0) {
        throw new RuntimeException("库存不足");
    }

    // 记录扣减事务
    database.saveTransaction(new InventoryTransaction(transactionId, productId, quantity));
}
```

### 模式 4：自然幂等模式

**适用场景**：设置操作、纯计算

```java
// ✅ 设置用户状态（自然幂等）
@Override
public void setUserStatus(String userId, String status) {
    // 设置操作：多次执行结果相同
    User user = database.findUser(userId);
    user.status = status;
    database.save(user);
    // 第 1 次：status = "active"
    // 第 2 次：status = "active"
    // 结果相同 ✅
}

// ✅ 纯计算函数（自然幂等）
@Override
public int calculateTotal(List<OrderItem> items) {
    // 纯函数：相同输入，相同输出
    return items.stream()
        .mapToInt(item -> item.price * item.quantity)
        .sum();
    // 多次计算结果相同 ✅
}

// ✅ 覆盖文件（自然幂等）
@Override
public void saveFile(String filePath, String content) {
    Files.writeString(Path.of(filePath), content);
    // 多次写入相同内容，结果相同 ✅
}
```

## 实际场景案例

### 案例 1：支付处理

```java
// ❌ 非幂等实现
@Override
public void processPayment(PaymentRequest request) {
    paymentGateway.charge(request.getAmount());
    database.updateOrderStatus(request.getOrderId(), "PAID");
}

// ✅ 幂等实现
@Override
public void processPayment(String paymentId, PaymentRequest request) {
    // 1. 检查支付状态
    Payment existing = database.findPaymentById(paymentId);
    if (existing != null && existing.getStatus() == "SUCCESS") {
        log.info("支付 {} 已完成，跳过", paymentId);
        return;  // 已支付
    }

    // 2. 幂等键请求
    PaymentResponse response = paymentGateway.chargeWithIdempotencyKey(
        paymentId,  // 幂等键
        request.getAmount(),
        request.getCreditCard()
    );

    // 3. 记录支付结果
    if (response.isSuccess()) {
        Payment payment = new Payment(paymentId, request.getOrderId(), "SUCCESS");
        database.savePayment(payment);
        database.updateOrderStatus(request.getOrderId(), "PAID");
    } else {
        throw new PaymentException("支付失败: " + response.getError());
    }
}
```

### 案例 2：发送通知

```java
// ❌ 非幂等实现
@Override
public void sendNotification(String userId, String message) {
    emailService.send(userId, message);
    smsService.send(userId, message);
}

// ✅ 幂等实现
@Override
public void sendNotification(String notificationId, String userId, String message) {
    // 检查是否已发送
    if (notificationService.isSent(notificationId)) {
        return;  // 已发送
    }

    // 发送通知
    try {
        emailService.send(userId, message);
        smsService.send(userId, message);

        // 标记为已发送
        notificationService.markAsSent(notificationId, userId, message);
    } catch (Exception e) {
        // 失败时清理状态，允许重试
        notificationService.markAsFailed(notificationId);
        throw e;
    }
}
```

### 案例 3：数据迁移

```java
// ❌ 非幂等实现
@Override
public void migrateData(String sourceTable, String targetTable) {
    List<Data> data = database.query(sourceTable);
    database.batchInsert(targetTable, data);
}

// ✅ 幂等实现
@Override
public void migrateData(String migrationId, String sourceTable, String targetTable) {
    // 1. 检查迁移状态
    MigrationRecord record = database.findMigration(migrationId);
    if (record != null && record.getStatus() == "COMPLETED") {
        log.info("迁移 {} 已完成", migrationId);
        return;
    }

    // 2. 使用临时表避免重复
    String tempTable = targetTable + "_temp_" + migrationId;

    // 3. 迁移数据到临时表
    List<Data> data = database.query(sourceTable);
    database.createTable(tempTable, getSchema(targetTable));
    database.batchInsert(tempTable, data);

    // 4. 原子性替换
    database.renameTable(tempTable, targetTable);

    // 5. 记录迁移完成
    database.saveMigration(new MigrationRecord(migrationId, "COMPLETED"));
}
```

### 案例 4：第三方 API 调用

```java
// ❌ 非幂等实现
@Override
public ApiResponse callExternalAPI(APIRequest request) {
    return httpClient.post("https://api.example.com/endpoint", request);
}

// ✅ 幂等实现
@Override
public ApiResponse callExternalAPI(String requestId, APIRequest request) {
    // 1. 检查是否已调用
    APICallRecord record = database.findAPICall(requestId);
    if (record != null) {
        log.info("API 调用 {} 已完成", requestId);
        return record.getResponse();  // 返回缓存的结果
    }

    // 2. 调用外部 API（使用幂等键）
    try {
        ApiResponse response = httpClient.post(
            "https://api.example.com/endpoint",
            request,
            Headers.of("X-Idempotency-Key", requestId)  // 传递幂等键
        );

        // 3. 缓存结果
        database.saveAPICall(new APICallRecord(requestId, response));

        return response;
    } catch (NetworkException e) {
        // 网络异常，可以安全重试
        throw new RetryableException("网络异常，稍后重试", e);
    }
}
```

## 长时间运行的 Activity

### 使用 Heartbeat 支持长时间操作

```java
@Override
public void processLargeDataset(String datasetId) {
    // 获取上次处理进度
    int lastProcessed = getHeartbeatDetails();

    List<Record> records = database.queryRecords(datasetId);

    for (int i = lastProcessed; i < records.size(); i++) {
        Record record = records.get(i);

        // 处理记录
        processRecord(record);

        // 每 100 条发送心跳
        if (i % 100 == 0) {
            Activity.getExecutionContext().heartbeat(i);
            log.info("已处理 {}/{} 条记录", i, records.size());
        }
    }

    log.info("数据集处理完成，共 {} 条记录", records.size());
}
```

**Heartbeat 工作原理：**

```
第 1 次执行：
  处理到第 500 条 → heartbeat(500)
  处理到第 1000 条 → heartbeat(1000)
  服务崩溃 ❌

Temporal 记录：lastHeartbeat = 1000

第 2 次执行（重试）：
  getHeartbeatDetails() 返回 1000
  从第 1001 条继续处理 ✅
```

## 幂等性检查清单

> [!check] 设计 Activity 时的检查清单

### 业务逻辑层面

- [ ] **识别副作用操作**：哪些操作会改变系统状态？
- [ ] **确定唯一标识**：使用什么作为幂等键？
- [ ] **检查已处理**：如何判断操作是否已执行？
- [ ] **设计补偿机制**：失败后如何清理状态？

### 数据库层面

- [ ] **唯一约束**：是否需要添加唯一索引？
- [ ] **事务隔离**：是否需要事务保证？
- [ ] **乐观锁**：是否需要版本号或时间戳？
- [ ] **事务记录表**：是否需要记录事务状态？

### 外部服务层面

- [ ] **幂等键支持**：外部 API 是否支持幂等键？
- [ ] **重试策略**：失败后如何重试？
- [ ] **超时处理**：如何处理超时但实际成功的情况？
- [ ] **结果缓存**：是否需要缓存响应结果？

### 监控和日志

- [ ] **日志记录**：是否记录幂等检查日志？
- [ ] **监控指标**：是否监控重复请求？
- [ ] **告警机制**：是否对异常重复处理告警？

## 常见陷阱

### 陷阱 1：时间依赖

```java
// ❌ 非幂等：依赖当前时间
@Override
public void createOrder(OrderData data) {
    Order order = new Order();
    order.setCreatedAt(LocalDateTime.now());  // ❌ 每次重试时间不同
    database.save(order);
}

// ✅ 幂等：使用确定性时间
@Override
public void createOrder(String orderId, OrderData data) {
    Order existing = database.findById(orderId);
    if (existing != null) {
        return existing;  // 返回已存在的订单
    }

    Order order = new Order(orderId, data);
    database.save(order);
}
```

### 陷阱 2：随机数

```java
// ❌ 非幂等：使用随机数
@Override
public String generateApiKey(String userId) {
    String apiKey = UUID.randomUUID().toString();  // ❌ 每次生成不同
    database.saveApiKey(userId, apiKey);
    return apiKey;
}

// ✅ 幂等：确定性生成
@Override
public String generateApiKey(String userId) {
    // 检查是否已生成
    String existingKey = database.getApiKey(userId);
    if (existingKey != null) {
        return existingKey;
    }

    // 使用确定性方法生成（如：HMAC）
    String apiKey = generateDeterministicKey(userId);
    database.saveApiKey(userId, apiKey);
    return apiKey;
}
```

### 陷阱 3：自增序列

```java
// ❌ 非幂等：使用自增 ID
@Override
public void createOrderItem(String orderId, ItemData data) {
    OrderItem item = new OrderItem();  // 自增 ID
    item.setOrderId(orderId);
    item.setData(data);
    database.insert(item);  // 每次插入新记录 ❌
}

// ✅ 幂等：使用业务 ID
@Override
public void createOrderItem(String itemId, String orderId, ItemData data) {
    OrderItem existing = database.findById(itemId);
    if (existing != null) {
        return;  // 已存在
    }

    OrderItem item = new OrderItem(itemId, orderId, data);
    database.insert(item);
}
```

## 幂等性测试方法

```java
@Test
public void testIdempotency() {
    String transactionId = "test-txn-123";
    String userId = "user-456";
    int points = 100;

    // 第 1 次执行
    activity.addBonusPoints(transactionId, userId, points);
    User user1 = database.findUser(userId);
    int pointsAfter1st = user1.getPoints();

    // 第 2 次执行（模拟重试）
    activity.addBonusPoints(transactionId, userId, points);
    User user2 = database.findUser(userId);
    int pointsAfter2nd = user2.getPoints();

    // 验证：积分应该相同
    assertEquals(pointsAfter1st, pointsAfter2nd, "积分应该相同，验证幂等性");
}
```

## 总结

> [!success] Activity 幂等性实现要点
>
> 1. **唯一标识**：为每个操作分配唯一 ID
> 2. **检查-执行**：先检查是否已处理，再执行
> 3. **结果缓存**：记录处理结果，避免重复执行
> 4. **条件更新**：使用 WHERE 条件确保幂等性
> 5. **自然幂等**：优先设计天然幂等的操作
> 6. **补偿机制**：失败后清理状态，允许重试

**关键记忆：**
- Activity 不需要原子性，但必须幂等性
- 幂等性 = 相同输入 → 相同结果
- 使用唯一 ID 是最通用的幂等性实现方式

## 相关资源

- [[Temporal工作流框架完整指南]]
- [[Activity 设计最佳实践]]
- [[分布式系统幂等性设计模式]]

## 标签

#temporal #activity #idempotency #distributed-systems #java #best-practices