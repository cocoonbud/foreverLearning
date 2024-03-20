## 在 iOS 项目中查找未使用的宏定义

一个项目时间久了，自然会有很多无用的杂七杂八的东西。有时间了需要去优化。遂写此小工具。如题所示，此脚本会搜索指定目录中的所有 `.m`、`.mm` 和 `.h` 文件，忽略注释掉的代码。脚本在搜索过程中显示加载小动画，并将搜索结果输出到一名为 unused_macro.txt 的文件。没有写对于 ``/* */`` 形式注释代码的忽略。

## 思路简单

1. 遍历目标目录并读取每个 `.m`、`.mm` 和 `.h` 文件。
2. 第一遍遍历通过正则获得所有的宏定义，并记录出现次数为 0。然后第二遍遍历获得已被查找到的宏定义出现的次数。
3. 搜索完成后，出现次数为 1 的宏定义就是没有使用过的宏定义。然后将结果输出。

## 主要代码如下

```python
def find_macros(directory, macros):
    for root, _, files in os.walk(directory):
        for file in files:
            if file.endswith(('.m', '.mm', '.h')):
                file_path = os.path.join(root, file)
                with open(file_path, 'r') as f:
                    lines = f.readlines()
                    for index, line in enumerate(lines, start=1):
                        if line.strip().startswith('//'):
                            continue

                        match = re.match(r'#define (\w+)', line)
                        if match:
                            macro_name = match.group(1)
                            macros[macro_name] = {'count': 0, 'file': file_path, 'line': index}

    for root, _, files in os.walk(directory):
        for file in files:
            if file.endswith(('.m', '.mm', '.h')):
                file_path = os.path.join(root, file)
                with open(file_path, 'r') as f:
                    lines = f.readlines()
                    for line in lines:
                        if line.strip().startswith('//'):
                            continue

                        for macro, info in macros.items():
                            if macro in line:
                                info['count'] += 1
```

## 完整代码

[find_unused_macros_in_iOS_project](https://github.com/cocoonbud/find_unused_macros_in_iOS_project)

