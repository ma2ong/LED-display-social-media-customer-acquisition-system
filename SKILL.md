# LED Display Social Media Customer Acquisition

## 任务定义

为深圳迈彩视觉有限公司（LED显示屏制造商）在 Instagram / Facebook 上开发海外 B2B 客户。
当前目标市场：**USA**（可在 `daily_runner.py` 的 `TARGET_COUNTRIES` 修改）。

**工作目录**：`C:\Users\Administrator\ai-topic-generator\output\leads\`

---

## 每日工作流程

```
Step 1（今天）: python daily_runner.py warmup
               → opencli 自动点赞3条 + 评论1条 + 关注目标账号

Step 2: 等待 24 小时

Step 3（明天）: python daily_runner.py send
               → claude -p 生成个性化消息 → QA检查 → 打开对方主页 → 人工粘贴发送 → 回车确认
```

### 辅助命令

```bash
python daily_runner.py plan    # 预览今日队列（不执行）
python daily_runner.py status  # 查看今日进度
python daily_runner.py send --force  # 跳过24h等待（调试用）
```

---

## 文件结构

```
output/leads/
├── pipeline_init.py       # 一次性初始化：从474条leads导入pipeline
├── daily_runner.py        # 主入口：plan / warmup / send / status
├── message_crafter.py     # 用 claude -p 生成个性化消息
├── qa_checker.py          # 发送前9项合规检查
└── pipeline/
    ├── instagram/
    │   ├── prospects.json   # 待触达队列（status=prospect）
    │   ├── warmed.json
    │   ├── messaged.json
    │   └── replies.json
    ├── facebook/
    │   └── (同上)
    ├── blacklist.json
    ├── hot_leads.json
    └── daily_log_YYYYMMDD.json
```

---

## 关键配置（daily_runner.py）

```python
TARGET_COUNTRIES = ["USA"]        # 目标国家，空列表=不过滤
DAILY_TARGETS = {
    "instagram": random.randint(5, 8),   # 每日IG发送量（上限8）
    "facebook":  random.randint(3, 5),   # 每日FB发送量（上限5）
}
```

---

## 合规红线（绝不超越）

| 平台 | 每日上限 | 最小间隔 |
|------|---------|---------|
| Instagram | 8条 DM | 20分钟 |
| Facebook | 5条 DM | 30分钟 |
| WhatsApp | 5条 | 40分钟 |
| 三平台合计 | 15人/天 | — |

- 同一人最多触达2次（首次 + 跟进，间隔≥72小时）
- 新账号前14天只做内容互动，不发私信
- 仅在目标时区 Tue-Thu 10:00-16:00 发送

---

## Warm-up 操作（opencli）

```bash
# Instagram 预热序列（点赞需间隔2-5分钟）
opencli instagram like "username" --index 1
opencli instagram like "username" --index 2
opencli instagram like "username" --index 3
opencli instagram comment "username" "专业评论" --index 1
opencli instagram follow "username"

# Facebook 预热
opencli facebook add-friend "username"  # 个人账号
# Page账号无需预热，可直接发消息
```

**注意**：使用 subagent 并行跑时，sleep 命令在沙盒中被跳过，会导致点赞过于密集触发 Instagram 限制。subagent 适合单步操作，不适合带延迟的多步序列。

---

## 消息生成（message_crafter.py）

通过 `claude -p` 调用，使用 Anthropic 订阅账号，无需 API Key。

```python
from message_crafter import craft_message, craft_followup, craft_reply_response

msg = craft_message(prospect_dict, "instagram")  # 40-60词
msg = craft_message(prospect_dict, "facebook")   # 50-80词
msg = craft_message(prospect_dict, "whatsapp")   # 30-50词

followup = craft_followup(prospect, "instagram", original_message)
reply = craft_reply_response(prospect, "instagram", their_reply, "hot")  # hot/warm/cold
```

**角色设定**：Allen Ma，LED 行业从业者，同行交流口吻，非工厂推销。
**禁止词**：leading manufacturer / best price / factory direct / I hope this message finds you well / dear sir

---

## QA 检查（qa_checker.py）

```python
from qa_checker import QAChecker, log_send

checker = QAChecker("instagram")
result = checker.check(prospect, message)
if result.passed:
    # 发送
    log_send("instagram", prospect, message)
```

9项检查：专属信息 / 每日限额 / 时区窗口(Tue-Thu 10-16) / 重复度<30% / blacklist / 间隔 / 字数 / 禁词 / 非中国账号

---

## 当前数据状态

| 国家 | Instagram | Facebook |
|------|-----------|---------|
| USA | 20 | 2 |
| Korea | 132 | — |
| Brazil | 70 | — |
| 合计 | 144 | 32 |

leads 来源：`generate_led_leads_v4.py` ～ `v17.py`（链式结构，每版只存增量）

---

## 依赖

```bash
npm install -g @jackwener/opencli   # v1.7.14+
# Python: 无需额外安装（使用 claude -p 替代 anthropic SDK）
```

Chrome 需登录 Instagram 和 Facebook（opencli 使用 cookie 策略）。

---

## 回复处理

| 回复类型 | 处理方式 | 标记 |
|---------|---------|------|
| 询价/规格/catalog | Claude生成回复 + 提议WhatsApp/Zoom | hot_lead |
| "send more info" | 发PDF/报价单 + 一个追问 | warm_lead |
| 消极/已读不回 | 不再触达 | cold → blacklist |
| 要视频通话/详细报价 | Allen 接手 | — |
| 订单预估 >$5000 | Allen 直接沟通 | — |
