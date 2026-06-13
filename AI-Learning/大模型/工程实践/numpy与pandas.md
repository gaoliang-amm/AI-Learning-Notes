---
aliases: [NumPy, Pandas, 数据分析]
tags: [AI/Learning, 工具, Python, NumPy, Pandas]
created: 2026-06-13
---

# NumPy 与 Pandas 工具参考

> **背诵版**：NumPy 核心是 ndarray，支持矢量化运算和广播机制，提供随机数生成、线性代数、统计函数。Pandas 核心是 Series（一维）和 DataFrame（二维表格），擅长数据清洗（缺失值/重复值）、分组聚合（groupby/agg）、数据合并（merge/concat/join）。两者配合是 Python 数据分析的基石：NumPy 负责底层数值计算，Pandas 负责结构化数据处理。

---

# Part 1 - NumPy

```python
import numpy as np
```

## 一、ndarray 创建

| 函数 | 说明 | 示例 |
|------|------|------|
| `np.array(data)` | 创建数组，**总是 copy** | `np.array([1,2,3])` |
| `np.asarray(data)` | 输入为 ndarray 时**不 copy** | `np.asarray(arr)` |
| `np.zeros(shape)` | 全 0 数组（默认 float64） | `np.zeros((2,3))` |
| `np.ones(shape)` | 全 1 数组 | `np.ones((2,3))` |
| `np.empty(shape)` | 未初始化（值不确定，慎用） | `np.empty((2,3))` |
| `np.zeros_like(arr)` | 与 arr 同形状同类型 | `np.zeros_like(arr)` |
| `np.ones_like(arr)` | 同上，全 1 | `np.ones_like(arr)` |
| `np.full(shape, val)` | 全 val 数组 | `np.full((2,3), 6)` |
| `np.full_like(arr, val)` | 同上，按 arr 形状 | `np.full_like(a, 5)` |

### 1.2 arange / linspace / logspace

| 函数 | 说明 | 示例 |
|------|------|------|
| `np.arange(start, stop, step)` | 等间隔一维，**不含 stop** | `np.arange(0,10,2)` -> `[0 2 4 6 8]` |
| `np.linspace(start, stop, num)` | 等差数列，**含 stop** | `np.linspace(0,10,5)` -> `[0 2.5 5 7.5 10]` |
| `np.logspace(start, stop, num, base)` | 等比数列 | `np.logspace(2,5,5,base=2)` |

> `endpoint=False` 时 linspace 不含 stop，间隔公式变为 `(stop-start)/num`。

### 1.3 随机数生成

| 函数 | 说明 | 示例 |
|------|------|------|
| `np.random.rand(d0,d1,...)` | [0,1) 均匀分布 | `np.random.rand(2,3)` |
| `np.random.randint(low, high, size)` | 整数 [low,high) | `np.random.randint(0,10,(2,3))` |
| `np.random.uniform(low, high, size)` | 浮点 [low,high) | `np.random.uniform(3,6,(2,3))` |
| `np.random.randn(d0,d1,...)` | 标准正态 | `np.random.randn(2,3)` |
| `np.random.normal(loc, scale, size)` | 自定义正态 | `np.random.normal(0,1,(2,3))` |
| `np.random.seed(42)` | 固定种子 | 保证可复现 |

> 训练模型前务必设置种子。新版本推荐 `np.random.default_rng(42)` 生成 Generator 对象。

### 1.4 matrix（不推荐）

```python
np.matrix("1 2; 3 4")   # 仅支持二维，已不推荐，用 ndarray + @ 即可
```

## 二、ndarray 属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `ndim` | 维度数 | `a.ndim` |
| `shape` | 形状（元组） | `a.shape` -> `(2, 3)` |
| `size` | 元素总数 | `a.size` -> `6` |
| `dtype` | 数据类型 | `a.dtype` |
| `itemsize` | 每个元素字节数 | `a.itemsize` |

> **ndarray 限制**：所有元素必须**同一类型**，创建后**大小不可变**，形状必须是**矩形**（非锯齿）。

## 三、数据类型与转换

### 常用 dtype 对照表

