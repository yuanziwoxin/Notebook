## 位置参数与关键字参数

在 Python 中，**位置参数**和**关键字参数**是两种传递参数的方式。下面是它们的定义和区别：

### 位置参数

- **定义**：位置参数是根据参数在函数定义中的位置来传递的参数。当你调用函数时，传递的参数值会依次赋给函数定义中的位置参数。

- 示例：

  ```python
  def greet(name):
      print(f"Hello, {name}!")
  
  greet("Alice")  # "Alice" 是位置参数
  ```

### 关键字参数

- **定义**：关键字参数是通过指定参数名称来传递的参数。在调用函数时，可以使用 `key=value` 的形式来传递参数，这样可以不必按照定义的顺序传递参数。

- 示例：

  ```python
  def greet(name, age):
      print(f"Hello, {name}! You are {age} years old.")
  
  greet(age=30, name="Bob")  # "name" 和 "age" 是关键字参数
  ```

### 区别

- **顺序性**：位置参数的顺序很重要，必须按照定义的顺序传递；关键字参数不受顺序限制，可以在调用时以任意顺序指定。
- **可读性**：使用关键字参数可以提高代码的可读性，特别是在函数参数较多时。

```python
# 1. 可变参数 *args（接收任意数量的位置参数，打包成元组）
def sum_all(*args):
    return sum(args)
 
print(sum_all(1, 2, 3, 4))  # 10
 
# 2. 关键字参数 **kwargs（接收任意数量的关键字参数，打包成字典）
def print_info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")
 
print_info(name="Alice", role="Engineer")
# name: Alice
# role: Engineer

print_info(role="Engineer", name="Alice")
# role: Engineer
# name: Alice
```

> *args 和 **kwargs都是可变参数
>
> - *args：可变位置参数
> - **kwargs：可变关键字参数（key=value）

### 总结

- **位置参数**：根据位置传递。
- **关键字参数**：根据名称传递。

## 特殊方法

在 Python 中，前后带下划线的方法被称为**特殊方法**（或魔法方法）。这些方法通常用于实现某些特定的功能，允许类的实例表现出某些特性或行为。

### 特殊方法

- **定义**：特殊方法是以双下划线开头和结尾的方法，例如 `__init__`、`__str__`、`__repr__` 等。这些方法具有特殊的意义，Python 解释器会在特定情况下自动调用它们。
- 用途：
  - `__init__`：构造函数，用于初始化对象的属性。
  - `__str__`：定义对象的“字符串表示”，当使用 `print()` 函数或 `str()` 函数时调用。
  - `__repr__`：定义对象的“官方字符串表示”，通常用于调试，能够准确地描述对象。

### self 变量

- **定义**：`self` 是**类实例的方法的第一个参数**，**代表当前对象的实例**。它并不是 Python 的关键字，而是一个**约定的名称**，通常使用 `self` 来表示。
- **引用实例属性**：通过 `self`，你可以直接引用实例的属性和方法。
  - 例如，`self.name` 访问的是当前实例的 `name` 属性。这是因为 `self` 指向当前对象，因此可以通过 `self` 来访问该对象的所有属性和方法。

### 示例

```python
class Agent:
    def __init__(self, name, model="gpt-4"):
        self.name = name  # 通过self将name属性与实例关联
        self.model = model
        self._memory = []

    def introduce(self):
        return f"我是 {self.name} ，使用 {self.model} 模型"

    def __str__(self):
        return f"Agent( {self.name} )"

    def __repr__(self):
        return f"Agent(name='{self.name}', model='{self.model}')"

# 创建一个Agent实例
agent = Agent("Monica")
print(agent.introduce())  # 输出: 我是 Monica ，使用 gpt-4 模型
print(agent)  # 输出: Agent( Monica )
```

**特殊方法的名称是固定的**。**这些名称遵循特定的命名约定，通常以双下划线开头和结尾**。以下是一些常见的特殊方法及其用途：

### 常见特殊方法

1. **`__init__`**：构造函数，用于初始化对象的属性。
2. **`__str__`**：定义对象的“字符串表示”，用于 `print()` 和 `str()` 函数。
3. **`__repr__`**：定义对象的“官方字符串表示”，通常用于调试和开发。
4. **`__len__`**：定义对象的长度（例如，使用 `len()` 函数时调用）。
5. **`__getitem__`**：定义对象的索引访问（例如，使用 `obj[key]` 时调用）。
6. **`__setitem__`**：定义对象的索引赋值（例如，使用 `obj[key] = value` 时调用）。
7. **`__delitem__`**：定义对象的索引删除（例如，使用 `del obj[key]` 时调用）。
8. **`__iter__`**：返回一个迭代器对象，用于支持迭代（例如，使用 `for` 循环时）。
9. **`__next__`**：定义迭代器的下一个值。
10. **`__call__`**：允许对象像函数一样被调用。

特殊方法的名称是固定的，遵循 Python 的约定。你可以在类中实现这些方法，以便自定义对象的行为。使用这些特殊方法可以让你的类更符合 Python 的数据模型，提供更好的用户体验。

### 总结

- **特殊方法**：以双下划线开头和结尾的方法，用于实现特定功能。
- **self**：代表当前对象的实例，通过它可以访问实例的属性和方法。

## Python四种核心内置数据结构

### 列表（list）— 可变有序集合

```python
# 创建
tools = ["search", "read_file", "write_code"]
mixed = [1, "hello", True, None]  # 可以混合类型
 
# 常用操作
tools.append("execute")           # 尾部添加
tools.insert(0, "plan")           # 指定位置插入
tools.pop()                       # 移除并返回最后一个
tools[0]                          # 索引访问（从 0 开始）
tools[-1]                         # 倒数第一个
tools[1:3]                        # 切片：[1] 和 [2]
```

### 字典（dict）— 键值对（类似 JS Object / Java HashMap）

```python
# 创建
config = {
    "model": "deepseek-v3",
    "temperature": 0.7,
    "max_tokens": 4096
}
 
# 常用操作
config["model"]                   # 访问（key 不存在会报错）
config.get("top_p", 1.0)          # 安全访问（不存在返回默认值）
config["stream"] = True           # 添加 / 修改
config.keys()                     # 所有键
config.values()                   # 所有值
config.items()                    # 所有键值对
```

在 Python 中，字典的键（key）必须是**不可变**（immutable）的数据类型。这意味着字典的键不能被修改。

字典的键（key）可以是如下类型：

- 字符串（str）
- 整数（int）
- 浮点数（float）
- 元组（tuple）
- 布尔值（bool）
- 冻结集合（frozenset)

不可以作为字典键的数据类型：

- 列表（list）
- 集合（set）
- 自定义对象（若未实现不可变特性）

```python
# 创建一个字典，包含不同类型的键
my_dict = {
    "name": "Alice",
    42: "The Answer",
    3.14: "pi",
    (1, 2): "Coordinates",
    True: "Affirmative",
    frozenset([1, 2, 3]): "set A"
}

# 输出字典内容
print(my_dict)
```

### 元组（tuple）— 不可变有序集合

```python
# 创建
point = (3, 4)
single = (1,)    # ← 单个元素的元组必须加逗号！
 
# 用途：函数返回多个值、作为字典的键、表示不会被修改的数据
def get_coordinates():
    return (116.40, 39.90)   # 实际返回的是一个元组
 
lat, lng = get_coordinates()  # 解包（unpacking）
```

### 集合（set）— 不重复无序集合

```python
# 创建
tags = {"python", "agent", "llm"}
 
# 常用操作
tags.add("fastapi")
tags.remove("llm")
"python" in tags       # True — O(1) 查找
 
# 集合运算
a = {1, 2, 3}
b = {2, 3, 4}
print(a & b)   # 交集：{2, 3}
print(a | b)   # 并集：{1, 2, 3, 4}
print(a - b)   # 差集：{1}
```

