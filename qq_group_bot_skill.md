# QQ 群管机器人 Skill — 技术文档

## 功能概述

基于 NapCat + OneBot11 协议的 QQ 群管机器人，功能：

1. **关键词自动回复**：监听群消息，匹配关键词后自动发送引导回复
2. **自动通过入群申请**：新成员申请加群时自动同意
3. **主动发群消息**：可从外部脚本调用，发通知/公告

---

## 架构

```
NapCat（本地/VPS）
  │
  ├── HTTP API（端口 3000）  ← 主动发消息用
  └── WebSocket（端口 3001） ← 事件监听用（群消息、入群申请）
        │
        ▼
    bot.py（常驻后台）
        │
        ├── 匹配关键词 → send_group_msg（自动回复）
        └── 入群申请事件 → set_group_add_request（自动同意）
```

---

## 第一步：部署 NapCat

### Docker 一键部署（VPS 推荐）

```bash
docker run -d \
  --name napcat \
  --network host \
  -e ACCOUNT=你的QQ号 \
  -v /opt/napcat/data:/app/napcat/data \
  mlikiowa/napcat-docker:latest

# 查看扫码登录二维码
docker logs -f napcat
# 用手机 QQ 扫码，登录成功后日志会显示「登录成功」
```

### 配置 NapCat 网络服务

浏览器打开 `http://服务器IP:6099`（NapCat WebUI）

→ **网络配置** → 分别添加：

**HTTP 服务器**（主动发消息）：
- 端口：`3000`
- Token：`your_secret_token`（自定义，记好）

**WebSocket 服务器**（事件监听）：
- 端口：`3001`
- Token：`your_secret_token`（同上）

保存后重启 NapCat。

### 验证

```bash
curl -X POST http://localhost:3000/get_login_info \
  -H "Authorization: Bearer your_secret_token" \
  -H "Content-Type: application/json" \
  -d '{}'
# 返回：{"status":"ok","retcode":0,"data":{"user_id":123456,"nickname":"..."}}
```

---

## 第二步：获取群号

```bash
curl -X POST http://localhost:3000/get_group_list \
  -H "Authorization: Bearer your_secret_token" \
  -H "Content-Type: application/json" \
  -d '{}'
# 返回群列表，找到目标群的 group_id
```

---

## .config.json 配置文件

```json
{
  "napcat_http_url": "http://localhost:3000",
  "napcat_ws_url": "ws://localhost:3001",
  "token": "your_secret_token",
  "groups": [123456789, 987654321],
  "keywords": [
    {
      "match": ["下载不了", "下载失败", "无法下载", "下载出错"],
      "reply": "下载遇到问题？请尝试：\n1. 清除浏览器缓存后重试\n2. 换用其他浏览器\n3. 检查网络连接\n如仍无法解决请联系客服。"
    },
    {
      "match": ["打不开", "无法访问", "网站挂了", "访问不了"],
      "reply": "网站访问异常？请尝试：\n1. 刷新页面重试\n2. 清除缓存\n3. 稍等几分钟再试\n如持续无法访问请联系管理员。"
    },
    {
      "match": ["怎么注册", "如何注册", "注册账号"],
      "reply": "注册教程请查看置顶公告，或访问官网帮助中心。"
    }
  ],
  "auto_approve_join": true,
  "reply_cooldown_seconds": 30
}
```

字段说明：
- `groups`：监听的群号列表（只处理这些群的消息）
- `keywords`：关键词规则，`match` 数组里任意一个匹配即触发 `reply`
- `auto_approve_join`：是否自动同意入群申请
- `reply_cooldown_seconds`：同一群同一关键词的回复冷却时间（防刷屏）

---

## bot.py 完整代码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
bot.py — QQ 群管机器人
功能：关键词自动回复 + 自动通过入群申请

依赖: pip install websocket-client requests

用法:
  python3 bot.py          # 启动机器人
  python3 bot.py --test   # 测试连接