| 类型 | 代码 | 说明 | 类型 | 代码 | 说明 |
|------|------|------|------|------|------|
| `bool` | `?` | 布尔 | `float16` | `f2` | 半精度浮点 |
| `int8/uint8` | `i1/u1` | 8 位整型 | `float32` | `f4/f` | 单精度浮点 |
| `int32/uint32` | `i4/u4` | 32 位整型 | `float64` | `f8/d` | 双精度浮点（默认） |
| `int64/uint64` | `i8/u8` | 64 位整型 | `complex128` | `c16` | 复数 |

### 类型指定与转换

```python
np.array([1, 2, 3], dtype=np.float64)   # [1. 2. 3.]
np.array([0.2, 2.5, 4.8], dtype="i8")   # [0 2 4]（截断为整数）
arr.astype(np.int64)    # 返回新数组，类型转换
```

> **性能提示**：深度学习中经常 `arr.astype(np.float32)` 节省显存。

## 四、索引与切片

### 4.1 一维数组索引

```python
arr = np.arange(10)       # [0 1 2 3 4 5 6 7 8 9]
arr[2]                    # 2
arr[2:9:2]                # [2 4 6 8]
arr[2:]                   # [2 3 4 5 6 7 8 9]
arr[slice(2, 9, 2)]       # 等价写法 [2 4 6 8]
```

### 4.2 二维数组索引

```python
arr2d = np.array([[1,2,3],[4,5,6],[7,8,9]])
arr2d[1]                  # [4 5 6]（取第 2 行）
arr2d[0, 1]               # 1（取第 0 行第 1 列）
arr2d[0:2, 1:3]           # [[2 3] [5 6]]（行切片+列切片）
```

### 4.3 布尔索引

```python
arr = np.array([1, 5, 3, 8, 2])
arr[arr > 3]              # [5 8]
np.where(arr > 3, arr, 0) # [0 5 0 8 0]（三元运算）
```

### 4.4 花式索引（Fancy Indexing）

```python
arr = np.array([10, 20, 30, 40, 50])
arr[[0, 2, 4]]            # [10 30 50]（用整数数组索引）

arr2d = np.array([[1,2],[3,4],[5,6]])
arr2d[[0, 2], [1, 0]]     # [2 5]（配对取：(0,1)和(2,0)）
```

## 五、运算

### 5.1 矢量化算术运算

```python
a = np.array([[1,2,3],[4,5,6]])
b = np.array([[7,8,9],[10,11,12]])
a + b       # 对位加
a * b       # 对位乘（不是矩阵乘法！）
a + 100     # 标量广播，每个元素+100
```

### 5.2 矩阵乘法

```python
a @ b.T                    # @ 运算符
np.dot(a, b.T)             # np.dot
a.dot(b.T)                 # 实例方法
# 要求：a 的列数 == b 的行数
```

> **易错点**：`a * b` 是对位乘法，`a @ b` 才是矩阵乘法。深度学习中权重矩阵相乘务必用 `@` 或 `np.dot`。

### 5.3 常用数学与统计函数

| 函数 | 说明 | 函数 | 说明 |
|------|------|------|------|
| `np.abs()` | 绝对值 | `np.sqrt()` | 平方根 |
| `np.ceil()` / `np.floor()` | 向上/向下取整 | `np.rint()` / `np.round()` | 四舍五入 |
| `np.isnan()` / `np.isinf()` | 判断 NaN / inf | `np.exp()` / `np.log()` | 指数 / 自然对数 |
| `np.sin()` / `np.cos()` | 三角函数 | `np.multiply()` / `np.divide()` | 对位乘/除 |
| `np.mean()` | 均值 | `np.sum()` | 求和 |
| `np.max()` / `np.min()` | 最大/最小值 | `np.std()` / `np.var()` | 标准差/方差 |
| `np.argmax()` / `np.argmin()` | 最大/小值索引 | `np.cumsum()` / `np.cumprod()` | 累计和/积 |
| `np.median()` | 中位数 | `np.percentile()` | 百分位数 |

