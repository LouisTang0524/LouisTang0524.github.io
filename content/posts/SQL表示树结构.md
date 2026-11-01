+++
date = '2026-11-01T19:38:38+08:00'
draft = false
title = 'SQL表示树结构'
+++

关系型数据库没有"树"这种数据类型，只有表和行。但在业务建模里，树几乎无处不在——尤其是公司组织架构：集团、事业部、部门、小组层层嵌套，员工之上还有一条"谁向谁汇报"的链。

于是"用 SQL 表示树"最终都会落到两件事上：**怎么存父子关系**，以及**怎么在一条查询里把这条关系走到底**。本文以公司人员组织系统为线索，依次讨论邻接表设计、递归 CTE 的写法与执行原理，以及物化路径、闭包表两种替代方案，最后给出"部门归属"与"人员汇报"两条树同时维护的工程方案。

## 一、问题背景

先看一个典型的组织结构：

```text
集团总部 (1)
├── 技术中心 (2)
│   ├── 平台组 (4)
│   └── 业务组 (5)
│       └── 数据小组 (7)
└── 市场中心 (3)
    └── 品牌组 (6)
```

围绕这棵树，业务系统至少要回答五类问题：

1. 某个部门下面**所有层级**的子部门有哪些？（数据权限、成本归集）
2. 某个部门的**完整路径**是什么？（面包屑、审批流转）
3. 某个人的**所有下属**（直接 + 间接）有哪些？（审批链、绩效）
4. 按部门**层级汇总**人数、编制、人力成本（报表）
5. 组织调整：部门合并、拆分、整体搬迁到另一个中心下

难点也很明确：

- 一条普通 `SELECT` 只能取到"一层"，无法表达不确定深度的链路；
- 层级数不固定，写不出固定数量的自连接（`JOIN` 到第三层就到头了）；
- 组织结构是**会变的**，方案必须让"移动子树"足够便宜，否则每次调整都要停机刷数据。

## 二、邻接表：一行一个父指针

最自然的建模是**邻接表（Adjacency List）**：每个节点存一个指向父节点的引用。

```sql
CREATE TABLE department (
    id        bigint      PRIMARY KEY,
    parent_id bigint      NULL REFERENCES department(id),
    name      varchar(64) NOT NULL,
    status    varchar(16) NOT NULL DEFAULT 'active'
);

CREATE INDEX idx_department_parent ON department (parent_id);
```

几个设计要点：

- `parent_id IS NULL` 表示根节点。也可以引入一个固定的 `id = 0` 虚拟根，好处是"根"不特殊，坏处是多一条无意义的数据，两种做法都常见。
- 自引用外键保证父节点一定存在；默认的 `RESTRICT` 行为还能阻止"删掉还有子部门的节点"。
- 但**外键只能保证"父存在"，不能保证"无环"**。`A → B → A` 在数据库层面是完全合法的，环必须靠应用校验、`CHECK` 约束（只能挡一跳）或查询时的兜底来防。
- `parent_id` 上的索引是必须的。没有它，每一次"找子节点"都会退化成全表扫描，而树查询要做很多次这样的查找。

邻接表最大的优点是**写入便宜**：新增一个节点就是 `INSERT` 一行；把一个部门连同整棵子树搬到别处，只需要 `UPDATE` 一行 `parent_id`。这一点在组织频繁调整的场景里非常关键。

代价在读取侧。要查整棵子树，应用层最常见的写法就是逐层查询（俗称 N+1）：

```java
List<Long> collectSubtree(Long rootId) {
    List<Long> result = new ArrayList<>();
    Deque<Long> queue = new ArrayDeque<>();
    queue.add(rootId);
    while (!queue.isEmpty()) {
        Long id = queue.poll();
        result.add(id);
        queue.addAll(deptMapper.selectChildren(id)); // 每层一次数据库往返
    }
    return result;
}
```

深度 10 的树要往返数据库 10 次；如果部门树只有几百个节点，其实不如一次 `SELECT * FROM department` 全量捞回内存建树——这往往是最省事也最快的做法。但当数据量大、或者需要和别的表做 join 时，就应该把"递归"交给数据库，也就是递归 CTE。

## 三、递归 CTE：让数据库自己走完这棵树

