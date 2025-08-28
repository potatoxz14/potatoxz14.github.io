---
title: 🎉 Shell
summary: Linux Shell脚本用法
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
---

## 生成固定大小的文件并读取产生page_cache

```shell
#!/bin/bash

# 文件大小（以MB为单位）
file_size_mb=300

# 生成文件
dd if=/dev/zero of=large_file bs=1M count=$file_size_mb
# 显示文件信息
ls -lh large_file

# 读取文件，产生大量page cache
cat large_file > /dev/null
```

## if使用

```shell
#!/bin/bash

# 定义一个字符串变量
my_string="1"

# 判断字符串变量是否等于1
if [ "$my_string" = "1" ]; then
  echo "字符串变量等于1"
else
  echo "字符串变量不等于1"
fi
```

## 字符集

```shell
#!/bin/bash

# 定义字符集
characters="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

# 定义目标目录
target_directory="/path/to/your/directory"

# 切换到目标目录
cd "$target_directory" || exit

# 生成所有可能的三个字符组合
for char1 in $(echo $characters | sed -e 's/\(.\)/\1 /g'); do
  for char2 in $(echo $characters | sed -e 's/\(.\)/\1 /g'); do
    for char3 in $(echo $characters | sed -e 's/\(.\)/\1 /g'); do
      # 拼接字符并创建文件
      filename="${char1}${char2}${char3}"
      touch "$filename"
    done
  done
done

echo "文件已创建在目录: $target_directory"
```

## for循环

```shell
#!/bin/bash

# 初始化空字符串
result=""

# 生成字母序列
for ((i = 65; i <= 122; i++)); do
    # 将 ASCII 码值转换为字符
    result+="$(printf \\$(printf '%03o' $i))"
done

# 打印生成的字符串
echo $result



#!/bin/bash

# 初始化空字符串
result=""

# 循环生成 A/B 循环
for ((i = 0; i < 4096; i++)); do
    # 使用求余运算符确定当前循环是 A 还是 B
    if ((i % 2 == 0)); then
        result+="A"
    else
        result+="B"
    fi
done

# 打印生成的字符串
echo $result

```