```python
np.where(arr > 3, 1, 0)           # 三元运算
np.mean(arr, axis=0)              # axis=0 按列求均值
np.mean(arr, axis=1)              # axis=1 按行求均值
```

> **axis 记忆**：`axis=0` 沿行方向（**沿列**计算，压扁行），`axis=1` 沿列方向（**沿行**计算，压扁列）。想象成"沿着哪个轴消减维度"。

### 5.5 比较、排序与去重

```python
np.any(arr > 3)       # True（至少一个满足）
np.all(arr > 3)       # False（不是全部满足）
arr.sort()            # 就地排序（修改原数组）
np.sort(arr)          # 返回排序副本
np.unique([1,2,2,3])  # [1 2 3]
```

### 5.7 线性代数

```python
np.linalg.inv(A)         # 逆矩阵
np.linalg.det(A)         # 行列式
np.linalg.eig(A)         # 特征值和特征向量
np.linalg.svd(A)         # 奇异值分解
np.linalg.solve(A, b)    # 解线性方程组 Ax=b
np.linalg.norm(A)        # 范数（默认 L2）
```

## 六、广播机制（Broadcasting）

允许不同形状数组元素级运算，无需复制数据。

**三条规则**：1. 维度数不同时，小维度在**最左边补 1**；2. 某维度为 1 时**复制扩展**匹配另一数组；3. 两个维度都不匹配且都不为 1，抛 `ValueError`。

```python
# (3,) + (3,1) -> (1,3) + (3,1) -> (3,3)
np.array([1,2,3]) + np.array([[4],[5],[6]])  # [[5 6 7] [6 7 8] [7 8 9]]
# (3,) + (2,) -> ValueError
```

> **直觉理解**：广播就像"对齐右端"——两个数组从右向左对齐维度，左边缺的补 1，大小为 1 的维度可以"拉伸"到和对方一样大。

## 七、文件读写

```python
# 二进制
np.save("arr.npy", arr)              # 保存单个数组
np.load("arr.npy")                   # 加载
np.savez("a.npz", a=a, b=b)         # 保存多个
np.load("a.npz")["a"]               # 按名加载

# 文本
np.savetxt("arr.txt", arr, delimiter=",")
np.loadtxt("arr.txt", delimiter=",")
np.genfromtxt("data.csv", delimiter=",", skip_header=1)
```

---

# Part 2 - Pandas

```python
import pandas as pd
```

## 一、Series

Series 是一维带标签数组，可以存储任何数据类型。

### 1.1 创建

```python
pd.Series([4, 7, -5, 3])                                        # 自动索引 0~N-1
pd.Series([4, 7, -5, 3], index=["a","b","c","d"], name="scores") # 指定索引和名称
pd.Series({"a": 4, "b": 7, "c": -5})                             # 字典键成为索引
```

### 1.2 索引

| 方式 | 说明 | 示例 |
|------|------|------|
| `s.loc["a"]` | **显式**标签索引 | `s.loc["a":"c"]`（**闭区间**） |
| `s.iloc[0]` | **隐式**位置索引 | `s.iloc[0:3]`（**左闭右开**） |
| `s.at["a"]` / `s.iat[0]` | 标签/位置取**单值**（快） | - |

### 1.3 属性与方法

```python
s.index / s.values / s.ndim / s.shape / s.size / s.dtype / s.name
```

| 方法 | 说明 | 方法 | 说明 |
|------|------|------|------|
| `s.head(n)` / `s.tail(n)` | 前/后 n 行 | `s.describe()` | 统计摘要 |
| `s.sum()` / `s.mean()` / `s.median()` | 求和/均值/中位数 | `s.min()` / `s.max()` / `s.std()` / `s.var()` | 最小/大/标准差/方差 |
| `s.value_counts()` / `s.count()` | 值计数/非空计数 | `s.unique()` / `s.nunique()` | 去重值/去重计数 |
| `s.sort_values()` / `s.sort_index()` | 按值/按索引排序 | `s.isin([v])` / `s.isna()` | 值匹配/缺失判断 |
| `s.replace(a, b)` / `s.sample(n)` | 替换/采样 | `s.to_frame()` / `s.corr(s2)` | 转 DataFrame/相关系数 |

