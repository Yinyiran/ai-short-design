# AI 漫剧创作小程序 · 统一版本后端实现方案

> **范围**：单一版本 = 漫剧创作链路（comic）+ 文生视频（t2v）+ 图生视频（i2v）+ 积分中心，共用一套后端底座。

---

## 一、技术栈与整体架构

| 层 | 选型 |
| --- | --- |
| 小程序端 | UniApp-X（微信小程序） |
| 后端 API | NestJS（monorepo，REST + JWT 双 token） |
| 管理后台 | Nest Admin（Vue3 + Element Plus） |
| 数据库 | MySQL 8（InnoDB，业务 + 账务全走事务） |
| 队列/缓存 | Redis + BullMQ |
| 对象存储 | 腾讯 COS（DB 只存 URL） |
| 第三方 AI | 多供应商统一网关（LLM/文生图/TTS/文生视频/图生视频） |
| 支付 | 微信支付 JSAPI v3（验签 + 幂等发货） |

**核心设计原则**：
1. **单任务模型**：`video_task` 一张表承载 comic/t2v/i2v 三类任务，`type` 区分，对外统一 `taskId` 轮询。
2. **同步只接单、异步干重活**：所有 AI 链路「提交 → 入队 → 立即返回 202」。
3. **两条队列**：按"时效性 + 成本"拆分，不按功能拆 5 条，避免过度设计。

```
UniApp-X ──HTTPS──▶ NestJS API（Auth/Guard/统一异常）
                        │
        ┌───────────────┼──────────────────┐
   ComicController  VideoController   Points/Pay/Distribution
        │               │                    │
        └─────┬─────────┘                    │
          AsyncService（入队前额度校验/积分冻结）
              │ addJob                        │
              ▼                               ▼
   Redis(BullMQ) ──▶ Worker 编排层 ──▶ 多供应商 AI 网关 ──▶ COS
   comic-pipeline / video-generate            │
              │                    ┌──────────┤
              ▼                    ▼          ▼
   MySQL(video_task)      Redis(task:progress)   微信内容安全/审核
```

---

## 二、核心时序（三条链路）

### 2.1 文生漫剧（comic）—— 最复杂链路

```
POST /comic/generate
  → ① 校验分镜/风格/余额
  → ② 积分预扣（冻结，原子 UPDATE）
  → ③ 创建 video_task(pending)
  → ④ addJob('comic-pipeline') → 返回 202 {taskId}
前端 GET /task/:id 轮询（读 Redis 进度，miss 回源 DB）
Worker（单 job 状态机串行推进）：
  script  → LLM 剧本转分镜 JSON（JSON mode + 校验 + 正则兜底）
  image   → 逐镜文生图（带 refImages 角色参考图 + 固定 seed）
  tts     → 逐镜台词 TTS（角色/旁白分音色，喵汪拟声预处理）
  compose → 逐镜图生视频片段（供应商异步，二级轮询）
  merge   → ffmpeg 拼接 + 转场 + BGM + AI 标识烧录
  done    → 写 result、生成封面、确认扣费 → success + videoUrl
```

### 2.2 文生视频（t2v）/ 图生视频（i2v）—— 单任务链路

```
POST /video/generate 或 /image2video
  → 校验参数/风格/余额 → 积分预扣 → 建 video_task → addJob('video-generate') → 202
Worker：
  submit   → 组装参数（prompt + 首帧图 + 风格/时长/比例）→ 供应商 /create → 拿供应商 taskId
  polling  → 每 10s 轮询供应商状态（供应商异步长任务 2–10 分钟）
  download → 取视频 URL 转存 COS（防供应商 URL 过期）
  done     → 回写 result（videoUrl/coverUrl/duration）→ 确认扣费
```

> 视频供应商为异步长任务，Worker 内是"二级异步"：先提交拿 taskId，再轮询其状态。

---

## 三、Worker 内部编排

### 3.1 漫剧流水线（队列 `comic-pipeline`）

| step | 动作 | 失败策略 | 进度权重 |
| --- | --- | --- | --- |
| script | LLM 剧本→分镜 JSON | 解析容错 + 重试 2 次 | 5% |
| image | 逐镜文生图（refImages + seed） | 单镜重试 1 次 | 15%→45% |
| tts | 逐镜 TTS 配音 | 重试 1 次 | 45%→70% |
| compose | 逐镜图生视频（可选） | 见超时策略 | 70%→90% |
| merge | ffmpeg 拼接 + 转场 + BGM + 标识烧录 | 重试 1 次 | 90%→100% |
| done | 写 result、封面、确认扣费 | — | 100% |

- 进度 = 已完成分镜数/总分镜数 × 阶段权重，写 Redis，轮询零 DB 压力。
- 图→TTS→合成必须串行，错序浪费 AI 调用。

### 3.2 文生/图生视频单任务（队列 `video-generate`）

