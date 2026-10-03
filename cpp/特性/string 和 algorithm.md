# C++ `string` 和 `algorithm`

这两个库几乎是字符串题的默认搭档：

- `std::string` 负责存储、切片、查找、替换。
- `<algorithm>` 负责排序、翻转、变换、去重、统计区间操作。

把它们配合起来，用很少的代码就能把很多字符串题写得很清晰。

## `string` 常用操作速查

```cpp
#include <string>
#include <iostream>

int main() {
    std::string s1 = "hello";
    std::string s2 = "world";

    std::string s3 = s1 + " " + s2;   // 拼接: "hello world"
    s1.append(" C++");                 // 追加: "hello C++"
    s1.insert(6, "modern ");           // 插入: "hello modern C++"
    s1.replace(6, 6, "awesome");       // 替换: "hello awesome C++"

    size_t pos = s1.find("awesome");   // 查找，返回 6
    std::string sub = s1.substr(6, 7); // 子串: "awesome"

    char ch = s1[1];                   // 下标访问: 'e'
    char front = s1.front();           // 首字符
    char back = s1.back();             // 尾字符

    size_t len = s1.size();            // 长度
    bool empty = s1.empty();           // 是否为空
}
```

```cpp
// 容量相关
size() / length()    // 长度
capacity()           // 当前容量
reserve(n)           // 预留容量
resize(n, c)         // 调整长度，不足时用 c 填充
shrink_to_fit()      // 请求回收多余容量

// 修改相关
push_back(c)         // 末尾追加一个字符
pop_back()           // 删除最后一个字符
append(str)          // 追加字符串
insert(pos, str)     // 插入字符串
erase(pos, len)      // 删除子串
replace(pos, len, str) // 替换子串
clear()              // 清空
swap(str)            // 交换

// 查找相关
find(str, pos)       // 从左往右找
rfind(str, pos)      // 从右往左找
find_first_of(str)   // 找到第一个属于 str 集合的字符
find_last_of(str)    // 找到最后一个属于 str 集合的字符
find_first_not_of(str) // 找到第一个不属于 str 集合的字符
find_last_not_of(str)  // 找到最后一个不属于 str 集合的字符

// 比较相关
compare(str)         // <0 / ==0 / >0
== != < > <= >=      // 直接支持字典序比较
```

## `string` 里的几个高频细节

### 1. `find` 找不到时返回 `npos`

```cpp
std::string s = "hello";
size_t pos = s.find("abc");

if (pos == std::string::npos) {
    std::cout << "not found\n";
}
```

不要拿它和 `-1` 直接做语义判断，虽然很多时候结果“看起来能对”，但标准写法应该是 `std::string::npos`。

### 2. `substr(pos, len)` 的第二个参数是长度，不是右边界

```cpp
std::string s = "abcdef";

std::cout << s.substr(2, 3) << '\n'; // "cde"
```

### 3. `operator[]` 不检查越界，`at()` 会检查

```cpp
std::string s = "abc";
char x = s[1];     // 快，但不做边界检查
char y = s.at(1);  // 越界会抛异常
```

### 4. 频繁拼接长字符串时，建议先 `reserve`

```cpp
std::string result;
result.reserve(1000);

for (int i = 0; i < 100; ++i) {
    result += "abc";
}
```

这样可以减少反复扩容带来的开销。

## `<algorithm>` 在字符串里的高频招式

```cpp
#include <algorithm>
#include <cctype>
#include <string>
```

### 1. 排序 `sort`

```cpp
std::string s = "dbca";
std::sort(s.begin(), s.end());
// s == "abcd"
```

常用于：

- 判断变位词 / 异位词
- 字符去重前的预处理
- 把字符串变成规范形态

### 2. 翻转 `reverse`

```cpp
std::string s = "abcde";
std::reverse(s.begin(), s.end());
// s == "edcba"
```

### 3. 计数 `count`

```cpp
std::string s = "banana";
int n = std::count(s.begin(), s.end(), 'a');
// n == 3
```

### 4. 批量变换 `transform`

```cpp
std::string s = "AbC";
std::transform(s.begin(), s.end(), s.begin(),
    [](unsigned char ch) {
        return static_cast<char>(std::tolower(ch));
    });
// s == "abc"
```

### 5. 删除类操作 `remove` / `remove_if`

`remove` 不会真的删元素，它只是把“保留的元素”挪到前面，真正删除要配合 `erase`。

```cpp
std::string s = "a b  c";
s.erase(std::remove(s.begin(), s.end(), ' '), s.end());
// s == "abc"
```

```cpp
std::string s = "a1b2c3";
s.erase(std::remove_if(s.begin(), s.end(),
    [](unsigned char ch) {
        return std::isdigit(ch);
    }), s.end());
// s == "abc"
```

### 6. 去重 `unique`

`unique` 只能去掉**相邻重复元素**，所以很多时候要先排序。