### 1.4 布尔索引与运算

```python
s[s > s.mean()]                     # 布尔索引筛选
s * 10                              # 标量运算
pd.Series([1,1,1,1]) + pd.Series([2,2,2,2], index=[1,2,3,4])  # 按索引对齐，不匹配填 NaN
```

> **Series 运算核心**：按索引标签自动对齐，对不齐用 NaN 填充。

## 二、DataFrame

DataFrame 是二维表格结构，可以理解为共享同一索引的 Series 字典。

### 2.1 创建

```python
pd.DataFrame({"id": [101,102], "name": ["张三","李四"], "age": [20,30]})
pd.DataFrame(data={"age": [20,30], "name": ["张三","李四"]}, columns=["name","age"], index=[101,102])
```

### 2.2 属性与索引

```python
df.index / df.columns / df.values / df.ndim / df.shape / df.size / df.dtypes / df.T
```

```python
df["name"]              # 选列 -> Series
df[["name","age"]]      # 选多列 -> DataFrame
df.loc[101]             # 按标签取行
df.loc[101:103, ["name","age"]]  # 行切片+列选择（loc 闭区间）
df.iloc[0]              # 按位置取行
df.iloc[0:2, 1]         # 前 2 行，第 1 列（iloc 左闭右开）
df.iloc[:, [0,2]]       # 所有行，第 0 和 2 列
df.at[101, "name"]      # 按标签取单值
df.iat[0, 1]            # 按位置取单值
```

### 2.3 修改操作

```python
df.set_index("id", inplace=True)         # 设置索引
df.reset_index(inplace=True)              # 重置索引
df.rename(columns={"age":"年龄"}, inplace=True)  # 重命名
df["phone"] = ["133...","144..."]          # 添加列
df.insert(0, "phone", vals)               # 指定位置插入列
df.drop("phone", axis=1, inplace=True)    # 删除列
df.drop(0, axis=0, inplace=True)          # 删除行（按标签）
```

### 2.4 常用方法

| 方法 | 说明 | 方法 | 说明 |
|------|------|------|------|
| `df.head()` / `df.tail()` | 前/后 n 行 | `df.info()` / `df.describe()` | 信息/统计摘要 |
| `df.isin([v])` / `df.isna()` | 值匹配/缺失判断 | `df.value_counts()` / `df.drop_duplicates()` | 值计数/去重 |
| `df.sum()` / `df.mean()` / `df.median()` | 聚合 | `df.sort_values(by="col")` | 按列排序 |
| `df.nlargest(n, "col")` / `df.nsmallest(n, "col")` | 最大/小 n 行 | `df.cumsum()` / `df.diff()` | 累计和/差分 |

> **axis 参数**：`axis=0` 沿行方向（对**列**操作），`axis=1` 沿列方向（对**行**操作）。

## 三、数据清洗

### 3.1 缺失值处理

**检测缺失值**：

```python
df.isnull()         # 等价 df.isna()
df.notnull()        # 等价 df.notna()
df.isnull().sum()   # 每列缺失值数量（最常用）
df.isnull().sum().sum()  # 总缺失值数
```

**缺失值类型**：`np.nan`（浮点特殊值）、`None`（Python 空值）、`pd.NA`（Pandas 缺失值标记），Pandas 统一识别为缺失值。

**删除缺失值**：

```python
df.dropna()                        # 删除任何含缺失值的行
df.dropna(axis=1)                  # 删除任何含缺失值的列
df.dropna(how="all")               # 只删全为缺失值的行
df.dropna(thresh=2)                # 至少 2 个非缺失值才保留
df.dropna(subset=["col1"])         # 仅看 col1 是否缺失
```

**填充缺失值**：

```python
df.fillna(0)                                    # 固定值填充
df.fillna({"col1": 0, "col2": "未知"})          # 分列指定值
df.fillna(df[["col1","col2"]].mean())           # 均值填充
df.fillna(method="ffill")                       # 前向填充（用前面的值）
df.fillna(method="bfill")                       # 后向填充（用后面的值）
df.ffill()                                      # 前向填充简写
df.bfill()                                      # 后向填充简写
df.interpolate()                                # 线性插值
```

