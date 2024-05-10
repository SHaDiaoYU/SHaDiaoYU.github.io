---
title: SpringBoot学习-ES相关内容
date: 2023-06-01 16:20:57
tags:
---

#### 1. ES 索引操作

使用方式：发送restful的web请求即可使用。假设我们要创建一个索引名为books:

* **创建索引：**通过发送PUT请求`http://localhost:9200/books`完成，返回结果如下：

  ```json
  {
      "acknowledged": true,
      "shards_acknowledged": true,
      "index": "books"
  }
  ```

  注意，索引值唯一，如果重复创建，es会报错。

* **查询索引：** 通过发送GET请求`http://localhost:9200/books`完成。返回该索引的各种信息：

  ```json
  {
      "books": {
          "aliases": {},
          "mappings": {},
          "settings": {
              "index": {
                  "routing": {
                      "allocation": {
                          "include": {
                              "_tier_preference": "data_content"
                          }
                      }
                  },
                  "number_of_shards": "1",
                  "provided_name": "books",
                  "creation_date": "1685608096323",
                  "number_of_replicas": "1",
                  "uuid": "uD2KU8OFRS2CvCj39kBCyg",
                  "version": {
                      "created": "8080099"
                  }
              }
          }
      }
  }
  ```

* **删除索引：**通过发送DELETE请求`http://localhost:9200/books`完成。返回结果：

  ```json
  {
      "acknowledged": true
  }
  ```

* **使用分词器：**

  * 使用ik分词器：将其解压到elasticsearch的plugins目录下即可使用（注意版本匹配。）

  * 重启elasticsearch，如果出现下列内容则说明ik分词器安装成功：

    ![image-20230602193459847](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230602193459847.png)

  * 创建索引并指定规则：添加请求体，postman上方式为Body->raw-JSON。为每个字段添加规则。注意，为了实现既对name字段查询，又对的description字段查询，引入了一个新的字段all，并为name以及description字段添加属性`"copy_to":"all"`以实现该效果。

  ```json
  {
      "mappings":{
          "properties":{
              "id":{"type":"keyword","copy_to":"all"},
              "name":{"type":"text","analyzer":"ik_max_word"},
              "type":{"type":"keyword","copy_to":"all"},
              "description":{"type":"text","analyzer":"ik_max_word"},
              "all":{"type":"text","analyzer":"ik_max_word"}
          }
      }
  }
  ```

#### 2. ES文档操作

es的文档是无模式的。

* **创建文档（请求方式为post）**

  * 方式1：请求地址为`http://localhost:9200/books/_doc`，这种方式使用的是系统生成的id。
  * 方式2：请求地址为`http://localhost:9200/books/_create/1`，这种方式使用的是指定的id（此处指定id为1）。
  * 方式3：请求地址为`http://localhost:9200/books/_doc/1`，这种方式使用指定的id，id不存在则创建，存在则更新。

  请求体格式：

  ```json
  {
      "name":"springboot",
      "type":"springboot",
      "description":"springboot"
  }
  ```

* **查询文档（请求方式为get）**

  * 方式1：请求地址为`http://localhost:9200/books/_doc/1`，查询指定id（此处为1）的文档。
  * 方式2：请求地址为`http://localhost:9200/books/_search`，查询全部的文档。

* **条件查询（请求方式为get）**

  * 请求地址为`http://localhost:9200/books/_search?q=name:springboot`

* **删除文档（请求方式为delete）**

  * 请求地址为`http://localhost:9200/books/_doc/1`，删除指定id（此处为1）的文档

* **修改文档（全量修改，请求方式为put）**

  * 请求地址为`http://localhost:9200/books/_doc/1`
  * 请求体格式：

  ```json
  {
      "name":"springboot good",
      "type":"springboot",
      "description":"springboot"
  }
  ```

* **修改文档（部分修改，请求方式为post）**

  * 请求地址为`http://localhost:9200/books/_update/1`
  * 请求体格式：

  ```json
  {
      "doc":{
          "name":"springboot"
      }
  }
  ```

  * 注意部分修改与全量修改的请求地址之间的区别。



