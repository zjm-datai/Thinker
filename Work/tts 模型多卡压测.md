
## 压测结论

>并发上限：指的是在平均首包延迟在 3 秒以内可容纳的最大并发（只并发一次，没有进行长时间测试）

| 配置              | 并发上限  | 平均首包延迟              | 显存占用   | 备注                 |
| --------------- | ----- | ------------------- | ------ | ------------------ |
| **单卡 · 1 实例**   | 10–20 | 10 并发 ~2s；20 并发 ~3s | ~30%   | 超过 20 并发延迟快速上升     |
| **单卡 · 2 实例**   | 20–30 | 30 并发 ~3.1s         | ~42%   | GPU 利用率接近 97%      |
| **单卡 · 3 实例**   | ≈30   | 30 并发 ~3s；40 并发 >6s | ~66%   | 性能未提升，反而不稳         |
| **双卡 · 各 1 实例** | ≈40   | 40 并发 ~2.9s         | ~35%/卡 | 50–60 并发时首包 3.5–5s |
| **双卡 · 各 2 实例** | ≈60   | 60 并发 ~3.1s         | ~55%/卡 | 70 并发能跑但首包 4–5s+   |
| **双卡 · 各 4 实例** | <60   | 60–70 并发直接卡死        | ~90%/卡 | 显存不足，延迟 >10s       |

---

**单卡单实例**：
- 极限并发 **10–20**，10 并发时首包 ~2s，20 并发时首包已逼近 3s。

**单卡双实例**：
- 极限并发 **20–30**，GPU 利用率约 97%，显存占用 ~42%。

**单卡三实例**：
- 极限并发与双实例接近（≈30），显存占用 ~66%，40 并发时首包延迟明显劣化。

**双卡单实例**：
- 极限并发 **≈40**。超过 50–60 并发时首包延迟快速上升至 3.5–5s。

**双卡双实例**：
- 极限并发 **≈60**。70 并发能跑通，但首包延迟升至 4–5s+，波动较大。

**双卡四实例/卡**：
- 显存占用 >90%，60–70 并发直接卡死，首包延迟 >10s，性能严重恶化。

**总体规律：**

- 并发能力随 **卡数和实例数增加** 而提升，但不是线性增长。
- **每卡 2 个实例**是比较合理的上限。显存最好保持 **<70%**，才能保证推理调度稳定。
- 每卡的稳态并发能力大致为 **20–30 并发 / GPU**（首包 ≤3s）。

#### 如何进行堆卡和实例部署

从实测结果看：

- 单卡可支撑 10–30 并发（取决于实例数）。
- 双卡扩展后可支撑 40–60 并发。
- 超过每卡 2 实例，显存竞争激烈，性能并没有得到提升。

**结论**：通过合理“堆卡”，确实能满足更高并发，但要控制实例数和显存占用。

#### 并发与卡的比例（经验数据）

在 **首包 ≤3s**的前提下：

- 单卡：20–30 并发
- 双卡：40–60 并发
- 推算公式：**GPU 数 ≈ ⌈目标并发 / 25⌉**

举例：

- 100 并发 → 约 4 卡
- 250 并发 → 约 10 卡

## 具体数据

### 接口设计

每个 fastapi app 挂载一个 tts model 

通过 uvcorn 启动多个 workers 也就是多个 app 多个 workers

多卡的启动设计为：用两个不同的端口启动 uvcorn ，然后通过一个 web 服务器作为网关对请求进行平均的分发

### 压测脚本及启动命令

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import asyncio
import argparse
import json
import random
import string
from time import perf_counter
from statistics import mean

import httpx

# -------- config defaults --------
DEF_URL = "http://127.0.0.1:7000/v1/audio/speech"
DEF_CONC = 50
DEF_N = 50
TEXT_BASE = "第 {i} 个请求。黑哥们的语言是不通的，坦克是没有后视镜的。"

def rand_suffix(k=6):
    return "".join(random.choices(string.ascii_lowercase + string.digits, k=k))

def percentile(values, p):
    if not values:
        return 0.0
    s = sorted(values)
    if p <= 0:
        return s[0]
    if p >= 100:
        return s[-1]
    k = (len(s)-1) * (p/100.0)
    f = int(k)
    c = min(f+1, len(s)-1)
    if f == c:
        return s[f]
    d0 = s[f] * (c-k)
    d1 = s[c] * (k-f)
    return d0 + d1

def fmt_ms(x): return f"{x:.1f}"

async def single_non_stream(
    client: httpx.AsyncClient, url: str, text: str, voice: str = "alloy"
):
    payload = {
        "model": "cosyvoice2",
        "input": text,
        "voice": voice,
        "response_format": "mp3",
        "speed": 1.0,
    }
    t0 = perf_counter()
    try:
        r = await client.post(url, json=payload, timeout=None)
        status = r.status_code
        if r.headers.get("content-type","").startswith("application/json"):
            # 失败/错误
            body = await r.aread()
            total_ms = (perf_counter()-t0)*1000
            return dict(
                ok=False, status=status, total_ms=total_ms,
                bytes=len(body), err=body.decode(errors="ignore")[:200]
            )
        body = await r.aread()
        total_ms = (perf_counter()-t0)*1000
        return dict(ok=True, status=status, total_ms=total_ms, bytes=len(body))
    except Exception as e:
        total_ms = (perf_counter()-t0)*1000
        return dict(ok=False, status=-1, total_ms=total_ms, bytes=0, err=str(e)[:200])

async def single_stream_pcm(
    client: httpx.AsyncClient, url: str, text: str, voice: str = "alloy"
):
    # 注意：你的服务要求 stream=true 且 response_format=pcm 才会流式
    payload = {
        "model": "cosyvoice2",
        "input": text,
        "voice": voice,
        "response_format": "pcm",
        "speed": 1.0,
        "stream": True
    }
    t0 = perf_counter()
    first_byte_ts = None
    total_bytes = 0
    status = None

    try:
        async with client.stream("POST", url, json=payload, timeout=None) as r:
            status = r.status_code
            ctype = r.headers.get("content-type","")
            # 如果后端返回 JSON 错误，直接读取文本
            if ctype.startswith("application/json"):
                body = await r.aread()
                total_ms = (perf_counter()-t0)*1000
                return dict(
                    ok=False, status=status, total_ms=total_ms,
                    first_byte_ms=None, bytes=len(body),
                    err=body.decode(errors="ignore")[:200]
                )

            async for chunk in r.aiter_bytes():
                if chunk:
                    if first_byte_ts is None:
                        first_byte_ts = perf_counter()
                    total_bytes += len(chunk)

        t1 = perf_counter()
        return dict(
            ok=True, status=status,
            first_byte_ms=(first_byte_ts - t0)*1000 if first_byte_ts else None,
            total_ms=(t1 - t0)*1000,
            bytes=total_bytes
        )
    except Exception as e:
        t1 = perf_counter()
        return dict(
            ok=False, status=status if status is not None else -1,
            first_byte_ms=(first_byte_ts - t0)*1000 if first_byte_ts else None,
            total_ms=(t1 - t0)*1000, bytes=total_bytes, err=str(e)[:200]
        )