递归 CTE 从 SQL:1999 起就是标准的一部分，语法骨架固定为"锚点 + 递归项"：

```sql
WITH RECURSIVE cte_name (col1, col2, ...) AS (
    <非递归项>        -- 锚点（anchor），只执行一次
    UNION [ALL]
    <递归项>          -- 唯一可以引用 cte_name 自身的地方
)
SELECT ... FROM cte_name;
```

几个语法约束：

- 非递归项和递归项必须有相同的列数与兼容的类型；
- `RECURSIVE` 关键字是标准的（Oracle 例外，它用 `WITH` 隐式支持递归）；
- 必须写 `UNION` 或 `UNION ALL` 把两部分连起来，`UNION` 会去重，`UNION ALL` 不去重。

### 3.1 自顶向下：查整棵子树

```sql
WITH RECURSIVE dept_tree AS (
    -- 锚点：从起点部门开始
    SELECT d.id, d.parent_id, d.name, 1 AS depth
    FROM department d
    WHERE d.id = 2
    UNION ALL
    -- 递归项：把上一层的每一行当作"父"，去表中找它的子
    SELECT c.id, c.parent_id, c.name, t.depth + 1
    FROM department c
    JOIN dept_tree t ON c.parent_id = t.id
)
SELECT id, parent_id, name, depth
FROM dept_tree
ORDER BY depth, id;
```

读法就是一句话：**先找到起点，再不停地把"上一层的结果"喂回给自己，直到找不出新行为止**。`depth` 不是必须的，但在分页、限制层级、按层聚合时很有用。

### 3.2 自底向上：查祖先链

把 join 的方向换一下，"父找子"就变成"子找父"，写法完全对称：

```sql
WITH RECURSIVE report_chain AS (
    SELECT e.id, e.name, e.manager_id, 0 AS distance
    FROM employee e
    WHERE e.id = 42
    UNION ALL
    SELECT m.id, m.name, m.manager_id, c.distance + 1
    FROM employee m
    JOIN report_chain c ON m.id = c.manager_id
)
SELECT id, name, distance
FROM report_chain
ORDER BY distance;
```

注意这里的 `JOIN` 条件是 `m.id = c.manager_id`：当前行提供"我的上级是谁"，再去表里取出这个上级的整行。方向调换后，同一条 SQL 模板就能回答"我的下属"和"我的上级"两个相反的问题。（`employee` 的完整字段见 6.2。）

### 3.3 顺带拼出路径和树形缩进

在递归项里累加一个路径数组，就能同时得到面包屑和树形排序：

```sql
WITH RECURSIVE dept_path AS (
    SELECT d.id, d.name, ARRAY[d.id] AS path, 1 AS depth
    FROM department d
    WHERE d.parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, p.path || d.id, p.depth + 1
    FROM department d
    JOIN dept_path p ON d.parent_id = p.id
)
SELECT
    id,
    depth,
    repeat('    ', depth - 1) || name AS tree_label,
    array_to_string(path, ' / ')      AS full_path
FROM dept_path
ORDER BY path;   -- 数组按元素逐个比较，天然是深度优先顺序
```

`ORDER BY path` 这个技巧很实用：数据库比较数组时是逐元素字典序，于是父节点一定排在子节点前面，结果直接就是"展开的树"。MySQL 没有数组类型，用 `CONCAT` 拼字符串也能达到同样效果，但要注意类型宽度问题（见 3.6）。

### 3.4 UNION 与环：一个容易踩的坑

`UNION ALL` 最快，但它**不防环**。数据里一旦出现 `4 → 5 → 4` 这样的脏数据，递归就永远不会结束，查询会一直跑到内存或磁盘被打满。

`UNION` 会按整行去重，看起来能挡住简单的环，但有两个问题：

1. 每一层都要做哈希或排序去重，代价比 `UNION ALL` 高；
2. **如果递归项里带了 `depth`（或任何随层数变化的列），整行永远不相等，去重就完全失效**。这是实践中最常见的错误——加了 `depth` 方便展示，却顺手把防环能力破坏了。

另外要区分"树"和"图"：如果业务上允许一个节点有多个父（菱形结构，比如一个人有两个上级、一个部门同时挂两个上级组织），`UNION ALL` 会输出**多条路径**而不是多个节点，结果行数可能随深度指数增长。需要按节点去重时，应只在 CTE 里输出节点列并配合 `UNION`，或者用路径判断。

