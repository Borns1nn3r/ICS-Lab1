# 计算机系统基础 Lab1：DataLab 实验报告

- **姓名**：陈星沅
- **学号**：25800190027
- **实验**：Lab1 DataLab
- **完成环境**：WSL2 Ubuntu（x86-64），gcc-multilib + libc6-dev-i386，`-m32 -fwrapv`

---

## 一、实验结果

### 1. 运算符与操作数检查

```
$ ./check_ops.py bits.c
signMask: 1/2 operators
bitXor: 7/8 operators
negativePart: 4/6 operators
copyByteWithin: 9/12 operators
logicalShift: 6/20 operators
swapNibblePairs: 9/18 operators
secondLowestZeroBit: 7/8 operators
oddParity: 12/56 operators
rotateRightBits: 12/16 operators
roundEvenPow2: 20/24 operators
midpointTowardFirst: 22/32 operators
isBetweenEitherOrder: 41/48 operators
mul5Sat: 23/30 operators
classifyAdd3: 32/52 operators
floatScaleThreeHalves: 41/60 operators
floatRoundEven: 34/65 operators
float_i2f: 37/40 operators
bitCount: 33/40 operators
bitReverse: 34/34 operators
All 19 functions passed operator checks.
```

> 截图：`images/check_ops.png`

### 2. 正确性测试

```
$ ./btest
Score   Rating  Errors  Function
 1      1       0       signMask
 2      2       0       bitXor
 3      3       0       negativePart
 4      4       0       copyByteWithin
 4      4       0       logicalShift
 4      4       0       swapNibblePairs
 4      4       0       secondLowestZeroBit
 5      5       0       oddParity
 5      5       0       rotateRightBits
 5      5       0       roundEvenPow2
 5      5       0       midpointTowardFirst
 7      7       0       isBetweenEitherOrder
 7      7       0       mul5Sat
 7      7       0       classifyAdd3
 7      7       0       floatScaleThreeHalves
 10     10      0       floatRoundEven
 10     10      0       float_i2f
 10     10      0       bitCount
 10     10      0       bitReverse
Total points: 110/110
```

> 截图：`images/btest.png`

以下实现思路中，所有“操作数”均以 `check_ops.py` 的实际计数为准。整数题遵守 32 位机器模型（补码、算术右移、只允许 `! ~ & ^ | + << >>` 与 0–255 常量）。

---

## 二、整数题实现思路

### P1 `signMask` — 返回 0x80000000

直接构造最高位：`1 << 31`。1 个操作符。

### P2 `bitXor` — 只用 `~` 和 `&` 实现异或

`x^y` 当且仅当两位“一者为一、另一者为零”。用德摩根式的写法：

```
~(~x & ~y) & ~(x & y)
```

`~x & ~y` 描述“两位都为 0”，取反后为“至少一位为 1”；`x & y` 描述“两位都为 1”，取反后为“并非两位都为 1”。两者取交，恰好是“恰好一位为 1”。共 7 个操作符。

### P3 `negativePart` — 负数返回相反数，否则返回 0

先取符号掩码 `s = x >> 31`（负数得 `0xFFFFFFFF`，非负得 0），再用 `~x + 1` 得到 `-x`，两者按位与：`(x >> 31) & (~x + 1)`。非负时 `-x` 被掩码清掉，负数时保留。4 个操作符。

### P4 `copyByteWithin` — 把第 src 字节复制到第 dst 字节

按字节号乘 8 得到位移量：

```
int ss = src << 3;
int ds = dst << 3;
int byte = (x >> ss) & 0xFF;
return (x & ~(0xFF << ds)) | (byte << ds);
```

先用 `~(0xFF << ds)` 把目标字节清零，再把取出的字节移到位后合并。

### P5 `logicalShift` — 逻辑右移

算术右移会补符号位，需要把多出来的高位清除。构造掩码：

```
(x >> n) & ~(((1 << 31) >> n) << 1)
```

`(1 << 31) >> n` 恰好是“算术右移后本该补进来的符号位”，左移一位再取反得到低 `32-n` 位为 1 的掩码，与结果相与即可。对 n=0 和 n=31 都成立。6 个操作符。

### P6 `swapNibblePairs` — 每个字节内交换高低半字节

掩码 `m = 0x0F0F0F0F`（用 `0x0F | (0x0F<<8)` 再 `|` 上 `<<16` 构造）。低半字节左移 4、高半字节右移 4 后合并：

```
((x & m) << 4) | ((x >> 4) & m)
```

### P7 `secondLowestZeroBit` — 次低 0 位的位置掩码

先求最低 0 位：设 `p` 为最低 0 位位置，则 `~x & (x+1)` 恰好只有第 p 位为 1（`x+1` 把从低位到 p 的连续 1 全部进位，`~x` 在 0 位处为 1）。把该位置 1 后再对新的值求最低 0 位，即得次低 0 位：

```
int z = x | (~x & (x + 1));
return ~z & (z + 1);
```

