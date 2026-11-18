# Presto/Trino 中 0.5 引发的精度谜团：Decimal 隐式类型推导与 MySQL 差异


在 Presto/Trino 中，`0.5` 默认不是 `DOUBLE`，而是 `DECIMAL`。这个容易被忽略的字面量类型，会让 `bigint * 0.5` 这类表达式整体走上 Decimal 算术规则，最终把结果 scale 锁死在 1 位小数。

本文从一个线上"精度丢失"问题切入，逐步拆解 Presto 的类型推导链与 Decimal 四则运算规则，再对比 MySQL 在加减乘除上的差异，最后给出可复用的排查建议。如果你曾在跨引擎或跨团队迁移时被精度问题困扰，这篇文章应该能帮到你。

<!--more-->

## 一个线上问题

某次排查中，用户反馈一个 Presto 查询结果出现"精度丢失"。

原表达式大致如下：

```sql
COALESCE(metric_10251313, 0) * 0.5 / COALESCE(metric_10251433, 0)
```

其中 `metric_10251313` 与 `metric_10251433` 都是 `bigint`。查询结果等于 `0.9`，但用户期望更精确的小数位。

表面上看起来是除法或乘法丢了精度，真正的原因是：**0.5 在 Presto 中被解析为 `DECIMAL`，而不是 `DOUBLE`。一旦表达式里混入 Decimal，整个表达式的类型推导就会按照 Decimal 算术规则走，最终结果的 scale 被限制为 1。**

这就是我们今天要聊的核心：**Presto/Trino 中 Decimal 字面量与整数类型隐式计算时的精度规则**，以及它与 MySQL 的差异。

## Presto 的类型推导链

我们把这个表达式拆开：

```sql
COALESCE(metric_10251313, 0) * 0.5
```

### 第一步：COALESCE 的返回类型

`COALESCE(bigint_column, 0)` 中，两个操作数都是整数类型，因此返回类型为 `bigint`。

> 在后续参与 Decimal 乘法时，`bigint` 会被隐式 cast 为 `DECIMAL(19, 0)`。

### 第二步：0.5 的字面量类型

Presto 的默认行为下，`0.5` 这种十进制字面量会被解析为 `DECIMAL`，而不是 `DOUBLE`。

也就是说：

| 写法 | 解析类型 |
|------|---------|
| `0.5` | `DECIMAL` |
| `0.5e0` | `DOUBLE` |

这是关键分水岭。如果 `0.5` 是 DOUBLE，后续会是浮点运算；但它是 DECIMAL，于是走上了精确算术规则。

### 第三步：bigint × DECIMAL 的类型提升

`bigint * DECIMAL` 时，Presto 会先把 `bigint` 隐式 cast 为 `DECIMAL(19, 0)`，然后应用 Decimal 乘法规则。

## Presto 的 Decimal 运算规则

与 MySQL 不同，Presto/Trino 的 `DECIMAL` 四则运算规则**统一且严格**：

- **结果标度（scale）**：总是取两个操作数标度的较大值 `max(xs, ys)`
- **结果精度（precision）**：按运算类型分别计算，并统一受 38 位上限约束

这里统一用 `DECIMAL(xp, xs)` 和 `DECIMAL(yp, ys)` 表示两个操作数：

- `xp`、`yp`：两个操作数的精度（总位数）
- `xs`、`ys`：两个操作数的标度（小数位数）

### 四则运算的结果类型

| 运算 | 结果精度 (precision) | 结果标度 (scale) |
|------|---------------------|-----------------|
| 加法（+） | `min(38, 1 + min(xs, ys) + min(xp - xs, yp - ys))` | `max(xs, ys)` |
| 减法（−） | 同加法 | `max(xs, ys)` |
| 乘法（*） | `min(38, xp + yp)` | `xs + ys` |
| 除法（/） | `min(38, xp + ys + max(0, ys - xs))` | `max(xs, ys)` |

### 公式解读

- **加法 / 减法**：标度取较大值，保证小数位不丢；精度在"整数部分最大位数"基础上加 1，预留进位或借位空间，再受 38 位上限截断。
- **乘法**：精度与标度分别是两个操作数对应值之和。
- **除法**：标度同样取较大值。精度公式可直观理解为"被除数整数位 + 除数小数位 + 可能的补偿位"，同时受 38 位上限约束。

### 关键注意事项

**结果溢出即报错。** 这是 Presto/Trino 与 MySQL 最显著的区别之一。如果数学计算结果无法用上述推导出的类型精确表示，Presto/Trino 会直接抛出 `Value is out of range` 异常，**不会静默截断或四舍五入**。

**类型提升的潜在风险。** 运算前，Presto/Trino 会先把不同精度和标度的操作数隐式转换为一个"公共超类型"。当类型接近最大精度 38 时，这种转换本身就可能引发溢出。例如 `DECIMAL(38, 0)` 和 `DECIMAL(38, 1)` 的公共超类型是 `DECIMAL(38, 1)`，但 `DECIMAL(38, 0)` 中的某些值无法无损地表示为 `DECIMAL(38, 1)`，从而导致错误。

### 回到最初的问题

代入最开始的表达式：

```sql
COALESCE(metric_10251313, 0) * 0.5
```

- `bigint` 隐式 cast 为 `DECIMAL(19, 0)`
- `0.5` 是 `DECIMAL(p, 1)`

按乘法规则：

