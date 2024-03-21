# iOS 头文件引用排序、去重和分堆

整理陈旧的项目代码时，常会面对一个 class 引用的头文件特别多。里面可能也偶尔会有重复引用。这个脚本用于自动排序和按一定规则分堆 iOS 项目中的头文件引用，并且支持去除重复引用。适用于 Objective-C 项目。使得代码清晰，整洁，头文件引用去除冗余。

## 功能特点

- **头文件引用排序**和分堆：根据一定的规则对头文件引用进行排序，提高代码可读性，也整洁。
- **去除重复引用**：自动检测并去除重复的头文件引用。
- 头文件引用少于 5 个的文件跳过。

## 排序和分堆规则

1. 如果头文件名以 `View` 结尾，归为一堆。（不区分大小写，以下同样
2. 如果头文件名以 `btn` 或 `button` 结尾，归为一堆。
3. 如果头文件名以 `VC` 或 `Viewcontroller` 结尾，归为一堆。
4. 如果头文件名中包含 `helper` 或 `manager`，归为一堆。
5. 如果头文件名中包含 `+` 号，归为一堆。
6. 其它头文件按字母排序，归为一堆。 （当然你可以修改脚本，自行定义。

## 主要代码

~~~python
def start_processing(directory):
    for root, _, files in os.walk(directory):
        for file in files:
            if any(file.endswith(suffix) for suffix in ['.m', '.mm', '.h']):
                file_path = os.path.join(root, file)
                print(f"处理文件: {file_path}")
                header_refs = find_header_refs(file_path)

                # 如果头文件引用数目少于 5 个，则跳过此文件
                if len(header_refs) < 5:
                    print(f"跳过文件 {file_path}，头文件引用数目少于 5 个")
                    continue

                sorted_header_groups = sort_and_group_headers(header_refs)

                # 读取文件内容
                with open(file_path, 'r') as f:
                    lines = f.readlines()

                # 查找插入排序后的起始位置
                first_import_position = -1
                last_import_position = -1
                for i, line in enumerate(lines):
                    if line.startswith('#import "') or line.startswith('#import <'):
                        if first_import_position == -1:
                            first_import_position = i
                        last_import_position = i

                # 删除原始的引用头文件
                del lines[first_import_position:last_import_position + 1]

                count = len(sorted_header_groups)
                # 插入sort后的头文件引用
                for i in range(count):
                    group = sorted_header_groups[i]
                    if group:
                        if i == count - 1:
                            lines.insert(first_import_position, '\n'.join(group) + '\n')
                        else:
                            lines.insert(first_import_position, '\n'.join(group) + '\n' + '\n')
                            first_import_position += 1

                # 把修改后的内容写回文件
                with open(file_path, 'w') as f:
                    f.writelines(lines)
~~~



## 示例

排序分堆去重前：

~~~objective-c
//
//  THSortImportHelper.m
//
//  Created by triple on 20xx/xx/xx.
//

#import "THAdd.h"
#import "THSortImportClass.h"
#import "ViewController.h"
#import "ViewController.h"
#import "ViewController.h"
#import "ViewController.h"
#import "TH1Btn.h"
#import "THCaange.h"
#import "THChange.h"
#import "NSString+str2.h"
#import "NSString+str2.h"
#import "THFind.h"
#import "TH2View.h"
#import "NSString+str1.h"

@implementation THSortImportHelper

@end
~~~

排序分堆去重后：

~~~objective-c
//
//  THSortImportHelper.m
//
//  Created by tripleEye on 20xx/xx/xx.
//

#import "TH2View.h"

#import "TH1Btn.h"

#import "ViewController.h"

#import "NSString+str1.h"
#import "NSString+str2.h"

#import "THAdd.h"
#import "THCaange.h"
#import "THChange.h"
#import "THFind.h"
#import "THSortImportClass.h"

@implementation THSortImportHelper

@end
~~~

## 注意事项

- 脚本会修改原始文件，请提前备份项目。
- 建议在对整个项目进行操作之前，在一个小范围的测试项目上尝试使用脚本。

## 项目地址

[sort_iOS_header_refs](https://github.com/cocoonbud/sort_iOS_header_refs)
