---
name: newspaper_download
description: 报纸/杂志 PDF 下载工具。基于 pick-read.vip 接口查询已收录的报刊更新，定位指定期次，获取 PDF 下载链接。查询不鉴权，下载需要 Import Token。Newspaper/magazine PDF download tool. Query collected issues via pick-read.vip API, locate specific issues, and get PDF download links. Queries are unauthenticated; downloads require Import Token.
metadata:
  {
    "openclaw": {
      "requires": {
        "bins": ["python3"]
      }
    }
  }
---

# 报刊 PDF 下载工具

## 零、快速配置（首次使用必读）

编辑工具目录下的 `config.json`，填入你的导入令牌后保存，之后所有命令无需再传 `--token` 参数：

```json
{
    "api_base": "https://pick-read.vip/api",
    "import_token": "imp-你的令牌填这里"
}
```

**令牌在哪里获取？** 登录 [pick-read.vip](https://pick-read.vip) → 账户页 → 导入令牌 → 点击「生成令牌」，复制后粘贴到上面的 `import_token` 字段。

**配置优先级**（从高到低）：命令行 `--token` 参数 > `IMPORT_TOKEN` 环境变量 > `config.json` > 内置默认值

---

## 一、三个核心能力

| 用户问题 | 推荐接口 | CLI 命令 |
|---|---|---|
| 今天更新了什么报刊 | `list_recent_updates()` | `updates` |
| 帮我找某天某份报纸 | `get_issue_info()` | `issue-info` |
| 给我今天的下载链接 | `get_download_links()` | `download-links` |

辅助接口：`resolve` — 按报纸名+日期定位 issue_id（底层辅助，一般不需要直接调用）

## 二、接口详情

### 1. `updates` / `list_recent_updates(issue_date=None, days=None, limit=None)`
- 作用：查询今天/近期更新了哪些报刊
- 返回：`type`、`issue_date`、`days`、`total`、`items[]`

### 2. `issue-info` / `get_issue_info(pub_name, issue_date=None, token=None)`
- 作用：按报纸名+日期定位某一期，返回基础信息+下载链接
- 返回：`type`、`matched`、`issue_id`、`pub_name`、`issue_date`、`page_count`、`download_url`
- 无 token 时 `download_url` 为 `null`，附 `note` 提示

### 3. `download-links` / `get_download_links(days=1, issue_date=None, pub_name=None, token=None, limit=None)`
- 作用：批量列出近期新增期次的 PDF 下载链接
- 返回：`type`、`has_token`、`total`、`items[]`
- 每个 item 含：`issue_id`、`pub_name`、`issue_date`、`page_count`、`download_url`
- 下载链接格式：`https://pick-read.vip/api/import-pdf/{issue_id}?token={your_import_token}`

### 4. `resolve` / `resolve_issue(pub_name, issue_date=None)`
- 作用：按报纸名+日期定位 issue（底层辅助）
- 返回：`matched`、`issue`

## 三、命令行调用

```bash
# 查看今天更新了什么
python3 {baseDir}/scripts/get_data.py updates --no-save

# 查看最近 3 天更新
python3 {baseDir}/scripts/get_data.py updates --days 3 --no-save

# 查询某一期的信息+下载链接
python3 {baseDir}/scripts/get_data.py issue-info "Financial Times" --issue-date 2026-03-27 --token imp-xxxx... --no-save

# 今天所有新增期次的下载链接
python3 {baseDir}/scripts/get_data.py download-links --token imp-xxxx... --no-save

# 指定日期 + 按刊物筛选
python3 {baseDir}/scripts/get_data.py download-links --issue-date 2026-03-31 --pub-name "Financial Times" --token imp-xxxx... --no-save

# 通过环境变量传令牌（推荐）
IMPORT_TOKEN=imp-xxxx... python3 {baseDir}/scripts/get_data.py download-links --days 2 --no-save

# 定位某期（底层辅助）
python3 {baseDir}/scripts/get_data.py resolve "Financial Times" --issue-date 2026-03-27 --no-save
```

## 四、Python 调用

```python
from scripts.get_data import list_recent_updates, get_issue_info, get_download_links

# 查看今天更新
updates = list_recent_updates(days=1)

# 查询某期信息+下载链接
info = get_issue_info("Financial Times", "2026-03-27", token="imp-xxxx...")
print(info["pub_name"], info["issue_date"], info["download_url"])

# 批量下载链接
links = get_download_links(days=1, token="imp-xxxx...")
for item in links["items"]:
    print(item["pub_name"], item["issue_date"], item["download_url"])
```

## 五、报纸名称匹配

支持模糊匹配与常见缩写：

| 输入 | 匹配结果 |
|---|---|
| `WSJ` | The Wall Street Journal |
| `NYT` | The New York Times |
| `FT` | Financial Times |
| `华盛顿邮报` | The Washington Post |
| `纽约时报` | The New York Times |
| `中国日报` | China Daily |

## 六、环境变量

| 变量 | 说明 | 默认 |
|---|---|---|
| `API_BASE` | API 地址 | `https://pick-read.vip/api` |
| `IMPORT_TOKEN` | 导入令牌 | 空 |

## 七、错误契约

```json
{
  "detail": {
    "error_code": "not_found",
    "message": "该期次不存在",
    "detail": "issue_id=xxx"
  }
}
```

| error_code | 含义 | HTTP 状态码 |
|---|---|---|
| `not_found` | 期次不存在 | 404 |
| `invalid_argument` | 参数不合法 | 422 |
| `ambiguous_match` | 多个匹配 | 409 |

## 八、行为规则

1. **查询不鉴权** — `updates`、`issue-info`（不含下载链接时）、`resolve` 均无需认证
2. **下载需鉴权** — 下载链接中的 token 由 Import Token 提供
3. **本工具只负责查询报刊更新与返回下载链接，不负责生成最终回答**
4. **无 token 时仍返回期次信息，但 download_url 为 null**

## 九、合规说明

- 检索失败时不得编造事实，应返回明确错误
- 输出应保持可追溯、可审计