虽不能通过索引访问集合，但是可以通过如下方式访问集合中的元素：

- 使用循环遍历
- 使用 `in`关键字检查某个元素是否存在于集合中
- 使用集合运算：并集、交集和差集等

### 总结

- 在 Python 的内置数据结构中，**数组（array）** 是唯一要求**子元素类型一致**的结构。其他数据结构（**列表、元组、集合和字典**）都允许包含不同类型的元素；
- 不可变的为元组（tuple)；
- 可变的为列表（list）、字典（dict）、集合（set）;
- 有序的为列表（list）、元组（tuple）, 可通过索引直接访问；
- 无序的为字典（dict）、集合（set），不可通过索引直接访问；
- 集合里的元素是唯一，不重复的；

> 数组要求子元素类型一致
>
> ```python
> import array
> my_array = array.array('i', [1, 2, 3])  # 所有元素必须是整数
> ```

## 数组（array）

在 Python 中，数组**并不是一种内置的数据结构**，像列表（list）和元组（tuple）是更常用的序列类型。不过，Python 提供了 `array` 模块来创建数组，主要用于**处理数值型数据**。

### 1. Python 数组的定义

Python 的数组是通过 `array` 模块实现的，主要用于**存储相同类型的数据**。与列表相比，数组在**内存使用和性能**上更为高效，尤其是**在处理大量数值数据时**。

#### 导入 `array` 模块

要使用数组，首先需要导入 `array` 模块：

```python
import array
```

### 2. 创建数组

使用 `array` 模块创建数组时，需要指定一个**类型代码（type code）**，它表示数组中元素的类型。

#### 类型代码

以下是一些常用的类型代码：

- `'i'`: 有符号整数（signed int）
- `'f'`: 浮点数（float）
- `'d'`: 双精度浮点数（double）
- `'u'`: Unicode 字符（在 Python 3 中通常不推荐使用）

#### 示例代码

```python
import array

# 创建一个整数类型的数组
int_array = array.array('i', [1, 2, 3, 4, 5])

# 创建一个浮点数类型的数组
float_array = array.array('f', [1.1, 2.2, 3.3])
```

### 3. 数组的基本操作

Python 数组支持多种操作，类似于列表，但有一些限制，因为数组要求所有元素类型一致。

#### 访问元素

可以使用索引访问数组中的元素：

```python
print(int_array[0])  # 输出: 1
```

#### 修改元素

可以通过索引修改数组中的元素：

```python
int_array[1] = 20
print(int_array)  # 输出: array('i', [1, 20, 3, 4, 5])
```

#### 数组的长度

可以使用 `len()` 函数获取数组的长度：

```python
length = len(int_array)
print(length)  # 输出: 5
```

#### 添加和删除元素

数组没有直接的添加和删除方法，但可以使用 `append()` 和 `remove()` 方法：

```python
int_array.append(6)  # 添加元素
print(int_array)  # 输出: array('i', [1, 20, 3, 4, 5, 6])

# 删除元素
int_array.remove(20)
print(int_array)  # 输出: array('i', [1, 3, 4, 5, 6])
```

### 4. 数组的优缺点

- 优点
  - **内存效率**：数组比列表在存储相同数量的数据时占用更少的内存。
  - **性能**：对于数值运算，数组的性能通常优于列表。

- 缺点
  - **类型限制**：数组要求所有元素类型一致，而列表可以存储不同类型的元素。
  - **功能限制**：数组提供的功能相对较少，不能像列表那样灵活。

### 5. 使用场景

- 适用于需要高效处理大量数值数据的场景，例如科学计算、数据分析等。

### 总结

Python 的数组通过 `array` 模块实现，主要用于存储相同类型的数值数据。虽然数组在内存使用和性能上优于列表，但它们的功能和灵活性相对较少。选择使用数组还是列表，取决于具体的应用场景和需求。

## 推导式（Comprehension）— Python 的杀手级特性

推导式可以将 for 循环压缩成一行，简洁、高效、可读。

```python
# ========== 列表推导式 ==========
# 普通写法
squares = []
for i in range(10):
    squares.append(i ** 2)
 
# 推导式 — 一行搞定
squares = [i ** 2 for i in range(10)]
# 结果：[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
 
# 带条件过滤
evens = [i for i in range(20) if i % 2 == 0]
# 结果：[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
 
# 嵌套循环（按顺序：外层在前）
pairs = [(x, y) for x in "ab" for y in "12"]
# 结果：[('a', '1'), ('a', '2'), ('b', '1'), ('b', '2')]
 
# ========== 字典推导式 ==========
# 构建一个工具名→工具描述映射
tool_map = {
    "search": "搜索网络",
    "read": "读取文件",
    "write": "写入文件",
}
# 调转键值对
reversed_map = {v: k for k, v in tool_map.items()}
# 结果：{'搜索网络': 'search', '读取文件': 'read', '写入文件': 'write'}
 
# ========== 集合推导式 ==========
unique_lengths = {len(word) for word in ["ai", "agent", "python", "code"]}
# 结果：{2, 4, 5, 6}
 
# ========== 生成器表达式（惰性求值） ==========
# 把方括号换成圆括号 — 不会一次性加载全部数据
big_sum = sum(i * i for i in range(1_000_000))  # 内存友好
```

> **生成器表达式** 是 Python 非常重要的内存优化技巧——处理大文件、大批 API 响应时，用 `(...)` 代替 `[...]` 能省大量内存。

### 生成器表达式

生成器表达式使用圆括号 `()` 而不是方括号 `[]`。

> `sum(i * i for i in range(1_000_000))` 是一个生成器表达式，其中的 `i * i for i in range(1_000_000)` 被圆括号包围。

- **方括号 `[]`**：用于创建列表。

  > 例如，`[i * i for i in range(1_000_000)]` 会创建一个**包含所有计算结果的列表**，这意味着所有数据会被一次性加载到内存中。

- **圆括号 `()`**：用于创建生成器表达式。

  > 例如，`(i * i for i in range(1_000_000))` 会创建一个生成器，**数据会在需要时（惰性求值）逐个生成**，而**不会一次性加载全部数据到内存中**。

具体示例

```python
# 使用列表推导式（方括号）
big_sum_list = sum([i * i for i in range(1_000_000)])  # 会一次性加载所有数据

# 使用生成器表达式（圆括号）
big_sum_gen = sum(i * i for i in range(1_000_000))  # 会逐个生成数据，更加节省内存
```

"把方括号换成圆括号" 是指将 `[]` 替换为 `()`，以便创建一个生成器表达式，从而实现惰性求值，节省内存。

## 类与面向对象

### 基本定义

```python
class Agent:
    """AI Agent 基类"""
 
    # 类属性（所有实例共享）
    category = "AI"
 
    # 初始化方法（构造函数）
    def __init__(self, name, model="gpt-4"):
        self.name = name          # 实例属性
        self.model = model
        self._memory = []         # 单下划线：约定为"内部使用"
 
    # 实例方法
    def introduce(self):
        return f"我是 {self.name}，使用 {self.model} 模型"
 
    # 特殊方法 — 定义"字符串表示"
    def __str__(self):
        return f"Agent({self.name})"
 
    def __repr__(self):
        return f"Agent(name='{self.name}', model='{self.model}')"
 
 
# 继承
class CodingAgent(Agent):
    def __init__(self, name, model="gpt-4", languages=None):
        super().__init__(name, model)     # 调用父类 __init__
        self.languages = languages or ["Python"]
 
    def code_review(self, code: str) -> str:
        return f"[{self.name}] 代码审查完成，发现 3 个优化建议"
 
# 使用
agent = CodingAgent("CodeBuddy", languages=["Python", "TypeScript"])
print(agent.introduce())   # 我是 CodeBuddy，使用 gpt-4 模型
print(agent.category)      # AI （继承自父类）
print(agent.code_review("print('hello')"))
```

