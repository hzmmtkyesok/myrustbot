# Polymarket Copy Trading Bot - 部署方案

## 一、系统要求

### 硬件要求
- **CPU**: 2核以上
- **内存**: 4GB以上 (推荐8GB)
- **存储**: 10GB以上可用空间
- **网络**: 稳定的网络连接，低延迟优先

### 推荐 VPS 配置
- **位置**: 阿姆斯特丹 (AMS) - Polymarket 有地区限制，AMS VPS 是常用选择
- **延迟**: Sub-10ms 到主要 Polygon 节点
- **供应商推荐**: [Trading VPS](https://app.tradingvps.io/aff.php?aff=60)

### 软件要求
- **操作系统**: Ubuntu 22.04 LTS / Debian 11+ / macOS
- **Rust**: 1.75+ (建议使用 rustup 安装)
- **Python**: 3.9+ (用于缓存脚本)

## 二、环境准备

### 1. 安装 Rust
```bash
# 安装 rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 验证安装
rustc --version  # 应显示 1.75+
cargo --version
```

### 2. 安装 Python 依赖 (用于缓存脚本)
```bash
pip3 install aiohttp
```

### 3. 克隆/上传项目
```bash
# 解压项目
unzip polymarket-kalshi-copy-trading-arbitrage-bot-main.zip
cd polymarket-kalshi-copy-trading-arbitrage-bot-main/rust
```

## 三、配置设置

### 1. 创建环境配置文件
```bash
cp .env.example .env
nano .env  # 或使用你喜欢的编辑器
```

### 2. 必填配置项

```env
# 你的钱包私钥 (64字符十六进制，无 0x 前缀)
PRIVATE_KEY=your_64_char_hex_private_key_here

# 你的钱包地址 (40字符十六进制)
FUNDER_ADDRESS=your_wallet_address_here

# 目标鲸鱼地址 (要跟单的地址，无 0x 前缀)
TARGET_WHALE_ADDRESS=whale_address_to_copy

# WebSocket RPC 提供商 (二选一)
ALCHEMY_API_KEY=your_alchemy_api_key
# 或
CHAINSTACK_API_KEY=your_chainstack_api_key
```

### 3. 可选配置项

```env
# 交易控制
ENABLE_TRADING=true          # 启用实际交易
MOCK_TRADING=false           # 模拟模式（不实际下单）

# 风控参数
CB_LARGE_TRADE_SHARES=1500   # 大单阈值（股数）
CB_CONSECUTIVE_TRIGGER=2     # 连续大单触发数
CB_SEQUENCE_WINDOW_SECS=30   # 检测窗口（秒）
CB_MIN_DEPTH_USD=200         # 最小深度（美元）
CB_TRIP_DURATION_SECS=120    # 熔断持续时间（秒）
```

### 4. 获取 API Key

**Alchemy (推荐新手)**:
1. 访问 https://www.alchemy.com/
2. 注册账号
3. 创建 App，选择 Polygon Mainnet
4. 复制 API Key 到 .env

**Chainstack (备选)**:
1. 访问 https://chainstack.com/
2. 注册账号
3. 创建 Polygon 节点
4. 复制 API Key

## 四、构建项目

### 1. 开发构建 (快速编译)
```bash
cargo build
```

### 2. 生产构建 (优化性能)
```bash
cargo build --release
```

### 3. 验证配置
```bash
cargo run --release --bin validate_setup
```

## 五、缓存预热

在首次运行前，需要预热市场缓存：

```bash
# 构建体育市场缓存
python3 scripts/build_sports_cache.py

# 构建实时状态缓存
python3 scripts/build_live_cache.py

# 构建 ATP 网球市场缓存
python3 scripts/fetch_categorized_atp.py

# 构建 Ligue 1 足球市场缓存
python3 scripts/fetch_ligue1.py
```

## 六、运行方式

### 方式一：直接运行
```bash
# 前台运行
cargo run --release

# 或使用编译好的二进制文件
./target/release/pm_bot
```

### 方式二：使用脚本运行
```bash
# Linux/macOS
chmod +x run.sh
./run.sh

# Windows
run.bat
```

### 方式三：后台运行 (推荐生产环境)

**使用 screen:**
```bash
screen -S polybot
cargo run --release
# 按 Ctrl+A, D 分离会话

# 重新连接
screen -r polybot
```

**使用 systemd (推荐):**

创建服务文件 `/etc/systemd/system/polybot.service`:
```ini
[Unit]
Description=Polymarket Copy Trading Bot
After=network.target

[Service]
Type=simple
User=your_username
WorkingDirectory=/path/to/rust
Environment="PATH=/home/your_username/.cargo/bin:/usr/bin"
ExecStart=/path/to/rust/target/release/pm_bot
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启用服务:
```bash
sudo systemctl daemon-reload
sudo systemctl enable polybot
sudo systemctl start polybot

# 查看状态
sudo systemctl status polybot

# 查看日志
journalctl -u polybot -f
```

## 七、定时任务设置

设置 crontab 定期刷新缓存：
```bash
crontab -e
```

添加以下内容:
```cron
# 每30分钟更新体育市场缓存
*/30 * * * * cd /path/to/rust && python3 scripts/build_sports_cache.py >> /tmp/cache_warmer.log 2>&1

# 每2分钟更新实时状态缓存
*/2 * * * * cd /path/to/rust && python3 scripts/build_live_cache.py >> /tmp/live_cache.log 2>&1

# 每4小时更新 ATP 市场缓存
0 */4 * * * cd /path/to/rust && python3 scripts/fetch_categorized_atp.py >> /tmp/atp_cache.log 2>&1

# 每30分钟更新 Ligue 1 市场缓存
*/30 * * * * cd /path/to/rust && python3 scripts/fetch_ligue1.py >> /tmp/ligue1_cache.log 2>&1
```

## 八、监控与日志

### 1. 交易日志
交易记录会保存到 `matches_optimized.csv`：
```bash
# 实时查看日志
tail -f matches_optimized.csv
```

### 2. 控制台输出含义
- `⚡` - 检测到交易事件
- `🔄` - 重试订单
- `✅` - 订单成功
- `❌` - 订单失败
- `🚀` - 程序启动

### 3. 颜色代码
- 绿色: 高填充率 (>90%)
- 黄色: 中等填充率 (75-90%)
- 橙色: 较低填充率 (50-75%)
- 红色: 低填充率 (<50%)

## 九、工具命令

### 测试订单类型
```bash
cargo run --release --bin test_order_types
```

### 验证配置
```bash
cargo run --release --bin validate_setup
```

### 监控你自己的交易
```bash
cargo run --release --bin trade_monitor
```

### Mempool 监控 (高级)
```bash
cargo run --release --bin mempool_monitor
```

## 十、故障排除

### 问题：WebSocket 连接失败
- 检查 API Key 是否正确
- 检查网络连接
- 尝试切换 RPC 提供商

### 问题：订单总是失败
- 检查钱包是否有足够的 USDC
- 检查钱包是否已授权 Polymarket
- 验证私钥和地址匹配

### 问题：缓存文件不存在
- 运行缓存预热脚本
- 检查 Python 依赖是否安装

### 问题：编译错误
- 确保 Rust 版本 >= 1.75
- 运行 `cargo clean && cargo build --release`

## 十一、安全建议

1. **私钥安全**: 
   - 永远不要分享你的私钥
   - 使用专门的交易钱包，不要用主钱包
   - 只放入准备承担风险的资金

2. **VPS 安全**:
   - 使用强密码
   - 启用 SSH 密钥认证
   - 配置防火墙

3. **交易安全**:
   - 先用 `MOCK_TRADING=true` 测试
   - 小资金测试至少 10-20 笔交易
   - 理解跟单策略的风险

## 十二、更新与维护

### 更新代码
```bash
# 备份配置
cp .env .env.backup

# 更新代码后
cargo clean
cargo build --release
```

### 日常维护
- 定期检查日志文件大小
- 监控钱包余额
- 检查缓存更新是否正常

---

## 联系与支持

- Telegram: [@soulcrancerdev](https://t.me/soulcrancerdev)
- X: [@soulcrancerdev](https://x.com/soulcrancerdev)

## 免责声明

本软件仅供学习和研究目的。加密货币交易存在高风险，过去的表现不代表未来的结果。请自行承担交易风险。
