# config/ 目录说明

本目录是 IPTV-API 的全部配置。**改配置前先看这张表，别改错文件**——上游更新会覆盖 `config.ini` 和 `demo.txt`，你的定制要写在 `user_*` 里。

## 一、文件地图

| 文件 | 作用 | 该不该改 |
|---|---|---|
| `config.ini` | 上游默认配置（220 行，全量参数说明） | ❌ 不动。上游更新会覆盖 |
| `user_config.ini` | 你的配置覆盖层，**在 config.ini 之后加载，同键覆盖** | ✅ 所有参数调整写这里 |
| `demo.txt` | 上游默认频道模板（79 行） | ❌ 不动 |
| `user_demo.txt` | 你的频道模板（318 行 / 14 个分类） | ✅ 调整频道清单写这里 |
| `subscribe.txt` | 订阅源列表 + `[WHITELIST]` 段 | ✅ 增删订阅源 |
| `whitelist.txt` | **流地址**白名单（不是订阅源！） | ⚠️ 一般不用动 |
| `blacklist.txt` | 流地址关键字黑名单（当前含 `av3a`） | ✅ 要屏蔽某类流时加 |
| `epg.txt` | 节目单源，最多生效 3 个 | ✅ 节目单失效时换源 |
| `local.txt` | 本地源，格式 `频道名称,接口地址` | 有自建源时才用 |
| `alias.txt` | 频道别名映射（2798 行，上游维护） | ❌ 不动 |
| `logo/` `local/` | 台标文件、本地源文件 | 按需 |

**加载顺序**：`config.ini` → `user_config.ini`（后者赢）。所以想改任何参数，加在 `user_config.ini` 的 `[Settings]` 里，不要改 `config.ini`。

## 二、私人源放哪里 —— 只放一个地方

```
config/subscribe.txt  →  文件末尾的 [WHITELIST] 段
```

三个容易搞混的概念，先分清：

| 概念 | 是什么 | 配在哪个文件 |
|---|---|---|
| **订阅源** | 一个 m3u/txt 地址，里面装着成百上千个频道 | `subscribe.txt` |
| **流地址 / 接口** | 单个频道的播放地址，形如 `http://x.com/cctv1.m3u8` | `whitelist.txt` / `blacklist.txt` |
| **频道模板** | 你要哪些频道、怎么分类命名 | `user_demo.txt` |

所以：

- ✅ **`subscribe.txt` 的 `[WHITELIST]` 段** —— 唯一正确位置。理由见下方。
- ❌ `subscribe.txt` 的普通列表 —— 会被测速和分辨率门槛刷掉。
- ❌ `whitelist.txt` 的 `[KEYWORDS]` —— 这里配的是流地址关键字，管不到订阅源。
  曾经误配过，结果 0 条入选，还让一次更新从 12 分钟拖到 38 分钟。

### 为什么私人源必须进 `[WHITELIST]` 段

私人源用的是**嵌套 m3u8 索引**：索引文件本身只有 1~2KB，里面再指向真实分片。测速器读这个 1KB 索引拿不到真实码率，会被 `min_speed` / `resolution_speed_map` 门槛全部过滤掉。

放进 `[WHITELIST]` 段后，该源抓到的接口 `origin` 会被标记为 `whitelist`，命中 `RETAIN_ORIGINS = {whitelist, hls}`，直接跳过速度与分辨率过滤。

实测对比：放普通列表 → **0 条**入选；放 `[WHITELIST]` → **164 条**入选、频道数 97 → 221。

### 写法

```ini
[WHITELIST]
https://example.com/iptv.m3u?userid=xxx&token=xxx UA="okHttp/Mod-1.5.0.0"
```

地址后面可以跟请求头：`UA=` / `User-Agent=` / `Cookie=` / `Authorization=`，值含空格用引号。

### 验证是否生效

```bash
cd D:/IPTV-API
.venv/Scripts/python.exe -c "import sys; sys.path.insert(0,'.'); from utils.tools import get_subscribe_entries; i,o = get_subscribe_entries('config/subscribe.txt'); print('白名单段', len(i), '普通段', len(o))"
```

## 三、⚠️ 凭据暴露问题

`subscribe.txt` 里的私人源 URL 带账号凭据，而本仓库是**公开**的。这个 URL 还会被写进 `output/log/unmatch.jsonl`，该文件每天作为 Release 资产公开发布。

也就是说：**仓库和 Release 两处都能拿到你的账号凭据**。要彻底解决，只能把私人源从公开仓库移走（改用私有仓库或 Secret 注入），目前尚未处理。

## 四、改完怎么生效

- **云端**：提交并推送到 fork 的 master，GitHub Actions 定时/手动运行；或手动触发 `main.yml`。
- **本地**：直接跑 `main.py`，配置即时生效。