> **实战建议**：数值列用均值/中位数填充，分类列用众数填充，时间序列用前后向插值。填充前先分析缺失原因（随机缺失 vs 系统缺失），避免引入偏差。

### 3.2 重复值处理

```python
df.duplicated()                         # 布尔 Series，标记重复行
df.duplicated(subset=["col1"])          # 按指定列判断重复
df.drop_duplicates()                    # 删除重复行（保留首次出现）
df.drop_duplicates(subset=["col1"])     # 按指定列去重
df.drop_duplicates(keep="last")         # 保留最后出现的
```

### 3.3 类型转换

```python
df["col"].astype(float)          # 转为浮点
df["col"].astype(str)            # 转为字符串
pd.to_datetime(df["date"])       # 转为日期
pd.to_numeric(df["col"], errors="coerce")  # 转数值，错误变 NaN
```

## 四、数据运算

### 4.1 apply 函数

`apply()` 对 Series 逐元素、对 DataFrame 逐列/逐行执行自定义函数。

```python
# Series.apply：接收单个元素
s = pd.Series([10, 20, 30])
s.apply(lambda x: x * 2)           # [20 40 60]
s.apply(lambda x, p: x * p, p=3)   # [30 60 90]（传额外参数）

# DataFrame.apply：接收一个 Series（整列/整行）
df = pd.DataFrame({"a": [10, 20], "b": [30, 40]})
df.apply(lambda s: s.sum())         # 按列求和（axis=0，默认）
df.apply(lambda s: s.sum(), axis=1) # 按行求和
df.apply(lambda s: s["a"] + s["b"], axis=1)  # 按行操作多列
```

> **向量化提速**：`apply` 本质是 Python 循环，大数据量下很慢。能用向量化操作（直接运算或 `np.where`）就不要用 `apply`。需要时可用 `np.vectorize()` 将普通函数向量化。

### 4.2 groupby 分组聚合

分组的三步走：**split（拆分）-> apply（应用函数）-> combine（合并结果）**。

```python
# 基本语法
df.groupby("分组列")["聚合列"].聚合函数()

# 单字段分组
df.groupby("department_id")["salary"].mean()

# 多字段分组（结果为复合索引）
df.groupby(["department_id", "job_id"])["salary"].mean()

# 多字段分组，不设为索引
df.groupby(["dept", "job"], as_index=False)["salary"].mean()

# 查看分组
df.groupby("department_id").groups           # {10: [0,1], 20: [2,3], ...}
df.groupby("department_id").get_group(10)   # 获取某组数据

# 遍历分组
for name, group in df.groupby("department_id"):
    print(name, group.shape)
```

### 4.3 agg 聚合

```python
# 一次计算多个统计值
df.groupby("department_id")["salary"].agg(["min", "median", "max"])

# 不同列用不同聚合函数
df.groupby("department_id").agg({
    "job_id": "nunique",
    "salary": ["mean", "max"]
})

# 自定义聚合函数
df.groupby("department_id")["last_name"].agg(lambda x: set(x.str[0]))
```

### 4.4 transform 转换

`transform` 返回与原数据等长的结果，常用于特征工程。

```python
# 数据标准化：每组内减均值
df["salary_z"] = df.groupby("department_id")["salary"].transform(lambda x: x - x.mean())

# 分组内用均值填充缺失值
df["salary"] = df.groupby("department_id")["salary"].transform(
    lambda x: x.fillna(x.mean())
)
```

> **agg vs transform**：`agg` 缩减数据（分几组返回几行），`transform` 保持数据等长（每行都有对应的变换结果）。

### 4.5 filter 过滤

```python
# 保留平均薪资大于 5000 的部门
df.groupby("department_id").filter(lambda g: g["salary"].mean() > 5000)
```

### 4.6 cut 分箱

