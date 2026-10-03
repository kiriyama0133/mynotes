Google Test (gtest) 的核心思想是通过**宏 来简化测试代码。要熟练使用 gtest，你需要掌握它的**命名规则、**核心断言**、**浮点数比较**、**字符串比较**以及进阶的**测试夹具**。

最长使用的两个宏： TEST和TEST_F，内部编写测试代码和断言。
```cpp
TEST(TestSuiteName, TestName) {
    // 测试代码与断言
}
```
- **TestSuiteName（测试套件名）：** 逻辑上的分类（如 `OrderManagerTest`、`MathUtilsTest`），同一类的测试用例用同一个套件名。
    
- **TestName（测试用例名）：** 具体的测试点（如 `CalculateTotalSuccess`、`DivideByZeroThrows`）。
    
- _注意：名称中不能包含下划线 `_`。_



gtest中的断言分为两类，一类是非致命断言：如果断言是失败的就会打印错误，允许**允许当前函数继续向下执行**。推荐绝大多数场景使用。一个是致命断言，发生错误，断言失败就会直接退出当前测试。

以上两个分别为EXPECT_* 和 ASSERT_* 。

|**基础断言 (EXPECT_ 可替换为 ASSERT_)**|**说明**|
|---|---|
|**`EXPECT_TRUE(condition)`**|期望条件为 `true`|
|**`EXPECT_FALSE(condition)`**|期望条件为 `false`|
|**`EXPECT_EQ(val1, val2)`**|期望 `val1 == val2`（两者相等）|
|**`EXPECT_NE(val1, val2)`**|期望 `val1 != val2`（两者不相等）|
|**`EXPECT_LT(val1, val2)`**|期望 `val1 < val2`（Less Than）|
|**`EXPECT_LE(val1, val2)`**|期望 `val1 <= val2`（Less Equal）|
|**`EXPECT_GT(val1, val2)`**|期望 `val1 > val2`（Greater Than）|
|**`EXPECT_GE(val1, val2)`**|期望 `val1 >= val2`（Greater Equal）|

由于计算机存储浮点数（`float`, `double`）存在精度误差，使用 `EXPECT_EQ(0.1 + 0.2, 0.3)` 往往会失败。gtest 提供了专用的浮点数比较宏：
```cpp
// 1. 自动判断（推荐）：判断两者是否“几乎相等”（默认在 4 个 ULP 误差范围内）
EXPECT_NEAR(val1, val2, abs_error) // 允许指定 abs_error 范围内的误差

// 2. 快捷宏
EXPECT_FLOAT_EQ(f1, f2)  // 针对 float 的近似比较
EXPECT_DOUBLE_EQ(d1, d2) // 针对 double 的近似比较
```