| 阶段 | progress | 说明 |
| --- | --- | --- |
| submit | 5% | 调供应商 /create，返回供应商 taskId |
| polling | 5%→90% | 每 10s 轮询；**视频任务 TTL 30 分钟**，超时标记 failed_timeout |
| download | 95% | 下载转存 COS |
| done | 100% | 回写结果，确认扣费 |

- 供应商多数只回 RUNNING → 用"轮询次数 + 估算"平滑递增（每次 +2%，90% 封顶）；支持 webhook 则以其为准 + 轮询兜底。
- 超时/取消调用 `internal/ai/video/cancel` 尽量取消供应商任务省钱。

---

## 四、关键接口定义

### 4.1 创作入口（均返回 202）

| 接口 | 说明 |
| --- | --- |
| `POST /api/v1/comic/generate` | `{storyboardId, styleId, voiceConfig, bgmUrl, ratio, generateVideo}` |
| `POST /api/v1/video/generate` | 文生视频 `{prompt, styleId, durationSec, ratio, motion, negativePrompt}` |
| `POST /api/v1/image2video` | 图生视频 `{imageUrl, prompt, styleId, durationSec, motion}` |

### 4.2 进度轮询

`GET /api/v1/task/:taskId`
```json
{ "taskId": "tk_8f3a", "type": "comic",          // comic | t2v | i2v
  "status": "processing",                         // pending|processing|success|failed|failed_timeout
  "stage": "image",                               // script|image|tts|compose|polling|merge
  "progress": 45, "costPoints": 60, "result": null, "error": null }
// success 时 result: { videoUrl, coverUrl, duration, aiMarked: true, shots:[...] }
```
- 优先读 Redis `task:progress:{taskId}`，miss 回源 DB 并回填；前端轮询 2s/次。

### 4.3 积分中心

| 接口 | 说明 |
| --- | --- |
| `GET /api/v1/points/balance` | `{balance, frozen}` |
| `GET /api/v1/points/transactions` | 流水明细（buy/exchange/gift/consume/refund） |
| `POST /api/v1/points/purchase` | 创建积分购买订单 → 微信支付 |
| `POST /api/v1/points/exchange` | 积分换次数包/会员 |
| `POST /api/v1/points/gift` | 转赠（事务双写，防超发） |
| `GET /api/v1/points/rules` | 兑换比例/上限/消耗单价（Admin 可配） |

### 4.4 内部 AI 接口（仅内网，Worker 经网关调用）

| 接口 | 说明 |
| --- | --- |
| `POST /internal/ai/llm/script` | 剧本→分镜 JSON（structured output） |
| `POST /internal/ai/image` | 文生图 `{prompt, refImages[], negativePrompt, ratio, seed}` |
| `POST /internal/ai/tts` | `{text, voiceId, speed}` |
| `POST /internal/ai/video/create` | 文生/图生视频提交，返回供应商 taskId |
| `GET /internal/ai/video/:supplierTaskId` | 轮询供应商进度 |
| `POST /internal/ai/video/cancel` | 取消供应商任务 |

---

## 五、核心数据表

```sql
-- 统一任务表（三类任务一张表）
video_task(
  id, task_id UNIQUE, user_id, type(comic|t2v|i2v), status, stage, progress,
  style_id, params JSON,           -- prompt/voiceConfig/ratio/seed/refImages
  supplier, supplier_task_id,      -- 供应商与供应商侧任务ID
  points_frozen INT, points_actual INT,   -- 冻结/实际扣减（对账依据）
  result JSON,                     -- {videoUrl, coverUrl, duration, shots, aiMarked}
  error, retry_count, created_at, updated_at, completed_at
  KEY(user_id,status), KEY(created_at)
)

storyboard(user_id, title, episodes, dialogue_mode, status)     -- 剧本
storyboard_shot(storyboard_id, shot_no, scene, action, dialogue,
                camera, character_asset_ids JSON, image_url, audio_url, prompt, seed)

materials(owner_id, type(character|scene|prop), image_url, is_official, status)  -- 素材库
video_styles(name, type(official|custom), params_json, cover_url, status, price_points) -- 风格
comic_work(user_id, task_id, title, video_url, cover_url, duration,
           is_public, ai_marked, status)                        -- 成品作品

-- 积分体系（一致性核心）
points_account(user_id PK, balance INT, frozen INT, version INT)  -- 乐观锁
points_transaction(user_id, biz_type(buy/exchange/gift/consume/freeze/unfreeze/admin_adjust),
                   amount, balance_after, ref_type, ref_id,
                   UNIQUE(ref_type, ref_id, biz_type))            -- 幂等防重放
points_rule(rule_key UNIQUE, rule_value, status)                  -- 可配置规则

orders(out_trade_no UNIQUE, user_id, type(vip|points|package), amount, status, wx_trade_no)
user_vips(user_id, package_id, start_at, end_at, status)
credit_packages / distribution_relations(level≤2) / commission_records / withdrawals
admin_users / roles / permissions / role_permissions / admin_audit_log
system_configs / ad_configs / ai_usage_stats
```