### 3.5 标准里的 CYCLE 与 SEARCH

SQL 标准后来补上了两个可选子句，专门解决"防环"和"排序"这两件事。PostgreSQL 从 14 开始支持，Oracle 也支持；MySQL 8.0 和 SQL Server 目前没有对应语法。

**CYCLE：检测并切断环**

```sql
WITH RECURSIVE dept_tree (id, parent_id, name) AS (
    SELECT id, parent_id, name FROM department WHERE id = 2
    UNION ALL
    SELECT d.id, d.parent_id, d.name
    FROM department d
    JOIN dept_tree t ON d.parent_id = t.id
)
CYCLE id SET is_cycle USING path
SELECT * FROM dept_tree
WHERE NOT is_cycle;
```

`CYCLE id SET is_cycle USING path` 会隐式追加两列：`is_cycle` 标记该行是否构成环，`path` 记录已经走过的 `id` 链。**一旦检测到重复，就只标记、不再对这一行继续递归**，所以需要 `WHERE NOT is_cycle` 把闭合那一行过滤掉。完整形式还可以写成 `CYCLE id SET is_cycle TO true DEFAULT false USING path`。

**SEARCH：生成排序键**

```sql
WITH RECURSIVE dept_tree (id, name, depth) AS (
    SELECT id, name, 1 FROM department WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, t.depth + 1
    FROM department d
    JOIN dept_tree t ON d.parent_id = t.id
)
SEARCH DEPTH FIRST BY id SET ordercol
SELECT * FROM dept_tree
ORDER BY ordercol;
```

`SEARCH DEPTH FIRST BY id SET ordercol` 等价于手写一个路径数组再排序，`SEARCH BREADTH FIRST BY id SET ordercol` 则等价于按 `(depth, id)` 排序。用标准子句的好处是不用自己维护路径列，也不会因为 `depth` 混进 CTE 输出而破坏 `UNION` 去重。

### 3.6 各数据库的差异

| 能力 | PostgreSQL | MySQL 8.0+ | SQL Server | Oracle |
| --- | --- | --- | --- | --- |
| 递归关键字 | `WITH RECURSIVE` | `WITH RECURSIVE` | `WITH RECURSIVE` | `WITH`（隐式） |
| CTE 列名列表 | 可选 | **必须** | 可选 | 可选 |
| 环检测 | `CYCLE` 子句 | 手动路径判断 | 手动 + `MAXRECURSION` | `CYCLE` 子句 |
| 深度兜底 | 自己写 `depth < n` | `cte_max_recursion_depth`（默认 1000） | `OPTION (MAXRECURSION n)`（默认 100） | 自己写 |
| 深度优先排序 | `SEARCH DEPTH FIRST` | 手动拼路径字符串排序 | 手动 | `SEARCH DEPTH FIRST` |
| 递归项限制 | 较少（自引用不能出现在子查询、外连接中，且不能多次引用） | 不能聚合、窗口函数、`GROUP BY`、`ORDER BY`、`DISTINCT` | 较多（聚合、`TOP`、`DISTINCT`、外连接、子查询等） | 较少 |

MySQL 用户最容易踩两个坑，值得单独记下来。

**坑一：递归列的类型宽度由锚点决定。** 路径列必须在非递归项里 `CAST` 到足够长，否则深层路径会被截断甚至直接报错：

```sql
WITH RECURSIVE dept_path (id, name, path, depth) AS (
    SELECT id, name, CAST(CONCAT('/', id, '/') AS CHAR(500)), 1
    FROM department WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, CONCAT(p.path, d.id, '/'), p.depth + 1
    FROM department d
    JOIN dept_path p ON d.parent_id = p.id
    WHERE p.depth < 100
      AND LOCATE(CONCAT('/', d.id, '/'), p.path) = 0   -- 手动环检测
)
SELECT * FROM dept_path
ORDER BY path;
```

**坑二：默认递归上限 1000 层。** 正常组织不会有 1000 层，但"环 + 上限"的组合会把一次误操作放大成 1000 轮 join。上线前把上限调小，是成本最低的保护：

```sql
SET SESSION cte_max_recursion_depth = 100;
```

