---
layout: post
title: "MySQL&MariaDB导入导出数据小贴士"
date: 2018-02-12 21:50:50 +0800
tags: [mysql, mariadb]
---

有时候需要用mysqldump来导数据，记录下省得查文档。

## 导出数据到sql文件
```
mysqldump --single-transaction --no-create-info --quick dbname tablename --where="t>a and t<b" > data.sql
```

## 导出数据到压缩文件
```
mysqldump --single-transaction --no-create-info --quick dbname tablename --where="t>a and t<b" | gzip > data.sql.gz
```

## 后台运行导出数据
```
nohup sh -c 'mysqldump --single-transaction --no-create-info --quick dbname tablename --where="t>a and t<b" | gzip > data.sql.gz' &
```

## 后台运行导入数据
```
nohup sh -c 'gunzip < data.sql.gz | mysql dbname' &
```

## 导入数据时显示进度：
```
pv data.sql.gz | gunzip | mysql dbname
```

* pv需要单独安装
* 在用户目录下使用.my.cnf来使用默认用户密码
