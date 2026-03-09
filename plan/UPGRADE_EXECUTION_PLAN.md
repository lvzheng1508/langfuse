# Langfuse 升级执行计划

## 升级策略：先升级数据库 Schema，再升级项目代码

### 前提条件确认
- ✅ 线上运行的代码版本**不包含**新字段定义
- ✅ 旧代码不会查询新字段，升级 Schema 后不会出错
- ✅ 新字段都有默认值或可为 NULL，不影响数据完整性

---

## 第一阶段：升级数据库 Schema

### 1. 准备工作

#### 1.1 备份数据库
```bash
# PostgreSQL 备份
pg_dump -h 10.186.14.198 -p 5438 -U postgres -d postgres > backup_postgres_$(date +%Y%m%d_%H%M%S).sql

# ClickHouse 备份（如果需要）
# 注意：ClickHouse 备份可能需要特殊工具，建议咨询 DBA
```

#### 1.2 验证当前数据库状态
```bash
# 检查 PostgreSQL organizations 表结构
psql -h 10.186.14.198 -p 5438 -U postgres -d postgres -c "\d organizations"

# 检查 ClickHouse observations 表结构
curl -u "clickhouse:clickhouse" "http://10.186.14.198:8123/?query=DESCRIBE+TABLE+observations"
```

### 2. 执行 PostgreSQL 迁移

```bash
cd packages/shared

# 运行 Prisma 迁移
pnpm run db:migrate
```

**预期结果：**
- organizations 表新增 5 个字段：
  - `cloud_billing_cycle_anchor`
  - `cloud_billing_cycle_updated_at`
  - `cloud_current_cycle_usage`
  - `cloud_free_tier_usage_threshold_state`
  - `ai_features_enabled`

**验证命令：**
```bash
psql -h 10.186.14.198 -p 5438 -U postgres -d postgres -c "\d organizations" | grep -E "(cloud_billing_cycle|ai_features_enabled)"
```

### 3. 执行 ClickHouse 迁移

```bash
cd packages/shared

# 运行 ClickHouse 迁移
pnpm run ch:up
```

**预期结果：**
- observations 表新增 5 个字段：
  - `usage_pricing_tier_id`
  - `usage_pricing_tier_name`
  - `tool_definitions`
  - `tool_calls`
  - `tool_call_names`

**验证命令：**
```bash
curl -u "clickhouse:clickhouse" "http://10.186.14.198:8123/?query=DESCRIBE+TABLE+observations" | grep -E "(tool_|usage_pricing)"
```

### 4. 验证 Schema 升级结果

#### 4.1 PostgreSQL 验证
```sql
SELECT 
  column_name, 
  data_type, 
  is_nullable, 
  column_default
FROM information_schema.columns 
WHERE table_name = 'organizations' 
  AND column_name IN (
    'cloud_billing_cycle_anchor',
    'cloud_billing_cycle_updated_at',
    'cloud_current_cycle_usage',
    'cloud_free_tier_usage_threshold_state',
    'ai_features_enabled'
  )
ORDER BY column_name;
```

#### 4.2 ClickHouse 验证
```bash
curl -u "clickhouse:clickhouse" "http://10.186.14.198:8123/?query=SELECT+name+FROM+system.columns+WHERE+database+%3D+%27default%27+AND+table+%3D+%27observations%27+AND+name+IN+%28%27usage_pricing_tier_id%27%2C+%27usage_pricing_tier_name%27%2C+%27tool_definitions%27%2C+%27tool_calls%27%2C+%27tool_call_names%27%29"
```

### 5. 验证旧代码仍能正常运行

#### 5.1 测试关键功能
- [ ] 用户登录功能
- [ ] 查看 traces
- [ ] 查看 observations
- [ ] 查看 scores
- [ ] API 调用

#### 5.2 监控错误日志
- 检查应用日志中是否有数据库相关错误
- 特别注意：
  - Prisma 查询错误
  - ClickHouse 查询错误
  - 字段不存在的错误

#### 5.3 验证数据完整性
- 检查数据是否正常
- 确保没有数据丢失或损坏

---

## 第二阶段：升级项目代码

### 1. 准备工作

#### 1.1 确认 Schema 升级成功
- [ ] PostgreSQL 迁移完成
- [ ] ClickHouse 迁移完成
- [ ] 旧代码运行正常
- [ ] 无错误日志

#### 1.2 准备代码部署
- [ ] 代码已构建
- [ ] 部署脚本已准备
- [ ] 回滚方案已准备

### 2. 部署新代码

根据你的部署方式执行：
- Docker 部署
- 直接部署
- 其他部署方式

### 3. 验证新代码功能

#### 3.1 功能验证清单
- [ ] 用户登录
- [ ] 查看 traces（包含工具调用相关功能）
- [ ] 查看 observations（验证新字段显示）
- [ ] 查看 scores
- [ ] API 调用
- [ ] 组织管理功能（验证新字段）

#### 3.2 性能监控
- [ ] 响应时间正常
- [ ] 数据库查询性能正常
- [ ] 无异常错误

---

## 回滚方案

### 如果 Schema 升级失败

#### PostgreSQL 回滚
```bash
cd packages/shared

# 查看迁移历史
pnpm prisma migrate status

# 回滚到上一个版本（需要手动执行 SQL）
# 或者恢复备份
psql -h 10.186.14.198 -p 5438 -U postgres -d postgres < backup_postgres_YYYYMMDD_HHMMSS.sql
```

#### ClickHouse 回滚
```sql
-- 手动删除新增字段（如果需要）
ALTER TABLE observations DROP COLUMN IF EXISTS usage_pricing_tier_id;
ALTER TABLE observations DROP COLUMN IF EXISTS usage_pricing_tier_name;
ALTER TABLE observations DROP COLUMN IF EXISTS tool_definitions;
ALTER TABLE observations DROP COLUMN IF EXISTS tool_calls;
ALTER TABLE observations DROP COLUMN IF EXISTS tool_call_names;
```

### 如果代码升级失败

- 回滚到旧代码版本
- 确保旧代码仍能正常运行（Schema 已升级，但旧代码不查询新字段）

---

## 注意事项

1. **备份优先**：升级前务必完成数据库备份
2. **分步验证**：每个步骤完成后都要验证
3. **监控日志**：升级过程中密切监控错误日志
4. **准备回滚**：提前准备好回滚方案和脚本
5. **测试环境**：如果可能，先在测试环境验证

---

## 时间估算

- **Schema 升级**：10-20 分钟
- **验证测试**：10-20 分钟
- **代码部署**：10-20 分钟
- **功能验证**：10-20 分钟

**总计**：约 40-80 分钟（不含备份时间）

---

## 联系信息

如有问题，请参考：
- 升级策略文档：`UPGRADE_STRATEGY.md`
- 数据库适配检查：`/tmp/db_compatibility_check.md`