### 各语言对照：类

| 概念            | Python               | JavaScript                   | Java                         |
| :-------------- | :------------------- | :--------------------------- | :--------------------------- |
| 构造函数        | `__init__(self)`     | `constructor()`              | 与类同名的方法               |
| `this` / `self` | `self`（必须显式写） | `this`（隐式）               | `this`（隐式）               |
| 继承            | `class B(A)`         | `class B extends A`          | `class B extends A`          |
| 调用父类        | `super().__init__()` | `super()` / `super.method()` | `super()` / `super.method()` |
| 私有约定        | `_name`（约定）      | `#name`（真私有）            | `private`（关键字）          |

> **Python 最大的不同：** `self` 必须**显式出现在每个实例方法的第一个参数位置**。这是 Python 的设计哲学——“显式优于隐式”（Explicit is better than implicit）。JS/Java 开发者刚接触时最容易忘记写 `self`。

## 装饰器（Decorator）

装饰器是 Python 最优雅的特性之一。一句话解释：**在不修改原函数的情况下，给函数添加额外功能。**

### 先理解"函数是一等公民"

```python
# Python 中，函数可以赋值给变量、作为参数传递、作为返回值
def greet(name):
    return f"Hello, {name}"
 
say_hello = greet            # 函数赋值给变量
print(say_hello("World"))    # Hello, World
 
def call_twice(func, arg):   # 函数作为参数
    return func(arg), func(arg)
```

### 手写一个装饰器

```python
import time
 
def timing_decorator(func):
    """测量函数执行时间的装饰器"""
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)   # 执行原函数
        elapsed = time.time() - start
        print(f"[计时] {func.__name__} 执行耗时：{elapsed:.3f} 秒")
        return result
    return wrapper
 
# 使用装饰器 — 等价于 slow_function = timing_decorator(slow_function)
@timing_decorator
def slow_function():
    time.sleep(1.5)
    return "完成"
 
print(slow_function())
# 输出：
# [计时] slow_function 执行耗时：1.501 秒
# 完成
```

### 带参数的装饰器

```python
def retry(max_attempts=3, delay=1):
    """重试装饰器 — 调用失败时自动重试"""
    import time
 
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise  # 最后一次还失败，抛出异常
                    print(f"第 {attempt + 1} 次失败：{e}，{delay}秒后重试...")
                    time.sleep(delay)
        return wrapper
    return decorator
 
# 使用
@retry(max_attempts=3, delay=2)
def call_unstable_api():
    # ... 可能失败的 API 调用 ...
    pass
```

> 要传参的话，外层还得再包一层函数

### 装饰器在 AI Agent 开发中的实际应用

```python
# 场景 1：记录每次工具调用的输入输出
def log_tool_call(func):
    def wrapper(*args, **kwargs):
        print(f"[Tool] 调用 {func.__name__}(args={args}, kwargs={kwargs})")
        result = func(*args, **kwargs)
        print(f"[Tool] {func.__name__} 返回 {result[:50]}...")
        return result
    return wrapper
 
@log_tool_call
def search_web(query: str) -> str:
    return f"搜索结果：关于 '{query}' 的 10 条相关信息..."
 
# 场景 2：限流保护
def rate_limit(max_calls_per_second):
    import time
    interval = 1.0 / max_calls_per_second
    last_called = [0]
 
    def decorator(func):
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < interval:
                time.sleep(interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator
```

> **类比 JS：** 装饰器 ≈ 高阶函数（HOC）。JS 中 `const enhanced = withLogging(Component)` 和 Python 的 `@log_tool_call` 本质是一样的——都是用一个函数"包装"另一个函数。

### **Decorator（装饰器）**

- **定义**: 装饰器是一种**用于修改或增强函数功能的函数**。它可以在不修改原始函数代码的情况下，添加一些额外的行为。
- **功能**: 装饰器通常**接收一个函数作为参数，返回一个新的函数（通常是 `wrapper` 函数）**，这个新函数是对原始函数的增强或修改。
- **使用场景**: 用于日志记录、权限检查、性能计时、缓存等场景。

**示例**:

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        # 在这里可以添加额外的功能
        print("调用前的操作")
        result = func(*args, **kwargs)  # 调用原始函数
        print("调用后的操作")
        return result
    return wrapper
```

### **Wrapper（包装器）函数**

- **定义**: 包装器函数是**装饰器内部定义的一个函数**，它通常**负责调用被装饰的原始函数**，**并在调用之前或之后执行额外的操作**。
- **功能**: `wrapper` 函数可以接收原始函数的参数，并控制其执行。它是实现装饰器逻辑的核心部分。
- **使用场景**: 在装饰器中，包装器函数用于添加功能，比如打印日志、修改参数、处理返回值等。

**示例**:

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("调用前的操作")  # 这是包装器的功能
        result = func(*args, **kwargs)  # 调用原始函数
        print("调用后的操作")  # 这是包装器的功能
        return result
    return wrapper
```

### Decorator和Wrapper的关系和区别

- **装饰器**是一个函数，用于接受另一个函数并返回一个新的函数（通常是 `wrapper`），以增强或修改原始函数的行为。
- **包装器**是装饰器内部定义的函数，负责实现具体的增强逻辑，通常会调用原始函数并在其前后执行额外的操作。

装饰器和包装器的关系

- 装饰器通常包含一个或多个包装器函数，以实现对被装饰函数的增强。**装饰器是一个更高层次的概念，而包装器是实现装饰器功能的具体实现**。

是否需要wrapper函数的场景判断：

- **使用 `wrapper` 函数**: 当需要在调用被装饰的函数之前或之后执行额外的操作时。
- **不使用 `wrapper` 函数**: 当装饰器的目的仅仅是注册、修改或替换函数，而不需要干预函数的执行过程时。

------

### 上下文管理器（with 语句）

上下文管理器让你**安全地管理资源**——确保文件被关闭、锁被释放、连接被归还，即使中间发生了异常。

### 最常用的场景：文件操作

```python
# ❌ 不好的写法 — 容易忘记关闭文件
f = open("data.txt", "r")
content = f.read()
f.close()  # 如果 read() 报错，这行永远不会执行
 
# ✅ Python 标准写法 — with 语句自动关闭
with open("data.txt", "r") as f:
    content = f.read()
# ← 缩进结束后，文件自动关闭（无论是否发生异常）
```

### 自定义上下文管理器

```python
class TimerContext:
    """一个计时上下文管理器"""
    import time
 
    def __enter__(self):
        self.start = time.time()
        return self
 
    def __exit__(self, exc_type, exc_val, exc_tb):
        elapsed = time.time() - self.start
        print(f"执行耗时：{elapsed:.3f} 秒")
        return False  # False = 不抑制异常（让异常正常传播）
 
# 使用
with TimerContext():
    total = sum(range(10_000_000))  # 耗时操作
# 输出：执行耗时：0.xxx 秒
```

### 使用 contextlib 快速创建上下文管理器

```python
from contextlib import contextmanager
 
@contextmanager
def temporary_env_var(key, value):
    """临时设置环境变量，退出时自动恢复"""
    import os
    old_value = os.environ.get(key)
    os.environ[key] = value
    try:
        yield                         # ← 这里是 with 块执行的地方
    finally:
        if old_value is None:
            del os.environ[key]
        else:
            os.environ[key] = old_value
 
# 使用
with temporary_env_var("OPENAI_API_KEY", "sk-test"):
    # 在这里 API key 是临时值
    call_api()
# 退出后自动恢复
```

### AI Agent 开发中的典型场景

