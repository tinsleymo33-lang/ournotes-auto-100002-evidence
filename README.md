# OurNotes-Auto · 100002 壱雫空 EXPERT 同步偏移取证材料

对应 issue：**壱雫空 EXPERT（100002）每局提前约 1.14 秒下按、7~10 秒生命值归零：小节线采样点不足导致过滤被静默关闭**
→ https://github.com/ChissaQAQ/OurNotes-Auto/issues （发帖后此链接会更新）

环境：OurNotes-Auto 0.2.3.2 / Windows 11 / MuMu 12 6.8.7.0 实例 1 / Android 15 / 1280×720 60fps / minitouch。

## 文件

| 文件 | 内容 |
|---|---|
| `tracked-object.png` | 同步求解那一帧的标注图：真音符（橙色 FLICK，距判定线 1.341 s）vs 被跟踪的中性灰小节线（game y≈439，距判定线 0.203 s），差 1.138 s |
| `death-202620.png` | 生命值归零瞬间原图（`debug/life/`） |
| `death-194424.png` | 更早一次归零瞬间 |
| `before-fix-excerpt.log` | 修复前 26 次归零的连续日志（2026-11-02 20:14~20:26） |
| `after-fix-excerpt.log` | 修复后 18 局 AP 的同步日志（21:02~21:47） |
| `ournotes.log` | 完整日志 |
| `evidence.json` | 全部回放数值、几何常量、采样点分布 |
| `fix-bar-line-probes.patch` | 修法：`_BAR_MIN_PROBES` 8 → 5 并补 debug 日志（已验证可正/反向应用到 0.2.3.2） |
| `sync-dump-202205.npz` | `debug/sync/` 里出问题那局的同步 dump（`crops`/`times`/`box`/`spans`/`first_ms`/`arrival`，120 帧） |

## 一句话结论

`100002_expert` 的 `first_hits()` 是 `[0,8] + [14,22]`，让开 `_BAR_GAP=1.5` 后 `_BAR_UNITS` 里只剩 `9.75~12.25` 六个采样点，低于 `_BAR_MIN_PROBES = 8`，小节线过滤被**静默**关闭 → 跟踪器锁到横贯全屏的中性灰小节线 → `t0` 早 1.138 s → 整首提前按、数秒归零。而拟合残差只有 2~5 ms，日志看不出异常。

## 复现

```python
import numpy as np
from dataclasses import replace
from ournotes_auto.config import load_config
from ournotes_auto.geometry import Geometry
from ournotes_auto.charts.model import Span
from ournotes_auto.player.sync import NoteTracker

cfg = load_config('config.yaml'); geo = Geometry(1280, 720, cfg.geometry)
d = np.load('sync-dump-202205.npz')
spans = [Span(float(a), float(b)) for a, b in d['spans']]
y0, y1, x0, x1 = (int(v) for v in d['box'])
tk = NoteTracker(geo, replace(cfg.play.sync, record_frames=0), float(d['first_ms']), spans)
crops = d['crops']
if crops.shape[2] == x1 - x0 and tk._probe_x is not None:
    px, py = tk._probe_x - x0, tk._probe_y - y0
    if px.min() >= 0 and px.max() < x1 - x0:
        crops = np.concatenate([crops, crops[:, py, px]], axis=2)
tk._baseline = np.median(crops, axis=0).astype(np.float32)
for t, c in zip(d['times'], crops):
    out = tk.feed_crop(c.astype(np.int16), float(t))
    if out is not None:
        print(len(out.samples), out.rms_ms, out.sigma_ms, out.ok, out.t0); break
```

`_BAR_MIN_PROBES` 为 8 时得到 `36 5.x 1.8 True 9333.138`（跟踪到小节线）；改成 5 后得到 `32 … 9334.276`（跟踪到 FLICK 本体下边缘）。差值 1138 ms。
