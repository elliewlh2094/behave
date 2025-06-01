# `behave`: Python 的行為樹套件

## 使用方式

使用時，需要將 `behave` 目錄放置在主程式相同的目錄內。

### 匯入 behave 的所有函式定義

````
from behave import *
````

### 定義條件節點（Condition node）

使用 `@condition` 宣告條件節點。條件節點會回傳布林值（`True`, `False`），分別對應執行狀態 `SUCCESS`, `FAILURE`。

````
@condition
def is_greater_than_10(x):
    return x > 10

@condition
def is_between_0_and_10(x):
    return 0 < x < 10

````

### 定義動作節點（Action node）

使用 `@action` 宣告動作節點。動作節點會回傳執行狀態（`SUCCESS`, `FAILURE`, `RUNNING`），如果不設定回傳值（相當於回傳 `None`）則視同回傳 `SUCCESS`，即動作節點預設為執行成功。

````
@action
def wow_large_number(x):
    print "WOW, %d is a large number!" % x

@action
def doomed(x):
    print "%d is doomed" % x
    return FAILURE
````

### 定義動作節點，生成器（Generator）版本

And you can define an action with a generator function:

* `yield FAILURE` fails the actions. 
* `yield` or `yield None` puts the action into `RUNNING` state.
* If the generator returns(stop iteration), action state will be set to `SUCCESS`

```
@action
def count_from_1(x):
    for i in range(1, x):
        print "count", i
        yield
    print "count", x

````

### 序列（Sequence）節點/運算子

在 `behave` 中，序列節點以運算子 `>>` 的形式實現。依序執行子節點；遇 `FAILURE` 立刻回傳 `FAILURE`；當全部子節點成功才回傳 `SUCCESS`。 

````
seq = is_greater_than_10 >> wow_large_number
````

### 選擇（Selector）節點/運算子

在 `behave` 中，序列節點以運算子 `|` 的形式實現。依序嘗試子節點；任一子節點回傳 `SUCCESS` 即回傳 `SUCCESS`；全部失敗才回傳 `FAILURE`。

````
sel = is_greater_than_10 | is_between_0_and_10
````

### 裝飾器節點（Decorator）

可用的裝飾器包含 `forever()`, `repeat(n)`, `succeeder()`, `failer()`。將修飾器包覆在節點外層；可用 * 將多個 decorator 串連。

````
decorated_1 = forever(count_from_1)
decorated_2 = succeeder(doomed)
decorated_3 = repeat(10)(count_from_1)
````

For readability reason, you can also use chaining style:

````
composite_decorator = repeat(3) * repeat(2)   # It's identical to repeat(6)

decorated_1 = forever * count_from_1
decorated_2 = succeeder * doomed
decorated_3 = repeat(10) * count_from_1
decorated_4 = failer * repeat(10) * count_from_1
````

### 組成行為樹

在行為樹中，每一個節點都視為一個子樹，一個子樹用 `()` 包起來。可以透過運算子 `>>`, `|` 組合成多層的樹。

````
tree = (
    is_greater_than_10 >> wow_large_number
    | is_between_0_and_10 >> count_from_1
    | failer * repeat(3) * doomed
)
````

### 行為樹執行器/黑板（Blackboard）

用 `tree.blackboard()` 方法建立行為樹黑板。並且使用 `tick()` 方法定期呼叫，tick 是行為樹執行一次的意思。最後透過 `assert` 來進行條件判斷，在做完一次 tick 後行為樹執行狀態應該要是 `SUCCESS` 或是 `FAILURE`，而不是 `RUNNING`。
````
bb = tree.blackboard(5) # Creates an run instance

# Now let the tree do its job, till job is done
state = bb.tick()
print "state = %s\n" % state
while state == RUNNING:
    state = bb.tick()
    print "state = %s\n" % state
assert state == SUCCESS or state == FAILURE
````

Output:

````
count 1
state = Running

count 2
state = Running

count 3
state = Running

count 4
state = Running

count 5
state = Success
````

### 除錯器（Debugger）

To debug the tree, you need to:

* Define a debugger function
* Create blackboard by calling `tree.debug(debugger, arg1, arg2...)` instead of `tree.blackboard(arg1, arg2...)`.

````
def my_debugger(node, state):
    print "[%s] -> %s" % (node.name, state)

bb = tree.debug(my_debugger, 5) # Creates an blackboard with debugger enabled

# Now let the tree do its job, till job is done
state = bb.tick()
print "state = %s\n" % state
while state == RUNNING:
    state = bb.tick()
    print "state = %s\n" % state
assert state == SUCCESS or state == FAILURE
````

Output:

````
[ is_greater_than_10 ] -> Failure
[ BeSeqence ] -> Failure
[ is_between_0_and_10 ] -> Success
count 1
[ count_from_1 ] -> Running
[ BeSeqence ] -> Running
[ BeSelect ] -> Running
state = Running

count 2
[ count_from_1 ] -> Running
[ BeSeqence ] -> Running
[ BeSelect ] -> Running
state = Running

count 3
[ count_from_1 ] -> Running
[ BeSeqence ] -> Running
[ BeSelect ] -> Running
state = Running

count 4
[ count_from_1 ] -> Running
[ BeSeqence ] -> Running
[ BeSelect ] -> Running
state = Running

count 5
[ count_from_1 ] -> Success
[ BeSeqence ] -> Success
[ BeSelect ] -> Success
state = Success

````

Too messy? Let put some comments into the tree:

````
tree = (
    (is_greater_than_10 >> wow_large_number) // "if x > 10, wow"
    | (is_between_0_and_10 >> count_from_1) // "if 0 < x < 10, count from 1"
    | failer * repeat(3) * doomed // "if x <= 0, doomed X 3, and then fail"
)
````

And make a little change to `my_debugger`:

````
def my_debugger(node, state):
    if node.desc:
        print "[%s] -> %s" % (node.desc, state)
````

Try it again:

````
[ if x > 10, wow ] -> Failure
count 1
[ if 0 < x < 10, count from 1 ] -> Running
state = Running

count 2
[ if 0 < x < 10, count from 1 ] -> Running
state = Running

count 3
[ if 0 < x < 10, count from 1 ] -> Running
state = Running

count 4
[ if 0 < x < 10, count from 1 ] -> Running
state = Running

count 5
[ if 0 < x < 10, count from 1 ] -> Success
state = Success
````

## 整合 ROS2 的注意事項

1. 節點宣告：將 `Condition` / `Action` 寫成類別方法，可直接存取成員變數（感測器狀態、publisher）。

2. 執行頻率：用 `rclpy.create_timer()` 依需求設定 tick 週期。

3. 非同步動作：若 `Action` 需等待回覆 (e.g. arm -> 狀態回報)，可用 Generator Action `yield RUNNING` 直到條件達成。

4. 行為切換：利用 `Selector` 的優先權，將高優先度安全檢查（如：電量、GPS 失效...）放在樹最前端。

## 模組解析

- `core.py`:  
    定義三大回傳狀態 `SUCCESS` / `FAILURE` / `RUNNING`、所有節點（`Action` / `Condition` / `Sequence` / `Selector` / `Decorator`...）核心邏輯，以及 `Blackboard` 執行器。

- `helper.py`  
    提供 `@action` / `@condition` 等 Python decorator 以簡化節點宣告，並含若干判斷函式（`is_action`...）。

- `decorator.py`  
    常用修飾節點 (decorator) 之實作，如 `forever` / `repeat` / `succeeder` / `failer`。
