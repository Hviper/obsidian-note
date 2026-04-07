# API 代理网关转换逻辑详解

## 一、完整转发流程（5步）

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: 接收请求                                             │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 入口: GenericForwardController                              │
│ 路径: /proxy/{prefix}/**                                    │
│ 方法: GET/POST/PUT/DELETE/PATCH                            │
│                                                             │
│ 示例请求:                                                   │
│ POST /proxy/employee-service/employee/create              │
│ Headers:                                                    │
│   Content-Type: application/json                           │
│   channel: feishu                                          │
│   session_key: user:session:123:nio:john.doe@nio.com      │
│ Body: {"emp_name": "张三", "department": {...}}          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 2: 验证白名单 & 获取配置                                │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 服务: PolarisService                                        │
│ 配置 Key: api_list.key.{prefix}.http_method.{method}       │
│          .original_path.{path}                              │
│                                                             │
│ 配置内容: ApiTemplateConfig                                 │
│   - url: http://internal.hr.com                            │
│   - path: /api/v2/employees/{empId}                        │
│   - ruleList: [转换规则列表]                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 3: 提取源字段                                           │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 服务: ProxySourceFieldService                               │
│                                                             │
│ 从 Headers 提取:                                            │
│   - channel: feishu / im                                   │
│   - session_key: 解析出 brand 和 emp_domain                │
│   - feishu_open_id: 映射到 emp_domain                      │
│                                                             │
│ 源字段映射:                                                 │
│   brand → nio                                              │
│   emp_domain → john.doe@nio.com                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 4: 字段转换                                             │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 工具: FieldReplaceUtil                                      │
│                                                             │
│ 支持 4 种转换类型:                                          │
│   1. PATH   - 路径参数替换                                  │
│   2. QUERY  - 查询参数替换/追加                             │
│   3. HEADER - Header 替换/追加                              │
│   4. BODY   - JSON 字段替换（嵌套+数组）                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Step 5: 转发请求                                             │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ Feign Clients:                                              │
│   - JsonForwardClient (JSON body)                          │
│   - UrlEncodedForwardClient (form-urlencoded)              │
│                                                             │
│ 最终请求:                                                   │
│ POST http://internal.hr.com/api/v2/employees/emp001        │
│   ?worker_user_id=alice                                     │
│   &emp_domain=john.doe@nio.com                             │
│ Headers:                                                    │
│   X-Auth-Token: Bearer token-abc123                        │
│ Body: {"name": "张三", "department": {"id": "D001"}}      │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、字段转换详解

### 1️⃣ PATH 转换（路径参数替换）

**功能**: 替换路径中的占位符 `{placeholder}`

**示例**:
```java
原始路径: /api/v2/employees/{empId}
转换规则:
  type: PATH
  targetField: empId
  sourceValue: emp_id (从 body 中获取值 "emp001")

转换结果: /api/v2/employees/emp001
```

**实现逻辑** (FieldReplaceUtil.applyPathReplacement):
```java
private String applyPathReplacement(String path, String targetField, String replaceValue) {
    String placeholder = "{" + targetField + "}";
    if (path.contains(placeholder)) {
        return path.replace(placeholder, replaceValue);
    }
    return path;
}
```

---

### 2️⃣ QUERY 转换（查询参数替换/追加）

**功能**: 替换或追加 URL 查询参数

**needAppend 参数控制**:
- `false`: 仅替换现有字段，不存在则忽略
- `true/null`: 存在则替换，不存在则追加

**示例**:
```java
原始 Query:
  user_id=alice
  location=SH

转换规则 1:
  type: QUERY
  targetField: worker_user_id
  sourceValue: user_id (从原始 query 获取 "alice")
  needAppend: false (仅替换)

转换规则 2:
  type: QUERY
  targetField: emp_domain
  sourceValue: emp_domain (从 sourceFieldMap 获取 "john.doe@nio.com")
  needAppend: true (追加)

转换结果:
  worker_user_id=alice (user_id 替换为 worker_user_id)
  emp_domain=john.doe@nio.com (新增追加)
  location=SH (保留原样)
```

**实现逻辑** (FieldReplaceUtil.applyQueryParamReplacement):
```java
private void applyQueryParamReplacement(
    Map<String, List<String>> queryParams,
    String targetField,
    String replaceValue,
    Boolean needAppend) {

    if (Objects.nonNull(needAppend) && !needAppend) {
        // 不追加：仅当字段存在时才替换
        List<String> valueList = queryParams.get(targetField);
        if (CollectionUtils.isEmpty(valueList)) {
            return; // 字段不存在，忽略
        }
    }
    // needAppend=true/null：存在则替换，不存在则追加
    queryParams.put(targetField, Collections.singletonList(replaceValue));
}
```

---

### 3️⃣ HEADER 转换（Header 替换/追加）

**功能**: 替换或追加 HTTP Headers

**示例**:
```java
原始 Headers:
  Authorization: Bearer token-abc123
  Content-Type: application/json

转换规则:
  type: HEADER
  targetField: X-Auth-Token
  sourceValue: Bearer token-abc123 (固定值或从原始 header 获取)
  needAppend: null (默认追加)

转换结果:
  X-Auth-Token: Bearer token-abc123
  Content-Type: application/json
```

**实现逻辑** (FieldReplaceUtil.applyHeaderReplacement):
```java
private void applyHeaderReplacement(
    Map<String, String> headers,
    String targetField,
    String replaceValue) {
    headers.put(targetField, replaceValue);
}
```

---

### 4️⃣ BODY 转换（JSON 字段替换）

**功能**: 替换 JSON Body 中的字段，支持：
- 简单字段: `emp_name → name`
- 嵌套字段: `department.dept_id → department.id`
- 数组字段: `employees[].salary → employees[].grade`

**示例**:

#### 简单字段替换
```java
原始 Body:
{
  "emp_name": "张三",
  "emp_id": "emp001"
}

转换规则:
  type: BODY
  targetField: name
  sourceValue: emp_name (从原始 body 获取 "张三")
  needAppend: false

转换结果:
{
  "name": "张三",
  "emp_id": "emp001"
}
```

#### 嵌套字段替换
```java
原始 Body:
{
  "department": {
    "dept_id": "D001",
    "dept_name": "研发部"
  }
}

转换规则:
  type: BODY
  targetField: department.id
  sourceValue: department.dept_id (从原始 body 获取 "D001")
  needAppend: false

转换结果:
{
  "department": {
    "id": "D001",
    "dept_name": "研发部"
  }
}
```

#### 数组字段替换
```java
原始 Body:
{
  "skills": [
    {"skill_name": "Java", "level": "高级"},
    {"skill_name": "Python", "level": "中级"}
  ]
}

转换规则:
  type: BODY
  targetField: skills.grade
  sourceValue: "高级" (固定值，应用到所有数组元素)
  needAppend: true

转换结果:
{
  "skills": [
    {"skill_name": "Java", "level": "高级", "grade": "高级"},
    {"skill_name": "Python", "level": "中级", "grade": "高级"}
  ]
}
```

**实现逻辑** (FieldReplaceUtil.applyNestedFieldAddition):
```java
private void applyNestedFieldAddition(
    JsonNode rootNode,
    String targetPath,  // 例如: "department.id" 或 "skills.grade"
    String value,
    ObjectMapper objectMapper,
    Boolean needAppend) {

    String[] parts = targetPath.split("\\.");

    // 递归遍历路径，支持数组和嵌套对象
    // needAppend 控制是否在字段不存在时追加

    if (current instanceof ObjectNode) {
        if (needAppend || current.has(lastField)) {
            ((ObjectNode) current).put(lastField, value);
        }
    }
}
```

---

## 三、needAppend 参数详解

### 📋 对照表

| needAppend | 字段存在 | 字段不存在 | 适用场景 |
|------------|----------|------------|----------|
| **false**  | 替换 ✓   | 忽略 ⚠️    | 严格映射，避免误添加字段 |
| **true**   | 替换 ✓   | 追加 ✓     | 灵活转换，补充缺失字段 |
| **null**   | 替换 ✓   | 追加 ✓     | 默认行为，等同于 true |

### 🎯 使用场景示例

#### 场景 1: needAppend=false（严格替换）
```
需求: 将 user_id 替换为 worker_user_id，但确保 user_id 必须存在

原始 Query:
  user_id=alice
  location=SH

规则:
  targetField: worker_user_id
  sourceValue: alice
  needAppend: false

结果:
  worker_user_id=alice  (user_id 存在，成功替换)
  location=SH

---

如果原始 Query 没有 user_id:
  location=SH

规则:
  targetField: worker_user_id
  needAppend: false

结果:
  location=SH  (user_id 不存在，worker_user_id 不会被添加)
```

#### 场景 2: needAppend=true（宽松替换）
```
需求: 确保所有请求都包含 emp_domain，缺失时从 sourceFieldMap 补充

原始 Query:
  user_id=alice

规则:
  targetField: emp_domain
  sourceValue: john.doe@nio.com (从 sourceFieldMap)
  needAppend: true

结果:
  user_id=alice
  emp_domain=john.doe@nio.com (追加成功)

---

如果 emp_domain 已存在:
  user_id=alice
  emp_domain=old@example.com

结果:
  user_id=alice
  emp_domain=john.doe@nio.com (替换成功)
```

---

## 四、完整转换示例

### 📝 示例场景

**原始请求**:
```
POST /proxy/employee-service/employee/create

Headers:
  Content-Type: application/json
  Authorization: Bearer token-abc123
  channel: feishu
  session_key: user:session:123:nio:john.doe@nio.com

Query Parameters:
  user_id=alice
  location=SH

Body:
{
  "emp_name": "张三",
  "emp_id": "emp001",
  "department": {
    "dept_id": "D001",
    "dept_name": "研发部"
  },
  "skills": [
    {"skill_name": "Java", "level": "高级"},
    {"skill_name": "Python", "level": "中级"}
  ]
}
```

**Polaris 配置规则**:
```json
{
  "url": "http://internal.hr.com",
  "path": "/api/v2/employees/{empId}",
  "ruleList": [
    {
      "type": "PATH",
      "targetField": "empId",
      "sourceValue": "emp_id",
      "needAppend": null
    },
    {
      "type": "QUERY",
      "targetField": "worker_user_id",
      "sourceValue": "user_id",
      "needAppend": false
    },
    {
      "type": "QUERY",
      "targetField": "emp_domain",
      "sourceValue": "emp_domain",
      "needAppend": true
    },
    {
      "type": "HEADER",
      "targetField": "X-Auth-Token",
      "sourceValue": "Bearer token-abc123",
      "needAppend": null
    },
    {
      "type": "BODY",
      "targetField": "name",
      "sourceValue": "emp_name",
      "needAppend": false
    },
    {
      "type": "BODY",
      "targetField": "department.id",
      "sourceValue": "department.dept_id",
      "needAppend": false
    }
  ]
}
```

**转换结果**:
```
最终 URL:
POST http://internal.hr.com/api/v2/employees/emp001
  ?worker_user_id=alice
  &emp_domain=john.doe@nio.com
  &location=SH

Headers:
  X-Auth-Token: Bearer token-abc123
  Content-Type: application/json

Body:
{
  "name": "张三",
  "emp_id": "emp001",
  "department": {
    "id": "D001",
    "dept_name": "研发部"
  },
  "skills": [
    {"skill_name": "Java", "level": "高级"},
    {"skill_name": "Python", "level": "中级"}
  ]
}
```

---

## 五、核心能力总结

### ✅ 动态路由验证
- Polaris 配置中心验证 API 白名单
- 配置 Key: `api_list.key.{prefix}.http_method.{method}.original_path.{path}`
- 返回 ApiTemplateConfig 包含目标 URL 和转换规则

### ✅ 多源字段提取
- Headers: channel, session_key, feishu_open_id
- Session Key: 解析 brand 和 emp_domain
- Feishu OpenId: 映射到固定域账号

### ✅ 灵活的字段转换引擎
- PATH: 占位符替换 `{field}`
- QUERY: 替换/追加，needAppend 控制
- HEADER: 替换/追加
- BODY: 嵌套字段、数组字段、多层路径

### ✅ 精细的追加控制
- needAppend=false: 严格模式，避免误添加
- needAppend=true/null: 宽松模式，灵活补充

### ✅ 支持复杂结构
- 嵌套对象: `user.department.id`
- 数组对象: `employees[].salary`
- 多层路径: `company.departments[].manager.name`

---

## 六、源字段数据来源

### 🔄 数据流向

```
┌─────────────────────────────────────────┐
│ 数据源                                   │
├─────────────────────────────────────────┤
│ 1. Request Headers                      │
│    - channel                            │
│    - session_key                        │
│    - feishu_open_id                     │
│                                         │
│ 2. Request Query                        │
│    - user_id                            │
│    - location                           │
│                                         │
│ 3. Request Body                         │
│    - emp_id                             │
│    - emp_name                           │
│    - department.dept_id                 │
│                                         │
│ 4. ProxySourceFieldService 提取         │
│    - brand (从 session_key)             │
│    - emp_domain (从 session_key/feishu) │
└─────────────────────────────────────────┘

sourceFieldMap = {
  "brand": "nio",
  "emp_domain": "john.doe@nio.com",
  "emp_id": "emp001",      (从 Body)
  "user_id": "alice"       (从 Query)
}
```

### 🔍 sourceValue 替换逻辑

```java
String replaceValue =
  StringUtils.isNotBlank(sourceFieldMap.get(rule.getSourceValue()))
    ? sourceFieldMap.get(rule.getSourceValue())
    : rule.getSourceValue();  // 固定值
```

**优先级**:
1. 先从 sourceFieldMap 获取值（动态值）
2. 如果不存在，使用 rule.sourceValue 作为固定值

---

## 七、关键文件路径

```
proxy-web/
  └─ controller/GenericForwardController.java  (入口)

proxy-core/
  ├─ service/GenericForwardService.java       (转发逻辑)
  ├─ service/PolarisService.java              (配置获取)
  ├─ service/ProxySourceFieldService.java     (源字段提取)
  └─ util/FieldReplaceUtil.java               (字段转换引擎)

proxy-common/
  ├─ bean/ApiTemplateConfig.java              (配置模型)
  ├─ bean/ApiTemplateReplace.java             (转换结果)
  └─ enums/FieldReplaceType.java              (转换类型枚举)
```

---

## 八、配置示例

### Polaris 配置 Key 格式
```
api_list.key.{prefix}.http_method.{method}.original_path.{path}

示例:
api_list.key.employee-service.http_method.POST.original_path./employee/create
```

### ApiTemplateConfig 结构
```json
{
  "url": "http://target-service.com",
  "path": "/api/v1/resource/{id}",
  "ruleList": [
    {
      "type": "PATH",
      "targetField": "id",
      "sourceValue": "resource_id",
      "needAppend": null
    },
    {
      "type": "QUERY",
      "targetField": "worker_id",
      "sourceValue": "user_id",
      "needAppend": false
    },
    {
      "type": "HEADER",
      "targetField": "X-Custom-Token",
      "sourceValue": "Authorization",
      "needAppend": null
    },
    {
      "type": "BODY",
      "targetField": "user.name",
      "sourceValue": "user.emp_name",
      "needAppend": false
    }
  ]
}
```

---

## 九、测试验证

测试文件位置:
```
proxy-core/src/test/java/.../GenericForwardServiceTest.java
```

包含 3 个详细测试:
1. `testCompleteForwardFlow` - 完整转发流程演示
2. `testNeedAppendBehavior` - needAppend 参数详解
3. `testBodyNestedAndArrayFieldReplacement` - BODY 复杂转换

运行命令:
```bash
mvn test -Dtest=GenericForwardServiceTest -pl proxy-core
```

---

**文档创建时间**: 2026-04-05
**Maven 版本**: 3.9.9
**项目**: up-mate-intelligence-proxy