async def run_phase(label, worker, client, url, n, conc):
    print(f"\n== {label} | total={n} | concurrency={conc} ==")
    # 生成请求文本
    texts = [TEXT_BASE.format(i=i) + " " + rand_suffix() for i in range(n)]

    sem = asyncio.Semaphore(conc)
    results = []

    async def task(i, text):
        async with sem:
            return await worker(client, url, text)

    t_batch0 = perf_counter()
    tasks = [asyncio.create_task(task(i, texts[i])) for i in range(n)]
    for i, t in enumerate(asyncio.as_completed(tasks), 1):
        res = await t
        results.append(res)
        if label.endswith("STREAM"):
            fb = res.get("first_byte_ms")
            fb_s = fmt_ms(fb) if fb is not None else "N/A"
            print(f"[{i:>3}/{n}] status={res['status']} fb_ms={fb_s} total_ms={fmt_ms(res['total_ms'])} bytes={res['bytes']}{' ERR' if not res['ok'] else ''}")
        else:
            print(f"[{i:>3}/{n}] status={res['status']} total_ms={fmt_ms(res['total_ms'])} bytes={res['bytes']}{' ERR' if not res['ok'] else ''}")
    t_batch1 = perf_counter()

    # 聚合统计
    oks = [r for r in results if r["ok"]]
    fails = [r for r in results if not r["ok"]]
    total_wall_ms = (t_batch1 - t_batch0)*1000
    rps = n / ((t_batch1 - t_batch0) if (t_batch1 > t_batch0) else 1)

    total_ms_list = [r["total_ms"] for r in results]
    fb_ms_list = [r["first_byte_ms"] for r in results if "first_byte_ms" in r and r["first_byte_ms"] is not None]

    def summary_line(name, arr):
        if not arr:
            return f"{name}: N/A"
        return (f"{name}: min={fmt_ms(min(arr))} avg={fmt_ms(mean(arr))} "
                f"p50={fmt_ms(percentile(arr,50))} p90={fmt_ms(percentile(arr,90))} "
                f"p95={fmt_ms(percentile(arr,95))} max={fmt_ms(max(arr))}")

    print("\n-- Summary --")
    print(f"OK={len(oks)} FAIL={len(fails)}  total_wall_ms={fmt_ms(total_wall_ms)}  RPS={rps:.2f}")
    print(summary_line("TOTAL_MS", total_ms_list))
    if fb_ms_list:
        print(summary_line("FIRST_BYTE_MS", fb_ms_list))
    if fails:
        print("\nFirst 3 errors:")
        for e in fails[:3]:
            print(f"  status={e['status']} err={e.get('err','')[:160]}")

async def main():
    parser = argparse.ArgumentParser(description="CosyVoice2 TTS load test (stream & non-stream)")
    parser.add_argument("--url", default=DEF_URL, help=f"TTS endpoint url (default: {DEF_URL})")
    parser.add_argument("--concurrency", "-c", type=int, default=DEF_CONC, help="concurrency (default: 10)")
    parser.add_argument("--n", type=int, default=DEF_N, help="number of requests per phase (default: 10)")
    parser.add_argument("--nonstream-only", action="store_true", help="only run non-stream mp3 phase")
    parser.add_argument("--stream-only", action="store_true", help="only run stream pcm phase")
    args = parser.parse_args()

    limits = httpx.Limits(max_connections=max(args.concurrency, args.n), max_keepalive_connections=args.concurrency)
    async with httpx.AsyncClient(limits=limits, timeout=None) as client:
        if not args.stream_only:
            await run_phase("NON-STREAM (mp3)", single_non_stream, client, args.url, args.n, args.concurrency)
        if not args.nonstream_only:
            await run_phase("STREAM (pcm)", single_stream_pcm, client, args.url, args.n, args.concurrency)

if __name__ == "__main__":
    asyncio.run(main())
```

#### 启动命令

```
python bench_tts.py --stream-only
```

### 单卡单模型实例

20 并发就不稳定，平均首包有时候会超过 3 s，最好是 10 并发在 2 s 左右

```
== STREAM (pcm) | total=10 | concurrency=10 ==
[  1/10] status=200 total_ms=6499.7 bytes=351360
[  2/10] status=200 total_ms=7832.9 bytes=384000
[  3/10] status=200 total_ms=7853.5 bytes=370560
[  4/10] status=200 total_ms=8415.0 bytes=420480
[  5/10] status=200 total_ms=8886.0 bytes=433920
[  6/10] status=200 total_ms=8982.8 bytes=428160
[  7/10] status=200 total_ms=9114.4 bytes=487680
[  8/10] status=200 total_ms=9180.6 bytes=478080
[  9/10] status=200 total_ms=9292.3 bytes=462720
[ 10/10] status=200 total_ms=9825.2 bytes=606720

-- Summary --
OK=10 FAIL=0  total_wall_ms=9830.2  RPS=1.02
TOTAL_MS: min=6499.7 avg=8588.2 p50=8934.4 p90=9345.5 p95=9585.4 max=9825.2
FIRST_BYTE_MS: min=2048.9 avg=2091.1 p50=2094.4 p90=2112.0 p95=2124.2 max=2136.5
```

```
== STREAM (pcm) | total=20 | concurrency=20 ==
[  1/20] status=200 total_ms=13507.1 bytes=351360
[  2/20] status=200 total_ms=13584.2 bytes=378240
[  3/20] status=200 total_ms=13770.0 bytes=362880
[  4/20] status=200 total_ms=14456.8 bytes=389760
[  5/20] status=200 total_ms=14854.5 bytes=380160
[  6/20] status=200 total_ms=15043.2 bytes=368640
[  7/20] status=200 total_ms=15082.3 bytes=412800
[  8/20] status=200 total_ms=15115.1 bytes=393600
[  9/20] status=200 total_ms=15177.5 bytes=412800
[ 10/20] status=200 total_ms=15273.7 bytes=387840
[ 11/20] status=200 total_ms=16023.4 bytes=428160
[ 12/20] status=200 total_ms=16210.8 bytes=424320
[ 13/20] status=200 total_ms=16357.4 bytes=433920
[ 14/20] status=200 total_ms=16413.1 bytes=414720
[ 15/20] status=200 total_ms=16422.5 bytes=414720
[ 16/20] status=200 total_ms=17098.0 bytes=501120
[ 17/20] status=200 total_ms=17178.2 bytes=497280
[ 18/20] status=200 total_ms=17353.8 bytes=514560
[ 19/20] status=200 total_ms=17395.9 bytes=554880
[ 20/20] status=200 total_ms=17412.0 bytes=533760

