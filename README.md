# 闲鱼 AI 客服机器人（Claude 版）

基于 [XianyuAutoAgent](https://github.com/shaxiu/XianyuAutoAgent) 开源项目，将默认的通义千问替换为 **Anthropic Claude**，实现 7×24 小时自动回复买家消息、智能议价、技术咨询等功能。

---

## 功能特性

- **自动回复**：实时接收买家消息，由 Claude 生成自然回复
- **智能议价**：根据可配置的价格比例，自动判断同意/拒绝砍价
- **技术咨询**：识别商品相关技术问题，给出专业回答
- **人工接管**：发送特定关键词（默认 `。`）可临时暂停自动回复
- **安全过滤**：自动拦截引导至微信/QQ 等站外交流的内容

---

## 目录结构

```
XianyuAutoAgent/
├── main.py              # 程序入口，WebSocket 连接管理
├── XianyuAgent.py       # AI Agent 核心逻辑（已改为 Claude）
├── XianyuApis.py        # 闲鱼平台 API 客户端
├── context_manager.py   # 对话历史管理（SQLite）
├── requirements.txt     # Python 依赖
├── .env                 # 配置文件（你需要填写）
└── prompts/             # 提示词目录
    ├── classify_prompt_example.txt   # 意图分类提示词（示例）
    ├── price_prompt_example.txt      # 议价提示词（示例）
    ├── tech_prompt_example.txt       # 技术咨询提示词（示例）
    └── default_prompt_example.txt    # 默认回复提示词（示例）
```

---

## 快速开始

### 第一步：安装依赖

```bash
pip3 install -r requirements.txt
```

### 第二步：配置 .env 文件

打开项目根目录的 `.env` 文件，填写以下必填项：

```env
# 必填：Anthropic API Key
ANTHROPIC_API_KEY=sk-ant-xxxx

# 必填：中转站地址（如使用官方 API 则删除此行）
ANTHROPIC_API_URL=xxxx

# 必填：闲鱼 Cookie（获取方式见下方）
COOKIES_STR=你的完整Cookie字符串
```

### 第三步：获取闲鱼 Cookie

> Cookie 是程序登录你闲鱼账号的凭证，**不要泄露给他人**。

**详细步骤：**

1. 用 Chrome/Edge 浏览器打开 [https://www.goofish.com](https://www.goofish.com)

2. 点击右上角「登录」，用手机扫码登录你的闲鱼账号

3. 登录成功后，点击顶部导航栏的「**消息**」按钮
   > ⚠️ 必须点消息！这一步会生成 `_m_h5_tk` 字段，缺少它程序无法运行

4. 如果出现滑块验证，**滑动完成验证**

5. 按 `F12` 打开开发者工具，切换到 **Network（网络）** 标签

6. 在页面上随便点击一下，触发网络请求

7. 在请求列表中找到任意一个请求（推荐找 `mtop.xxx` 开头的），点击它

8. 在右侧 **Headers（标头）** → **Request Headers** 中找到 `Cookie:` 字段

9. **右键 → 复制值**，复制完整的 Cookie 字符串（很长，正常）

10. 将复制的内容粘贴到 `.env` 文件的 `COOKIES_STR=` 后面

**验证 Cookie 是否完整：** Cookie 中必须包含以下字段：
```
unb=...          # 用户 ID（必须有）
_m_h5_tk=...     # API 签名 token（必须有）
cookie2=...      # 会话 cookie（必须有）
```

### 第四步：启动程序

```bash
cd /pathto/XianyuAutoAgent
python3 main.py
```

看到以下日志说明启动成功：
```
INFO | 成功加载所有提示词
INFO | WebSocket connected
INFO | 开始监听消息...
```

---

## 详细配置说明

### 模型选择

在 `.env` 中修改 `MODEL_NAME`：

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| `claude-haiku-4-5-20251001` | 最快、最便宜 | 流量大、回复简单 |
| `claude-sonnet-4-6` | 性价比最佳 ✅ 推荐 | 日常使用 |
| `claude-opus-4-6` | 最聪明、最贵 | 复杂商品咨询 |

### 议价规则配置

在 `.env` 中修改两个比例参数：

```env
PRICE_EASY_RATIO=0.9   # 买家出价 >= 原价90%，直接同意
PRICE_FLOOR_RATIO=0.8  # 买家出价 < 原价80%，坚决拒绝
```

**示例：** 商品原价 100 元，默认配置下：
- 买家出 90 元及以上 → AI 直接同意
- 买家出 80~89 元 → AI 说"已经很低了"但同意
- 买家出 79 元及以下 → AI 拒绝，说"80 折是最低了"

**调整策略：**
- 不想让步太多：提高 `PRICE_FLOOR_RATIO`（如改为 0.85）
- 更灵活成交：降低 `PRICE_FLOOR_RATIO`（如改为 0.7）

### 自定义提示词

在 `prompts/` 目录创建不带 `_example` 后缀的文件，会覆盖默认提示词：

```bash
# 复制示例文件并修改
cp prompts/default_prompt_example.txt prompts/default_prompt.txt
cp prompts/price_prompt_example.txt prompts/price_prompt.txt
```

建议在 `default_prompt.txt` 中补充你的商品描述，例如：
```
你是一位二手数码产品卖家的客服助手。
主营：iPhone、MacBook、AirPods 等苹果产品。
风格：简洁友好，不超过 30 字/条。
```

### 人工接管模式

当你需要亲自回复时，在闲鱼 APP 中向该买家发送 `。`（句号），AI 会暂停回复该对话 1 小时。

再次发送 `。` 可恢复自动回复。

可在 `.env` 中修改触发关键词：
```env
TOGGLE_KEYWORDS=暂停    # 改为发送"暂停"时触发
```

### 模拟人工打字

启用后，AI 回复前会有随机延迟，避免买家察觉是机器人：
```env
SIMULATE_HUMAN_TYPING=True
```

---

## 完整 .env 配置参考

```env
# ===== 必填 =====
ANTHROPIC_API_KEY=sk-ant-你的key
ANTHROPIC_API_URL=xxx   # 中转站，使用官方则删除
MODEL_NAME=claude-sonnet-4-6
COOKIES_STR=你的完整Cookie

# ===== 议价规则 =====
PRICE_EASY_RATIO=0.9    # 轻松接受线（>= 此比例直接同意）
PRICE_FLOOR_RATIO=0.8   # 最低底线（< 此比例坚决拒绝）

# ===== 可选配置 =====
TOGGLE_KEYWORDS=。              # 人工接管触发词
SIMULATE_HUMAN_TYPING=True      # 模拟打字延迟
HEARTBEAT_INTERVAL=15           # WebSocket 心跳间隔（秒）
LOG_LEVEL=INFO                  # 日志级别：DEBUG/INFO/WARNING
```

---

## 常见问题

### Q：启动后报 `KeyError: 'unb'`
**原因：** Cookie 不完整，缺少 `unb` 字段。
**解决：** 重新获取 Cookie，确保已点击「消息」页面后再复制。

### Q：报 `RGV587_ERROR::SM::哎哟喂,被挤爆啦`
**原因：** 触发了闲鱼风控验证。
**解决：**
1. 浏览器打开闲鱼网页版 → 点击「消息」→ 完成滑块验证
2. 重新复制 Cookie 更新 `.env`
3. 重启程序

### Q：程序运行正常但 AI 没有回复
**可能原因：**
- 当前对话处于人工接管模式（发送 `。` 恢复）
- 买家消息被判定为无需回复（如系统消息、表情包）
- 检查 `LOG_LEVEL=DEBUG` 查看详细日志

### Q：Cookie 多久过期？
大约 **7 天**，过期后会报 token 刷新错误，重新提取 Cookie 即可。

### Q：AI 回复的议价价格不准确？
注意：AI 只能在文字上"同意"某个价格，**不会自动修改闲鱼商品价格**。买家仍需按挂牌价下单，成交后你再手动协商/改价。

---

## 注意事项

- **Cookie 安全**：不要将 `.env` 文件提交到 git，已在 `.gitignore` 中排除
- **账号风险**：本项目为个人学习使用，大量刷消息可能触发风控
- **免责声明**：AI 回复内容仅供参考，交易决策请自行判断

---

## 技术架构

```
买家发消息
    ↓
WebSocket（main.py）接收
    ↓
意图识别（ClassifyAgent）
    ├── 议价 → PriceAgent（动态温度 + 价格规则）
    ├── 技术咨询 → TechAgent
    └── 其他 → DefaultAgent
    ↓
调用 Claude API（XianyuAgent.py）
    ↓
安全过滤（屏蔽站外联系方式）
    ↓
通过 WebSocket 发送回复
```
