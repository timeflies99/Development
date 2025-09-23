

## 一、什么是AC自动机？

**AC自动机**（Aho-Corasick Automaton）是一种高效的**多模式字符串匹配算法**，由 Alfred Aho 和 Margaret Corasick 于1975年提出。它能够在一次扫描中同时匹配多个模式串，时间复杂度接近 O(n)，其中 n 是文本长度。

### ✅ 核心优势：
- 一次遍历完成多关键词匹配
- 支持失败跳转，避免回溯
- 可扩展性强，支持多种匹配策略

### 🌐 典型应用场景：

| 场景             | 说明                                                                 |
|------------------|----------------------------------------------------------------------|
| 敏感词过滤       | 构建敏感词库，实时检测并替换违规内容（如评论、弹幕、聊天系统）         |
| 生物信息学       | 快速定位DNA序列中的特定基因片段                                     |
| 网络入侵检测     | 匹配攻击特征码，识别恶意流量                                         |
| 日志分析         | 批量提取日志中的关键事件或错误码                                     |
| 搜索引擎索引     | 快速建立关键词倒排索引                                               |

---

## 二、AC自动机的核心结构

AC自动机由三部分构成：

1. **Trie树（字典树）**：存储所有模式串。
2. **失败指针（Failure Pointer）**：匹配失败时的跳转路径，类似KMP中的next数组。
3. **输出指针（Output List）**：记录当前节点可匹配的完整模式串集合。

---

## 三、基础实现：Python版AC自动机

```python
from collections import defaultdict, deque


class TrieNode:
    def __init__(self):
        self.children = defaultdict(TrieNode)  # 子节点映射
        self.fail = None                       # 失败指针
        self.is_end = False                    # 是否为模式串结尾
        self.output = []                       # 匹配到的完整模式串列表


class AhoCorasick:
    def __init__(self):
        self.root = TrieNode()

    def add_pattern(self, pattern: str):
        """添加一个模式串到Trie树"""
        node = self.root
        for char in pattern:
            node = node.children[char]
        node.is_end = True
        node.output.append(pattern)

    def build_failure_links(self):
        """BFS构建失败指针和输出合并"""
        queue = deque()
        
        # 初始化：根节点的直接子节点失败指针指向根
        for child in self.root.children.values():
            child.fail = self.root
            queue.append(child)

        while queue:
            current = queue.popleft()
            for char, child in current.children.items():
                # 寻找当前字符在失败路径上的匹配
                fail_node = current.fail
                while fail_node and char not in fail_node.children:
                    fail_node = fail_node.fail
                
                child.fail = fail_node.children[char] if fail_node else self.root
                
                # 合并失败路径上的输出（重要！）
                child.output.extend(child.fail.output)
                queue.append(child)

    def search(self, text: str) -> list[tuple[int, str]]:
        """
        在文本中查找所有匹配的模式串
        返回：[(起始位置, 匹配模式), ...]
        """
        node = self.root
        results = []

        for i, char in enumerate(text):
            # 失败跳转直到匹配或回到根
            while node and char not in node.children:
                node = node.fail
            if not node:
                node = self.root
                continue

            node = node.children[char]
            
            # 输出当前节点所有匹配模式
            for pattern in node.output:
                start_index = i - len(pattern) + 1
                results.append((start_index, pattern))

        return results
```

> ✅ **说明：**
> - `output` 列表在构建失败指针时已合并，因此搜索时无需递归查找。
> - 时间复杂度：构建 O(Σ|pattern|)，搜索 O(n + m)，其中 m 是匹配结果总数。

---

## 四、高级匹配策略

### 1️⃣ 最长匹配优先（Longest Match First）

在关键词过滤中，我们常希望优先匹配**最长关键词**，避免短词误匹配（如“北京” vs “北京市”）。

```python
def search_longest(self, text: str) -> tuple[int, str] | None:
    """
    返回文本中第一个最长匹配的模式串
    """
    node = self.root
    best_match = None
    best_length = 0

    for i, char in enumerate(text):
        while node and char not in node.children:
            node = node.fail
        if not node:
            node = self.root
            continue

        node = node.children[char]

        # 检查当前节点是否有匹配模式
        for pattern in node.output:
            length = len(pattern)
            if length > best_length:
                best_length = length
                best_match = (i - length + 1, pattern)

    return best_match
```

---

### 2️⃣ 跳过任意字符匹配（Wildcard Matching）

允许在匹配过程中**跳过任意字符**，适用于模糊匹配场景（如“h*llo”匹配“hello”、“hallo”等）。

⚠️ **注意：此算法复杂度较高，仅适用于短文本或小模式集。**

```python
def search_with_any_skips(self, text: str) -> list[tuple[str, int]]:
    """
    允许跳过任意字符进行匹配（广度优先搜索状态空间）
    返回所有可能匹配结果
    """
    from collections import deque
    matches = []
    queue = deque([(self.root, 0, [])])  # (当前节点, 位置, 已匹配字符列表)

    while queue:
        node, pos, matched = queue.popleft()
        if pos >= len(text):
            continue

        char = text[pos]

        # 尝试匹配当前字符
        if char in node.children:
            next_node = node.children[char]
            new_matched = matched + [char]
            if next_node.is_end:
                pattern = ''.join(new_matched)
                start = pos - len(new_matched) + 1
                matches.append((pattern, start))
            else:
                queue.append((next_node, pos + 1, new_matched))

        # 尝试跳过当前字符（无论是否匹配）
        queue.append((node, pos + 1, matched))

    return matches
```