-- Summary --
OK=20 FAIL=0  total_wall_ms=17416.8  RPS=1.15
TOTAL_MS: min=13507.1 avg=15686.5 p50=15648.5 p90=17358.0 p95=17396.7 max=17412.0
FIRST_BYTE_MS: min=2695.2 avg=2994.1 p50=3030.9 p90=3093.6 p95=3103.7 max=3108.9
```

平均首包延迟在 3 秒内的极限是 10 到 20 的样子

### 单卡双模型实例

30 并发推理时候的显存占用：GPU 97% 的使用率，显存占用 42%

![[Pasted image 20250825172927.png]]

平均首包延迟在 3 秒内的极限是 20 - 30 的样子

```
== STREAM (pcm) | total=30 | concurrency=30 ==
[  1/30] status=200 total_ms=13287.3 bytes=337920
[  2/30] status=200 total_ms=13688.3 bytes=393600
[  3/30] status=200 total_ms=14150.3 bytes=376320
[  4/30] status=200 total_ms=14288.0 bytes=391680
[  5/30] status=200 total_ms=15533.3 bytes=445440
[  6/30] status=200 total_ms=15681.9 bytes=403200
[  7/30] status=200 total_ms=15944.8 bytes=428160
[  8/30] status=200 total_ms=15960.3 bytes=391680
[  9/30] status=200 total_ms=16071.4 bytes=401280
[ 10/30] status=200 total_ms=16117.9 bytes=445440
[ 11/30] status=200 total_ms=16183.2 bytes=387840
[ 12/30] status=200 total_ms=16237.4 bytes=405120
[ 13/30] status=200 total_ms=16630.9 bytes=472320
[ 14/30] status=200 total_ms=16688.3 bytes=474240
[ 15/30] status=200 total_ms=16806.2 bytes=485760
[ 16/30] status=200 total_ms=16858.6 bytes=495360
[ 17/30] status=200 total_ms=17119.2 bytes=424320
[ 18/30] status=200 total_ms=17353.8 bytes=520320
[ 19/30] status=200 total_ms=17397.5 bytes=460800
[ 20/30] status=200 total_ms=17485.9 bytes=428160
[ 21/30] status=200 total_ms=17614.7 bytes=554880
[ 22/30] status=200 total_ms=17643.9 bytes=533760
[ 23/30] status=200 total_ms=17675.3 bytes=531840
[ 24/30] status=200 total_ms=17918.3 bytes=480000
[ 25/30] status=200 total_ms=18164.2 bytes=499200
[ 26/30] status=200 total_ms=18841.2 bytes=539520
[ 27/30] status=200 total_ms=18886.4 bytes=518400
[ 28/30] status=200 total_ms=19088.2 bytes=597120
[ 29/30] status=200 total_ms=19238.9 bytes=558720
[ 30/30] status=200 total_ms=19354.9 bytes=631680

-- Summary --
OK=30 FAIL=0  total_wall_ms=19363.0  RPS=1.55
TOTAL_MS: min=13287.3 avg=16797.0 p50=16832.4 p90=18906.6 p95=19171.1 max=19354.9
FIRST_BYTE_MS: min=2235.7 avg=3138.9 p50=3694.0 p90=3913.2 p95=3915.6 max=3927.9
```

### 单卡三模型实例

显存占用 66%

40 并发顶不住，30 尚可但是和双模型差不多

```
== STREAM (pcm) | total=30 | concurrency=30 ==
[  1/30] status=200 total_ms=12186.5 bytes=389760
[  2/30] status=200 total_ms=12364.3 bytes=347520
[  3/30] status=200 total_ms=12536.4 bytes=341760
[  4/30] status=200 total_ms=13077.8 bytes=416640
[  5/30] status=200 total_ms=14227.7 bytes=451200
[  6/30] status=200 total_ms=14273.2 bytes=424320
[  7/30] status=200 total_ms=14529.9 bytes=418560
[  8/30] status=200 total_ms=14746.9 bytes=451200
[  9/30] status=200 total_ms=14949.7 bytes=481920
[ 10/30] status=200 total_ms=14990.7 bytes=462720
[ 11/30] status=200 total_ms=15239.7 bytes=478080
[ 12/30] status=200 total_ms=15619.2 bytes=462720
[ 13/30] status=200 total_ms=16059.4 bytes=560640
[ 14/30] status=200 total_ms=16166.0 bytes=539520
[ 15/30] status=200 total_ms=16350.3 bytes=531840
[ 16/30] status=200 total_ms=16411.1 bytes=579840
[ 17/30] status=200 total_ms=16759.7 bytes=625920
[ 18/30] status=200 total_ms=17034.7 bytes=410880
[ 19/30] status=200 total_ms=17736.4 bytes=451200
[ 20/30] status=200 total_ms=18338.8 bytes=416640
[ 21/30] status=200 total_ms=18477.9 bytes=430080
[ 22/30] status=200 total_ms=18572.3 bytes=439680
[ 23/30] status=200 total_ms=18646.4 bytes=441600
[ 24/30] status=200 total_ms=18945.2 bytes=501120
[ 25/30] status=200 total_ms=19203.9 bytes=499200
[ 26/30] status=200 total_ms=19538.9 bytes=551040
[ 27/30] status=200 total_ms=19877.3 bytes=528000
[ 28/30] status=200 total_ms=20002.0 bytes=539520
[ 29/30] status=200 total_ms=20034.0 bytes=516480
[ 30/30] status=200 total_ms=20123.3 bytes=566400

