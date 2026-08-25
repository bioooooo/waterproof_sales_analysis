# Matplotlib / Seaborn 图表规范（MyEDA 项目验证版）

> 本套规范来自建材销售 EDA 项目（Sales_analysis.ipynb）全部 20+ 张图的实际落地，已通过 PIL 像素验证与 SimHei 警告检查。新项目直接复制对应模板即可。

## 1. 全局配置（每个 notebook 开头执行一次）

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams["font.family"] = ["SimHei"]     # 中文字体（Windows 自带）
plt.rcParams["axes.unicode_minus"] = False   # 负号正常显示（否则显示为方块）
```

> ⚠️ Matplotlib 3.11：SimHei 无粗体，不要依赖 `fontweight='bold'`。
> ⚠️ 验证模式（排查字体问题时用）：`warnings.simplefilter('error', UserWarning)`——SimHei 缺失警告直接变报错，脚本跑通即代表字体没问题。

## 2. 调色板（固定 9 色，语义化使用）

| 变量 | 色值 | 用途 |
|---|---|---|
| 蓝 | `#2a78d6` | 主系列、常规柱、正向 |
| 深蓝 | `#1b4f96` | 强调（TOP1）、深色文字标签 |
| 橙 | `#e3963e` | 对比系列（第二组）、新客、辅助折线 |
| 深橙 | `#b36b1e` | 橙色系文字标签、注释 |
| 深红 | `#b3371e` | 负值、警示、流失 |
| ink | `#0b0b0b` | 主文字（标题、关键数字） |
| secondary | `#52514e` | 次要文字、注释、刻度 |
| axis | `#c3c2b7` | 坐标轴边框、灰色系系列线 |
| grid | `#e1e0d9` | 网格线 |
| surface | `#fcfcfb` | 图表背景 |

## 3. 通用图骨架（每张图的基础）

```python
fig, ax = plt.subplots(figsize=(13, 6.5), facecolor='#fcfcfb')
ax.set_facecolor('#fcfcfb')

# —— 坐标轴：去上右框、低调边框、仅横向网格 ——
ax.spines['top'].set_visible(False)
ax.spines['right'].set_visible(False)
ax.spines['left'].set_color('#c3c2b7')
ax.spines['bottom'].set_color('#c3c2b7')
ax.grid(axis='y', color='#e1e0d9', lw=0.8)   # 横向条形图改 axis='x'
ax.set_axisbelow(True)                        # 网格垫在数据下层
ax.tick_params(labelsize=10)

# —— 标题 ——
ax.set_title('标题（含单位与口径，如：万元、%、对数刻度）', fontsize=14, pad=14)
ax.set_xlabel('...', fontsize=11)
ax.set_ylabel('...', fontsize=11)

plt.tight_layout()
plt.show()
```

**原则**：背景统一 `#fcfcfb`；去 top/right 边框；网格只留横向且低调；**关键数字必须直接标注在图上**（不让读者自己读坐标轴）。

## 4. 常用图型模板

### 4.1 折线图（时间序列）

```python
ax.plot(years, vals, color='#2a78d6', lw=2, marker='o', ms=6, zorder=3)
# 每个点上方标注数值（峰值用更大字号）
for y, v in zip(years, vals):
    ax.annotate(f'{v:.1f}', (y, v), xytext=(0, 9), textcoords='offset points',
                ha='center', fontsize=10.5 if v == vals.max() else 9.5, color='#0b0b0b')
# 峰值注释（箭头指向）
ax.annotate('历史峰值', xy=(peak_y, peak_v), xytext=(peak_y - 1.5, peak_v + 1.1),
            fontsize=10, color='#52514e', ha='center',
            arrowprops=dict(arrowstyle='->', color='#898781', lw=1))
ax.set_ylim(0, vals.max() * 1.15)   # 顶部为标注留空间
```

多序列对比：主序列深蓝粗线（`#1b4f96, lw=2.8`），次序列橙色（`#e3963e, lw=2.0`），其余灰色系（`#c3c2b7, lw=1.4`），图例 `frameon=False`。

### 4.2 柱状图