## 四、执行原理：火山模型下的递归 CTE

写得出递归 CTE 是一回事，知道它怎么跑是另一回事。后者决定了你为什么要把 `parent_id` 加上索引、为什么深树会突然变慢，以及为什么加了 `LIMIT` 有时快有时不快。

### 4.1 火山模型回顾

Volcano / 迭代器模型（Graefe, 1993）把执行计划组织成一棵**算子树**，每个算子只实现三个方法：

- `open()`：初始化自己，并递归调用子算子的 `open()`；
- `next()`：返回**一行**；
- `close()`：释放资源。

数据是**自底向上拉取**的：顶层算子反复调用 `next()`，算子内部再去调子算子的 `next()`。这就是所谓 pull-based 执行。它的好处是天然支持"要多少算多少"——`LIMIT 10` 只拉 10 行就停，不会先算完整张表。

现代引擎在这个骨架之上做了向量化（一次一批）或编译执行（把算子内联成循环），但 `open/next/close` 的语义没有变。理解这一点，就足够解释递归 CTE 的行为。

### 4.2 递归 CTE 对应的物理算子

在 PostgreSQL 里，递归 CTE 被翻译成 `RecursiveUnion` 节点；不同引擎命名各有不同（MySQL 的执行计划里叫 `Recursive union`，SQL Server 则由 `Spool` 算子实现）。它们都有两个子计划：

```text
RecursiveUnion
├── 锚点分支      -- 只执行一次
└── 递归分支      -- 每轮重扫一次，输入是 working table
```

数据库文档里描述的求值算法，翻译成算子的 `next()` 就是一个"换代表"的状态机：

```text
状态：working（当前层）、pending（本层待返回的行）

open():
    working ← 执行锚点分支的全部结果
    pending ← working

next():
    if pending 还有未返回的行:
        return 下一行                  -- 本层还没吐完，继续吐
    if working 为空:
        return EOF                     -- 没有下一层了，递归结束
    -- 本层已吐完：把 working 当作递归分支的输入，一次算出下一层
    working ← 执行递归分支(working)     -- 产出即文档里的 intermediate table
    pending ← working
    return next()
```

用第一节那棵树、从部门 2 出发走一遍：

| 轮次 | working table（输入） | 递归分支产出 | 累计结果 |
| --- | --- | --- | --- |
| 0（锚点） | — | `{2}` | 2 |
| 1 | `{2}` | `{4, 5}` | 2, 4, 5 |
| 2 | `{4, 5}` | `{7}`（4 无子节点，5 的子节点是 7） | 2, 4, 5, 7 |
| 3 | `{7}` | `∅` | 同上，循环结束 |

可以看到，递归在物理上就是**一个 while 循环**，`RECURSIVE` 只是让 SQL 能表达"自我引用"的语法糖。

### 4.3 这个模型给出的四个工程结论

**第一，`parent_id` 没有索引会被放大成"层数 × 全表扫描"。** 每一轮迭代都要为 working table 里的每一行去找子节点。一次全表扫描是 O(N)，h 层就是 O(h·N)；有索引时是 O(log N + 命中行数)。这也是邻接表在生产上唯一必须做的调优。

**第二，它是"半阻塞"算子，存在层间屏障。** 普通算子可以一行进一行出（`Filter`、`Project` 都是纯流水线），而 `RecursiveUnion` 必须以"整层"为推进单位：上一层没有全部产出，下一层就无法开始。这个屏障让递归 CTE 很难向量化，也意味着它不适合要求极低首行延迟的流式输出场景。（不过顶层带 `LIMIT` 时，算子仍可以在攒够行数后提前停止，不必跑完全部层级——这正是"有时快有时不快"的原因。）

**第三，中间结果会落到临时存储。** working / intermediate table 本质是元组存储，`work_mem` 不够就落盘；用 `UNION` 去重还额外需要哈希表或排序。所以一棵很大的子树，递归 CTE 的 I/O 和内存开销会明显高于同等行数的普通 join——这也是"读多写少"场景愿意改用闭包表的原因。

**第四，`UNION ALL` 加环等于死循环，而且没有隐式保护。** 火山模型下只有 working table 变空才返回 `EOF`，环意味着永远存在"下一行"。生产代码必须在三件事里至少做一件：用 `CYCLE` 子句、手动判断路径、或者硬性 `depth` 上限。