- 结果 scale = `0 + 1 = 1`
- 结果 precision = `min(38, 19 + p)`

因此最终类型为 `DECIMAL(20, 1)`：**scale 被固定为 1，只能保留一位小数。** 业务若期望两位或更多小数位，自然就觉得"精度不够"。

## 如何改变行为

Presto 提供了会话属性 `parse_decimal_literals_as_double`。

```sql
SET SESSION parse_decimal_literals_as_double = true;
```

开启后，`0.5` 会被解析为 `DOUBLE`，与 `bigint` 的乘除会走浮点运算路径，得到更多小数位。

但这也意味着：**结果不再是精确 Decimal 运算，而是浮点近似值。** 是否需要开启，取决于你的业务对"精确"与"小数位"的取舍。

## 与 MySQL 的差异

同样是 `DECIMAL` 运算，MySQL 的规则与 Presto 并不相同。下面做一组对比，其中 Presto 的公式采用上文统一形式。

### 1. 乘法规则

| 系统 | 结果精度 | 结果标度 |
|------|---------|---------|
| MySQL | `p1 + p2` | `s1 + s2` |
| Presto/Trino | `min(38, xp + yp)` | `xs + ys` |

两者在乘法上的思路一致：精度与标度分别相加。

示例：`DECIMAL(2, 1) * DECIMAL(2, 1)`，即 `9.9 * 9.9`。

- 结果精度 = `2 + 2 = 4`
- 结果标度 = `1 + 1 = 2`
- 结果 = `98.01`，类型为 `DECIMAL(4, 2)`

### 2. 除法规则

| 系统 | 结果精度 | 结果标度 |
|------|---------|---------|
| MySQL | 内部固定 65 | `s1 + div_precision_increment` |
| Presto/Trino | `min(38, xp + ys + max(0, ys - xs))` | `max(xs, ys)` |

MySQL 的除法受系统变量 `div_precision_increment` 控制，默认值为 4。

示例：`5.05 / 0.014`

- 被除数 scale = 2
- `div_precision_increment` = 4
- 结果 scale = `2 + 4 = 6`
- 结果 = `360.714286`

也就是说，MySQL 的除法结果标度通常比被除数多 4 位，这是由配置决定的。

### 3. 加法与减法规则

| 系统 | 结果精度 | 结果标度 |
|------|---------|---------|
| MySQL | `max(p1-s1, p2-s2) + max(s1, s2) + 1` | `max(s1, s2)` |
| Presto/Trino | `min(38, 1 + min(xs, ys) + min(xp - xs, yp - ys))` | `max(xs, ys)` |

以 `DECIMAL(10, 2) + DECIMAL(8, 2)` 为例：

- 结果标度 = `max(2, 2) = 2`
- 结果精度 = `max(10-2, 8-2) + max(2, 2) + 1 = 8 + 2 + 1 = 11`
- 最终类型 = `DECIMAL(11, 2)`

再举一例，标度不同：`DECIMAL(10, 2) + DECIMAL(8, 4)`

- 结果标度 = `max(2, 4) = 4`
- 结果精度 = `max(10-2, 8-4) + 4 + 1 = 8 + 4 + 1 = 13`
- 最终类型 = `DECIMAL(13, 4)`

公式里的 `+1` 是为了预留进位空间。例如 `99.99 + 0.01 = 100.00`，整数部分会多出一位。

### 4. 对比总表

| 运算 | MySQL 结果精度 | MySQL 结果标度 | Presto/Trino 结果精度 | Presto/Trino 结果标度 |
|------|---------------|---------------|----------------------|----------------------|
| 加法 / 减法 | `max(p1-s1, p2-s2) + max(s1, s2) + 1` | `max(s1, s2)` | `min(38, 1 + min(xs, ys) + min(xp - xs, yp - ys))` | `max(xs, ys)` |
| 乘法 | `p1 + p2` | `s1 + s2` | `min(38, xp + yp)` | `xs + ys` |
| 除法 | 内部固定 65 | `s1 + div_precision_increment` | `min(38, xp + ys + max(0, ys - xs))` | `max(xs, ys)` |

## 线上排查建议

当你再次遇到 Presto/Trino 中的"精度丢失"问题时，可以按下面三步快速定位：

1. **确认表达式中是否存在 Decimal 字面量**：如 `0.5`、`1.23` 这种未加任何类型后缀的十进制数字。
2. **查看类型推导结果**：用 `EXPLAIN (TYPE VALIDATE)` 或 `DESCRIBE` 查看中间结果类型。
3. **明确取舍**：如果业务需要精确计算，保留 Decimal 但显式指定字面量的 scale；如果可以接受浮点近似，使用 `0.5e0` 或开启 `parse_decimal_literals_as_double`。

## 结论

Presto/Trino 中 `0.5` 默认是 `DECIMAL`，不是 `DOUBLE`。这个看似不起眼的字面量类型，会在 `bigint * 0.5` 这样的表达式中引发连锁的类型提升，最终把结果 scale 锁死在 1 位小数。

同样的 `DECIMAL` 计算，在 MySQL 中又有另一套规则，尤其是除法受 `div_precision_increment` 控制。跨引擎迁移或对比结果时，**不能假设它们的精度行为一致**，需要针对每种引擎分别验证。

## 参考资料

- [Presto DECIMAL Type](https://prestodb.io/docs/current/language/types.html#decimal)