```python
pd.cut(df["salary"], bins=3)                    # 等宽分 3 箱
pd.cut(df["salary"], [0, 5000, 10000, 20000])   # 自定义边界
pd.cut(df["salary"], 3, labels=["低","中","高"])  # 自定义标签
```

## 五、数据合并

### 5.1 merge（类似 SQL JOIN）

```python
# 一对一合并（自动找同名列）
pd.merge(df1, df2)

# 指定合并键
pd.merge(df1, df2, on="employee")

# 列名不同时分别指定
pd.merge(df1, df2, left_on="emp_name", right_on="name")

# 用索引合并
pd.merge(df1, df2, left_index=True, right_index=True)

# 连接方式
pd.merge(df1, df2, how="inner")  # 内连接（默认，取交集）
pd.merge(df1, df2, how="outer")  # 外连接（取并集，NaN 填充）
pd.merge(df1, df2, how="left")   # 左连接
pd.merge(df1, df2, how="right")  # 右连接

# 处理重名列
pd.merge(df1, df2, on="name", suffixes=("_左", "_右"))
```

> **merge 连接类型**：一对一是标准合并；多对一将重复值展开保留；多对多产生笛卡尔积（行数=左组数 x 右组数），数据量会爆炸，慎用。

### 5.2 concat（堆叠拼接）

```python
# 按行堆叠（axis=0，默认）
pd.concat([df1, df2])

# 按列拼接（axis=1）
pd.concat([df1, df2], axis=1)

# 重置索引
pd.concat([df1, df2], ignore_index=True)

# 交集列（仅保留共有列）
pd.concat([df1, df2], join="inner")
```

### 5.3 join（按索引合并）

```python
# df.join() 按索引合并，列名冲突需加后缀
df1.join(df2, lsuffix="_left", rsuffix="_right")
```

> **选择建议**：按**列值**关联用 `merge`，按**索引**关联用 `join`，简单上下拼接用 `concat`。

## 六、数据透视表

```python
df.pivot_table(values="待聚合列", index="行分组列", columns="列分组列",
               aggfunc="mean", fill_value=0, margins=True)
```

```python
# 示例：不同睡眠时间、不同压力等级下的睡眠质量均值
df.pivot_table(values="sleep_quality",
    index=pd.cut(df["sleep_duration"], [0,6,7,8,9,12]),
    columns=pd.cut(df["stress_level"], 4), aggfunc="mean")
```

## 七、文件读写

### CSV

```python
df = pd.read_csv("data.csv")
df = pd.read_csv("data.csv", index_col="id")          # 指定行索引
df = pd.read_csv("data.csv", sep="\t", parse_dates=[0], usecols=["name","age"])
df.to_csv("output.csv", index=False)                   # 不保存行索引
```

### Excel / JSON / Pickle / 其他

```python
df = pd.read_excel("data.xlsx", sheet_name="Sheet1")   # 需 openpyxl
pd.read_json("data.json") / df.to_json("out.json")
pd.read_pickle("data.pkl") / df.to_pickle("out.pkl")   # Python 专用，速度快
pd.read_feather("data.feather") / df.to_feather("out.feather")  # 列式存储，极快
pd.read_hdf("data.h5", key="df") / df.to_hdf("out.h5", key="df")

# SQL
from sqlalchemy import create_engine
engine = create_engine("sqlite:///data.db")
pd.read_sql("SELECT * FROM table", engine)
df.to_sql("table", engine, if_exists="replace", index=False)
```

### 日期处理

```python
df["date"] = pd.to_datetime(df["date"])    # 字符串转日期
df = pd.read_csv("data.csv", parse_dates=["date"])  # 读取时解析
df["year"] = df["date"].dt.year            # 提取年
df["month"] = df["date"].dt.month          # 提取月
df["quarter"] = df["date"].dt.to_period("Q")  # 转为季度周期
```

---

## 相关笔记

- [[00. 大模型知识体系导航]]
- NumPy 是深度学习框架（PyTorch/TensorFlow）的底层数据结构基础，理解 ndarray 有助于理解 Tensor 操作
- Pandas 在数据预处理、特征工程、EDA（探索性数据分析）中不可或缺，是机器学习流水线的第一步