## 五、其他方案：物化路径与闭包表

递归 CTE 解决了"能不能查"的问题，但没有解决"查得快不快"。当树被高频读取（数据权限、组织架构页、报表），把递归的结果提前物化就是自然的想法。业界有两条成熟路线。

### 5.1 物化路径（Materialized Path）

把祖先链直接写进每一行，用字符串（或数组）表示"我从根到我的路径"：

```sql
CREATE TABLE department_mp (
    id        bigint      PRIMARY KEY,
    parent_id bigint      NULL,
    name      varchar(64) NOT NULL,
    path      varchar(500) NOT NULL UNIQUE,   -- 例：'/1/2/5/'
    depth     int          NOT NULL
);
CREATE INDEX idx_dept_mp_path ON department_mp (path);

-- 整棵子树：一次前缀匹配，不需要递归
SELECT * FROM department_mp
WHERE path LIKE '/1/2/%'
ORDER BY path;

-- 全部祖先（含自身）：一次前缀比较
SELECT * FROM department_mp
WHERE (SELECT path FROM department_mp WHERE id = 5) LIKE path || '%'
ORDER BY depth;
```

特点很鲜明：

- **读极快**。查子树、查祖先都是一次索引范围扫描，代价只和结果行数有关，没有递归、没有层间屏障。（PostgreSQL 里要注意排序规则：非 `C` 排序规则下前缀匹配默认走不了 btree 索引，需要给 `path` 建 `text_pattern_ops` 的索引，或者让列使用 `COLLATE "C"`。）
- **写很贵**。移动子树的代价是"更新这棵子树里所有行的 `path`"，行数越多越慢；而且这是一条大范围 `UPDATE`，容易产生长事务和锁等待。
- 路径长度受列宽限制，等价于给深度设了上限。为了让字符串比较正确，`id` 最好定宽（补零）或用固定位数的编码，否则 `/1/10/` 和 `/1/1/` 的字典序会错乱，前缀匹配也会误伤。
- PostgreSQL 可以用 `ltree` 扩展把路径变成原生类型，配合 GiST 索引：`WHERE path <@ '1.2'::ltree` 直接查子树。

一句话总结：**用"写放大"换"读极简"**，适合目录、分类、URL 路径这类读多写少且很少搬动的树。

### 5.2 闭包表（Closure Table）

闭包表把"所有祖先-后代对"单独存成一张表，本质上是**把递归的结果物化下来**：

```sql
CREATE TABLE dept_closure (
    ancestor_id   bigint NOT NULL REFERENCES department(id),
    descendant_id bigint NOT NULL REFERENCES department(id),
    depth         int    NOT NULL,   -- 0 表示自身
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX idx_closure_desc ON dept_closure (descendant_id, depth);
```

插入节点时，把它到所有祖先的关系一次性写进去：

```sql
-- 在 5（业务组）下新增 100（数据小组）
INSERT INTO department (id, parent_id, name) VALUES (100, 5, '数据小组');

INSERT INTO dept_closure (ancestor_id, descendant_id, depth)
SELECT ancestor_id, 100, depth + 1 FROM dept_closure WHERE descendant_id = 5
UNION ALL
SELECT 100, 100, 0;      -- 自身到自身，depth = 0
```

之后任意方向的查询都是普通 join，复杂度只与结果集大小相关：

```sql
-- 子树（不含自身）
SELECT d.*
FROM dept_closure c
JOIN department d ON d.id = c.descendant_id
WHERE c.ancestor_id = 2 AND c.depth > 0
ORDER BY c.depth;

-- 祖先链（不含自身）
SELECT d.*
FROM dept_closure c
JOIN department d ON d.id = c.ancestor_id
WHERE c.descendant_id = 5 AND c.depth > 0
ORDER BY c.depth DESC;
```

移动子树要"先切断、再重建"：删掉所有"子树内部节点 → 子树外部祖先"的关系，再用新祖先链与子树做笛卡尔积插回去。