"""

import argparse
import json
import logging
import os
import requests
import threading
import time
import websocket
from pathlib import Path

SKILL_DIR = Path(__file__).resolve().parent.parent
DEFAULT_CONFIG = SKILL_DIR / ".config.json"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)
log = logging.getLogger("qq-bot")


def load_config(path=DEFAULT_CONFIG):
    if not path.exists():
        raise FileNotFoundError(f"配置文件不存在: {path}")
    return json.loads(path.read_text(encoding="utf-8"))


def api_call(cfg, endpoint, payload):
    """调用 NapCat HTTP API"""
    url = cfg["napcat_http_url"].rstrip("/") + "/" + endpoint.lstrip("/")
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {cfg.get('token', '')}",
    }
    r = requests.post(url, json=payload, headers=headers, timeout=10)
    return r.json()


def send_group_msg(cfg, group_id, text):
    result = api_call(cfg, "/send_group_msg", {
        "group_id": group_id,
        "message": [{"type": "text", "data": {"text": text}}]
    })
    if result.get("retcode") == 0:
        log.info(f"✅ 发送成功 → 群{group_id}")
    else:
        log.warning(f"❌ 发送失败 群{group_id}: {result}")
    return result


def approve_join_request(cfg, flag, sub_type):
    """同意入群申请"""
    result = api_call(cfg, "/set_group_add_request", {
        "flag": flag,
        "sub_type": sub_type,
        "approve": True
    })
    if result.get("retcode") == 0:
        log.info(f"✅ 已自动同意入群申请 flag={flag}")
    else:
        log.warning(f"❌ 同意申请失败: {result}")


class QQBot:
    def __init__(self, cfg):
        self.cfg = cfg
        self.groups = set(cfg.get("groups", []))
        self.keywords = cfg.get("keywords", [])
        self.auto_approve = cfg.get("auto_approve_join", True)
        self.cooldown = cfg.get("reply_cooldown_seconds", 30)
        # 冷却记录：{(group_id, keyword_index): last_reply_time}
        self._cooldown_map = {}
        self._lock = threading.Lock()

    def match_keywords(self, text):
        """匹配关键词，返回第一个匹配的 reply，没有则返回 None"""
        for i, rule in enumerate(self.keywords):
            for kw in rule.get("match", []):
                if kw in text:
                    return i, rule["reply"]
        return None, None

    def is_cooled_down(self, group_id, kw_index):
        """检查是否在冷却中"""
        key = (group_id, kw_index)
        with self._lock:
            last = self._cooldown_map.get(key, 0)
            now = time.time()
            if now - last < self.cooldown:
                return True
            self._cooldown_map[key] = now
            return False

    def handle_event(self, data):
        post_type = data.get("post_type")

        # 群消息
        if post_type == "message" and data.get("message_type") == "group":
            group_id = data.get("group_id")
            if self.groups and group_id not in self.groups:
                return  # 不在监听列表，忽略

            # 提取纯文本
            raw = data.get("message", "")
            if isinstance(raw, list):
                text = "".join(
                    seg["data"].get("text", "")
                    for seg in raw if seg.get("type") == "text"
                )
            else:
                text = str(raw)

            if not text.strip():
                return

            kw_index, reply = self.match_keywords(text)
            if reply:
                if not self.is_cooled_down(group_id, kw_index):
                    log.info(f"触发关键词 群{group_id} 消息: {text[:50]}")
                    send_group_msg(self.cfg, group_id, reply)
                else:
                    log.debug(f"冷却中，跳过回复 群{group_id}")

        # 入群申请
        elif post_type == "request" and data.get("request_type") == "group":
            sub_type = data.get("sub_type", "add")
            group_id = data.get("group_id")
            user_id = data.get("user_id")
            flag = data.get("flag", "")

            if self.groups and group_id not in self.groups:
                return

            if self.auto_approve and flag:
                log.info(f"入群申请 群{group_id} 用户{user_id}，自动同意")
                approve_join_request(self.cfg, flag, sub_type)

    def run(self):
        ws_url = self.cfg["napcat_ws_url"]
        token = self.cfg.get("token", "")
        if token:
            ws_url = ws_url.rstrip("/") + f"?access_token={token}"

        log.info(f"连接 WebSocket: {ws_url.split('?')[0]}")

        def on_message(ws, message):
            try:
                data = json.loads(message)
                self.handle_event(data)
            except Exception as e:
                log.error(f"处理事件失败: {e}")

        def on_open(ws):
            log.info("✅ WebSocket 已连接，机器人运行中...")

        def on_error(ws, error):
            log.error(f"WebSocket 错误: {error}")

        def on_close(ws, code, msg):
            log.warning(f"WebSocket 断开 ({code})，5秒后重连...")
            time.sleep(5)
            self.run()

        ws = websocket.WebSocketApp(
            ws_url,
            on_open=on_open,
            on_message=on_message,
            on_error=on_error,
            on_close=on_close,
        )
        ws.run_forever(ping_interval=30, ping_timeout=10)


def cmd_test(cfg):
    log.info("测试 HTTP 连接...")
    try:
        result = api_call(cfg, "/get_login_info", {})
        if result.get("retcode") == 0:
            d = result["data"]
            log.info(f"✅ 连接成功 QQ: {d.get('user_id')} 昵称: {d.get('nickname')}")
        else:
            log.error(f"❌ 响应异常: {result}")
    except Exception as e:
        log.error(f"❌ HTTP 连接失败: {e}")
        return

    log.info("获取群列表...")
    try:
        result = api_call(cfg, "/get_group_list", {})
        groups = result.get("data", [])
        log.info(f"已加入 {len(groups)} 个群:")
        for g in groups[:15]:
            log.info(f"  {g.get('group_id')}  {g.get('group_name')}")
    except Exception as e:
        log.error(f"获取群列表失败: {e}")


def main():
    parser = argparse.ArgumentParser(description="QQ 群管机器人")
    parser.add_argument("--config", default=None)
    parser.add_argument("--test", action="store_true", help="测试连接")
    args = parser.parse_args()

    cfg_path = Path(args.config) if args.config else DEFAULT_CONFIG
    cfg = load_config(cfg_path)

    if args.test:
        cmd_test(cfg)
        return

    bot = QQBot(cfg)
    try:
        bot.run()
    except KeyboardInterrupt:
        log.info("机器人已停止")


if __name__ == "__main__":
    main()
```

---

## 部署步骤

### 1. 安装依赖

```bash
pip install websocket-client requests
```

### 2. 创建配置文件

```bash
cat > .config.json << 'EOF'
{
  "napcat_http_url": "http://localhost:3000",
  "napcat_ws_url": "ws://localhost:3001",
  "token": "your_secret_token",
  "groups": [123456789],
  "keywords": [
    {
      "match": ["下载不了", "下载失败", "无法下载"],
      "reply": "下载遇到问题？请清除缓存后重试，或更换浏览器。仍有问题请联系客服。"
    },
    {
      "match": ["打不开", "无法访问", "网站挂了"],
      "reply": "访问异常？请刷新页面或稍后重试。如持续无法访问请联系管理员。"
    }
  ],
  "auto_approve_join": true,
  "reply_cooldown_seconds": 30
}
EOF
chmod 600 .config.json
```

### 3. 测试连接

```bash
python3 scripts/bot.py --test
```

### 4. 启动机器人

```bash
# 前台运行（测试用）
python3 scripts/bot.py

# 后台运行（生产用）
nohup python3 scripts/bot.py > /var/log/qq-bot.log 2>&1 &
echo $! > /tmp/qq-bot.pid
```

### 5. 查看日志

```bash
tail -f /var/log/qq-bot.log
```

### 6. 停止机器人

```bash
kill $(cat /tmp/qq-bot.pid)
```

---

## 自定义关键词

编辑 `.config.json` 中的 `keywords` 数组，格式：

```json
{
  "match": ["关键词1", "关键词2"],
  "reply": "自动回复的内容\n支持换行"
}
```

修改后重启 bot.py 生效。

---

## 目录结构

```
skills/qq-group-bot/
├── SKILL.md
├── .config.json        ← 凭据和规则配置（不进 git）
└── scripts/
    └── bot.py
```

---

## 注意事项

1. **NapCat 需要持续运行**：Docker 部署最稳，重启自动恢复
2. **QQ 账号风控**：新号容易被封，建议用 1 年以上的老号
3. **冷却时间**：`reply_cooldown_seconds` 防止同一关键词短时间内刷屏回复
4. **groups 留空**：`"groups": []` 表示监听所有群（慎用）
5. **WebSocket 断线重连**：bot.py 内置自动重连，5 秒后重试
