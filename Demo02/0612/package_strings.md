## strings.Contains() 源码

```go
// Contains 判断substr是否在s内，返回bool值
func Contains(s, substr string) bool {  
    return Index(s, substr) >= 0  
}

// Index 查找substr子串在s中首次出现的位置
func Index(s, substr string) int {  
    n := len(substr)  
    switch {  
    case n == 0:  
       return 0  
    case n == 1:
	   // 判断substr的第一个元素在s中的位置
	   // 这里的IndexByte调用了IndexByteString方法
	   // 该方法使用汇编语言编写的，提高性能	 
       return IndexByte(s, substr[0])  
    case n == len(s):  
       if substr == s {  
          return 0  
       }  
       return -1  
    case n > len(s):  
       return -1  
    case n <= bytealg.MaxLen: // 判断是否整数溢出
       // MaxBruteForce 小字符串的暴力搜索阈值
       if len(s) <= bytealg.MaxBruteForce {  
		  // 判断substr字符串在s中的位置
          return bytealg.IndexString(s, substr)  
       }  
       c0 := substr[0]  
       c1 := substr[1]  
       i := 0  
       t := len(s) - n + 1  
       fails := 0  
       for i < t {  
          if s[i] != c0 {  
             // 判断c0是否存在于i之后的字符中，返回c0第一次出现在s中的索引位置
             o := IndexByte(s[i+1:t], c0)
             if o < 0 {  
                return -1  
             }  
             i += o + 1  
          }
          // 先快速比较子串第二个字符是否匹配
          // 匹配，比较整个子串
          if s[i+1] == c1 && s[i:i+n] == substr {  
             return i  
          }  
          // 当前i位置匹配失败，增加失败计数器
          fails++  
          i++  
          // 检查是否超过最大失败的尝试阈值
          if fails > bytealg.Cutover(i) {  
             r := bytealg.IndexString(s[i:], substr)  
             if r >= 0 {  
                return r + i  
             }  
             return -1  
          }  
       }  
       return -1  
    }  
    // 当n>bytealg.MaxLen时，会执行到该逻辑
    c0 := substr[0]  
    c1 := substr[1]  
    i := 0  
    t := len(s) - n + 1  
    fails := 0  
    for i < t {  
       if s[i] != c0 {  
          o := IndexByte(s[i+1:t], c0)  
          if o < 0 {  
             return -1  
          }  
          i += o + 1  
       }  
       if s[i+1] == c1 && s[i:i+n] == substr {  
          return i  
       }  
       i++  
       fails++  
       if fails >= 4+i>>4 && i < t {  
          // 使用IndexRabinKarp算法处理大字串
          j := bytealg.IndexRabinKarp(s[i:], substr)  
          if j < 0 {  
             return -1  
          }  
          return i + j  
       }  
    }  
    return -1  
}
```

#### Rabin-Karp算法：是一种基于哈希的字符串搜索算法。
##### 核心思想：
1. **滚动哈希**：计算子串的哈希值，并在主串中滑动窗口计算哈希值。
2. **哈希匹配**：当哈希值匹配时，再进行精确比较（防止哈希冲突）。
3. **高效更新**：利用前一个窗口的哈希值快速计算新窗口的哈希值。

## strings.Replace() 源码

```go
// Replace 将字符串s中的前n个old子串替换为new子串。
// s：原始字符串  
// old：需要被替换的子串
// new：替换后的新子串
// n：替换的次数（如果 `n < 0`，则替换所有匹配项）
func Replace(s, old, new string, n int) string {  
    if old == new || n == 0 {  
       return s // 避免分配新的内存空间  
    }  
  
    // old在s中出现的次数
    if m := Count(s, old); m == 0 {  
       return s 
    } else if n < 0 || m < n {  
       n = m  
    }  
  
    // 初始化缓冲区 用于将结果写入缓冲区
    var b Builder
    // 预先分配足够的内存空间，避免多次扩容  
    b.Grow(len(s) + n*(len(new)-len(old)))  
    start := 0  
    for i := 0; i < n; i++ {  
       j := start  
       if len(old) == 0 { // 空字符串特殊处理  
          if i > 0 {  
             // 返回s[start:]中第一个Unicode字符以及字节宽度
             // 因为Unicode是变长编码，不同的Unicode字符占用不同的字节数
             _, wid := utf8.DecodeRuneInString(s[start:])  
             j += wid  
          }  
       } else {  
          j += Index(s[start:], old) // 找到old在剩余字符串中的位置
       }  
       b.WriteString(s[start:j]) // 写入从start到j未替换的部分
       b.WriteString(new) // 写入替换后的new子串
       start = j + len(old) // 更新start
    }  
    b.WriteString(s[start:]) // 处理剩余部分  
    return b.String() 
}


// Count 计算substr在s字符串中出现的次数
func Count(s, substr string) int {  
    // 如果substr为 ""，返回s中Unicode字符数量+1
    if len(substr) == 0 {  
       return utf8.RuneCountInString(s) + 1  
    }  
    // 如果是单个字符，调用CountString方法进行统计
    // bytealg.CountString 是 Go 标准库底层优化过的函数，
    // 用于快速计算单个字节在字符串中的出现次数
    if len(substr) == 1 {  
       return bytealg.CountString(s, substr[0])  
    }  
    n := 0  
    for {  
       i := Index(s, substr)  
       if i == -1 {  
          return n  
       }  
       n++  
       s = s[i+len(substr):]  
    }  
}
```

## strings.Split() 源码

```go
// Split 字符串分割方法
// 
// s 需要被分割的原始字符串
// sep 用作分隔符的字符串
func Split(s, sep string) []string { return genSplit(s, sep, 0, -1) }
```

```go
// genSplit 分割方法
//
// sepSave = 0：表示不保留分隔符
// n = -1：表示分割所有出现的分隔符
func genSplit(s, sep string, sepSave, n int) []string {  
    if n == 0 {  
       return nil  
    }  
    // 处理sep为 "" 的情况
    if sep == "" {
       // 将s拆分成一个UTF-8字符串切片
       // 每个Unicode字符对应一个字符串  
       return explode(s, n)  
    }  
    if n < 0 {
	   // 获取sep在s中的出现的次数 + 1  
       n = Count(s, sep) + 1  
    }  
    // 分割次数太大 重置为最大分割次数
    if n > len(s)+1 {  
       n = len(s) + 1  
    }  
    a := make([]string, n)  
    n--  
    i := 0  
    // 主分割循环
    for i < n {
	   // 获取sep出现在s中的第一次的索引	  
       m := Index(s, sep)  
       if m < 0 {  
          break  
       }  
       a[i] = s[:m+sepSave]  
       s = s[m+len(sep):]  
       i++  
    } 
    // 处理剩余部分 
    a[i] = s  
    return a[:i+1]  
}

// explode 将字符串分割成单个 Unicode 字符（rune）组成的切片
// 
// s：需要被分割的原始字符串
// n：控制返回结果的最大长度
func explode(s string, n int) []string {  
	// 计算字符串的字符数量
    l := utf8.RuneCountInString(s)
    // 处理n的边界情况
    if n < 0 || n > l {  
       n = l  
    }  
    // 创建字符串切片
    a := make([]string, n)  
    for i := 0; i < n-1; i++ { 
       // 获取第一个Unicode字符及其字符宽度
       _, size := utf8.DecodeRuneInString(s)
       // 根据字符宽度就可以跳过当前元素到下一个  
       a[i] = s[:size]  
       s = s[size:]  
    } 
    // 将剩余部分存入切片最后一个位置    
    if n > 0 {  
       a[n-1] = s  
    }  
    return a  
}
```