# WorkBuddy 每日自动签到（GitHub Actions 云端版）

参考 [appcctv/wnflb-checkin](https://github.com/appcctv/wnflb-checkin) 的思路：把无头签到脚本放到 GitHub Actions，
**无需开机**，每天定时自动签到 + 晚间复查补签，可选微信推送通知。

## 文件结构

```
workbuddy签到工具/
├── .github/workflows/
│   └── checkin.yml          # GitHub Actions 定时任务（09:00 / 22:00 北京时间）
├── checkin.py               # 无头签到核心脚本（由 green_checkin.py 而来）
├── requirements.txt         # Python 依赖（requests）
└── README.md
```

> `签到工具.exe` 是有界面的版本（需要手动点按钮），**不能**直接被定时任务驱动；
> 云端自动化用的是无头的 `checkin.py`（即 `green_checkin.py` 的云端适配版）。

## ⚠️ 安全（重要）
- `accounts.json` / `签到token.json` / `签到工具.exe` 含真实 token，**已被 .gitignore 忽略，切勿提交到仓库**。
- 云端只读 GitHub Secret `WORKBUDDY_ACCOUNTS`，不依赖本地文件；本地文件仅用于本机调试。
- 仓库务必设为 **Private**。

## 本地调试
```bash
pip install requests
# 方式一：直接读本地 accounts.json（同目录）
python checkin.py
# 方式二：用环境变量模拟云端（推荐，验证 Secret 路径）
WORKBUDDY_ACCOUNTS="$(cat 签到token.json)" python checkin.py
```
已用真实 token 实测：接口 `POST /v2/billing/meter/daily-checkin` 鉴权有效，
返回 `{"code":10001,"msg":"今天已签到，请明天再来"}` 正确判定为当日已签。

## 部署步骤

### 1. 准备 checkin.py
把无头签到逻辑放进 `checkin.py`：读取 `WORKBUDDY_ACCOUNTS` 环境变量（JSON，含 `accounts` 数组，
每项 `{name, token}`），用 JWT 作为鉴权逐个账号签到。

### 2. 创建私有仓库并上传
1. GitHub 新建 **Private** 仓库（Cookie/token 是敏感信息）。
2. 把本目录内容 `git push` 上去（含 `checkin.py`、`.github/workflows/checkin.yml`、`requirements.txt`）。

### 3. 配置 Secrets
仓库 → Settings → Secrets and variables → Actions → New repository secret：

| Name                  | Value                                            | 必填 |
|-----------------------|--------------------------------------------------|------|
| `WORKBUDDY_ACCOUNTS`  | 整个 `签到token.json`（或 `accounts.json`）的内容 | 必填 |
| `PUSHPLUS_TOKEN`      | PushPlus token（微信推送，见下）                 | 可选 |
| `SERVERCHAN_KEY`      | Server酱 SendKey（微信推送备选）                 | 可选 |

> 注意：`accounts.json` 里带 `_status`/`_detail` 字段，脚本只用到 `name`+`token`，不影响。

### 4. 测试运行
Actions → 左侧 Workflow → Run workflow → 看日志是否有 `签到成功` / `今日已签到`。

## 微信推送（可选）
- PushPlus：访问 www.pushplus.plus 扫码，复制 token → 存入 `PUSHPLUS_TOKEN`。
- Server酱：访问 sct.ftqq.com 扫码，复制 SendKey → 存入 `SERVERCHAN_KEY`。

## 定时说明
| cron (UTC) | 北京时间 | 动作           |
|------------|----------|----------------|
| `0 1 * * *`  | 09:00    | 早起签到       |
| `0 14 * * *` | 22:00    | 晚间复查补签   |

GitHub Actions 定时任务通常有 5~15 分钟延迟，不影响签到效果。

## 维护：Token 过期
从 `签到token.json` 的 JWT 看，`exp` 大约在 **2027-07-10 ~ 2027-07-14**（约一年有效期）。
过期后重新在 WorkBuddy 签到工具里导出 token → 更新 `WORKBUDDY_ACCOUNTS` Secret 即可。