-- Summary --
OK=30 FAIL=0  total_wall_ms=20130.4  RPS=1.49
TOTAL_MS: min=12186.5 avg=16567.3 p50=16380.7 p90=19889.7 p95=20019.6 max=20123.3
FIRST_BYTE_MS: min=2129.5 avg=3060.3 p50=2682.7 p90=4386.9 p95=4414.3 max=4450.7
```

```
== STREAM (pcm) | total=40 | concurrency=40 ==
[  1/40] status=200 total_ms=4110.9 bytes=389760
[  2/40] status=200 total_ms=4430.6 bytes=426240
[  3/40] status=200 total_ms=4544.7 bytes=422400
[  4/40] status=200 total_ms=4972.5 bytes=410880
[  5/40] status=200 total_ms=5411.5 bytes=458880
[  6/40] status=200 total_ms=5801.8 bytes=497280
[  7/40] status=200 total_ms=5979.0 bytes=541440
[  8/40] status=200 total_ms=24585.8 bytes=405120
[  9/40] status=200 total_ms=24849.1 bytes=412800
[ 10/40] status=200 total_ms=24908.6 bytes=405120
[ 11/40] status=200 total_ms=25306.0 bytes=405120
[ 12/40] status=200 total_ms=25327.4 bytes=407040
[ 13/40] status=200 total_ms=25426.8 bytes=399360
[ 14/40] status=200 total_ms=25923.8 bytes=407040
[ 15/40] status=200 total_ms=26878.0 bytes=432000
[ 16/40] status=200 total_ms=27062.6 bytes=428160
[ 17/40] status=200 total_ms=27714.9 bytes=428160
[ 18/40] status=200 total_ms=27742.1 bytes=447360
[ 19/40] status=200 total_ms=27878.8 bytes=422400
[ 20/40] status=200 total_ms=27967.9 bytes=414720
[ 21/40] status=200 total_ms=28009.9 bytes=455040
[ 22/40] status=200 total_ms=28053.5 bytes=443520
[ 23/40] status=200 total_ms=28312.9 bytes=437760
[ 24/40] status=200 total_ms=28398.0 bytes=433920
[ 25/40] status=200 total_ms=28697.0 bytes=439680
[ 26/40] status=200 total_ms=29481.1 bytes=476160
[ 27/40] status=200 total_ms=29548.9 bytes=487680
[ 28/40] status=200 total_ms=29609.4 bytes=485760
[ 29/40] status=200 total_ms=29764.7 bytes=504960
[ 30/40] status=200 total_ms=29777.4 bytes=468480
[ 31/40] status=200 total_ms=29846.8 bytes=493440
[ 32/40] status=200 total_ms=29854.8 bytes=503040
[ 33/40] status=200 total_ms=29911.2 bytes=487680
[ 34/40] status=200 total_ms=29966.6 bytes=481920
[ 35/40] status=200 total_ms=30073.1 bytes=487680
[ 36/40] status=200 total_ms=30891.3 bytes=587520
[ 37/40] status=200 total_ms=30925.6 bytes=597120
[ 38/40] status=200 total_ms=31341.5 bytes=618240
[ 39/40] status=200 total_ms=31394.9 bytes=618240
[ 40/40] status=200 total_ms=31456.2 bytes=691200

-- Summary --
OK=40 FAIL=0  total_wall_ms=31466.4  RPS=1.27
TOTAL_MS: min=4110.9 avg=24303.4 p50=27988.9 p90=30894.8 p95=31344.2 max=31456.2
FIRST_BYTE_MS: min=1159.0 avg=6796.5 p50=7960.6 p90=8085.9 p95=8093.5 max=8110.3
```

### 双卡，每张卡单模型实例

平均首包延迟在3 秒内的极限是 40

```
== STREAM (pcm) | total=40 | concurrency=40 ==
[  1/40] status=200 total_ms=11889.1 bytes=360960
[  2/40] status=200 total_ms=12280.3 bytes=357120
[  3/40] status=200 total_ms=13296.4 bytes=368640
[  4/40] status=200 total_ms=14917.7 bytes=432000
[  5/40] status=200 total_ms=15009.3 bytes=414720
[  6/40] status=200 total_ms=15273.8 bytes=458880
[  7/40] status=200 total_ms=15347.4 bytes=432000
[  8/40] status=200 total_ms=15384.4 bytes=432000
[  9/40] status=200 total_ms=15415.7 bytes=447360
[ 10/40] status=200 total_ms=15566.5 bytes=405120
[ 11/40] status=200 total_ms=15702.6 bytes=393600
[ 12/40] status=200 total_ms=15808.7 bytes=414720
[ 13/40] status=200 total_ms=16243.8 bytes=418560
[ 14/40] status=200 total_ms=16410.5 bytes=428160
[ 15/40] status=200 total_ms=16455.5 bytes=470400
[ 16/40] status=200 total_ms=16505.4 bytes=491520
[ 17/40] status=200 total_ms=16539.9 bytes=493440
[ 18/40] status=200 total_ms=16647.4 bytes=418560
[ 19/40] status=200 total_ms=16656.1 bytes=478080
[ 20/40] status=200 total_ms=16668.5 bytes=495360
[ 21/40] status=200 total_ms=16924.3 bytes=432000
[ 22/40] status=200 total_ms=17031.2 bytes=510720
[ 23/40] status=200 total_ms=17162.3 bytes=456960
[ 24/40] status=200 total_ms=17199.9 bytes=516480
[ 25/40] status=200 total_ms=17238.0 bytes=529920
[ 26/40] status=200 total_ms=17250.3 bytes=535680
[ 27/40] status=200 total_ms=17622.8 bytes=418560
[ 28/40] status=200 total_ms=18718.6 bytes=483840
[ 29/40] status=200 total_ms=18779.3 bytes=466560
[ 30/40] status=200 total_ms=18886.1 bytes=506880
[ 31/40] status=200 total_ms=18964.4 bytes=480000
[ 32/40] status=200 total_ms=19102.8 bytes=472320
[ 33/40] status=200 total_ms=19174.5 bytes=504960
[ 34/40] status=200 total_ms=19622.4 bytes=562560
[ 35/40] status=200 total_ms=19811.0 bytes=545280
[ 36/40] status=200 total_ms=19892.6 bytes=537600
[ 37/40] status=200 total_ms=19906.1 bytes=526080
[ 38/40] status=200 total_ms=20235.4 bytes=562560
[ 39/40] status=200 total_ms=20351.0 bytes=566400
[ 40/40] status=200 total_ms=20500.8 bytes=633600