```cpp
std::string s = "cbbaac";
std::sort(s.begin(), s.end()); // "aabbcc"
s.erase(std::unique(s.begin(), s.end()), s.end());
// s == "abc"
```

### 7. 条件判断 `all_of` / `any_of` / `none_of`

```cpp
std::string s = "12345";
bool allDigit = std::all_of(s.begin(), s.end(),
    [](unsigned char ch) {
        return std::isdigit(ch);
    });
```

### 8. 查找满足条件的字符 `find_if`

```cpp
std::string s = "   hello";
auto it = std::find_if(s.begin(), s.end(),
    [](unsigned char ch) {
        return !std::isspace(ch);
    });

if (it != s.end()) {
    std::cout << *it << '\n'; // 'h'
}
```

## 常见用例

下面这些基本就是刷题和日常代码里最常见的套路。

### 1. 判断回文串

```cpp
#include <string>

bool isPalindrome(const std::string& s) {
    int left = 0;
    int right = static_cast<int>(s.size()) - 1;

    while (left < right) {
        if (s[left] != s[right]) {
            return false;
        }
        ++left;
        --right;
    }
    return true;
}
```

如果题目要求“忽略大小写和非字母数字字符”，可以这样写：

```cpp
#include <cctype>
#include <string>

bool isPalindromeIgnore(const std::string& s) {
    int left = 0;
    int right = static_cast<int>(s.size()) - 1;

    while (left < right) {
        while (left < right &&
               !std::isalnum(static_cast<unsigned char>(s[left]))) {
            ++left;
        }
        while (left < right &&
               !std::isalnum(static_cast<unsigned char>(s[right]))) {
            --right;
        }

        char lc = static_cast<char>(
            std::tolower(static_cast<unsigned char>(s[left])));
        char rc = static_cast<char>(
            std::tolower(static_cast<unsigned char>(s[right])));

        if (lc != rc) {
            return false;
        }
        ++left;
        --right;
    }
    return true;
}
```

### 2. 判断两个字符串是否为异位词

方法一：排序，代码最短。

```cpp
#include <algorithm>
#include <string>

bool isAnagram(std::string a, std::string b) {
    if (a.size() != b.size()) {
        return false;
    }
    std::sort(a.begin(), a.end());
    std::sort(b.begin(), b.end());
    return a == b;
}
```

方法二：计数，时间复杂度更好，适合只含小写字母的题。

```cpp
#include <array>
#include <string>

bool isAnagram2(const std::string& a, const std::string& b) {
    if (a.size() != b.size()) {
        return false;
    }

    std::array<int, 26> cnt{};
    for (char ch : a) {
        ++cnt[ch - 'a'];
    }
    for (char ch : b) {
        --cnt[ch - 'a'];
    }

    for (int x : cnt) {
        if (x != 0) {
            return false;
        }
    }
    return true;
}
```

### 3. 删除字符串中的所有空格

```cpp
#include <algorithm>
#include <string>

void removeSpaces(std::string& s) {
    s.erase(std::remove(s.begin(), s.end(), ' '), s.end());
}
```

如果要删除所有空白字符，包括空格、换行、制表符，用 `remove_if`：

```cpp
#include <algorithm>
#include <cctype>
#include <string>

void removeAllWhitespace(std::string& s) {
    s.erase(std::remove_if(s.begin(), s.end(),
        [](unsigned char ch) {
            return std::isspace(ch);
        }), s.end());
}
```

### 4. 去掉首尾空白

```cpp
#include <algorithm>
#include <cctype>
#include <string>

std::string trim(std::string s) {
    auto left = std::find_if(s.begin(), s.end(),
        [](unsigned char ch) {
            return !std::isspace(ch);
        });

    auto right = std::find_if(s.rbegin(), s.rend(),
        [](unsigned char ch) {
            return !std::isspace(ch);
        }).base();

    if (left >= right) {
        return "";
    }
    return std::string(left, right);
}
```

### 5. 全部转小写 / 大写

```cpp
#include <algorithm>
#include <cctype>
#include <string>

void toLower(std::string& s) {
    std::transform(s.begin(), s.end(), s.begin(),
        [](unsigned char ch) {
            return static_cast<char>(std::tolower(ch));
        });
}

void toUpper(std::string& s) {
    std::transform(s.begin(), s.end(), s.begin(),
        [](unsigned char ch) {
            return static_cast<char>(std::toupper(ch));
        });
}
```

### 6. 替换所有子串

比如把所有 `"ab"` 替换成 `"XY"`：

```cpp
#include <string>

void replaceAll(std::string& s, const std::string& from, const std::string& to) {
    if (from.empty()) {
        return;
    }

    size_t pos = 0;
    while ((pos = s.find(from, pos)) != std::string::npos) {
        s.replace(pos, from.size(), to);
        pos += to.size();
    }
}
```

