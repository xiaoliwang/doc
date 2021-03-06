# Pull Request 规范

[TOC]

## 一般规范

提交一个新的 PR，需要参照中文文档规范 https://github.com/sparanoid/chinese-copywriting-guidelines。并按下面的要求进行处理。

1.  title 里概括性的描述 PR 具体实现的是什么功能。
2. 在 comment 里列出具体实现了哪些功能。

下面是一个例子：

> title:
>
> 登录模块
>
> comment:
>
> 1. 实现账户，密码登录
> 2. 实现第三方登录（包括 QQ 登录，微博登录）
>    1. 对第三方 token 进行验证
>    2. balabala...



如果在该 PR 对**配置文件**，**其他项目**进行了相应修改，或添加新的约定。需要在 comment 中标示出来。并提醒 Reviewer 添加具体的 *labels*，在合并后对线上进行修改。例如：

>（续上）
>
>本次改动对**配置文件**进行了修改。具体如下：
>
>1. 在 `key.php` 添加新的变量
>
>   ```php
>   define('KEY', 'value');
>   ```
>
>   ​
>
>2. 在 `conf.php` 配置新的组件
>
>   ```php
>   'test' => 'CLASS', // 136行
>   ```
>
>
>
>这次修改需要同时更新 **SSO** 项目及 **drama** 项目
>
>
>
>这次修改同时约定了新的错误代码
>
>| 错误代码      | 对应 http code | 详细描述  |
>| --------- | ------------ | ----- |
>| 200130001 | 404          | 剧集不存在 |
>
>



如果 PR 中有新建数据表或对数据表进行修改，也需要在 comment 中进行说明：

> （续上）
>
> 添加新表
>
> ```sql
> CREATE TABLE xxx ....
> ```
>
> 

如果 PR 中设计较复杂的**查询语句**，必须在 comment 中附上该 SQL 的 EXPLAIN 信息。

1. `type` 字段至少要达到 range 级别，要求是 ref 级别，如果可以是 consts  最好。
2. `Extra` 字段中不能出现 `Using filesort`。
3. 查询语句如果没有特殊需求，单词返回数量不能超过 500 条。且多条查询不能返回 `text` 字段。



## 特殊情况

### 提交引入第三方 SDK 的 PR

在引入第三方 SDK 的情况下，至少需要提交三次 commit：

1. 引入第三方 SDK
2. 添加配置文件及对 SDK 的封装组件
3. 组件的具体应用