```python
bars = ax.bar(labs, pct, color='#2a78d6', edgecolor='#fcfcfb', lw=0.5, width=0.58)
for b, p in zip(bars, pct):
    ax.annotate(f'{p:.1f}%', (b.get_x() + b.get_width() / 2, p),
                xytext=(0, 4), textcoords='offset points', ha='center',
                fontsize=9, color='#0b0b0b')
ax.set_ylim(0, pct.max() * 1.2)     # 为柱顶标签留空间
```

### 4.3 横向条形图（品类/排名类）

```python
# 排序：底部最大（升序后 barh）
data = data.sort_values('值', ascending=True)
ax.barh(range(len(data)), data['值'], color=['#1b4f96' if TOP1 else '#2a78d6' ...], height=0.62)
for i, v in enumerate(data['值']):
    ax.text(v + 5, i, f'{v:.1f}', va='center', fontsize=10, color='#0b0b0b')
ax.set_yticks(range(len(data))); ax.set_yticklabels(data.index, fontsize=10)
ax.margins(x=0.2)                   # 长标签/全角字符防溢出（横向条形必备）
```

### 4.4 双条对比（两组效应，如数量效应 vs 价格效应）

```python
y = np.arange(len(df)); h = 0.36
ax.barh(y + h / 2, df['A'], height=h, color='#2a78d6', label='A（口径说明）')
ax.barh(y - h / 2, df['B'], height=h, color='#e3963e', label='B（口径说明）')
# 标签：|值| ≥ 阈值才标注，正负值左右错开
for i, row in df.iterrows():
    if abs(row['A']) >= 3:
        ax.text(row['A'] + (5 if row['A'] >= 0 else -5), i + h / 2, f"{row['A']:.0f}",
                va='center', ha='left' if row['A'] >= 0 else 'right', fontsize=8, color='#1b4f96')
ax.axvline(0, color='#898781', lw=1)
ax.legend(fontsize=9.5, frameon=False, loc='lower right')
ax.margins(x=0.25)
```

### 4.5 热力图（相关性/留存率/品类季节）

```python
# 方式一：seaborn（矩阵标注）
sns.heatmap(overlap, annot=True, fmt='.0f', cmap='Blues', vmin=0, vmax=100,
            square=True, linewidths=0.5, linecolor='#fcfcfb',
            cbar_kws={'label': '...（%）'}, ax=ax)
ax.set_xticklabels(ax.get_xticklabels(), rotation=45, ha='right')

# 方式二：imshow（手动控制格内文字颜色）
im = ax.imshow(data, cmap='Blues', vmin=0, vmax=50, aspect='auto')
for i in range(n):
    for j in range(m):
        v = data[i, j]
        ax.text(j, i, f'{v:.0f}', ha='center', va='center', fontsize=9,
                color='white' if v > 55 else '#0b0b0b')   # 深色格子白字
fig.colorbar(im, ax=ax, shrink=0.8, label='... %')
```

### 4.6 气泡图（散点 + 象限分割）

```python
ax.scatter(x, y, s=10 + 6 * np.sqrt(weight), c='#2a78d6', alpha=0.55, zorder=3)
ax.axvline(thr, color='#898781', ls='--', lw=1.2, zorder=2)   # 分割线
ax.axhline(ovr, color='#898781', ls='--', lw=1.2, zorder=2)
# 象限说明文字：四角 ax.text(...)
ax.set_xscale('log')   # 长尾数据用对数刻度 + 显式 set_xticks/set_xticklabels
# 图例用 Line2D 手动构造（避免散点 legend 圆点过小）
handles = [plt.Line2D([0], [0], marker='o', color='w', markerfacecolor=c, ms=7, label=l)
           for l, c in colors.items()]
ax.legend(handles=handles, fontsize=10, frameon=False, loc='upper left')
```

### 4.7 瀑布桥图（增量分解）

