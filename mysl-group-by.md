# 聊一聊 MySQL 的 group by

## group by 和 order by

首先我们知道 `order by` 可以用来排序，可以针对一列或多列进行 `asc` `desc` 排序。
而 `group by` 可以用来分组，合并一列中重复的内容。

有时候，我们需要在 `group by` 的时候，希望先排序一下……

所以直觉的写出 `group by class order by score desc` 来获取每个班级分数最高的那位。

但是你会发现，出来的结果并非你想要的。因为 `order by` 排序失效了。

是的，因为在执行顺序上，是先执行 `group by`，而 `order by` 只是对结果集进行排序而已。

那你会到百度搜索，搜索出来大多数的答案是：子查询。

写出来的语句大概是：

```
select * from 

(select * from student order by score desc) as c

group by c.class
```

但是当你安装了一个较新的 MySQL 版本，5.7以上，你会发现，这样的结果跟上面的一样。

子查询语句的 `order by` 也失效了。

然后再去搜索，大概率会看到 

`select class, max(score) as max_score from student group by class`

这样就能达到目的了。

## 加上联表呢？

接上一小节，聚合函数的确能帮我们很方便的实现。

但是当你需要联表的时候，直觉得写出：

```
select s.id, s.class, max(score) as max_score, si.name 

from student as s

left join student_info as si on s.id = si.student_id

group by class
```

当你安装的是 5.7 以上版本，你会发现报错了。。。

一个关于 `ONLY_FULL_GROUP_BY`，大概率会叫你也得把 id 加入 `group by`。

但是那样出来的结果也不是我们想要的，怎么办呢？照样去百度搜索……

你会找到很多教你如何如何关闭 `ONLY_FULL_GROUP_BY`。

当然关闭了，自然就搞定了……

## MySQL 5.7 的 ONLY_FULL_GROUP_BY

关于 `ONLY_FULL_GROUP_BY` 一搜一大堆，大概是关于语义上的事。

就是以前太松散了，现在要严格限制……

语法大概是这样：`select 选取分组中的列+聚合函数 from 表名 group by 分组的列`

那确实得把 `id` 放进去 `group by` 才不会报错，但是结果集就不是我们所期望的了。

## 又是子查询

那我们再试试子查询吧

```
select s.id, s.class, sub.max_score, si.name 

from (select class, max(score) as max_score from student group by class) as sub

inner join student as s on sub.class = s.class and sub.max_score = s.score

left join student_info as si on s.id = si.student_id
```

虽然麻烦了些，但是总算是解决了问题。