```python
# 场景：确保工具调用的资源被正确释放
class ToolSession:
    def __enter__(self):
        print("[Session] 开始工具调用会话")
        self.results = []
        return self
 
    def __exit__(self, *args):
        print(f"[Session] 会话结束，共执行 {len(self.results)} 次工具调用")
        # 清理临时文件、释放资源等
 
with ToolSession() as session:
    session.results.append(search_web("AI Agent"))
    session.results.append(read_file("config.json"))
```

> **对比 JS：** Python 的 `with` ≈ JS 的 `try-finally` 或 Go 的 `defer`。但 `with` 更优雅——它把"获取资源"和"释放资源"封装在了一起。

## 上下文管理器（with 语句）

上下文管理器让你**安全地管理资源**——确保文件被关闭、锁被释放、连接被归还，即使中间发生了异常。

### 最常用的场景：文件操作

```python
# ❌ 不好的写法 — 容易忘记关闭文件
f = open("data.txt", "r")
content = f.read()
f.close()  # 如果 read() 报错，这行永远不会执行
 
# ✅ Python 标准写法 — with 语句自动关闭
with open("data.txt", "r") as f:
    content = f.read()
# ← 缩进结束后，文件自动关闭（无论是否发生异常）
```

### 自定义上下文管理器

```python
class TimerContext:
    """一个计时上下文管理器"""
    import time
 
    def __enter__(self):
        self.start = time.time()
        return self
 
    def __exit__(self, exc_type, exc_val, exc_tb):
        elapsed = time.time() - self.start
        print(f"执行耗时：{elapsed:.3f} 秒")
        return False  # False = 不抑制异常（让异常正常传播）
 
# 使用
with TimerContext():
    total = sum(range(10_000_000))  # 耗时操作
# 输出：执行耗时：0.xxx 秒
```

### 使用 contextlib 快速创建上下文管理器

```python
from contextlib import contextmanager
 
@contextmanager
def temporary_env_var(key, value):
    """临时设置环境变量，退出时自动恢复"""
    import os
    old_value = os.environ.get(key)
    os.environ[key] = value
    try:
        yield                         # ← 这里是 with 块执行的地方
    finally:
        if old_value is None:
            del os.environ[key]
        else:
            os.environ[key] = old_value
 
# 使用
with temporary_env_var("OPENAI_API_KEY", "sk-test"):
    # 在这里 API key 是临时值
    call_api()
# 退出后自动恢复
```

### AI Agent 开发中的典型场景

```python
# 场景：确保工具调用的资源被正确释放
class ToolSession:
    def __enter__(self):
        print("[Session] 开始工具调用会话")
        self.results = []
        return self
 
    def __exit__(self, *args):
        print(f"[Session] 会话结束，共执行 {len(self.results)} 次工具调用")
        # 清理临时文件、释放资源等
 
with ToolSession() as session:
    session.results.append(search_web("AI Agent"))
    session.results.append(read_file("config.json"))
```

### 文件备份案例

```python
import os
import shutil
from contextlib import contextmanager

class FileBackup:
    def __init__(self, file_path):
        self.file_path = file_path
        self.backup_path = file_path + '.bak'

    def __enter__(self):
        # 备份文件
        shutil.copy(self.file_path, self.backup_path)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            print(f"发生异常: {exc_val}")
        print(f"备份文件路径: {self.backup_path}")

# 使用示例
if __name__ == "__main__":
    original_file = 'example.txt'  # 指定要备份的文件

    with FileBackup(original_file) as backup:
        # 在此处可以进行对原始文件的操作
        print(f"正在处理文件: {original_file}")
```

> FileBackup 类
>
> - `__init__` 方法接收一个文件路径并初始化备份路径。
> - `__enter__` 方法在进入上下文时备份指定文件。
> - `__exit__` 方法在退出上下文时打印备份文件的路径。
>
> 通过在 `__enter__` 方法中使用 `return self`，上下文管理器允许用户在 `with` 块内直接使用上下文管理器的实例，这使得在 `with` 块内可以方便地访问上下文管理器的属性和方法。
>
> ```python
> # 使用示例
> with FileBackup('example.txt') as backup:
>     print(backup.backup_path)  # 访问上下文管理器实例的属性
> ```
>
> with块中产生异常后，Python 会跳过 `with` 块中剩余的代码，并立即调用 `__exit__` 方法。在 `__exit__` 方法中，可以访问异常的类型、值和 traceback，这样就可以进行相应的处理。



> **对比 JS：** Python 的 `with` ≈ JS 的 `try-finally` 或 Go 的 `defer`。但 `with` 更优雅——它把"获取资源"和"释放资源"封装在了一起。



## 类型注解（Type Hints）基础

Python 是动态类型语言，但**从 Python 3.5 开始支持可选的类型注解**。写了不会影响运行时行为，但能带来巨大的开发体验提升——尤其是配合大模型使用。

### 为什么类型注解对 AI Agent 开发尤其重要？

1. **大模型代码补全更准：** 有类型注解时，大模型能更好地理解你的意图，生成的代码更准确
2. **FastAPI 强依赖：** FastAPI 用类型注解自动生成 API 文档、做请求校验——你没写类型注解，FastAPI 的一半能力就废了
3. **Pydantic 的基础：** Pydantic（1.5 节会学）完全建立在类型注解之上
4. **团队协作和 IDE 支持：** 类型检查能提前发现参数传递错误

### 基本用法

```python
# 变量注解
name: str = "Agent"
count: int = 42
config: dict[str, any] = {"model": "gpt-4"}
 
# 函数注解
def greet(name: str, times: int = 1) -> str:
    """参数和返回值都有注解"""
    return f"Hello, {name}! " * times
 
def search_web(query: str, max_results: int = 10) -> list[str]:
    """搜索结果返回字符串列表"""
    return [f"结果{i}: 关于 {query}" for i in range(max_results)]
```

### 复合类型

```python
from typing import Optional, Union, List, Dict, Tuple
 
# Optional — 可以是 None
def get_user(id: int) -> Optional[str]:
    """可能返回 None"""
    users = {1: "Alice", 2: "Bob"}
    return users.get(id)
 
# Union — 多种类型之一
def process(data: Union[str, bytes]) -> str:
    """接受 str 或 bytes，返回 str"""
    if isinstance(data, bytes):
        return data.decode("utf-8")
    return data
 
# List / Dict（Python 3.9+ 可直接用小写 list / dict）
tools: list[str] = ["search", "read_file"]
config: dict[str, float] = {"temperature": 0.7, "top_p": 0.9}
 
# Tuple — 固定长度和类型的元组
coord: tuple[float, float] = (116.40, 39.90)
```

### 更实用的例子

```python
from typing import TypedDict, Literal, Callable
 
# TypedDict — 定义字典的"形状"
class ToolResult(TypedDict):
    tool_name: str
    status: Literal["success", "error"]   # 只能是这两个值之一
    output: str
    duration_ms: float
 
# Callable — 标注函数类型
ToolHandler = Callable[[str, dict], ToolResult]  # (参数类型) -> 返回类型
 
def register_tool(name: str, handler: ToolHandler) -> None:
    """注册一个工具处理器"""
    print(f"已注册工具：{name}")
 
# 自引用类型（用字符串延迟求值）
class TreeNode:
    def __init__(self, value: int, left: "TreeNode" = None, right: "TreeNode" = None):
        self.value = value
        self.left = left
        self.right = right
```

### 类型检查工具

```bash
# pip install mypy
# mypy your_file.py  — 静态检查类型是否正确
```

> **学习建议：** 先掌握基本注解（`str`、`int`、`bool`、`list[str]`、`Optional`、`Union`），复杂类型用到时再查文档。在 1.5 节学 Pydantic 时，你会看到类型注解的真正威力。

## Python核心概念简单速查

### 一句话总结 Python 的"性格"

> **Python 是"可执行的伪代码"。** 它的设计哲学是"显式优于隐式"（`self` 必须写）、“简单优于复杂”（没有 `switch`，用 `if/elif` 替代）、“可读性很重要”（缩进是语法的一部分）。