---

### 3️⃣ 跳过指定关键词（Skip Keywords）

在匹配前**跳过某些干扰词**（如“的”、“是”、“了”等停用词），提升语义匹配准确性。

```python
def search_skip_keywords(self, text: str, skip_words: set[str]) -> list[tuple[int, str]]:
    """
    匹配时跳过指定关键词
    """
    node = self.root
    i = 0
    results = []

    while i < len(text):
        # 检查当前位置是否为跳过词开头
        skipped = False
        for word in skip_words:
            if text[i:i+len(word)] == word:
                i += len(word)  # 跳过整个词
                skipped = True
                break

        if skipped:
            continue

        char = text[i]

        # 正常AC匹配流程
        while node and char not in node.children:
            node = node.fail
        if not node:
            node = self.root
        else:
            node = node.children[char]

        # 输出匹配结果
        for pattern in node.output:
            start = i - len(pattern) + 1
            results.append((start, pattern))

        i += 1

    return results
```

---

### 4️⃣ 限制跳过字符数匹配（Bounded Skip）

允许在模式中**跳过最多 N 个字符**，适用于“手__机”匹配“手机号”、“手提机”等场景。

```python
def search_with_bounded_skips(self, text: str, max_skip: int) -> list[tuple[str, int]]:
    """
    允许跳过最多 max_skip 个字符进行匹配
    """
    from collections import deque
    matches = []
    # 状态: (节点, 位置, 已匹配字符, 已跳过字符数)
    queue = deque([(self.root, 0, [], 0)])

    while queue:
        node, pos, matched, skipped = queue.popleft()
        if pos >= len(text) or skipped > max_skip:
            continue

        char = text[pos]

        # 尝试匹配当前字符
        if char in node.children:
            next_node = node.children[char]
            new_matched = matched + [char]
            if next_node.is_end:
                pattern = ''.join(new_matched)
                start = pos - len(new_matched) + 1
                matches.append((pattern, start))
            else:
                queue.append((next_node, pos + 1, new_matched, 0))  # 重置跳过计数

        # 尝试跳过当前字符（仅当已有匹配时才允许跳过，避免无限跳过开头）
        if matched:  # 避免从空开始无限跳
            queue.append((node, pos + 1, matched, skipped + 1))

    return matches
```

---

## 五、性能优化策略

### 🔽 1. Trie树压缩（Path Compression）

合并单子节点路径，减少节点数，提高缓存命中率。

```python
def compress_trie(self, node: TrieNode = None):
    """递归压缩Trie树（后序遍历）"""
    if node is None:
        node = self.root

    # 先递归子节点
    for char in list(node.children.keys()):
        self.compress_trie(node.children[char])

    # 如果只有一个子节点，尝试压缩
    if len(node.children) == 1:
        char, child = next(iter(node.children.items()))
        if not node.is_end and not node.output:  # 非终止节点才可压缩
            # 合并子节点
            node.children = child.children
            node.output.extend(child.output)
            node.is_end = child.is_end
            # 重新压缩当前节点（可能继续压缩）
            self.compress_trie(node)
```

> ⚠️ 压缩后需重新构建失败指针！

---

### ⚡ 2. 双数组Trie（Double-Array Trie）

使用 `base[]` 和 `check[]` 两个数组实现Trie，内存连续、访问高效，适合大规模词典。

```python
class DoubleArrayTrie:
    def __init__(self):
        self.base = [0]
        self.check = [0]
        self.size = 1

    def build(self, patterns):
        """构建双数组Trie（此处为示意，完整实现较复杂）"""
        # 实际实现需处理冲突、动态扩展等
        raise NotImplementedError("完整实现请参考DATrie开源库")
```

> ✅ 推荐生产环境使用成熟库如 [`datrie`](https://pypi.org/project/datrie/) 或 [`pyahocorasick`](https://pypi.org/project/pyahocorasick/)。

---

## 六、关于 `pyahocorasick` 库

> **`pyahocorasick` 是工业级AC自动机实现，推荐用于生产环境。**

### ✅ 优点：
- C语言底层实现，速度极快
- 内存效率高
- 支持Unicode、序列化、模糊匹配等

### ❌ 局限：
- 不易定制匹配逻辑（如跳过、最长优先等）
- 无法直接修改内部结构

> 📌 **建议：学习时手写AC自动机，生产时优先使用 `pyahocorasick`。**

---

## 七、总结与建议

| 项目           | 建议                                                                 |
|----------------|----------------------------------------------------------------------|
| 学习阶段       | 手写AC自动机，理解失败指针与输出合并机制                             |
| 生产环境       | 使用 `pyahocorasick` 或 `ahocorasick-rs`（Rust）等成熟库             |
| 性能瓶颈       | 优先考虑压缩Trie、双数组Trie、预编译模式                             |
| 匹配策略       | 根据业务选择“最长匹配”、“跳过匹配”等策略                             |
| 扩展性         | 可结合正则、BM、后缀数组等构建混合匹配引擎                           |

