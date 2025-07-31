# Aptos DeFi交换数据分析师指南（第二部分）

## 识别DeFi交换交易

在Aptos区块链上分析DeFi交换交易时，我们需要重点关注自动化做市商（AMM）池的操作。通过BigQuery公开数据集，我们可以使用以下SQL语句筛选Liquidswap交易：

```sql
SELECT *
FROM `bigquery-public-data.crypto_aptos_mainnet_us.transactions`
WHERE tx_type = 'user'
AND success
AND payload.entry_function_id_str = '0x9dd974aea0f927ead664b9e1c295e4215bd441a9fb4e53e5ea0bf22f356c8a2b::router::swap_exact_coin_for_coin_x1'
AND block_timestamp BETWEEN '2025–01–01' AND '2025–01–31'
LIMIT 100
```

在处理地址格式时，需要注意AIP-40标准要求的0填充逻辑。以下SQL函数可用于规范化地址格式：

```sql
IF(LENGTH(SPLIT(payload.entry_function_id_str, '::')[0]) != 64,
 '0x' || LPAD(LTRIM(SPLIT(payload.entry_function_id_str, '::')[0],'0x'),64, '0') ||
 SUBSTR(payload.entry_function_id_str, LENGTH(SPLIT(payload.entry_function_id_str, '::')[0])+1,
 LENGTH(payload.entry_function_id_str)),
 payload.entry_function_id_str) AS entry_function_id_str_normalized
```

## 资产交换分析方法论

### 交易上下文获取
通过以下区块链数据源获取交易上下文：
- **事件表**：记录交易触发的消息
- **资源表**：显示链上状态变化

对于闪电贷等复杂交易，事件记录尤为重要。例如一笔闪电贷交易可能包含以下事件流：
1. 资产借出事件
2. 中间交易处理
3. 资产归还事件

### 资产类型差异分析
Aptos区块链上存在两种主要资产标准：
| 资产类型       | 存储机制               | 事件特征              |
|----------------|------------------------|-----------------------|
| Coin标准       | CoinStore资源          | 支持直接存取事件      |
| 可替代资产标准 | 多重FungibleStore资源  | 需通过元数据关联      |

在分析过程中，需要特别注意地址派生逻辑。例如主FungibleStore地址计算公式：
```
SHA3-256(account_address | metadata_address | 0xFC)
```

## 实战案例解析

### Liquidswap交易分析（2224014200）
该笔交易包含以下关键事件流：
1. APT资产从发送方CoinStore转出
2. 池费用计算（0.1%费率）
3. Oracle价格更新
4. 流动性交换执行
5. MOOMOO代币存入接收方账户

通过分析资源变化，我们可以观察到：
- APT储备减少16.182个
- MOOMOO储备增加48,000个
- 池费用存储更新

👉 [深入了解DeFi交易分析工具](https://bit.ly/okx_welcome)

### Cellana交换案例（2293960645）
该笔交易展示了FA标准的典型应用：
1. 13.387 APT存入Cellana金库
2. 铸造Cellana FA标准APT
3. 通过池完成APT-USDt交换
4. USDT存入用户账户

关键资源变化包含：
- Cellana池的4个FungibleStore更新
- 用户账户的APT与USDT余额变动
- 池参数元数据更新

## 聚合器交易解析（Panora案例）

### 跨版本资产交换
在分析Panora聚合器交易（2221267359）时，我们观察到：
1. lzUSDC → MOD → THL → APT的多跳交换路径
2. ThalaSwap v1（Coin）与v2（FA）池的协同工作
3. 资产迁移过程：
   - Coin → FA的透明转换
   - 最终APT结算

### 费用分析模型
通过事件流可追踪各环节费用：
- Thala v1池：3层费用抽取
- Thala v2池：FA标准费用记录
- 聚合器服务费：单独事件标记

## 数据分析工具链

### 原始数据查询方法
使用BigQuery进行价格数据查询的示例：
```sql
SELECT 
  tx_version,
  block_timestamp,
  JSON_VALUE(key.name, '$.bytes') AS price_feed_id,
  CAST(JSON_VALUE(value.content, "$.price_feed.price.price.magnitude") AS BIGNUMERIC) AS price
FROM `bigquery-public-data.crypto_aptos_mainnet_us.table_items`
WHERE address = '0xd1321c17eebcaceee2d54d5f6ea0f78dae846689935ef53d1f0c3cff9e2d6c49'
```

### 链下索引方案
推荐使用以下工具提升分析效率：
1. **Dune仪表盘**：可视化交易流
2. **GraphQL接口**：实时查询FungibleStore状态
3. **自定义解析器**：处理复杂事件组合

👉 [探索专业级区块链分析平台](https://bit.ly/okx_welcome)

## 常见问题解答

### Q1：如何区分Coin和FA标准交易？
A：主要通过存储结构和事件类型判断：
- Coin交易涉及CoinStore资源的存取事件
- FA交易需要追踪FungibleStore变化并关联元数据

### Q2：跨版本资产迁移如何追踪？
A：通过观察：
1. Coin供应量减少事件
2. 对应FA储备增加
3. 静默迁移过程中的元数据更新

### Q3：如何计算AMM池价格影响？
A：使用储备变化公式：
```
价格影响 = (ΔOut/Out)/(ΔIn/In)
```
通过对比交易前后的池状态计算得到

### Q4：聚合器交易分析关键点？
A：需关注：
- 路径优化算法特征
- 跨协议交互顺序
- 最终结算保证机制

## 高级分析技巧

### 地址派生验证代码（Python示例）
```python
def get_primary_fs_address(account_address, metadata_address):
    account_bytes = bytes.fromhex(account_address[2:])
    metadata_bytes = bytes.fromhex(metadata_address[2:])
    fc_byte = bytes.fromhex("fc")
    input_bytes = account_bytes + metadata_bytes + fc_byte
    sha3_hash = hashlib.sha3_256(input_bytes).digest()
    return "0x" + sha3_hash.hex()
```

### 跨链资产识别
主要桥接资产特征：
- LayerZero桥接资产：`0xf22bede2...`前缀
- Wormhole桥接资产：多链版本标识

👉 [获取区块链分析实战指南](https://bit.ly/okx_welcome)

## 行业趋势洞察

### 稳定币发展动态
- 原生稳定币采用率提升
- 多签资产（MOD）创新应用
- RWA（现实世界资产）上链加速

### Oracle服务比较
| 提供商       | 数据结构       | 更新机制       | 服务特点           |
|--------------|----------------|----------------|--------------------|
| Pyth         | 表项存储       | 定期推送       | 高频价格更新       |
| Switchboard  | 资源存储       | 轮次确认       | 去中心化预言机网络 |
| Chainlink    | 智能表         | 请求-响应      | 企业级解决方案     |

本指南展示了如何通过事件溯源和资源变化追踪，深入解析Aptos区块链上的DeFi交易。通过系统化的分析方法，可以有效识别交易模式、评估协议风险，并为投资决策提供数据支持。