-- Summary --
OK=40 FAIL=0  total_wall_ms=20512.3  RPS=1.95
TOTAL_MS: min=11889.1 avg=17059.8 p50=16796.4 p90=19894.0 p95=20241.2 max=20500.8
FIRST_BYTE_MS: min=1328.3 avg=2941.6 p50=3000.4 p90=3269.2 p95=3285.0 max=3301.5
```

```
== STREAM (pcm) | total=50 | concurrency=50 ==
[  1/50] status=200 total_ms=15689.9 bytes=334080
[  2/50] status=200 total_ms=16766.4 bytes=345600
[  3/50] status=200 total_ms=17540.4 bytes=405120
[  4/50] status=200 total_ms=18148.4 bytes=372480
[  5/50] status=200 total_ms=18526.0 bytes=433920
[  6/50] status=200 total_ms=18682.1 bytes=455040
[  7/50] status=200 total_ms=18741.9 bytes=447360
[  8/50] status=200 total_ms=18756.3 bytes=407040
[  9/50] status=200 total_ms=19143.9 bytes=393600
[ 10/50] status=200 total_ms=19285.4 bytes=412800
[ 11/50] status=200 total_ms=19489.6 bytes=435840
[ 12/50] status=200 total_ms=19701.3 bytes=432000
[ 13/50] status=200 total_ms=19937.7 bytes=437760
[ 14/50] status=200 total_ms=19978.8 bytes=393600
[ 15/50] status=200 total_ms=20173.3 bytes=435840
[ 16/50] status=200 total_ms=20225.4 bytes=420480
[ 17/50] status=200 total_ms=20251.2 bytes=458880
[ 18/50] status=200 total_ms=20384.4 bytes=422400
[ 19/50] status=200 total_ms=20492.3 bytes=443520
[ 20/50] status=200 total_ms=21137.8 bytes=476160
[ 21/50] status=200 total_ms=21463.3 bytes=420480
[ 22/50] status=200 total_ms=21488.0 bytes=480000
[ 23/50] status=200 total_ms=21513.6 bytes=487680
[ 24/50] status=200 total_ms=21630.4 bytes=426240
[ 25/50] status=200 total_ms=21680.2 bytes=472320
[ 26/50] status=200 total_ms=21781.3 bytes=504960
[ 27/50] status=200 total_ms=21788.7 bytes=489600
[ 28/50] status=200 total_ms=21943.5 bytes=455040
[ 29/50] status=200 total_ms=21964.2 bytes=441600
[ 30/50] status=200 total_ms=22119.4 bytes=432000
[ 31/50] status=200 total_ms=22165.2 bytes=551040
[ 32/50] status=200 total_ms=22239.0 bytes=524160
[ 33/50] status=200 total_ms=22290.4 bytes=514560
[ 34/50] status=200 total_ms=22350.7 bytes=537600
[ 35/50] status=200 total_ms=22403.7 bytes=414720
[ 36/50] status=200 total_ms=22491.6 bytes=564480
[ 37/50] status=200 total_ms=22824.9 bytes=481920
[ 38/50] status=200 total_ms=23306.1 bytes=470400
[ 39/50] status=200 total_ms=23452.3 bytes=504960
[ 40/50] status=200 total_ms=23580.1 bytes=462720
[ 41/50] status=200 total_ms=23719.5 bytes=481920
[ 42/50] status=200 total_ms=24329.8 bytes=537600
[ 43/50] status=200 total_ms=24699.3 bytes=518400
[ 44/50] status=200 total_ms=24774.5 bytes=524160
[ 45/50] status=200 total_ms=24887.4 bytes=551040
[ 46/50] status=200 total_ms=25435.0 bytes=558720
[ 47/50] status=200 total_ms=25509.5 bytes=566400
[ 48/50] status=200 total_ms=25525.0 bytes=572160
[ 49/50] status=200 total_ms=25833.1 bytes=650880
[ 50/50] status=200 total_ms=25917.5 bytes=700800

-- Summary --
OK=50 FAIL=0  total_wall_ms=25925.3  RPS=1.93
TOTAL_MS: min=15689.9 avg=21563.2 p50=21730.7 p90=24942.1 p95=25518.1 max=25917.5
FIRST_BYTE_MS: min=2269.8 avg=3489.9 p50=3578.3 p90=3694.1 p95=3714.7 max=3720.3
```

```
== STREAM (pcm) | total=60 | concurrency=60 ==
[  1/60] status=200 total_ms=29306.2 bytes=360960
[  2/60] status=200 total_ms=31648.7 bytes=372480
[  3/60] status=200 total_ms=31676.4 bytes=456960
[  4/60] status=200 total_ms=31924.4 bytes=382080
[  5/60] status=200 total_ms=32107.5 bytes=389760
[  6/60] status=200 total_ms=32238.5 bytes=366720
[  7/60] status=200 total_ms=32446.3 bytes=376320
[  8/60] status=200 total_ms=32872.1 bytes=407040
[  9/60] status=200 total_ms=32938.9 bytes=376320
[ 10/60] status=200 total_ms=33097.9 bytes=382080
[ 11/60] status=200 total_ms=33136.0 bytes=412800
[ 12/60] status=200 total_ms=33189.1 bytes=405120
[ 13/60] status=200 total_ms=33387.7 bytes=387840
[ 14/60] status=200 total_ms=33531.6 bytes=405120
[ 15/60] status=200 total_ms=34601.7 bytes=414720
[ 16/60] status=200 total_ms=34738.5 bytes=432000
[ 17/60] status=200 total_ms=34950.8 bytes=453120
[ 18/60] status=200 total_ms=35110.0 bytes=451200
[ 19/60] status=200 total_ms=35173.2 bytes=433920
[ 20/60] status=200 total_ms=35185.6 bytes=414720
[ 21/60] status=200 total_ms=35249.7 bytes=447360
[ 22/60] status=200 total_ms=35278.8 bytes=455040
[ 23/60] status=200 total_ms=35337.5 bytes=455040
[ 24/60] status=200 total_ms=35371.9 bytes=460800
[ 25/60] status=200 total_ms=35405.8 bytes=432000
[ 26/60] status=200 total_ms=35549.4 bytes=432000
[ 27/60] status=200 total_ms=35575.6 bytes=453120
[ 28/60] status=200 total_ms=35618.8 bytes=447360
[ 29/60] status=200 total_ms=35705.9 bytes=430080
[ 30/60] status=200 total_ms=37102.4 bytes=501120
[ 31/60] status=200 total_ms=37125.2 bytes=487680
[ 32/60] status=200 total_ms=37176.1 bytes=464640
[ 33/60] status=200 total_ms=37210.2 bytes=489600
[ 34/60] status=200 total_ms=37242.9 bytes=487680
[ 35/60] status=200 total_ms=37271.2 bytes=485760
[ 36/60] status=200 total_ms=37288.3 bytes=503040
[ 37/60] status=200 total_ms=37306.8 bytes=487680
[ 38/60] status=200 total_ms=37396.5 bytes=478080
[ 39/60] status=200 total_ms=37484.4 bytes=476160
[ 40/60] status=200 total_ms=37566.0 bytes=508800
[ 41/60] status=200 total_ms=37560.5 bytes=472320
[ 42/60] status=200 total_ms=37574.7 bytes=495360
[ 43/60] status=200 total_ms=37576.7 bytes=499200
[ 44/60] status=200 total_ms=37642.5 bytes=487680
[ 45/60] status=200 total_ms=37694.5 bytes=512640
[ 46/60] status=200 total_ms=37710.2 bytes=491520
[ 47/60] status=200 total_ms=38232.2 bytes=537600
[ 48/60] status=200 total_ms=38286.0 bytes=528000
[ 49/60] status=200 total_ms=38336.3 bytes=549120
[ 50/60] status=200 total_ms=38360.1 bytes=535680
[ 51/60] status=200 total_ms=38382.8 bytes=516480
[ 52/60] status=200 total_ms=38382.6 bytes=533760
[ 53/60] status=200 total_ms=38425.3 bytes=533760
[ 54/60] status=200 total_ms=38486.0 bytes=514560
[ 55/60] status=200 total_ms=38557.1 bytes=512640
[ 56/60] status=200 total_ms=38597.9 bytes=522240
[ 57/60] status=200 total_ms=39004.1 bytes=604800
[ 58/60] status=200 total_ms=39093.2 bytes=572160
[ 59/60] status=200 total_ms=39351.4 bytes=618240
[ 60/60] status=200 total_ms=39734.6 bytes=721920