### 7. 统计每个字符出现次数

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

void countChars(const std::string& s) {
    std::unordered_map<char, int> freq;
    for (char ch : s) {
        ++freq[ch];
    }

    for (const auto& [ch, cnt] : freq) {
        std::cout << ch << ": " << cnt << '\n';
    }
}
```

如果字符集固定，比如只统计小写字母，用数组更快。

```cpp
#include <array>
#include <string>

std::array<int, 26> countLowerLetters(const std::string& s) {
    std::array<int, 26> cnt{};
    for (char ch : s) {
        if ('a' <= ch && ch <= 'z') {
            ++cnt[ch - 'a'];
        }
    }
    return cnt;
}
```

### 8. 对字符串排序后去重

这个套路很常见，经常用于“规范化”一个字符串。

```cpp
#include <algorithm>
#include <string>

std::string normalize(std::string s) {
    std::sort(s.begin(), s.end());
    s.erase(std::unique(s.begin(), s.end()), s.end());
    return s;
}
```

### 9. 分割字符串

标准库里最常见的是用 `std::stringstream`。

```cpp
#include <sstream>
#include <string>
#include <vector>

std::vector<std::string> splitBySpace(const std::string& s) {
    std::stringstream ss(s);
    std::vector<std::string> result;
    std::string word;

    while (ss >> word) {
        result.push_back(word);
    }
    return result;
}
```

如果按指定分隔符切分，比如逗号：

```cpp
#include <sstream>
#include <string>
#include <vector>

std::vector<std::string> splitByChar(const std::string& s, char delim) {
    std::stringstream ss(s);
    std::vector<std::string> result;
    std::string token;

    while (std::getline(ss, token, delim)) {
        result.push_back(token);
    }
    return result;
}
```

### 10. 压缩连续重复字符

例如 `"aaabbc"` 变成 `"a3b2c1"`：

```cpp
#include <string>

std::string compressString(const std::string& s) {
    if (s.empty()) {
        return "";
    }

    std::string result;
    for (size_t i = 0; i < s.size(); ) {
        size_t j = i;
        while (j < s.size() && s[j] == s[i]) {
            ++j;
        }
        result += s[i];
        result += std::to_string(j - i);
        i = j;
    }
    return result;
}
```

### 11. 找到第一个不重复字符

```cpp
#include <string>
#include <unordered_map>

int firstUniqChar(const std::string& s) {
    std::unordered_map<char, int> freq;
    for (char ch : s) {
        ++freq[ch];
    }

    for (int i = 0; i < static_cast<int>(s.size()); ++i) {
        if (freq[s[i]] == 1) {
            return i;
        }
    }
    return -1;
}
```

### 12. 比较两个字符串的公共前缀

```cpp
#include <string>

std::string commonPrefix(const std::string& a, const std::string& b) {
    size_t i = 0;
    while (i < a.size() && i < b.size() && a[i] == b[i]) {
        ++i;
    }
    return a.substr(0, i);
}
```

## 一眼就该想到的套路

遇到字符串题时，可以先想这几个方向：

1. 只关心字符出现次数：用数组或哈希表统计。
2. 只关心字符组成是否一样：排序后比较，或者计数比较。
3. 涉及左右对称：双指针。
4. 涉及批量删字符：`erase + remove_if`。
5. 涉及大小写统一：`transform`。
6. 涉及按规则重排：`sort`。
7. 涉及连续段处理：双指针或分组遍历。
8. 涉及查找某段子串：`find` / `rfind` / `substr`。

## 常见坑

### 1. `remove` 不是真删

错误理解：

```cpp
std::remove(s.begin(), s.end(), ' ');
```

正确写法：

```cpp
s.erase(std::remove(s.begin(), s.end(), ' '), s.end());
```

### 2. `tolower` / `toupper` / `isdigit` / `isspace` 最好接 `unsigned char`

推荐这样写：

```cpp
std::tolower(static_cast<unsigned char>(ch));
```

这是为了避免某些编译器和字符集环境下的未定义行为。

### 3. `sort` 需要随机访问迭代器

所以它可以直接排：

- `std::string`
- `std::vector`
- `std::deque`

但不能直接排 `std::list`。

### 4. 修改字符串时要注意迭代器失效

比如 `insert`、`erase`、`replace` 后，之前保存的迭代器或下标可能不再可靠，尤其是在循环里边删边改时要小心。

## 一个小结

`std::string` 负责“存”和“改”，`<algorithm>` 负责“批量处理”和“套路化操作”。

如果你把下面这些操作练熟，大多数基础字符串题都会顺很多：

- `find / substr / replace`
- `sort / reverse / count / transform`
- `erase + remove_if`
- 双指针
- 哈希计数
- 排序去重

真正写题时，不要一上来就手搓最底层循环，先想标准库里有没有现成套路，这样代码更短，也更不容易出错。
