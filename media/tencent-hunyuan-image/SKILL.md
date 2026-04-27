---
name: tencent-hunyuan-image
description: 使用腾讯混元生图 API（aiart）生成 AI 图片。含 TC3-HMAC-SHA256 签名、SubmitTextToImageJob/QueryTextToImageJob 调用、免费额度说明。
tags: [tencent, hunyuan, image, aiart, cover, text-to-image]
version: 1.0
triggers:
  - "生成封面图"
  - "混元生图"
  - "腾讯云AI画图"
  - "公众号封面"
---

# 腾讯混元生图 API

## 适用场景
- 生成公众号封面图
- 任何需要 AI 文生图的场景
- 腾讯云账号已有 SecretId/SecretKey

## 免费额度

| 接口 | 免费次数 | 说明 |
|------|---------|------|
| 混元生图 3.0 | 50 次 | 质量最好，首选 |
| 混元生图 2.0 | 50 次 | 备选风格 |
| 混元生图极速版 | 50 次 | 快速出图 |

共 150 次免费，有效期 1 年。

## API 核心参数

### 服务发现（容易踩坑）

腾讯混元生图的 API **不是** `hunyuan.tencentcloudapi.com`，而是：

```
Service:  aiart
Host:     aiart.tencentcloudapi.com
Version:  2022-12-29
Action:   SubmitTextToImageJob / QueryTextToImageJob
```

### 认证方式

使用腾讯云标准 **TC3-HMAC-SHA256** 签名。需要 SecretId 和 SecretKey（从 [腾讯云 CAM](https://console.cloud.tencent.com/cam/capi) 获取）。

环境变量（存到 `~/.hermes/.env`）：
```
TENCENT_SECRET_ID=AKIDxxxxxxxx
TENCENT_SECRET_KEY=xxxxxxxx
```

## 调用流程

### Step 1: 提交生图任务

```python
import hashlib, hmac, datetime, json, urllib.request

secret_id = "你的 SecretId"
secret_key = "你的 SecretKey"
service = "aiart"
host = "aiart.tencentcloudapi.com"
action = "SubmitTextToImageJob"
version = "2022-12-29"
region = "ap-guangzhou"

# Prompt 建议英文（效果更好），但中文也可
payload = json.dumps({
    "Prompt": "Minimalist tech cover, glowing chip on dark blue, clean modern, 16:9",
    "Resolution": "1024:576",   # 16:9 比例
    "LogoAdd": 0,               # 0=不加水印
})

# --- TC3 签名（标准模板，复制即用）---
algorithm = "TC3-HMAC-SHA256"
timestamp = str(int(datetime.datetime.now().timestamp()))
date = datetime.datetime.utcnow().strftime("%Y-%m-%d")

ct = "application/json; charset=utf-8"
canonical_headers = f"content-type:{ct}\nhost:{host}\nx-tc-action:{action.lower()}\n"
signed_headers = "content-type;host;x-tc-action"
hashed_payload = hashlib.sha256(payload.encode("utf-8")).hexdigest()
canonical_request = f"POST\n/\n\n{canonical_headers}\n{signed_headers}\n{hashed_payload}"

credential_scope = f"{date}/{service}/tc3_request"
hashed_canonical = hashlib.sha256(canonical_request.encode("utf-8")).hexdigest()
string_to_sign = f"{algorithm}\n{timestamp}\n{credential_scope}\n{hashed_canonical}"

def sign(key, msg):
    return hmac.new(key, msg.encode("utf-8"), hashlib.sha256).digest()

secret_date = sign(("TC3" + secret_key).encode("utf-8"), date)
secret_service = sign(secret_date, service)
secret_signing = sign(secret_service, "tc3_request")
signature = hmac.new(secret_signing, string_to_sign.encode("utf-8"), hashlib.sha256).hexdigest()

authorization = f"{algorithm} Credential={secret_id}/{credential_scope}, SignedHeaders={signed_headers}, Signature={signature}"

# --- 发送请求 ---
req = urllib.request.Request(f"https://{host}", data=payload.encode("utf-8"))
for h, v in [
    ("Authorization", authorization), ("Content-Type", ct), ("Host", host),
    ("X-TC-Action", action), ("X-TC-Version", version),
    ("X-TC-Timestamp", timestamp), ("X-TC-Region", region)
]:
    req.add_header(h, v)

with urllib.request.urlopen(req, timeout=60) as resp:
    result = json.loads(resp.read())
    job_id = result["Response"]["JobId"]
    print(f"JobId: {job_id}")
```

### Step 2: 轮询结果 & 下载

```python
action = "QueryTextToImageJob"

for attempt in range(10):  # 最多等 50 秒
    time.sleep(5)
    
    payload = json.dumps({"JobId": job_id})
    # ... 重新签名（timestamp 会变，必须重新算）...
    
    result = json.loads(resp.read())["Response"]
    status = result.get("JobStatusCode", "")
    
    if status == "5":  # 成功
        for img_url in result.get("ResultImage", []):
            img_data = urllib.request.urlopen(img_url).read()
            with open("/tmp/cover.jpg", "wb") as f:
                f.write(img_data)
            print(f"✅ Downloaded: {len(img_data)} bytes")
        break
    elif status == "7":  # 失败
        print(f"Failed: {result.get('JobErrorMsg')}")
        break
```

### Step 3: 不同尺寸对比

| 用途 | Resolution | 说明 |
|------|-----------|------|
| 公众号封面 | `1024:576` | 16:9 |
| 方形配图 | `1024:1024` | 1:1 |
| 竖版海报 | `720:1280` | 9:16 |

### Prompt 技巧

- ✅ 英文 prompt 效果通常更好
- ✅ 加 `16:9 cinematic` 或 `minimalist` 等风格词
- ✅ `no text, no watermark` 避免生成多余文字
- ❌ 太复杂的中文描述有时不如简洁英文
- ❌ 不要用 `photorealistic`——混元在写实风格上不够好

## 费用

免费额度用完后按量计费。具体价格见 [腾讯云混元生图定价](https://buy.cloud.tencent.com/price/aiart)。

## 踩过的坑

1. **Service 不是 hunyuan**：试了 `hunyuan.tencentcloudapi.com` 报 `InvalidAction`，正确是 `aiart`
2. **Region 必传**：不加 `X-TC-Region` 报 `MissingParameter`
3. **签名每次都要重算**：timestamp 变了签名就变，轮询时不能复用
4. **图片 URL 24 小时过期**：下载到本地保存，不要只存 URL