```sql
-- 把 100 及其后代整体挂到 3 下
-- 1) 删除跨越边界的旧关系
DELETE FROM dept_closure
WHERE descendant_id IN (SELECT descendant_id FROM dept_closure WHERE ancestor_id = 100)
  AND ancestor_id   IN (SELECT ancestor_id   FROM dept_closure WHERE descendant_id = 100)
  AND ancestor_id <> 100;   -- 保留子树内部关系

-- 2) 用「新祖先链 × 子树」重建关系
INSERT INTO dept_closure (ancestor_id, descendant_id, depth)
SELECT sup.ancestor_id, sub.descendant_id, sup.depth + sub.depth + 1
FROM dept_closure sup
JOIN dept_closure sub ON sub.ancestor_id = 100
WHERE sup.descendant_id = 3;   -- 3 及它的所有祖先
```

代价与收益同样明确：

- **写**：插入代价 ≈ 节点深度；移动代价 ≈ 子树节点数 × 祖先链长度。比邻接表贵一个数量级。
- **读**：任意方向都是索引 join，单层、稳定、可预测；"两个部门的最近公共祖先"这类问题也只需要一次 join。
- **存储**：O(节点数 × 平均深度)。组织树深度一般不超过 10，这份膨胀完全可以接受。
- **一致性**：闭包表是**冗余数据**，必须和邻接表在同一个事务里维护，否则会出现"部门树里已经搬走了、闭包表还停在原地"的脏数据。这是它最大的工程风险。

### 5.3 三种方案对比

| 维度 | 邻接表 | 物化路径 | 闭包表 |
| --- | --- | --- | --- |
| 表数量 | 1 | 1 | 2（主表 + 关系表） |
| 查子树 | 递归 CTE | 一次前缀扫描 | 一次 join |
| 查祖先 | 递归 CTE | 一次前缀比较 | 一次 join |
| 移动子树 | **1 行 UPDATE** | 更新整棵子树的 path | 删 + 插，O(子树 × 深度) |
| 插入节点 | 1 行 | 1 行（需算出 path） | 1 + 深度 行 |
| 深度上限 | 无 | 受路径列宽限制 | 无 |
| 环风险 | 有（靠约束或 CTE 兜底） | 低（移动时校验） | 有（写入时校验） |
| 典型场景 | 通用建模、变更频繁 | 目录、分类、URL 路径 | 组织架构、权限树、报表 |

这三者并不互斥。最常见的生产组合是：**邻接表作为唯一事实来源（写），闭包表或物化路径作为读模型（读）**，两者在同一事务内更新；如果读压力不大，就只保留邻接表，靠缓存挡住高频查询。

## 六、实战：部门归属与人员汇报两条树

前面讨论的都是"一棵树"。真实的人力系统里，问题往往不是一棵树，而是**两棵语义不同、还允许交叉的树**。

### 6.1 两条正交的树

- **部门归属树**：员工"属于"哪个部门。这是编制、成本中心、考勤、预算的口径。
- **人员汇报树**：员工"向谁汇报"。这是审批链、绩效、任务分派的口径。

在矩阵式组织里，这两条链**可以不一致**：平台组的工程师可能虚线汇报给业务组负责人；某个新业务部门的负责人可能同时挂在两个中心之下；总部直属的专家可能"归属"在技术中心，但汇报给 CTO 办公室。由此得到三条建模原则：

1. 不要用部门树去推导汇报链（也不要反过来），两条关系都显式存储。
2. 不要把员工挂进部门树当叶子节点（即"全公司一棵树"），否则一次部门调整会顺带改掉一批人的汇报关系。
3. 允许不一致，但要把"允许到什么程度"写成可校验的规则，否则报表口径会互相打架。

### 6.2 表结构

```sql
CREATE TABLE department (
    id        bigint      PRIMARY KEY,
    parent_id bigint      NULL REFERENCES department(id),
    name      varchar(64) NOT NULL,
    leader_id bigint      NULL,           -- 部门负责人，指向 employee
    status    varchar(16) NOT NULL DEFAULT 'active'
);
CREATE INDEX idx_department_parent ON department (parent_id);

CREATE TABLE employee (
    id         bigint      PRIMARY KEY,
    name       varchar(64) NOT NULL,
    dept_id    bigint      NOT NULL REFERENCES department(id),
    manager_id bigint      NULL REFERENCES employee(id),
    status     varchar(16) NOT NULL DEFAULT 'active',   -- active / left
    CONSTRAINT chk_emp_not_self CHECK (manager_id <> id)
);
CREATE INDEX idx_employee_dept    ON employee (dept_id);
CREATE INDEX idx_employee_manager ON employee (manager_id);
```