---

## 六、积分扣减一致性（两阶段冻结）

```
① 入队前（API 线程，事务）：
   UPDATE points_account SET frozen = frozen + :cost
   WHERE user_id=:uid AND balance >= :cost AND frozen + :cost <= :ceiling;  -- 原子防超发
   affected=0 → 拒绝（引导充值）；写流水 freeze（幂等键）
② 成功（Worker done，事务）：balance -= cost, frozen -= cost；流水 → finished（幂等）
③ 失败/超时/取消（事务）：仅解冻 frozen -= cost；流水 → unfreeze
```
- **防超发**：`balance >= cost` 写在单条 UPDATE（InnoDB 行锁串行）；**防重放**：`uk_biz` 唯一键；**回退**：失败只解冻不扣 balance。

---

## 七、异步基础设施

| 队列 | 用途 | attempts | TTL | 并发 |
| --- | --- | --- | --- | --- |
| `comic-pipeline` | 漫剧整条流水线（单 job 状态机） | 3（指数退避） | 600s | 2–4 |
| `video-generate` | 文生/图生视频单任务 | 2 | **1800s** | 按供应商配额 |
| `internal-ai-callback` | 供应商 webhook 回调 | 5 | 60s | 高 |
| `cron` | 对账/过期清理/会员降级 | 1 | — | — |

**错误分级**：`RETRYABLE`（超时/5xx/限流）→ 走队列重试；`FATAL`（参数错/内容违规）→ 立即 failed 不烧重试费。

**成本护栏分层**：
- 用户层：积分预扣 + 单用户并发上限（漫剧 2、视频 3）+ 免费额度/每日次数
- 供应商层：平台日预算熔断（超限排队）+ QPS 令牌桶
- 平台层：队列积压监控，积压超阈值动态扩容 Worker 或降级为"排队中"

---

## 八、第三方 AI 网关

统一 `AiProvider` 接口（health/call/submit/poll）+ Provider 注册表 + 降级链：

| 能力 | 主供应商 | 降级链 | 要点 |
| --- | --- | --- | --- |
| LLM 剧本/改写 | 豆包/混元 | DeepSeek → 通义 | JSON mode + schema 校验 |
| 文生图 | 即梦/万相 | SD 自托管 | refImages 图生图保一致性 + 固定 seed |
| TTS | 火山 | 腾讯云 TTS | 多说话人，喵汪拟声预处理 |
| 文生视频 | Seedance | 可灵 → CogVideo | 异步 submit/poll，参数映射 |
| 图生视频 | Seedance | 可灵 → 其他 | 首帧图 + prompt + motion |

- 轮询超时/熔断 → 顺延下一家重试 1 次；成功率/单价写入 `ai_usage_stats` 供看板动态调主备。
- 视频类提交前做成本估算并返回 `costPoints`，扣费按实际用量。

---

## 九、支付 / 会员 / 分销 / 合规

**微信支付**：`POST /order/create` → 建订单（out_trade_no 唯一）→ JSAPI 下单 → 回调验签 → 幂等发货（积分/会员到账）→ 上报发货信息 → 触发分销佣金结算；每日对账任务比对微信流水；管理员退款扣回并记流水。

**会员/次数包**：`user_vips` + 权益表 + VipGuard 守卫高级功能；定时任务到期降级；次数包与积分双轨（次数优先扣）。

**分销**：注册写 `distribution_relations`（代码强制 level≤2、防自邀防环）；支付发货成功同事务结算佣金；提现审核流 + 实名信息。

**合规（硬性）**：
- ffmpeg 合成阶段烧录「AI 生成」角标 + 元数据标识；`comic_work.ai_marked=1` 才可公开
- 生成前 prompt 过微信内容安全 API，生成后图片/视频走天御/人工审核，先审后上架
- 全链路请求日志 + AI 调用日志 + 管理操作审计；备案资质由运营方办理

---

## 十、关键风险与对策

| # | 风险 | 对策 |
| --- | --- | --- |
| 1 | 视频生成成本高（单条约 1–5 元） | 积分两阶段冻结 + 失败只解冻；并发限流；平台日预算熔断；多供应商比价动态选主 |
| 2 | 任务积压/长尾 | 双队列拆分 + 视频进度估算平滑 + 积压监控动态扩 Worker + 超时自动取消 |
| 3 | 积分一致性（超发/重放/不回退） | 原子 UPDATE 扣减 + uk_biz 幂等键 + 两阶段冻结 + 每日对账 |
| 4 | 角色跨镜一致性 | 先 POC 锁供应商参数；允许选参考帧人工介入；预留 LoRA 路径 |
| 5 | 供应商不稳定 | 统一网关 + 熔断 + 降级链 + webhook/轮询双通道 + 成功率进统计动态调主备 |
