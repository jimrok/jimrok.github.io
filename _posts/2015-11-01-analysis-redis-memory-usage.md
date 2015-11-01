---
layout: post
title:  "分析Redis的内存占用"
date:   2015-11-01 21:55:25
categories: Redis memery rdbtools
---

Redis可以将内存中的数据保存成为一个.rdb的快照文件。可以对快照文件进行分析，发现Redis中那些数据项占用了大量的内存。

首先，安装rdbtools这个工具。这是一个python写的分析工具。对于ruby党来说，python的安装就直接使用系统的python，不再折腾另外的python环境了。

1.先安装pip，在Mac下执行

```bash
$sudo easy_install pip

```

2.安装rdbtools

```bash
$sudo pip install rdbtools

```

下面开始分析rdb文件的内存占用，将rdb文件中的key使用的内存导出到一个csv文件中。

```bash
$rdb -c memory dump.rdb > redis_memory.csv

```  

导出后，redis_memory.csv的格式如下：  

```bash
database,type,key,size_in_bytes,encoding,num_elements,len_largest_element
0,set,"clientid:2934501:oumsg:set",109,intset,1,8
0,list,"conv:msg:queue:29258",109,ziplist,1,8
0,set,"a_users:8621",260,hashtable,1,6
0,sortedset,"thd:558",129,ziplist,4,8
0,sortedset,"conv_feed:577",109,ziplist,1,8
0,string,"user:first_login:3002",94,string,8,8
```
database 是Redis数据库的编号  
type 是Redis中的数据类型：set,sortedset,string,list,hash  
key Redis中使用的Key  
size_in_bytes 内存中使用的bytes数  
encoding 编码方式  
num_elements 元素个数  
len_largest_element 最大元素的长度  

剩下的事情就是写个程序去分析一下那个类型的key占了最多的内存。ruby党自然是用ruby去写。

```ruby

#!/usr/bin/env ruby
# -*- coding: utf-8 -*-


type_sum = {}

i_index = 0
File.open(File.expand_path("../redis_memory.csv", __FILE__), "r") do |f|
  f.each_line do |line|
    items = line.split(",")
    next if items[0] != "0"  # 只分析第一个数据库
    # next if items[1] != "sortedset"
    # next if items[2][0,5] != "\"feed:"

    _key = items[1] # 按照类型进行分组统计
    t = type_sum[_key]
    if t then
    	type_sum[_key] = t + items[3].to_i
    else
    	type_sum[_key] = items[3].to_i
    end

    i_index = i_index + 1
    puts(i_index) if i_index % 100 == 0
  end
end
# File is closed automatically at end of block

puts "items:#{type_sum.inspect}"

```