# LED Display Social Media Customer Acquisition

## 任务定义

为深圳迈彩视觉有限公司（LED显示屏制造商）在 Instagram / Facebook 上开发海外 B2B 客户。
**当前目标市场：USA，目标 500 个客户（采集 + 触达同步进行）**。

**工作目录**：`C:\Users\Administrator\ai-topic-generator\output\leads\`
**Excel 记录表**：`C:\Users\Administrator\ai-topic-generator\output\leads\LED_Display_Leads_v17.xlsx`

---

## 核心目标

| 指标 | 目标 |
|------|------|
| 美国客户总数 | 500（采集 + 触达） |
| 当前已采集 USA | ~40（持续增加） |
| Warm-up 完成 | 每批次采集后立即安排 |
| DM 发送 | Warm-up 后 24h 开始 |

**边采集边触达**：不等名单凑满 500 再行动。每采集 10-20 个新客户，立即启动 warmup；warmup 完 24h 后立即发 DM。

---

## 每日工作流程

```
Step 1 — 采集新 USA 客户:
  opencli instagram search "LED display USA" --limit 10
  opencli instagram profile "username"
  → 通过 dedup 检查后写入 pipeline/instagram/prospects.json

Step 2 — Warm-up（可与采集并行）:
  python daily_runner.py warmup
  → opencli 自动点赞3条 + 评论1条 + 关注目标账号
  → 每个账号用 subagent 并行跑，加速批量预热

Step 3 — 等待 24 小时

Step 4 — 发送 DM:
  python daily_runner.py send
  → claude -p 生成个性化消息 → QA检查 → 打开对方主页 → 人工粘贴发送 → 回车确认

Step 5 — 更新 Excel:
  → 每完成一批 warmup/send，更新 LED_Display_Leads_v17.xlsx 追踪列
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
├── pipeline_init.py           # 一次性初始化：从474条leads导入pipeline
├── daily_runner.py            # 主入口：plan / warmup / send / status
├── message_crafter.py         # 用 claude -p 生成个性化消息
├── qa_checker.py              # 发送前9项合规检查
├── LED_Display_Leads_v17.xlsx # 客户总台账（含7个外拓追踪列）
└── pipeline/
    ├── instagram/
    │   ├── prospects.json     # 待触达队列（status=prospect）
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

## Excel 追踪（LED_Display_Leads_v17.xlsx）

表格 "LED Leads - All Markets" 包含 7 个外拓追踪列（最右侧）：

| 列名 | 说明 |
|------|------|
| Warmup Done | TRUE/FALSE |
| Warmup Date | 日期 YYYY-MM-DD |
| DM Sent Date | 发送日期 |
| DM Platform | instagram / facebook / whatsapp |
| DM Message | 消息前 100 字预览 |
| Touch Count | 触达次数（最多 2） |
| Lead Status | prospect / warmed / messaged / hot / cold |

**更新时机**：每批 warmup / send 完成后立即更新，按 Instagram 用户名或公司名匹配行。

```python
# 更新示例（openpyxl）
from openpyxl import load_workbook
wb = load_workbook("LED_Display_Leads_v17.xlsx")
ws = wb["LED Leads - All Markets"]
# 找到目标行，更新 Warmup Done / Warmup Date 列
wb.save("LED_Display_Leads_v17.xlsx")
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
# 等待 3 分钟（用 Python，不用 shell sleep）
python -c "import time; time.sleep(180)"
opencli instagram like "username" --index 2
python -c "import time; time.sleep(180)"
opencli instagram like "username" --index 3
python -c "import time; time.sleep(120)"
opencli instagram comment "username" "专业评论" --index 1
opencli instagram follow "username"

# Facebook 预热
opencli facebook add-friend "username"  # 个人账号
# Page账号无需预热，直接标记 warmup_done=True，可直接发消息
```

**关键**：subagent 中必须用 `python -c "import time; time.sleep(N)"` 而非 shell `sleep`——shell sleep 在沙盒被跳过会导致点赞过密触发 Instagram 限制。

### 并行 Warmup（推荐方式）

每个账号启动独立 subagent（run_in_background=True），所有账号同时处理：

```python
# 主 agent 为每个 USA 账号各启动一个 subagent
Agent(description="Warmup @username", prompt="...", run_in_background=True)
```

每个 subagent 完成后自动更新 prospects.json 中该账号的 warmup_done/status。

---

## 新客户采集（USA 专项）

### 搜索关键词（Instagram）

```bash
opencli instagram search "LED display USA" --limit 10
opencli instagram search "LED video wall rental" --limit 10
opencli instagram search "LED screen rental USA" --limit 10
opencli instagram search "LED signage company USA" --limit 10
opencli instagram search "LED wall events" --limit 10
opencli instagram search "church LED screen" --limit 10
opencli instagram search "LED display installer" --limit 10
opencli instagram search "outdoor LED billboard" --limit 10
opencli instagram search "fine pitch LED" --limit 10
opencli instagram search "LED video wall supplier" --limit 10
```

### 筛选标准

| 条件 | 要求 |
|------|------|
| 业务相关 | LED屏/视频墙/数字标牌/AV租赁/集成商 |
| 美国确认 | bio有城市/美国电话+1/美国网站 |
| 账号活跃 | 粉丝≥50 或有联系方式 |
| 不重复 | dedup 检查通过 |
| 非制造商 | 不收录中国LED制造商 |

### Dedup 检查

```python
python -c "
import json, sys
sys.stdout.reconfigure(encoding='utf-8')
base = r'C:\Users\Administrator\ai-topic-generator\output\leads\pipeline'
usernames = set()
for plat in ['instagram', 'facebook']:
    for f in ['prospects', 'warmed', 'messaged']:
        try:
            data = json.load(open(f'{base}/{plat}/{f}.json', encoding='utf-8'))
            for p in data:
                usernames.add(p.get('username', '').lower())
        except: pass
print('EXISTING:', sorted(usernames))
"
```

### 新条目格式（写入 prospects.json）

```json
{
  "username": "username",
  "platform": "instagram",
  "company_en": "Company Name",
  "country": "USA",
  "region": "Americas",
  "city": "City",
  "phone_whatsapp": "",
  "email": "",
  "website": "",
  "business": "Brief description. Instagram @username (N followers)",
  "status": "prospect",
  "warmup_done": false,
  "touch_count": 0,
  "added_date": "YYYY-MM-DD"
}
```

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

## 当前数据状态（2026-05-08）

| 国家 | Instagram | Facebook | 目标 | 缺口 |
|------|-----------|---------|------|------|
| **USA** | **~40+** | **2** | **500** | **~458** |
| Korea | 132 | — | — | — |
| Brazil | 70 | — | — | — |
| 其他 | — | — | — | — |

**优先级**：集中火力攻 USA，其他市场暂停采集。

leads 来源：`generate_led_leads_v4.py` ～ `v17.py`（链式结构）+ 新增直接写入 prospects.json

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