-- Summary --
OK=60 FAIL=0  total_wall_ms=39747.6  RPS=1.51
TOTAL_MS: min=29306.2 avg=35991.9 p50=37113.8 p90=38493.1 p95=39008.5 max=39734.6
FIRST_BYTE_MS: min=5016.1 avg=5713.2 p50=5632.3 p90=5982.2 p95=5985.3 max=6000.5
```

### 双卡，每张卡双模型实例

平均首包延迟在 3 s 内的极限是 60 并发

```
CUDA_VISIBLE_DEVICES=0 uvicorn server.app:app --host 0.0.0.0 --port 8002 --workers 2
```

```
CUDA_VISIBLE_DEVICES=6 uvicorn server.app:app --host 0.0.0.0 --port 8001 --workers 2
```

```
== STREAM (pcm) | total=40 | concurrency=40 ==
[  1/40] status=200 total_ms=6602.5 bytes=372480
[  2/40] status=200 total_ms=6926.6 bytes=401280
[  3/40] status=200 total_ms=7260.1 bytes=458880
[  4/40] status=200 total_ms=7615.7 bytes=487680
[  5/40] status=200 total_ms=7859.1 bytes=397440
[  6/40] status=200 total_ms=7917.3 bytes=476160
[  7/40] status=200 total_ms=8005.3 bytes=520320
[  8/40] status=200 total_ms=8207.6 bytes=428160
[  9/40] status=200 total_ms=8252.2 bytes=408960
[ 10/40] status=200 total_ms=9890.5 bytes=529920
[ 11/40] status=200 total_ms=9955.6 bytes=535680
[ 12/40] status=200 total_ms=9966.8 bytes=554880
[ 13/40] status=200 total_ms=10084.1 bytes=526080
[ 14/40] status=200 total_ms=11062.9 bytes=387840
[ 15/40] status=200 total_ms=11515.6 bytes=385920
[ 16/40] status=200 total_ms=12189.1 bytes=401280
[ 17/40] status=200 total_ms=12818.3 bytes=443520
[ 18/40] status=200 total_ms=12831.1 bytes=445440
[ 19/40] status=200 total_ms=12950.7 bytes=455040
[ 20/40] status=200 total_ms=12995.1 bytes=441600
[ 21/40] status=200 total_ms=13147.6 bytes=416640
[ 22/40] status=200 total_ms=13231.9 bytes=445440
[ 23/40] status=200 total_ms=13307.1 bytes=453120
[ 24/40] status=200 total_ms=13319.3 bytes=443520
[ 25/40] status=200 total_ms=13352.7 bytes=416640
[ 26/40] status=200 total_ms=13538.9 bytes=441600
[ 27/40] status=200 total_ms=13666.4 bytes=491520
[ 28/40] status=200 total_ms=13718.4 bytes=462720
[ 29/40] status=200 total_ms=13901.0 bytes=418560
[ 30/40] status=200 total_ms=14289.1 bytes=652800
[ 31/40] status=200 total_ms=14405.9 bytes=480000
[ 32/40] status=200 total_ms=14532.5 bytes=474240
[ 33/40] status=200 total_ms=14599.7 bytes=491520
[ 34/40] status=200 total_ms=14654.0 bytes=485760
[ 35/40] status=200 total_ms=14807.9 bytes=483840
[ 36/40] status=200 total_ms=15056.5 bytes=789120
[ 37/40] status=200 total_ms=15237.8 bytes=510720
[ 38/40] status=200 total_ms=15496.3 bytes=583680
[ 39/40] status=200 total_ms=15516.5 bytes=587520
[ 40/40] status=200 total_ms=16014.8 bytes=675840