### 各语言"翻译"总表

| 你想做的事 | Python               | JavaScript              | Java                   | Shell            |
| :--------- | :------------------- | :---------------------- | :--------------------- | :--------------- |
| 定义函数   | `def f(x):`          | `function f(x)`         | `T f(T x)`             | `f() { ... }`    |
| 循环 N 次  | `for i in range(N):` | `for(let i=0;i<N;i++)`  | `for(int i=0;i<N;i++)` | `for i in ...`   |
| 检查类型   | `isinstance(x, int)` | `typeof x === "number"` | `x instanceof Integer` | —                |
| 数组操作   | `arr.append(x)`      | `arr.push(x)`           | `list.add(x)`          | `arr+=($x)`      |
| 空值检查   | `if x is None:`      | `if (x === null)`       | `if (x == null)`       | `if [ -z "$x" ]` |
| 字符串拼接 | `f"{a} {b}"`         | ``${a} ${b}``           | `a + " " + b`          | `"$a $b"`        |
| Try-Catch  | `try/except`         | `try/catch`             | `try/catch`            | `trap` / `       |

### 最容易踩的 5 个坑

1. **缩进混用：** Python 用缩进定义代码块，不能用 tab 和空格混用。统一用 4 个空格。
2. **可变默认参数：** `def f(lst=[])` —— 默认值只计算一次，所有调用共享同一个 list。请用 `def f(lst=None)`。
3. **`is` vs `==`：** `is` 比较对象身份（是否同一个对象），`==` 比较值。检查 None 用 `is None`。
4. **`self` 不能忘：** 实例方法第一个参数必须是 `self`，调用时不用传，但定义时必须写。
5. **列表切片是浅拷贝：** `new = old[:]` 创建新列表，但里面的对象还是原来的引用。

#### 可变默认参数的坑！！！

在 Python 中，默认参数的行为可能会导致一些意外的结果，尤其是对于可变对象（如列表、字典等）。当你定义一个函数时，默认参数的值只会在函数定义时计算一次。如果使用可变对象作为默认参数，那么所有调用该函数时，如果没有提供该参数，将共享同一个对象。这可能会导致意外的修改。

```python
def f(lst=[]):
    lst.append(1)
    return lst

print(f())  # 输出: [1]  (第一次调用，使用的是新的空列表)
print(f())  # 输出: [1, 1]  (第二次调用，仍然是同一个列表，添加了 1)
print(f())  # 输出: [1, 1, 1]  (第三次调用，继续在同一个列表上添加 1)
```

在这个例子中，虽然 `lst` 是局部变量，但它引用的是同一个列表对象，导致了意外的共享状态。

`lst` 是函数的局部变量，但这里的关键在于 Python 如何处理默认参数的赋值。

**默认参数的计算**

在 Python 中，当你定义一个函数时，**默认参数的值是在函数定义时计算的**，**而不是在函数调用时**。具体来说：

1. **函数定义时**:
   - 当你定义函数 `def f(lst=[])` 时，`[]`（空列表）会被创建并赋值给 `lst`。这意味着在函数定义的那一刻，Python 会生成一个新的列表对象，并将其作为默认值存储。
2. **函数调用时**:
   - 当你调用 `f()` 而不提供参数时，Python **会使用这个在函数定义时创建的列表对象**。
   - 因此，所有后续的调用（如果不提供参数）都会使用这个同一个列表对象，而不是创建新的列表。

**局部变量与共享引用**

- 局部变量:

  - 在函数内部，`lst` 确实是一个局部变量，它的作用域仅限于该函数内部。

- 共享引用:

  - 但 `lst` 引用的是**在函数定义时创建的列表对象**。由于这个列表是可变的，任何对 `lst` 的修改都会影响到这个列表对象本身。
  - 所以，即使 `lst` 是局部的，所有对这个列表的修改都会反映在所有后续的函数调用中，因为它们都**引用同一个列表对象**。

  > 有点类似于指针一样，lst这个变量的指针地址是一样的，后面f()函数未传参数时对lst的操作都是在同一个指针地址指向的lst变量上操作。

**总结**

所以，关键在于默认参数的创建时机。在函数定义时创建的默认参数（尤其是可变对象）会在后续调用中被共享，而不是在每次调用时重新创建。为了避免这种问题，**建议使用 `None` 作为默认参数，然后在函数内部创建新的可变对象**。这样可以确保每次调用都得到一个全新的对象，避免共享状态。

```python
def f(lst=None):
    if lst is None:
        lst = []  # 每次调用时创建一个新的列表
    lst.append(1)
    return lst

print(f())  # 输出: [1]
print(f())  # 输出: [1]
print(f())  # 输出: [1]
```

## Python的一些关键字含义和作用

### assert

在 Python 中，`assert` 是一个用于调试和测试的关键字。它用于验证某个条件是否为真，如果条件为假，程序会抛出一个 `AssertionError` 异常。`assert` 通常用于测试代码的正确性，确保程序在运行时符合预期的行为。

```python
assert condition, message
```

- **`condition`**：要检查的条件表达式。如果该条件为 `False`，则会触发 `AssertionError`。
- **`message`**（可选）：当条件为 `False` 时，抛出的异常消息，可以帮助调试。

主要用途：

- **调试**：在开发过程中，`assert` 可以帮助开发者发现潜在的错误或不一致。例如，确保某个变量的值在预期范围内。
- **单元测试**：在测试代码时，`assert` 用于验证函数或方法的输出是否符合预期。例如，检查返回值、状态码等。

示例：

```python
# 示例 1: 基本使用
x = 5
assert x > 0  # 如果 x <= 0，将抛出 AssertionError

# 示例 2: 带消息的断言
y = -1
assert y > 0, "y should be greater than 0"  
# 抛出 AssertionError，消息为 "y should be greater than 0"
```

在测试中的使用：

```python
def test_health_check():
    """测试健康检查接口"""
    response = client.get("/health")
    
    assert response.status_code == 200  # 验证响应状态码是否为 200
    assert response.json() == {"status": "healthy"}  # 验证响应内容是否为 {"status": "healthy"}
```

- **`assert response.status_code == 200`**：检查 HTTP 响应的状态码是否为 200（表示请求成功）。
- **`assert response.json() == {"status": "healthy"}`**：检查返回的 JSON 数据是否与预期的字典相同。

如果任一 `assert` 条件失败，测试将抛出 `AssertionError`，并且测试框架（如 pytest）将报告该测试失败。这有助于确保 API 的健康检查功能按预期工作。

`assert` 是一个强大的工具，用于**在代码中执行条件检查**，帮助开发者捕获错误和验证程序行为。在测试环境中，它提供了**一种简单的方式来确保代码的正确性和可靠性**。



## 异步编程入门

### 同步、异步、并发和并行

|                 | 同步                 | 异步                                 | 并发                                                   | 并行                       |
| :-------------- | :------------------- | :----------------------------------- | :----------------------------------------------------- | :------------------------- |
| **简述**        | 一件事做完再做下一件 | 发起一个操作后不干等，切换到其他任务 | 一段时间内，多个任务交替执行（但同一时刻只有一个在跑） | 同一时刻，多个任务同时在跑 |
| **核心问题**    | 怎么顺序执行？       | 怎么不浪费等待时间？                 | 怎么管理多个任务？                                     | 怎么同时执行多个任务？     |
| **关系**        | 执行方式             | 执行方式                             | 程序设计                                               | 硬件执行                   |
| **Python 实现** | 普通 `def`           | `async def` + `await`                | `asyncio` / 线程                                       | `multiprocessing`          |
| **关键不同**    | 阻塞等待             | 非阻塞等待                           | 交替执行                                               | 同时执行                   |

> **对我们最重要的结论：** AI Agent 开发中，90% 的等待时间都在等网络（调 API）、等磁盘（读文件）。Python 的 `asyncio` 就是为这类 **IO 密集型** 场景设计的。如果是 CPU 密集型（图像处理、科学计算），你才需要多进程并行——那不是预备课的内容。

### `async` / `await` 核心语法

```python
# async def：定义一个"协程函数"（coroutine function）
#            调用它不会立即执行，而是返回一个"协程对象"（coroutine object）
async def my_func():
    return "hello"
 