值得说明的几个点：

- `dept_id` 和 `manager_id` 是**两个独立维度**，不要合并成一个 `parent_id`。合并之后就再也表达不了"归属在 A 部门、汇报给 B 部门"这种合法状态。
- `leader_id`（部门负责人）和 `manager_id`（我的直接上级）语义不同：前者是**角色**，一个人可以负责多个部门；后者是**关系**，一人只有一个直接上级。两者同时存在时，必须明确谁是权威源——通常 `leader_id` 应该由"该部门某人的 manager 关系"派生，或者用流程保证一致，否则就是典型的双写不一致。
- `status` 很重要：离职员工不能出现在汇报链的递归路径上，否则会把链路"接断"。
- `CHECK (manager_id <> id)` 只能挡住一跳自环，多跳环需要写入前校验（见 6.4）。

### 6.3 四个高频查询

**(a) 部门子树的在编人数**（报表与编制管理）

```sql
WITH RECURSIVE sub_dept AS (
    SELECT id FROM department WHERE id = 2
    UNION ALL
    SELECT d.id FROM department d JOIN sub_dept s ON d.parent_id = s.id
)
SELECT d.id, d.name, count(e.id) AS headcount
FROM sub_dept s
JOIN department d ON d.id = s.id
LEFT JOIN employee e ON e.dept_id = d.id AND e.status = 'active'
GROUP BY d.id, d.name
ORDER BY d.id;
```

两个细节：用 `LEFT JOIN` 是为了让"没有人的部门"也出现在结果里（计数为 0）；过滤条件 `e.status = 'active'` 必须写在 `ON` 里，写到 `WHERE` 会把它变成内连接，空部门又被过滤掉了。

**(b) 某人的完整汇报链**（向上，带距离）

```sql
WITH RECURSIVE report_chain AS (
    SELECT e.id, e.name, e.manager_id, e.dept_id, 0 AS distance
    FROM employee e
    WHERE e.id = 42
    UNION ALL
    SELECT m.id, m.name, m.manager_id, m.dept_id, c.distance + 1
    FROM employee m
    JOIN report_chain c ON m.id = c.manager_id
)
SELECT id, name, dept_id, distance
FROM report_chain
ORDER BY distance;
```

**(c) 某人的全部下属**（向下，含间接）

```sql
WITH RECURSIVE subordinates AS (
    SELECT e.id, e.name, e.dept_id, e.manager_id, 1 AS level
    FROM employee e
    WHERE e.manager_id = 42 AND e.status = 'active'
    UNION ALL
    SELECT e.id, e.name, e.dept_id, e.manager_id, s.level + 1
    FROM employee e
    JOIN subordinates s ON e.manager_id = s.id
    WHERE e.status = 'active'
)
SELECT * FROM subordinates
ORDER BY level, id;
```

汇报关系在正常数据下必然是一棵树（每个人只有一个 `manager_id`），所以这里用 `UNION ALL` 是安全的，性能也最好。但仍建议加一个 `s.level < 20` 的兜底：脏数据成环时，业务方宁愿看到"层级被截断"，也不愿意数据库连接池被打爆。

**(d) 一致性校验：找出跨部门汇报的人员**

大多数公司允许跨部门汇报，但要求"上级必须在本部门或其祖先部门内"。这类规则不适合放在线上接口，适合做成每日跑一次的数据质量任务：

```sql
WITH RECURSIVE dept_path AS (
    SELECT id, ARRAY[id] AS ancestors FROM department WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, p.ancestors || d.id
    FROM department d
    JOIN dept_path p ON d.parent_id = p.id
)
SELECT e.id, e.name, e.dept_id, m.dept_id AS manager_dept_id
FROM employee e
JOIN employee m ON m.id = e.manager_id
JOIN dept_path dp ON dp.id = e.dept_id
WHERE e.status = 'active'
  AND m.dept_id <> ALL (dp.ancestors);   -- 上级部门不在我的祖先链（含自身）里
```

`ancestors` 包含了自身，所以"上级和我同部门"是合法的，不会被误报。规则本身可以按公司实际调整：有的公司只要求"上级在同一个一级事业部内"，那就把祖先链截断到一级再比较。