-- Summary --
OK=40 FAIL=0  total_wall_ms=16023.7  RPS=2.50
TOTAL_MS: min=6602.5 avg=12017.5 p50=13071.3 p90=15074.6 p95=15497.3 max=16014.8
FIRST_BYTE_MS: min=1262.7 avg=2423.7 p50=2805.8 p90=2926.8 p95=2943.6 max=2958.3
```

```
== STREAM (pcm) | total=50 | concurrency=50 ==
[  1/50] status=200 total_ms=9114.7 bytes=412800
[  2/50] status=200 total_ms=10402.7 bytes=447360
[  3/50] status=200 total_ms=10433.0 bytes=443520
[  4/50] status=200 total_ms=10912.9 bytes=455040
[  5/50] status=200 total_ms=11069.4 bytes=468480
[  6/50] status=200 total_ms=11169.3 bytes=466560
[  7/50] status=200 total_ms=11295.9 bytes=360960
[  8/50] status=200 total_ms=11301.8 bytes=499200
[  9/50] status=200 total_ms=11509.4 bytes=518400
[ 10/50] status=200 total_ms=11674.5 bytes=512640
[ 11/50] status=200 total_ms=11970.2 bytes=360960
[ 12/50] status=200 total_ms=12563.9 bytes=362880
[ 13/50] status=200 total_ms=12794.1 bytes=389760
[ 14/50] status=200 total_ms=13084.3 bytes=403200
[ 15/50] status=200 total_ms=13324.2 bytes=407040
[ 16/50] status=200 total_ms=13436.8 bytes=407040
[ 17/50] status=200 total_ms=13461.9 bytes=405120
[ 18/50] status=200 total_ms=13683.7 bytes=460800
[ 19/50] status=200 total_ms=13942.8 bytes=410880
[ 20/50] status=200 total_ms=13991.0 bytes=439680
[ 21/50] status=200 total_ms=14277.2 bytes=408960
[ 22/50] status=200 total_ms=14354.1 bytes=451200
[ 23/50] status=200 total_ms=14366.9 bytes=451200
[ 24/50] status=200 total_ms=14585.7 bytes=435840
[ 25/50] status=200 total_ms=14628.6 bytes=443520
[ 26/50] status=200 total_ms=14706.7 bytes=443520
[ 27/50] status=200 total_ms=14740.2 bytes=503040
[ 28/50] status=200 total_ms=14982.9 bytes=439680
[ 29/50] status=200 total_ms=15011.6 bytes=445440
[ 30/50] status=200 total_ms=15046.4 bytes=441600
[ 31/50] status=200 total_ms=15344.3 bytes=458880
[ 32/50] status=200 total_ms=15353.9 bytes=506880
[ 33/50] status=200 total_ms=15360.8 bytes=462720
[ 34/50] status=200 total_ms=15531.5 bytes=464640
[ 35/50] status=200 total_ms=15569.6 bytes=489600
[ 36/50] status=200 total_ms=15772.8 bytes=508800
[ 37/50] status=200 total_ms=15809.9 bytes=510720
[ 38/50] status=200 total_ms=15820.2 bytes=506880
[ 39/50] status=200 total_ms=15825.1 bytes=472320
[ 40/50] status=200 total_ms=15827.6 bytes=547200
[ 41/50] status=200 total_ms=15861.8 bytes=510720
[ 42/50] status=200 total_ms=15883.8 bytes=468480
[ 43/50] status=200 total_ms=15917.2 bytes=503040
[ 44/50] status=200 total_ms=16023.7 bytes=529920
[ 45/50] status=200 total_ms=16027.0 bytes=520320
[ 46/50] status=200 total_ms=16104.3 bytes=566400
[ 47/50] status=200 total_ms=16161.3 bytes=524160
[ 48/50] status=200 total_ms=16267.9 bytes=545280
[ 49/50] status=200 total_ms=16349.4 bytes=579840
[ 50/50] status=200 total_ms=19722.2 bytes=1228800

-- Summary --
OK=50 FAIL=0  total_wall_ms=19729.7  RPS=2.53
TOTAL_MS: min=9114.7 avg=14167.4 p50=14667.6 p90=16034.7 p95=16219.9 max=19722.2
FIRST_BYTE_MS: min=2290.4 avg=2839.2 p50=2900.6 p90=3125.2 p95=3149.3 max=3165.1
```

```
== STREAM (pcm) | total=60 | concurrency=60 ==
[  1/60] status=200 total_ms=12846.0 bytes=382080
[  2/60] status=200 total_ms=12898.9 bytes=408960
[  3/60] status=200 total_ms=13021.8 bytes=403200
[  4/60] status=200 total_ms=13250.4 bytes=395520
[  5/60] status=200 total_ms=13559.2 bytes=351360
[  6/60] status=200 total_ms=13681.5 bytes=366720
[  7/60] status=200 total_ms=14298.9 bytes=384000
[  8/60] status=200 total_ms=14350.1 bytes=432000
[  9/60] status=200 total_ms=14455.7 bytes=387840
[ 10/60] status=200 total_ms=14648.7 bytes=433920
[ 11/60] status=200 total_ms=14647.9 bytes=410880
[ 12/60] status=200 total_ms=14660.1 bytes=460800
[ 13/60] status=200 total_ms=14867.5 bytes=410880
[ 14/60] status=200 total_ms=14888.6 bytes=422400
[ 15/60] status=200 total_ms=14927.4 bytes=376320
[ 16/60] status=200 total_ms=14967.6 bytes=384000
[ 17/60] status=200 total_ms=15177.5 bytes=405120
[ 18/60] status=200 total_ms=15216.4 bytes=407040
[ 19/60] status=200 total_ms=15300.6 bytes=385920
[ 20/60] status=200 total_ms=15418.2 bytes=508800
[ 21/60] status=200 total_ms=15801.3 bytes=491520
[ 22/60] status=200 total_ms=15808.8 bytes=478080
[ 23/60] status=200 total_ms=16023.6 bytes=432000
[ 24/60] status=200 total_ms=16046.7 bytes=439680
[ 25/60] status=200 total_ms=16125.2 bytes=443520
[ 26/60] status=200 total_ms=16172.1 bytes=445440
[ 27/60] status=200 total_ms=16232.2 bytes=426240
[ 28/60] status=200 total_ms=16279.3 bytes=432000
[ 29/60] status=200 total_ms=16408.9 bytes=430080
[ 30/60] status=200 total_ms=16403.5 bytes=556800
[ 31/60] status=200 total_ms=16423.3 bytes=458880
[ 32/60] status=200 total_ms=16461.9 bytes=441600
[ 33/60] status=200 total_ms=16481.8 bytes=458880
[ 34/60] status=200 total_ms=16480.8 bytes=460800
[ 35/60] status=200 total_ms=16634.1 bytes=414720
[ 36/60] status=200 total_ms=16645.7 bytes=572160
[ 37/60] status=200 total_ms=16712.5 bytes=437760
[ 38/60] status=200 total_ms=16842.0 bytes=478080
[ 39/60] status=200 total_ms=16859.4 bytes=579840
[ 40/60] status=200 total_ms=16944.2 bytes=474240
[ 41/60] status=200 total_ms=17001.5 bytes=481920
[ 42/60] status=200 total_ms=17002.2 bytes=606720
[ 43/60] status=200 total_ms=17163.2 bytes=493440
[ 44/60] status=200 total_ms=17223.8 bytes=506880
[ 45/60] status=200 total_ms=17303.2 bytes=478080
[ 46/60] status=200 total_ms=17391.9 bytes=474240
[ 47/60] status=200 total_ms=17422.4 bytes=474240
[ 48/60] status=200 total_ms=17454.8 bytes=487680
[ 49/60] status=200 total_ms=17498.5 bytes=483840
[ 50/60] status=200 total_ms=17519.4 bytes=466560
[ 51/60] status=200 total_ms=17539.1 bytes=600960
[ 52/60] status=200 total_ms=17746.8 bytes=520320
[ 53/60] status=200 total_ms=17979.3 bytes=524160
[ 54/60] status=200 total_ms=18413.8 bytes=606720
[ 55/60] status=200 total_ms=18614.5 bytes=514560
[ 56/60] status=200 total_ms=18660.8 bytes=529920
[ 57/60] status=200 total_ms=18695.3 bytes=547200
[ 58/60] status=200 total_ms=18908.9 bytes=591360
[ 59/60] status=200 total_ms=19022.4 bytes=570240
[ 60/60] status=200 total_ms=19110.9 bytes=620160