若不存在次低 0 位（如全 1、仅一个 0），结果为 0。7 个操作符。

### P8 `oddParity` — 奇偶校验位（1 的个数为偶数返回 1）

把 32 位对折异或，逐层缩小到最低位：`x ^= x>>16; x ^= x>>8; x ^= x>>4; x ^= x>>2; x ^= x>>1;`。此时最低位是全部 32 位的异或（1 的个数奇偶性）。题目要求“偶数个 1 返回 1”，故返回 `~y & 1`。12 个操作符。

### P9 `rotateRightBits` — 循环右移（n 按 32 取模）

`k = n & 31`，右半部分用逻辑右移（复用了 P5 的掩码技巧），左半部分左移 `32-k`：

```
int r = x << ((~k + 1) & 31);
int l = (x >> k) & ~(((1 << 31) >> k) << 1);
return l | r;
```

`(~k + 1) & 31` 即 `(-k) mod 32`，保证 k=0 时位移量为 0 不会越界。12 个操作符。

### P10 `roundEvenPow2` — 四舍六入五取偶到 2^n 的倍数

`q = x >> n` 为商，`r = x & ((1<<n)-1)` 为余数，`half = 2^(n-1)` 为半程。只有 `r > half`，或 `r == half` 且 `q` 为奇数时才进一位：

```
int mask = (1 << n) + ~0;
int q = x >> n;
int r = x & mask;
int half = 1 << (n + ~0);
int d = half + ~r + 1;          /* half - r */
int gt = (d >> 31) & 1;         /* r > half 时 d<0 */
int eq = !(r ^ half);
int roundup = gt | (eq & (q & 1));
return (q + roundup) << n;
```

### P11 `midpointTowardFirst` — 不溢出的 (x+y)/2，平局偏向 x

向下取整的平均值有经典无溢出写法：`base = (x & y) + ((x ^ y) >> 1)`。当 `x+y` 为奇数（`(x^y)&1`）且 `x > y` 时，数学中点在小数上，应向 x 侧进位。判断 `x > y` 用溢出安全的符号法：同号看 `x-y` 的符号，异号看 x 是否非负。

```
int t = x ^ y;
int base = (x & y) + (t >> 1);
int dx = x + ~y + 1;
int same = ~((x >> 31) ^ (y >> 31));
int raw = (same & ~(dx >> 31)) | (~same & ~(x >> 31));
int inc = (t & 1) & (raw & 1);
return base + inc;
```

22 个操作符。

### P12 `isBetweenEitherOrder` — 判断 x 是否落在 [a,b]（端点无序）

思路：`x` 在区间内 ⟺ `(x≥a 且 b≥x) 或 (x≥b 且 a≥x)`。关键是**比较不能直接用减法判断符号，因为 `INT_MIN - INT_MAX` 这类会溢出**。采用溢出安全的“小于”判定：

```
lt(u,v) = (u & ~v) | (~(u ^ v) & (u - v))    /* 最高位为 1 表示 u<v */
```

- 若 u、v 异号，`u<v` 等价于 u 为负（`u & ~v` 的最高位）；
- 若同号，减法不会溢出，最高位就是 `u-v` 的符号。

对 (x,a)、(a,x)、(x,b)、(b,x) 各做一次（保留其符号掩码），再组合：

```
int lxa = ... >> 31;   /* x < a */
int lax = ... >> 31;   /* a < x */
int lxb = ... >> 31;   /* x < b */
int lbx = ... >> 31;   /* b < x */
return ((~lxa & ~lbx) | (~lxb & ~lax)) & 1;
```

41 个操作符（上限 48）。

### P13 `mul5Sat` — x*5，溢出则饱和

饱和阈值是 `INT_MAX/5 = 0x19999999`。先求 `|x|`（`(x^s) + ~s + 1`，`s=x>>31`；`INT_MIN` 会得到自身，恰好仍落在溢出区），与阈值比较：

```
int c = (0x19 << 24) + (0x99 << 16) + (0x99 << 8) + 0x99;  /* 0x19999999 */
int e = m + ~c;          /* |x| - c - 1，e >= 0 表示 |x| > c */
int ov = ~(e >> 31);     /* 溢出掩码 */
int sat = ~(1 << 31) ^ s;/* x>=0 取 INT_MAX，否则 INT_MIN */
int r = x + (x << 2);    /* 5x 的低 32 位 */
return r ^ ((r ^ sat) & ov);
```

23 个操作符。

### P14 `classifyAdd3` — 判断 x+y+z 相对 int 范围的大小

不能开更宽的类型，于是先算 `t=x+y` 并记录进位方向，再算 `t+z` 并记录第二次进位，把两次“溢出方向”累加得到 `K`（即精确和相对 32 位结果的 2^32 倍数）。`K>0` 表示精确和大于 `INT_MAX`，`K<0` 表示小于 `INT_MIN`，`K=0` 表示在范围内。