print(my_func())   # <coroutine object my_func at 0x...>  ← 不是 "hello"！
# 协程对象需要用 await 或 asyncio.run() 来驱动执行
 
 
# await：暂停当前协程，等待另一个协程完成
#        在等待期间，事件循环可以去执行其他协程
async def greet():
    print("开始打招呼...")
    await asyncio.sleep(1)    # ← 暂停 1 秒，让出控制权
    print("你好！")
```

### 协程的三种运行方式

```python
import asyncio
 
async def say_hello(name: str) -> str:
    await asyncio.sleep(0.5)
    return f"Hello, {name}!"
 
 
# ===== 方式 1：asyncio.run() — 程序的"入口点" =====
# 从同步世界进入异步世界。整个程序只需要一个 asyncio.run()
result = asyncio.run(say_hello("World"))
print(result)                         # Hello, World!
 
 
# ===== 方式 2：await — 在已有的协程中调用另一个协程 =====
async def main():
    result = await say_hello("Agent")  # ← 等待 say_hello 完成，拿到返回值
    print(result)                      # Hello, Agent!
 
asyncio.run(main())
 
 
# ===== 方式 3：asyncio.gather() — 并发运行多个协程 =====
async def main():
    results = await asyncio.gather(
        say_hello("Alice"),
        say_hello("Bob"),
        say_hello("Charlie"),
    )
    print(results)  # ['Hello, Alice!', 'Hello, Bob!', 'Hello, Charlie!']
    # 三个任务并发执行，总耗时 ≈ 0.5 秒（不是 1.5 秒）
 
asyncio.run(main())
```

#### 总结对比

| 特性         | `asyncio.run()`        | `await`                    | `asyncio.gather()`           |
| ------------ | ---------------------- | -------------------------- | ---------------------------- |
| **功能**     | 启动并运行单个协程     | 在协程中等待另一个协程完成 | 并发执行多个协程并收集结果   |
| **调用方式** | 直接调用并阻塞         | 在协程内部使用             | 在协程内部使用               |
| **适用场景** | 从同步代码进入异步代码 | 在协程中调用其他协程       | 同时执行多个 I/O 密集型任务  |
| **返回值**   | 返回单个协程的结果     | 返回被调用协程的结果       | 返回所有被调用协程的结果列表 |

简而言之

- **`asyncio.run()`** 适合作为程序的入口点，启动异步代码。
- **`await`** 用于在已有的协程中等待其他协程的结果。
- **`asyncio.gather()`** 适合并发执行多个协程，提高执行效率。

### 用 `asyncio.create_task()` 实现"发出但不等"

```python
import asyncio
 
async def download_file(filename: str, duration: float):
    print(f"[开始] 下载 {filename}...")
    await asyncio.sleep(duration)     # 模拟下载耗时
    print(f"[完成] 下载 {filename}")
    return f"{filename} 的内容"
 
async def main():
    start = time.time()
 
    # create_task()：立即创建并调度任务，不等待
    # 注意：任务在事件循环的"下一次机会"就会开始执行
    task_a = asyncio.create_task(download_file("A.pdf", 3))
    task_b = asyncio.create_task(download_file("B.pdf", 2))
    task_c = asyncio.create_task(download_file("C.pdf", 1))
 
    print("三个下载任务已启动，我可以干别的事了...")
    await asyncio.sleep(0.5)          # 干点别的事
    print("现在等待所有下载完成...")
 
    # 再逐个等待结果
    content_a = await task_a           # task_a 可能已经完成了，这里不额外等待
    content_b = await task_b
    content_c = await task_c
 
    print(f"全部完成！耗时：{time.time() - start:.1f} 秒")
    # 输出顺序：C(1s) → B(2s) → A(3s)，总耗时 3 秒
 
asyncio.run(main())
```

### `gather()` vs `create_task()` vs `as_completed()`

```python
import asyncio
 
async def fetch(url: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"来自 {url} 的响应"
 
# ===== gather()：等所有完成，返回列表（保持顺序） =====
async def demo_gather():
    results = await asyncio.gather(
        fetch("url_a", 3),
        fetch("url_b", 1),
        fetch("url_c", 2),
    )
    print(results)   # ['来自 url_a 的响应', '来自 url_b 的响应', '来自 url_c 的响应']
    # 总耗时 3 秒，返回顺序 = 传入顺序
 
 
# ===== as_completed()：哪个先完成就先处理哪个 =====
async def demo_as_completed():
    tasks = [
        fetch("url_a", 3),
        fetch("url_b", 1),
        fetch("url_c", 2),
    ]
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"收到结果：{result}")
        # 输出顺序：
        # 收到结果：来自 url_b 的响应    ← 1 秒后
        # 收到结果：来自 url_c 的响应    ← 2 秒后
        # 收到结果：来自 url_a 的响应    ← 3 秒后
        # 总耗时 3 秒，但快的先显示！
 
asyncio.run(demo_as_completed())
```

> **选择指南：**
>
> - 你必须拿到所有结果才能继续 → `gather()`
> - 你希望"有一个算一个"，先完成的先处理 → `as_completed()`
> - 你想发出任务后干点别的，稍后再收结果 → `create_task()`

### 不能 await 普通函数

```python
# ❌ 错误示范
def sync_func():
    return "hello"
 
async def main():
    result = await sync_func()   # TypeError: object str can't be used in 'await' expression
 
# sync_func() 返回的是普通字符串 "hello"，不是协程对象，不能 await
 
# ✅ 正确的是：在协程中调用普通函数不需要 await
async def main():
    result = sync_func()         # 直接调用即可
    print(result)
```

### 你能 await 什么？（Awaitable 的三类对象）

```python
# 1. 协程对象（async def 函数的返回值）
coro = async_func()     # 协程对象
await coro
 
# 2. Task（create_task 返回的对象）
task = asyncio.create_task(async_func())  # Task 对象
await task
 
# 3. Future（底层对象，日常开发很少直接使用）
future = asyncio.Future()
await future
```

### 事件循环（Event Loop）

创建事件循环是异步编程中的一个核心概念，特别是在使用 Python 的 `asyncio` 库时。事件循环负责**管理和调度异步任务的执行**。

事件循环（Event Loop）是 `asyncio` 的心脏。它的工作原理可以用一个简短的口诀概括：

- **"挑一个协程执行，直到它遇到 await → 切换下一个协程 → 循环往复。"**

```
                    ┌──────────────────────────┐
                    │      事件循环 (Event Loop)    │
                    │                            │
  协程 A: ──await──▶ │  ① 协程 A 遇到 await       │
                    │  ② 把 A 挂起，切换去执行 B   │
  协程 B: ──run────▶ │  ③ B 遇到 await           │
                    │  ④ 把 B 挂起，看 A 好了没？  │
                    │  ⑤ A 还没好，看 C           │
  协程 C: ──run────▶ │  ... 循环 ...              │
                    │  ⑥ A 好了！恢复 A 继续执行   │
  协程 A: ◀──resume── │                            │
                    └──────────────────────────┘