```python
steps = [('起点', v0, None), ('流失', -v1, 'lost'), ('新增', +v2, 'new'), ('终点', v3, None)]
vals = [s[1] for s in steps]
cum = np.cumsum([0] + vals[:-1])          # 每段的底部 = 之前累积
for i, (v, c0) in enumerate(zip(vals, cum)):
    ax.bar(i, v, bottom=c0, color='#2a78d6' if v >= 0 else '#b3371e', width=0.62, zorder=3)
    if i in (0, len(vals) - 1):           # 端点：柱顶外深色标签
        ax.text(i, c0 + v + 18, f'{v:.0f}', ha='center', va='bottom', fontsize=11, color='#0b0b0b')
    else:                                 # 中间增量：柱内白字
        ax.text(i, c0 + v / 2, f'{v:+.0f}', ha='center', va='center', fontsize=10.5, color='white')
```

### 4.8 堆叠柱状图（构成变化）

```python
ax.bar(x, 存量, color='#2a78d6', label='存量')
ax.bar(x, 新增, bottom=存量, color='#e3963e', label='新增')   # bottom= 堆叠
# 条件标注：占比 > 12% 才在段内写占比文字（避免小段文字重叠）
if 新增/总量 > 0.12:
    ax.text(i, 存量[i] + 新增[i]/2, f'{占比:.0%}', ha='center', va='center', fontsize=8, color='#b36b1e')
```

### 4.9 直方图（金额等长尾分布）

```python
# 金额横跨 0~26 万，线性分箱会挤成一根柱 → 对数分箱
bins = np.logspace(np.log10(s[s > 0].min()), np.log10(s.max()), 50)
ax.hist(s[s > 0], bins=bins, color='#2a78d6', edgecolor='#fcfcfb', lw=0.5)
ax.set_xscale('log')
ax.set_xlabel('金额（元，对数刻度）')
# 中位数/均值参考线 + 标注
ax.axvline(med, color='#898781', ls='--', lw=1)
ax.annotate(f'中位数 {med:.0f} 元', xy=(med, top), xytext=(6, 0), textcoords='offset points',
            fontsize=10, color='#52514e', va='top')
```

### 4.10 明细表（替代图表的表格展示）

```python
from IPython.display import display
display(df.style.format({'营收万': '{:,.0f}', '占比%': '{:.1f}'})
        .bar(subset=['占比%'], color='#2a78d6')           # 单元格色条
        .apply(lambda x: ['font-weight: bold' if x.name == len(df) - 1 else '' for _ in x]))  # 合计行加粗
```

## 5. 交付前验证清单

1. **无 SimHei 警告**：脚本头部 `warnings.simplefilter('error', UserWarning)`，跑通即字体 OK
2. **图不越界**：长标签加 `ax.margins(x=0.2~0.25)`；`plt.tight_layout()` 必写
3. **PIL 像素验证**（可选，批量出图时用）：

```python
from PIL import Image
img = np.array(Image.open(path).convert('RGB'))
bg = ((np.abs(img - np.array([252, 252, 251])) < 8).all(axis=2)).mean()   # 背景占比应 > 85%
blue = ((np.abs(img[:, :, 0].astype(int) - 42) < 30) & (np.abs(img[:, :, 1].astype(int) - 120) < 30)
        & (np.abs(img[:, :, 2].astype(int) - 214) < 30)).mean()            # 主色存在
nonbg = ~((np.abs(img - np.array([252, 252, 251])) < 8).all(axis=2))
edge = np.concatenate([nonbg[:4].ravel(), nonbg[-4:].ravel(), nonbg[:, :4].ravel(), nonbg[:, -4:].ravel()])
print(f'背景 {bg:.1%}，主色 {blue:.3%}，边缘溢出 {edge.sum()}（应为 0）')
```

4. **保存图**：`plt.savefig(path, dpi=110)`（验证脚本用 `matplotlib.use('Agg')` 免弹窗）

## 6. Windows / 中文注意事项

- **控制台 GBK 乱码**：脚本开头 `import sys, io; sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')`
- **字体**：SimHei（Windows 自带）；macOS/Linux 需换 `PingFang SC` / `Noto Sans CJK SC` 并确认系统已装
- **全角字符比半角宽**：中文标签 + 负值 label 特别容易溢出，横向图默认加 `margins(x=0.2)`
- **数字口径写进标题**：如"（万元）""（对数刻度）""（客户口径，排除零售）"——读者不需要猜