```
int c1 = (up1 & 1) + dn1;   /* 第一次加法溢出方向：+1 / -1 / 0 */
int c2 = (up2 & 1) + dn2;
int K  = c1 + c2;
```

最后把 K 的符号映射为 1 / -1 / 0（`neg | pos`）。32 个操作符。

---

## 三、浮点题实现思路

浮点题允许任意整数常量、比较和控制流，但不允许浮点类型/常量/运算，也不允许强制类型转换。

### P15 `floatScaleThreeHalves` — f * 1.5（就近偶数舍入）

把浮点数看成 `m * 2^e`：规格化数 `m=(1<<23)|man, e=exp-150`，非规格化数 `m=man, e=-149`。乘以 1.5 即 `1.5m * 2^e`。

- 规格化输入：`n = 3m`，其最高位必在第 24 或 25 位。据此决定右移 1 位还是 2 位来归一化，并在丢弃低位时做“就近偶数”舍入；必要时指数进位，指数到 255 则返回无穷。
- 非规格化输入：`n = 3*man`，结果单位是 `2^-149`，即 `round(n/2)` 作为尾数；若尾数达到 `2^23` 则进位为最小规格化数。
- `±0` 保持符号原样返回；`NaN`/`Inf`（阶码 0xFF）直接返回参数。

### P16 `floatRoundEven` — 舍入到最近整数（就近偶数）

按阶码分类处理：

- 阶码 ≥ 150：本身已是整数，直接返回。
- 阶码 ≤ 125：绝对值 < 0.5，结果为 0，**保留符号**（因此负数得到 -0）。
- 阶码 = 126：值在 [0.5, 1)，恰好 0.5 时（尾数为 0）取偶返回 0，否则返回 ±1.0。
- 127 ≤ 阶码 ≤ 149：丢弃低 `k = 150-e` 位得到整数部分，用“就近偶数”舍入，再把得到的整数重新规格化成浮点编码。

### P17 `float_i2f` — int 转 float

取绝对值后找最高位 `hb`：

- `hb ≤ 23`：直接左移补齐，尾数无舍入。
- `hb > 23`：右移 `hb-23` 位，用 **guard / sticky** 做就近偶数舍入；若舍入导致尾数溢出到 24 位，则指数加一并把尾数重置为隐含的 1。

`INT_MIN` 通过无符号取负正确处理（`absx` 为 32 位无符号）。

---

## 四、难题实现思路

### P18 `bitCount` — 统计 1 的个数

经典“分治并行加法”（SWAR）：

1. 把相邻两位相加得到 2 位计数值；
2. 再按 4 位、8 位、16 位分别相加；

需要的掩码 `0x5555...`、`0x3333...`、`0x0F0F...`、`0x00FF00FF` 用“移位后自或”逐层扩展（如 `0x55 → 0x5555 → 0x55555555`）。最后把两个 16 位计数值相加后取低 8 位。33 个操作符。

### P19 `bitReverse` — 二进制位逆序（34 个操作符，卡线通过）

采用 5 层“分块交换”：依次交换距离 1、2、4、8、16 的相邻块。其中距离 16 的交换会受算术右移的符号扩展影响，因此也做成带掩码的安全写法 `(x<<16) | ((x>>16) & M16)`。

亮点在于**掩码的自推导链**，把 5 个掩码的操作数从 14 压到 10：

```
M16 = 0xFF | (0xFF << 8);      // 0x0000FFFF
M8  = M16 ^ (M16 << 8);        // 0x00FF00FF
M4  = M8  ^ (M8  << 4);        // 0x0F0F0F0F
M2  = M4  ^ (M4  << 2);        // 0x33333333
M1  = M2  ^ (M2  << 1);        // 0x55555555
```

每一层都是“相邻块半周期整体平移后异或”，只需 2 个操作符即可得到下一个掩码。5 个掩码共 10 个操作符，加上 5 段交换（4+5+5+5+5）共 34 个，正好满足上限。

---

## 五、参考资料

- Randal E. Bryant, David R. O'Hallaron. *Computer Systems: A Programmer's Perspective* (3rd ed.), 第 2 章“信息的表示和处理”。
- 复旦大学 ICS 2026 秋 Lab1 DataLab 实验文档与 `README.md`。
- CMU CS:APP Data Lab 官方说明。

---

## 六、实验感受（可选）

DataLab 让我把课本里“补码、掩码、移位、IEEE 754”从抽象定义变成了手上的操作。印象最深的两点：一是**算术右移的符号扩展**几乎处处是坑，从 `logicalShift` 到 `bitReverse` 都必须显式处理；二是**溢出**在比较和求和类题目里必须专门防御，例如 P12 不能用减法符号直接比较、P14 要靠进位去向推断精确和的大小。把 P19 从“能跑”优化到卡满 34 个操作符的过程，也让我体会到掩码之间可以互相推导，好的位运算写法在正确之外还追求紧凑。