```

#### 什么是事件循环？

事件循环是**一个运行在单线程中的循环**，**负责执行异步任务并处理 I/O 操作**。它的主要职责包括：

- **调度任务**: 管理和调度待执行的协程（异步函数）和回调函数。
- **处理 I/O 事件**: 监控 I/O 操作（如网络请求、文件读写），**当某个操作完成时，触发相应的回调或协程**。
- **控制程序的执行顺序**: 确保在合适的时机执行任务，以避免阻塞主线程。

#### 事件循环的工作原理

1. **任务注册**: 当你调用一个异步函数（例如使用 `await` 或 `asyncio.create_task()`），事件循环会将这个任务注册到内部队列中。
2. **轮询执行**: 事件循环不断检查任务队列，执行已注册的任务。当遇到 `await` 或 I/O 操作时，当前任务会被挂起，事件循环会去执行其他任务。
3. **事件处理**: 一旦 I/O 操作完成，事件循环会将相应的回调或协程放回任务队列中，等待下一次轮询执行。

>### "单线程 + 事件循环"为什么快？
>
>“只有一个线程，怎么能同时处理 1000 个请求？”
>
>答案：**因为它不等。** 传统多线程模式下，每个线程在等网络响应时也占着 CPU 资源。单线程 + 事件循环的模式下，等网络响应时 CPU 就去干别的活了。网络很慢（毫秒级），CPU 很快（纳秒级）——在等一个网络请求回来的时间里，CPU 可以切换几千次。

#### 如何创建事件循环？

在 Python 中，使用 `asyncio` 库可以创建和管理事件循环。以下是创建事件循环的基本步骤：

1. **使用 `asyncio.run()`**（推荐方式）:

   ```python
   import asyncio
   
   async def main():
       print("Hello, World!")
   
   asyncio.run(main())  # 创建事件循环并运行 main 协程
   ```

2. **手动创建事件循环**（较低级的方式）:

   ```python
   import asyncio
   
   async def say_hello():
       print("Hello, World!")
   
   # 手动创建事件循环
   loop = asyncio.new_event_loop()  # 创建新事件循环
   asyncio.set_event_loop(loop)      # 设置为当前事件循环
   
   try:
       loop.run_until_complete(say_hello())  # 运行协程
   finally:
       loop.close()  # 关闭事件循环
   ```

#### 总结

- **事件循环** 是异步编程的核心，负责调度和管理异步任务的执行。
- 在 `asyncio` 中，可以使用 `asyncio.run()` 轻松创建和管理事件循环，推荐用于大多数简单场景。
- 对于更复杂的情况，可以手动创建事件循环，但需要注意在使用后关闭它以释放资源。

事件循环使得 Python 能够高效地处理并发任务，特别是在 I/O 密集型应用中，通过非阻塞的方式实现高效的并发执行。

### 常用 asyncio 工具函数速查

```python
import asyncio
 
# asyncio.sleep(seconds)  — 异步等待（不阻塞事件循环）
await asyncio.sleep(1.5)
 
# asyncio.gather(*coros)  — 并发运行多个协程，等全部完成
results = await asyncio.gather(coro1(), coro2())
 
# asyncio.create_task(coro) — 调度协程，立即返回 Task
task = asyncio.create_task(some_coro())
 
# asyncio.wait_for(coro, timeout) — 给协程加超时
try:
    result = await asyncio.wait_for(slow_api_call(), timeout=5.0)
except asyncio.TimeoutError:
    print("API 调用超时！")
 
# asyncio.as_completed(coros) — 遍历协程，哪个先完成先返回哪个
for coro in asyncio.as_completed([task1(), task2()]):
    result = await coro
 
# asyncio.run(main()) — 从同步代码启动异步程序（Python 3.7+）
asyncio.run(main())
```

### 常见坑点

#### 坑 1：在同步函数里调用 async 函数

```python
async def async_func():
    return "hello"
 
def sync_func():
    result = async_func()       # ❌ TypeError: 不能直接调用协程
    # 正确：用 asyncio.run() 作为入口
    result = asyncio.run(async_func())   # ✅ 但 asyncio.run() 不能在已有事件循环中使用
```

规则：`asyncio.run()` 是程序异步部分的入口。一个程序通常只有一个 `asyncio.run()`（在 `main()` 或 `if __name__ == "__main__"` 中）。

#### 坑 2：忘了 await

```python
async def main():
    task = asyncio.create_task(download())   # 创建了任务
 
    # ❌ 没有 await task！程序直接结束，任务可能还没跑完
    print("main 结束")
 
    # ✅ 正确：确保任务完成
    await task
```

#### 坑 3：在协程中用同步阻塞操作

```python
async def bad_async():
    print("开始")
    time.sleep(5)                # ❌ 同步阻塞！整个事件循环被冻结 5 秒
    print("结束")
 
async def good_async():
    print("开始")
    await asyncio.sleep(5)       # ✅ 异步等待，不阻塞其他协程
    print("结束")
```

> **黄金法则：协程里，所有"等待"操作都用 `await`。** `time.sleep()` → `await asyncio.sleep()`，`requests.get()` → `await httpx.AsyncClient.get()`。

#### 坑 4：在 Jupyter Notebook 中运行 asyncio.run()

Jupyter 自带一个运行中的事件循环，所以你不能再调用 `asyncio.run()`。用 `await` 直接在 cell 里写：

```python
# Jupyter Notebook 中 ❌ 不要用
asyncio.run(main())
 
# ✅ 直接用 await
await main()
```

## Pydantic

### 数据建模

| 概念        | Pydantic (Python)      | TypeScript               | Java (Spring)          | Go               |
| :---------- | :--------------------- | :----------------------- | :--------------------- | :--------------- |
| 模型定义    | `class X(BaseModel)`   | `interface X` / `type X` | `class X` + `@Entity`  | `type X struct`  |
| 必填字段    | `name: str`            | `name: string`           | `@NotNull String name` | `Name string`    |
| 可选字段    | `name: str = "x"`      | `name?: string`          | `@Nullable`            | `*string`        |
| 校验        | `Field(ge=0, le=100)`  | zod / class-validator    | `@Min` / `@Max`        | validator 库     |
| 序列化      | `.model_dump()`        | `JSON.stringify()`       | Jackson ObjectMapper   | `json.Marshal()` |
| JSON Schema | `.model_json_schema()` | zod.toJSONSchema()       | —                      | —                |

## FastAPI

FastAPI 是 Python 目前最快的 Web 框架之一。它的"快"不是指运行速度（虽然也很快），而是**开发速度**。

**和 Flask 的对比：**

|          | FastAPI                  | Flask                   |
| :------- | :----------------------- | :---------------------- |
| 请求校验 | 自动（基于类型注解）     | 手动写                  |
| API 文档 | 自动生成 Swagger + ReDoc | 需要 Flask-RESTX 等插件 |
| 异步支持 | 原生 `async/await`       | 需要额外配置            |
| 序列化   | 内建（基于 Pydantic）    | 需要 marshmallow 等     |
| 学习曲线 | 略陡（要懂类型注解）     | 平缓                    |

### 请求体

```python
from pydantic import BaseModel, Field

# 定义请求模型（这就是 1.5 节学的内容）
class CreateAgentRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=50)
    model: str = Field(default="gpt-4o")
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    tools: list[str] = Field(default_factory=list)


class AgentResponse(BaseModel):
    id: int
    name: str
    model: str
    temperature: float
    tools: list[str]


@app.post("/agents", response_model=AgentResponse)   # ← response_model 控制输出
async def create_agent(agent: CreateAgentRequest):   # ← Pydantic 自动校验请求体
    # 假装保存到数据库...
    saved = AgentResponse(
        id=1,
        name=agent.name,
        model=agent.model,
        temperature=agent.temperature,
        tools=agent.tools,
    )
    return saved    # FastAPI 自动序列化为 JSON