-- Summary --
OK=60 FAIL=0  total_wall_ms=19129.7  RPS=3.14
TOTAL_MS: min=12846.0 avg=16209.0 p50=16416.1 p90=18433.9 p95=18705.9 max=19110.9
FIRST_BYTE_MS: min=2823.9 avg=3193.6 p50=3215.1 p90=3285.9 p95=3316.6 max=3329.2
```

```
== STREAM (pcm) | total=70 | concurrency=70 ==
[  1/70] status=200 total_ms=3201.5 bytes=362880
[  2/70] status=200 total_ms=3326.6 bytes=385920
[  3/70] status=200 total_ms=3632.4 bytes=422400
[  4/70] status=200 total_ms=15415.8 bytes=407040
[  5/70] status=200 total_ms=15411.5 bytes=343680
[  6/70] status=200 total_ms=15494.6 bytes=364800
[  7/70] status=200 total_ms=15897.4 bytes=378240
[  8/70] status=200 total_ms=16239.5 bytes=384000
[  9/70] status=200 total_ms=17063.8 bytes=384000
[ 10/70] status=200 total_ms=17256.1 bytes=432000
[ 11/70] status=200 total_ms=17351.0 bytes=445440
[ 12/70] status=200 total_ms=17421.7 bytes=428160
[ 13/70] status=200 total_ms=18003.1 bytes=432000
[ 14/70] status=200 total_ms=18327.5 bytes=449280
[ 15/70] status=200 total_ms=18356.4 bytes=460800
[ 16/70] status=200 total_ms=18383.9 bytes=418560
[ 17/70] status=200 total_ms=18541.9 bytes=430080
[ 18/70] status=200 total_ms=18863.0 bytes=422400
[ 19/70] status=200 total_ms=18974.1 bytes=428160
[ 20/70] status=200 total_ms=19058.2 bytes=418560
[ 21/70] status=200 total_ms=19203.1 bytes=468480
[ 22/70] status=200 total_ms=19271.8 bytes=466560
[ 23/70] status=200 total_ms=19513.9 bytes=347520
[ 24/70] status=200 total_ms=19538.6 bytes=418560
[ 25/70] status=200 total_ms=19547.5 bytes=487680
[ 26/70] status=200 total_ms=19589.7 bytes=464640
[ 27/70] status=200 total_ms=19813.2 bytes=510720
[ 28/70] status=200 total_ms=20059.8 bytes=470400
[ 29/70] status=200 total_ms=20107.4 bytes=547200
[ 30/70] status=200 total_ms=20320.9 bytes=489600
[ 31/70] status=200 total_ms=20356.7 bytes=504960
[ 32/70] status=200 total_ms=20444.3 bytes=474240
[ 33/70] status=200 total_ms=20594.0 bytes=508800
[ 34/70] status=200 total_ms=20664.1 bytes=633600
[ 35/70] status=200 total_ms=20669.9 bytes=355200
[ 36/70] status=200 total_ms=20788.7 bytes=625920
[ 37/70] status=200 total_ms=20965.9 bytes=510720
[ 38/70] status=200 total_ms=21187.9 bytes=577920
[ 39/70] status=200 total_ms=21364.7 bytes=560640
[ 40/70] status=200 total_ms=21621.2 bytes=405120
[ 41/70] status=200 total_ms=21706.5 bytes=693120
[ 42/70] status=200 total_ms=22594.9 bytes=397440
[ 43/70] status=200 total_ms=23103.2 bytes=378240
[ 44/70] status=200 total_ms=23137.3 bytes=395520
[ 45/70] status=200 total_ms=25919.1 bytes=418560
[ 46/70] status=200 total_ms=25996.0 bytes=428160
[ 47/70] status=200 total_ms=26006.7 bytes=420480
[ 48/70] status=200 total_ms=26412.5 bytes=445440
[ 49/70] status=200 total_ms=26433.4 bytes=445440
[ 50/70] status=200 total_ms=26492.3 bytes=422400
[ 51/70] status=200 total_ms=26517.1 bytes=437760
[ 52/70] status=200 total_ms=26561.7 bytes=414720
[ 53/70] status=200 total_ms=26660.3 bytes=443520
[ 54/70] status=200 total_ms=26864.1 bytes=422400
[ 55/70] status=200 total_ms=26922.0 bytes=499200
[ 56/70] status=200 total_ms=27775.9 bytes=478080
[ 57/70] status=200 total_ms=28147.9 bytes=472320
[ 58/70] status=200 total_ms=28198.5 bytes=495360
[ 59/70] status=200 total_ms=28221.1 bytes=472320
[ 60/70] status=200 total_ms=28404.0 bytes=464640
[ 61/70] status=200 total_ms=28441.4 bytes=504960
[ 62/70] status=200 total_ms=28442.1 bytes=478080
[ 63/70] status=200 total_ms=28997.0 bytes=514560
[ 64/70] status=200 total_ms=29115.8 bytes=529920
[ 65/70] status=200 total_ms=29198.0 bytes=545280
[ 66/70] status=200 total_ms=29242.2 bytes=539520
[ 67/70] status=200 total_ms=29377.0 bytes=537600
[ 68/70] status=200 total_ms=29413.7 bytes=528000
[ 69/70] status=200 total_ms=29442.4 bytes=545280
[ 70/70] status=200 total_ms=29488.2 bytes=600960

-- Summary --
OK=70 FAIL=0  total_wall_ms=29488.7  RPS=2.37
TOTAL_MS: min=3201.5 avg=21786.8 p50=20729.3 p90=29008.9 p95=29316.3 max=29488.2
FIRST_BYTE_MS: min=1358.4 avg=4303.6 p50=3830.5 p90=5584.2 p95=5601.0 max=5613.6
```

### 双卡，每卡四模型实例

拉的一批，并发 70 直接平均首包延迟卡死了都破十了，然后一看显存直接到 90 了，可以看出来足够的显存剩余对计算还是很重要的。60 并发也卡死。

