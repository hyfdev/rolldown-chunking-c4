# rolldown-chunking-c4

Rolldown 的 chunk 生成流程，两个版本画在一起：今天的实现，和分层之后的实现。
模型用 [LikeC4](https://likec4.dev) 写，图上的文字是英文。

今天那一版对照 rolldown `main`，commit `6a54a0531`。

## 看图

```bash
npm install     # 首次
npm run dev     # http://localhost:5173
```

改 `src/*.c4` 会热更新。点节点看完整说明——节点框里的文字会被截断，说明只在侧栏完整。

## 两个视图

| 视图 | 内容 |
| --- | --- |
| `index` | 上下对比：上面今天，下面分层之后，都从左往右 |
| `proposal_only` | 只看分层之后的流程 |

## 图上的颜色

| 颜色 | 标记 | 含义 |
| --- | --- | --- |
| 红 | `back-edge` | 跨了分层。两种形态：后面的步骤使前面的失效或反过来约束它（箭头往回指），或者可选优化喂给强制放置（箭头往前指） |
| 橙 | `own-judgment` | 自己回答"哪个 chunk import 哪个 chunk"，不读同一份定义；或者判断的范围小于它改动的范围 |
| 橙 | `direct-write` | 直接在共享状态上赋值，没有值表示"一次修改" |
| 绿 | `bounded-loop` | 有回路，但是局部的：单调、有上界、中间态不外流 |

标题里的 `(strict)` 表示这一步只在 `output.strictExecutionOrder` 打开时存在，而它默认是关的。

## 一眼要看到的

**上面一行**：四个橙色小方块是四份各自为政的答案——`avoidRedundantChunkLoads` 自己的 atom 图、
`mergeCommonChunks` 自己的临时图、runtime 并入的按需遍历、lowering 之后的真实边。
共用的那份 `ChunkLoadGraph` 是最右边一个节点，封存之后才算出来。
圆柱形那个节点收着十条写入边：每个写入者都直接在它上面赋值。

**下面一行**：没有跨分层的东西。`ChunkLoadGraph` 紧跟在放置之后算出来，所有判断都读它。
每次改动都提给同一个判断，唯一往回指的箭头是那个判断把通过的改动落到 placement 上（绿色）。

同一个概念在两行用同一个名字，位置不同——这就是要看的东西。比如 `Compute load sets` 和
`Automatic code splitting`：今天那行两者之间夹着三样东西，其中 `avoidRedundantChunkLoads` 是可选的
而且改写了 load set；分层之后两者相邻，四条放置规则并列进同一次划分，manual 对它匹配到的模块优先。

## 文件

```
src/spec.c4        记号：元素种类、标记、关系种类
src/1-today.c4     今天的流程
src/2-proposal.c4  分层之后的流程
src/views.c4       两个视图
```

`index` 视图里 `include` 的顺序是反的（`proposed.*, now.*`），因为渲染时后列的排在上面。

## 其它命令

```bash
npm run validate   # 语法与引用检查
npm run fmt        # 格式化
npm run png        # 导出 out/*.png
```

`npm run png` 用 headless chromium 渲染，首次要先装浏览器：
`./node_modules/.bin/playwright install chromium-headless-shell`。
它的画布在 16384px 处截断。`index` 视图用 `size sm` 把节点画小，内容宽度约 14.2k，
还有两列左右的余量；再加节点接近这条线时，导出的图右端会被切，`npm run dev` 不受限制。
`npm run dev` 没有这个限制。