### 6.4 变更流程与一致性

组织系统的 bug 大多不出在查询上，而出在"两条树同时被改"的时候。几条实践中的规则：

- **部门合并/拆分只动部门树**：改 `parent_id`（或闭包表的边），人员只调整真正受影响的那部分 `dept_id`，`manager_id` 不动。汇报关系不会因为组织调整自动改变，不要"顺手"一起改。
- **人员调动只动人员表**：改 `dept_id`，必要时改 `manager_id`，部门树不动。这两件事在业务上是两笔操作，合成一条隐式规则会很难追溯。
- **部门搬迁要防"移到自己子树下"**：移动前先做一次子树查询，确认目标父节点不在自己的后代集合里；否则会制造出一个孤立的环。
- **管理者离职要先转移下属**：把直属下属的 `manager_id` 指向接任者，再把该员工置为 `left` 并清空 `manager_id`。如果外键用 `ON DELETE SET NULL`，直接删人会**静默丢失整条汇报链**；用默认的 `RESTRICT` 更安全，逼你走流程。
- **闭包表/物化路径必须与邻接表同事务更新**：它们是派生数据。更稳妥的做法是把"移动子树"封装成一个存储过程或领域服务方法，任何入口都只能调用它，禁止业务代码直接 `UPDATE parent_id`。
- **多跳环要在写入前校验**：新增/修改 `manager_id` 时，沿上级链向上走，一旦遇到自己就拒绝；部门移动同理。这一步用递归 CTE 做一次"祖先探测"即可，比事后巡检便宜得多。

### 6.5 现实中的取舍

- 组织树通常**浅而宽**（深度 5\~10，节点数百到数万），人员树更浅。这个量级下，邻接表 + 递归 CTE 完全够用，**不必为了性能上闭包表**。真正贵的往往不是递归本身，而是"每个请求都递归一遍"——加一层 5\~10 分钟的缓存或一个物化视图就能解决。
- 需要"历史某月的组织架构"时，三种方案都只表达**当前状态**。正确做法是在部门与汇报关系上加 `valid_from` / `valid_to` 做时态表，或者每天落一份快照到数仓；不要指望递归 CTE 帮你穿越时间。
- **数据权限**是树查询最典型的落地点：登录人的可见范围 = "所在部门子树" ∪ "自己的下属链"。两条递归 CTE 的 `UNION` 一次算出 id 集合，再作为 `IN` 条件使用；集合很大时改用临时表或缓存，避免超长 `IN` 列表把 SQL 解析拖慢。
- 报表侧不要直接打业务库。把"部门路径"在 ETL 里展开成宽表（每行带一级~五级部门 id 和名称），之后所有按层级汇总都用普通 `GROUP BY`，比反复递归便宜得多。这也是数仓里最常见的组织维度建模方式。

## 七、小结

- 树在关系库里没有原生表示。**邻接表是"唯一事实来源"的首选**：一行一个 `parent_id`，写入便宜、移动子树只改一行；代价是读取要递归，而 `parent_id` 上的索引是硬性要求。
- **递归 CTE** 把"走到底"交给数据库，骨架就是 `锚点 UNION [ALL] 递归项`。标准已提供 `CYCLE`（防环）与 `SEARCH`（排序），PostgreSQL 14+ 和 Oracle 支持；MySQL 则要靠路径字符串加 `cte_max_recursion_depth` 自己兜底。
- 从**火山模型**看，递归 CTE 就是 `RecursiveUnion` 算子：锚点跑一次，然后以 working table 为输入反复重扫递归分支，直到某一层产出为空。它以"整层"为推进单位、中间结果落临时存储、难以向量化——这三点决定了它的性能特征，也解释了深树查询为什么会突然变慢。
- 想读得更快，就把递归结果物化：**物化路径**用写放大换极简读，**闭包表**用存储放大换任意方向的索引 join；两者都必须和邻接表保持强一致。
- 真实系统里要把两条树分开：**部门归属树**管编制与成本，**人员汇报树**管审批与绩效，允许跨部门汇报但用校验任务守住口径。存储用邻接表，读侧按需叠加缓存、闭包表或数仓宽表，是成本与复杂度最平衡的组合。