```



### 请求体 + 路径参数 + 查询参数混合

```python
@app.put("/agents/{agent_id}", response_model=AgentResponse)
async def update_agent(
    agent_id: int,                              # 路径参数
    agent: CreateAgentRequest,                  # 请求体
    notify: bool = Query(default=False),        # 查询参数
):
    """PUT /agents/42?notify=true  +  JSON 请求体"""
    # FastAPI 自动区分：URL 模板里的 → 路径参数
    #                 函数签名里的 Pydantic 模型 → 请求体
    #                 基础类型带默认值 → 查询参数
    print(f"更新 Agent {agent_id}，通知 ={notify}")
    return AgentResponse(id=agent_id, **agent.model_dump())
```

1. **客户端发送请求**：
   - 客户端（例如前端应用或其他服务）会向 `/agents/{agent_id}` 发送一个 PUT 请求，并在请求体中包含一个 JSON 对象，该对象的结构应符合 `CreateAgentRequest` 模型的定义。
2. **FastAPI 自动解析**：
   - FastAPI 会根据请求的 Content-Type（通常为 `application/json`）自动解析请求体中的 JSON 数据，并将其映射到 `CreateAgentRequest` 模型的字段上。
3. **Pydantic 验证**：
   - 在解析过程中，Pydantic 会验证请求体数据的有效性，确保它符合 `CreateAgentRequest` 模型中定义的字段类型和约束。

#### Put和Post的区别

PUT 和 POST 请求在 HTTP 协议中有不同的语义和用途。以下是它们的主要区别以及为什么在某些情况下选择使用 PUT 而不是 POST：

##### 主要区别

1. **目的**：
   - **POST**：**用于创建新的资源**。它通常用于提交数据以创建一个新的实体，服务器会根据请求体中的数据生成一个新的资源。
   - **PUT**：**用于更新现有资源或创建一个指定 URI 的资源**。它通常用于提供完整的资源表示，**服务器将用该表示替换现有资源**。
2. **幂等性**：
   - **POST**：**不幂等**。多次执行相同的 POST 请求可能会创建多个资源，导致状态变化。
   - **PUT**：**幂等**。多次执行相同的 PUT 请求会产生相同的结果，即资源的状态不会因为多次请求而改变。
3. **URI 的使用**：
   - **POST**：请求 URI 通常指向一个集合（例如 `/agents`），服务器会在该集合中创建新资源。返回的 URI 通常由服务器生成。
   - **PUT**：请求 URI 通常指向一个特定的资源（例如 `/agents/42`），客户端知道要更新哪个资源。

##### 为什么选择 PUT 而不是 POST

- **更新现有资源**：在你的例子中，`update_agent` 函数是用于更新一个已有的代理（agent）。使用 PUT 可以明确表示这是一个更新操作，而不是创建新资源。
- **明确性**：使用 PUT 可以清晰地指明要更新的资源的 URI，避免了可能由于使用 POST 而导致的混淆。
- **幂等性需求**：如果你的应用程序需要确保同样的请求不会导致不同的结果（例如，更新操作），使用 PUT 是更合适的选择。

##### 总结

- 当你需要创建新资源时，使用 POST。
- 当你需要更新现有资源或替换资源时，使用 PUT。

### 四类参数来源速查

| 来源     | 定义方式                       | 示例                         |
| :------- | :----------------------------- | :--------------------------- |
| 路径参数 | URL 模板 `{name}` + 函数签名   | `@app.get("/{id}")`          |
| 查询参数 | 函数签名中非路径参数的基础类型 | `page: int = 1`              |
| 请求体   | 函数签名中的 Pydantic 模型参数 | `agent: CreateAgentRequest`  |
| Header   | `Header()`                     | `user_agent: str = Header()` |
| Cookie   | `Cookie()`                     | `session_id: str = Cookie()` |

### 路由组织

当 API 多了，把所有路由堆在 `main.py` 里是灾难。FastAPI 用 `APIRouter` 拆分。

#### 拆分成模块

```python
# ===== routers/agents.py =====
from fastapi import APIRouter
 
router = APIRouter(prefix="/agents", tags=["Agents"])
 
# 虚构的"数据库"
fake_db = {}
 
@router.get("/")
async def list_agents():
    return list(fake_db.values())
 
@router.get("/{agent_id}")
async def get_agent(agent_id: int):
    if agent_id not in fake_db:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="Agent 不存在")
    return fake_db[agent_id]
 
@router.post("/")
async def create_agent(agent: CreateAgentRequest):
    agent_id = len(fake_db) + 1
    fake_db[agent_id] = AgentResponse(id=agent_id, **agent.model_dump())
    return fake_db[agent_id]
 
@router.delete("/{agent_id}")
async def delete_agent(agent_id: int):
    if agent_id not in fake_db:
        raise HTTPException(status_code=404, detail="Agent 不存在")
    del fake_db[agent_id]
    return {"ok": True}
 
 
# ===== routers/tasks.py =====
router = APIRouter(prefix="/tasks", tags=["Tasks"])
 
@router.get("/")
async def list_tasks():
    return {"tasks": []}
 
 
# ===== main.py =====
from fastapi import FastAPI
from routers import agents, tasks
 
app = FastAPI(title="AI Agent API")
 
app.include_router(agents.router)
app.include_router(tasks.router)
```

现在 `/docs` 里会自动按 tags 分组展示，结构清晰。

### 依赖注入（Depends）

依赖注入是 FastAPI 最强大的设计模式。一句话解释：**把"获取共享数据"的逻辑从路由函数中抽出来，变成可复用的"依赖函数"。**

#### 从重复代码到 Depends

```python
# ❌ 没有 Depends —— 每个路由里都要写一遍"从 Header 获取 API Key"
@app.get("/agents")
async def list_agents(x_api_key: str = Header(...)):
    if x_api_key not in valid_keys:
        raise HTTPException(status_code=401)
    # ... 业务逻辑 ...
 
@app.get("/tasks")
async def list_tasks(x_api_key: str = Header(...)):
    if x_api_key not in valid_keys:
        raise HTTPException(status_code=401)
    # ... 业务逻辑 ...
 
# ✅ 用 Depends —— 认证逻辑写一次，到处复用
from fastapi import Depends
 
async def verify_api_key(x_api_key: str = Header(...)):
    """验证 API Key，验证通过返回它"""
    valid_keys = {"sk-test", "sk-prod"}
    if x_api_key not in valid_keys:
        raise HTTPException(status_code=401, detail="无效的 API Key")
    return x_api_key
 
@app.get("/agents")
async def list_agents(api_key: str = Depends(verify_api_key)):
    return {"key": api_key[:7] + "..."}
 
@app.get("/tasks")
async def list_tasks(api_key: str = Depends(verify_api_key)):
    return {"key": api_key[:7] + "..."}
```

Depends的3种常见用途

- 认证/授权
- 数据库连接
- 配置/设置

> 核心作用是只需要做一次，后续不需要每次每个使用到的地方都做。

```python	
# 1. 认证 / 授权
async def get_current_user(authorization: str = Header(...)):
    """从 JWT Token 中解析用户"""
    token = authorization.replace("Bearer ", "")
    # 解析 token，返回用户对象...
    return {"user_id": 1, "name": "Alice"}

@app.get("/me")
async def me(user: dict = Depends(get_current_user)):
    return {"user": user}


# 2. 数据库连接
async def get_db():
    """每个请求创建一个数据库连接，请求结束时自动关闭"""
    db = await create_connection()
    try:
        yield db                 # ← yield 是关键
    finally:
        await db.close()         # ← 请求结束自动执行

@app.get("/agents")
async def list_agents(db = Depends(get_db)):
    agents = await db.fetch_all("SELECT * FROM agents")
    return agents


# 3. 配置 / 设置
from functools import lru_cache

@lru_cache()     # 只读一次配置文件
def get_settings():
    return {"model": "gpt-4o", "max_tokens": 4096}

@app.get("/config")
async def config(settings: dict = Depends(get_settings)):
    return settings
```

