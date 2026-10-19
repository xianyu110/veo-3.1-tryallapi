# Google Veo 3.1 API 调用教程：用 TryAllAPI 做异步视频生成

> **Veo 3.1**（模型 ID：`veo_3_1` / 快档 `veo_3_1-fast`）是 Google 侧可见的视频生成模型。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场选型、读按次价格字段，到 `POST /v1/videos`（OpenAI video format）创建任务并轮询出片的 **异步视频生成 API** 接入流程。价格与分组倍率**以页面实时为准**；示例仅为请求形态说明，未跑通真实出片。

**Keywords:** Veo 3.1 · Google 视频 API · TryAllAPI · 异步视频生成 · `veo_3_1` · `/v1/videos`

---

## 目录

- [为什么用聚合站调视频模型](#一为什么用聚合站调视频模型)
- [在模型广场找到 Veo 3.1](#二在模型广场找到-veo-31)
- [模型怎么选：veo_3_1 vs fast vs components](#三模型怎么选veo_3_1-vs-fast-vs-components)
- [价格怎么读](#四价格怎么读)
- [调用流程：Key → POST /v1/videos → 轮询](#五调用流程key--post-v1videos--轮询)
- [curl 最小示例](#六curl-最小示例)
- [常见注意](#七常见注意)
- [FAQ](#faq)
- [小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调视频模型

视频生成几乎都是 **异步任务**：先 `POST` 创建任务拿任务 ID，再轮询直到状态成功，最后取出成片 URL。若业务同时要接文本、生图、多厂商视频，为每个厂商维护一套 Key、签名与重试逻辑会很散。

聚合站（如 [TryAllAPI](https://tryallapi.com/)）的常见价值是：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可试不同档位；
2. **异步形态可对照**：广场标明「异步 / 视频」标签与支持的 endpoint 类型；
3. **模型广场可对照**：标准档 / fast / components 同屏可见，便于降级。

下面以广场可见的 **`veo_3_1`** 为主线，并对照 `veo_3_1-fast`。注册试用：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在模型广场找到 Veo 3.1

打开 [价格页](https://tryallapi.com/pricing) / 模型广场，侧栏按供应商 **Google**、类型 **音视频**、标签 **视频 / 异步 / 首尾帧** 筛选。详情卡要点（公开定价接口快照，**截图待补**）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `veo_3_1` |
| **供应商** | Google |
| **描述 gist** | 高质量模式；可伴随音频；支持首尾帧控制与文生视频；约 8 秒片段 |
| **标签** | 异步、视频、首尾帧 |
| **API（广场 endpoint 类型）** | OpenAI video format → `POST /v1/videos` |
| **计费** | 按次（`quota_type=1`）；分组 **Discounted-Gemini-2** |

同屏还能看到：`veo_3_1-fast`、`veo_3_1-components`。文中 UI 品牌可能为 momoAPIPro / 模型广场，实际对接以 [TryAllAPI](https://tryallapi.com/) 为准。

---

## 三、模型怎么选：veo_3_1 vs fast vs components

只根据广场可见信息做选型对照，**不编造未出现的能力差异**：

| 模型 ID（可见） | 怎么理解（接入视角） |
| --- | --- |
| **`veo_3_1`** | 高质量主推卡；标签含首尾帧；endpoint 类型为 OpenAI video format |
| **`veo_3_1-fast`** | 同屏 fast 档；描述强调性价比 / 更快；同 endpoint 类型 |
| **`veo_3_1-components`** | 参考图（最多三张）驱动；标签含「参考生视频」 |

实务上：默认锁 `veo_3_1` 或 `veo_3_1-fast` 写进配置；容量不足时在同供应商可见 ID 间降级，尽量不改 path。

---

## 四、价格怎么读

按次计费模型在公开接口里用 **`model_price`** 字段（单位与最终展示以广场详情卡为准）。快照如下（**以页面实时为准**）：

| 模型 ID | model_price（快照） | 分组 | 分组倍率（快照） |
| --- | --- | --- | --- |
| **`veo_3_1`** | **0.768** | Discounted-Gemini-2 | 0.28722 |
| **`veo_3_1-fast`** | **0.576** | Discounted-Gemini-2 | 0.28722 |
| **`veo_3_1-components`** | **0.768** | Discounted-Gemini-2 | 0.28722 |

读表时注意：

1. 快照来自站点公开定价 JSON，**不是**替你算出的最终账单；控制台可能再乘分组倍率或另有展示价。
2. 描述写明约 **8 秒** 片段；时长 / 分辨率限额以详情卡为准。
3. 换分组或活动价后数字会变——下单前再看一眼价格页。

---

## 五、调用流程：Key → POST /v1/videos → 轮询

广场将该系列的 endpoint 类型标为 **OpenAI video format**，创建路径为：

```text
控制台创建 API Key（分组需覆盖 Discounted-Gemini-2 等可见组）
  → POST https://tryallapi.com/v1/videos   （创建，拿 id / task id）
  → 间隔轮询 GET …/v1/videos/{id}         （字段名以实际 JSON / 控制台为准）
  → status 成功后取成片 URL
  → 失败则读 error，按业务退款 / 重试
```

建议：

- **超时与退避**：视频渲染可能几十秒到数分钟，轮询间隔 2–5s，并设总超时；
- **幂等**：把任务 ID 落库，避免前端连点重复创建；
- **对照控制台**：若详情卡或文档给出独立查询 path / Webhook，**以控制台为准**，勿死记本文骨架。

---

## 六、curl 最小示例

以下为 OpenAI Videos 风格的最小骨架：Bearer 鉴权、模型 ID 与广场一致、占位 Key `sk-xxx`。请求体字段名若与控制台示例不一致，**以控制台 / 模型广场详情卡为准**。

### 1）创建任务（POST）

```bash
curl -sS -X POST 'https://tryallapi.com/v1/videos' \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "veo_3_1",
    "prompt": "A quiet seaside morning, soft light, gentle camera push-in, 8s."
  }'
```

快档可将 `"model"` 换成 `"veo_3_1-fast"`。若控制台示例还要求 `size` / `seconds` / 首尾帧图片字段，请按详情卡补全——本文不编造未在广场出现的字段名。

成功响应里应能拿到任务 ID（常见 `id` / `task_id`，以实际 JSON 为准）。

### 2）轮询任务（GET 骨架）

```bash
VIDEO_ID='替换为上一步返回的 id'

curl -sS -X GET \
  "https://tryallapi.com/v1/videos/${VIDEO_ID}" \
  -H 'Authorization: Bearer sk-xxx'
```

当 `status` 表示完成时，从响应中的视频 URL 字段取出可下载地址（具体键名以实际 JSON / 控制台为准）。若控制台文档给出不同查询 path，优先文档。

---

## 七、常见注意

1. **异步**：不要把创建接口当同步渲染；前端应展示进度并轮询。
2. **fast vs 标准**：同 endpoint，换 `model` 即可；先小流量对比画质与扣费。
3. **首尾帧 / 参考图**：`veo_3_1` 与 `components` 标签不同；带参考前先读详情卡字段。
4. **分组**：快照挂在 Discounted-Gemini-2；403 / 无货时对照令牌分组。
5. **合规**：仅使用你有权使用的素材；成片标明 AI 生成；遵守平台与当地法规。

---

## FAQ

### Veo 3.1 的模型 ID 是什么？

广场可见主 ID 为 **`veo_3_1`**；快档 **`veo_3_1-fast`**；参考图档 **`veo_3_1-components`**。

### endpoint 是什么？

广场 endpoint 类型：**OpenAI video format** → 创建 `POST /v1/videos`。Base：`https://tryallapi.com`。轮询 path 建议对照控制台；下文 GET 骨架仅作 OpenAI Videos 习惯写法。

### 为什么要异步轮询？

渲染通常需要数十秒到数分钟。创建接口只返回任务 ID，需轮询到成功后再取成片 URL。

### 如何试用 TryAllAPI？

首页：[https://tryallapi.com/](https://tryallapi.com/)；注册：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 Veo 3.1 接到业务里，核心是：**在广场确认 model id 与 OpenAI video format → 读懂按次 `model_price` 与分组 → 用 POST/GET 任务协议出片**。`veo_3_1` + `/v1/videos` 已足够搭通最小链路骨架。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [https://tryallapi.com/pricing](https://tryallapi.com/pricing) |

控制台 Token、定价与文档更新后，以线上为准。**截图待补**。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非 Google 官方产品。
