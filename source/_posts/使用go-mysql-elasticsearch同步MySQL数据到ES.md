---
title: 使用go-mysql-elasticsearch同步MySQL数据到ES
date: 2020-07-05 20:55:31
tags: ["ElasticSearch", "MySQL"]
---
[ElasticSearch](https://www.elastic.co/)是一个基于[Lucene](http://lucene.apache.org)的分布式搜索引擎和大数据近实时分析引擎,我们常常用它来构建全文搜索引擎，结合Logstash+Kibana等搭建日志分析平台等。

使用ES做全文搜索引擎时,首先需要把数据存储到ES中,我们可以直接使用ES提供的Rest API管理数据,但也有些场景我们的数据是存储在关系型数据库如MySQL中,这时候我们需要有工具可以帮我们自动把MySQL的存量和增量数据同步到ES中,本文将介绍如何基于开源项目[go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch)实现该功能。
<!-- more -->
## 需求概述

## 定义MySQL表及插入数据
假设我们已经有一张文章表,表结构如下:
```sql

```

## 安装Elasticsearch及IK分词器

## 创建ES索引及Mapping

## go-mysql-elasticsearch简介

## 安装go-mysql-elasticsearch

## 配置go-mysql-elasticsearch

## 启动&验证
supervisor
## 总结
