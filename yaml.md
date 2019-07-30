YAML语法简介
=======


## YAML基础
YAML是专门用来写配置文件的语言,非常简介和强大,相对JSON格式它更易读写。

## 基本语法规则
* 大小写敏感
* 使用缩进表示层级关系
* 缩进时不允许使用Tab键,只允许使用空格
* 缩进的空格数目不重要,只要相同层级的元素左侧对齐即可
* \#号表示注释,从这个字符一直到行尾,都会被解析器忽略

## 支持三种数据结构

* **对象** : 键值对的集合,又称为映射(mapping)/哈希(hashes)/字典(dictionary)
* **数组** ：一组按次序排列的值,又称为序列(sequence)/列表(list)
* **纯量(scalars)** : 单个的、不可再分的值

## 对象
对象就是一组键值对,使用冒号结构表示。
``` json
role: web
env: test
region: cn-beijing
zone: cn-beijing-f
```
转为javascript如下
``` json
{role: 'web', env: 'test', region: 'cn-beijing', zone: 'f'}
```
## 数组
以中线开头的行组成一个数组
``` json
command:
- wget
- "-O"
- "/work-dir/index.html:
- http://kubernetes.io
```
转为JavaScript如下:
``` json
['wget', '-O', '/work-dir/index.html', 'http://kubernetes.io']
```
## 复合结构
```json
containers:
- name: nginx
  image: nginx:latest
- name: php
  image:php:latest
```
转为JavaScript如下
``` json
{
  containers:
  [
    {name: 'nginx', image: 'nginx:latest'}
    {name: 'php', image: 'php:latest'}
  ]
}
```

## 纯量
纯量是最基本的、不可再分的值。
* 字符串 --'我是字符串'
* 布尔值 -- true或 false
* 整数 -- 10
* 浮点数 -- 10.30
* Null -- null
* 时间 -- 21:59:00
* 日期 -- 1976-07-31

## 多行字符串
多行字符串可以使用 |保留换行,+表示保留文字块末尾的换行， -表示删除字符串末尾的换行。

``` json
data:
  nginx-conf: |-
    server {
        listen 80;
        server_name hello.hipstershop.cn;
        location / {
            include uwsgi_params;
            uwsgi_pass unix:///tmp/uwsgi.sock;
        }
    }

```

## 多个配置写到同一文件
多个配置写到同一文件，可用连续三个连子号---区分多个文件
``` json
apiVersion: v1
kind: Deployment
...
---
apiVersion: v1
kind: Service
```
