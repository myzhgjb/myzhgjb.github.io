---
title: 308 家企业、1406 个岗位：我把一场招聘会的岗位表做成了一个能检索的库
date: 2026-11-11
tags: [前端, 数据可视化, 求职, JavaScript, 效率工具]
summary: 主办方发来一份 6 列的 Excel，1382 行。三小时后它变成了一个手机能打开、断网能用的岗位检索库。
cover: https://myzhgjb.github.io/suda-1111-jobfair/docs/shots/library-desktop.png
---

上周去苏州大学参加「2026 校园苏州日」秋招专场，主办方在群里丢了一份 `岗位需求.xlsx`。

308 家企业，1377 行岗位记录。打开一看，6 列：

| 单位名称 | 单位简介 | 岗位名称 | 专业要求 | 学历要求 | 需求人数 |
|---|---|---|---|---|---|

没有企业规模，没有岗位 JD，没有联系人，没有展位号。**这就是一场招聘会能给你的全部信息。**

现场只有 2 小时 50 分钟（14:40–17:30），308 家企业。靠翻 Excel 是不可能看完的，所以我花了三个小时把它做成了一个库。

## 最后做出来的三个东西

**① 全量检索库**：308 家企业 / 1406 个岗位，左侧 20 个类别点一下就筛，支持搜索企业名 / 岗位 / 专业。

![企业岗位全库](https://myzhgjb.github.io/suda-1111-jobfair/docs/shots/library-desktop.png)

**② 岗位视图**：切成表格，一屏能扫十几行，标注类别、专业、学历、人数。

![岗位视图](https://myzhgjb.github.io/suda-1111-jobfair/docs/shots/library-jobs.png)

**③ 手机投递清单**：PWA，装到桌面像原生 App，投完打勾，状态存本地，走的时候一键导出。

<img src="https://myzhgjb.github.io/suda-1111-jobfair/docs/shots/pwa-mobile.png" width="330">

## 几个真正踩到坑的地方

### 1. 合并单元格会把数据吃掉

Excel 里「单位名称」是合并单元格，**只有每家企业第一行有值**。直接读出来就是 308 个名字 + 1000 多个空白。

必须向下填充：

```python
cur = None
for r in rows:
    name = clean(r[0])
    if name and (cur is None or name != cur['n']):
        cur = {'n': name, 'jobs': []}
    if not r[2]:        # 岗位名称为空就跳过
        continue
    cur['jobs'].append(...)
```

判断「是不是新企业」不能只看 `name` 非空——因为合并单元格填充后，第二行开始 `name` 就是空的。用「名字变了」作为切换条件才对。

### 2. 一格塞了 23 个岗位

有的企业把整个招聘计划塞进一个单元格：

```
世界模型算法工程师 基座模型算法工程师 具身感知算法工程师 VLA算法工程师 SLAM算法工程师 ...
```

按 `、` `,` `;` 拆，但**括号里的逗号不能拆**，否则「质量工程师（苏州、嘉兴）」会被劈成两条。

所以写了个带括号深度的拆分器：

```python
def split_out(t):
    out, buf, depth = [], [], 0
    for ch in t:
        if ch in '（(【[':  depth += 1; buf.append(ch)
        elif ch in '）)】]': depth = max(0, depth-1); buf.append(ch)
        elif ch in '、，,;；\n' and depth == 0:
            out.append(''.join(buf).strip()); buf = []
        else: buf.append(ch)
```

### 3. 数据体积：688KB → 388KB

第一版我把每个岗位的技术要求都完整写进 JSON，结果数据文件 688KB，HTML 接近 800KB，手机加载会卡。

改成**模板化**：16 个类别各存一份「职责描述 + 5 个技能点」，岗位只存一个类别 id 和一个变体 id（0=默认 / 1=Agent / 2=前端 / 3=后端 / 4=数据）。

```json
{ "cats": [...], "tpl": {"软件研发": ["描述", ["技能1","技能2"]]},
  "c": [{"n":"公司名","j":[{"j":"岗位","c":7,"v":3}]}] }
```

同样一份数据，388KB，页面渲染逻辑不变。

### 4. 手机端 390px 的横向溢出

页面在电脑上好好的，手机上右边被切掉一块。

排查下来是 Grid 的锅：`grid-template-columns: 236px 1fr` 里的 `1fr` 最小值是 `auto`，也就是**内容的 min-content 宽度**。侧栏里有一排横向滚动的类别标签，它的 min-content 把整列撑到了 2000 多像素。

```css
.layout { grid-template-columns: 236px minmax(0, 1fr); }
.layout > div { min-width: 0; }
aside { min-width: 0; }
```

三行 `min-width: 0` 解决。验证方法不是看截图，是量：

```js
document.body.scrollWidth === document.documentElement.clientWidth  // true 才算过
```

### 5. 无头截图骗了我半小时

用 Chrome `--headless --window-size=390,880` 截图，右边总是缺一块，我以为页面又溢出了。

实际上 Windows 显示缩放 125% 时，**`--window-size` 是设备像素，CSS 视口不是 390**。测量一下：

```js
document.documentElement.clientWidth  // 489，不是 390
```

后来改成「外面套一个 390px 的 iframe，再截图裁切」，才拿到真正的 390 CSS 像素视图。

**教训：不要拿截图宽度当视口宽度，永远用 `clientWidth` 量。**

## 数据诚实性：没有的字段不要编

这份表没有企业规模，也没有 JD。做页面时最容易犯的错就是「补一个看起来合理的」。

我的处理是：

- 企业规模：只有 24 家能从简介里正则读到「员工 XXX 人」，其余一律标**「规模未披露」**
- 岗位技术要求：按类别模板 + 岗位名推断，页面顶部明写**「推断，不是官方 JD，投递前以现场沟通为准」**
- 行业：按企业名称与简介推断，注明**不是官方展区划分**

少写几个字段不会有人骂你，编错了会有人按着你写的去准备面试。

## 代码

- 数据 + 页面：<https://github.com/myzhgjb/suda-1111-jobfair>
- 在线看：<https://myzhgjb.github.io/suda-1111-jobfair/>
- 生成这套东西的 Skill：<https://github.com/myzhgjb/jobfair-job-screening>

原始 CSV 也在仓库里（`data/` 三个文件：原始表 / 企业表 / 岗位表），下一步打算把「展位路线」也补进去。
