---
title: 构建Text2SQL多轮对话的benchmark
toc: true
date: 2024-07-02 15:27:17
categories: 技术
excerpt: 最近因项目需要做的一些调研，顺手记录一下了
cover:
---
# 数据集
## 已有数据集
经调研，目前已有的较完善的数据集有Chase[^1]、SParC[^2]、CoSQL[^3]

其中SParC和CoSQL都是耶鲁大学与SalesForce合作的成果，Chase是西交和微软合作的成果，三个数据集中只有它是中文的

## 统计[^4]
|数据集|数据量|1轮|2轮|3轮|4轮|5轮|6轮|更多|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|Chase|3949|0|697|1858|1033|352|9|0|
|SParC|3034|8|793|1551|633|44|5|0|
|CoSQL|2458|2|5|64|874|675|456|382|

```python Chase\SParC
import json
data=json.load(open("train.json"))
count={}
for i in list(data):
    l=len(i["interaction"])
    count[l]=count.get(l,0)+1
for i in list(sorted(count)):
    print(i," ",count[i])
```

```python CoSQL
import json
data=json.load(open("cosql_all_info_dialogs.json"))
count={}
for i in list(data):
    i=data[i]
    l=0
    for j in i["turns"]:
        if j["isUser"]:
            l+=1
    count[l]=count.get(l,0)+1
for i in list(sorted(count)):
    print(i," ",count[i])
```

## exact set match metrics
浏览数据集的时候发现，chase的ground truth以两种形式同时呈现

比如，这个语句`SELECT name FROM department GROUP BY departmentID ORDER BY count(departmentID) DESC LIMIT 1;`就会被拆分成：

```json
"sql": {
    "orderBy": [
        "desc", 
        [
            [
                0, 
                [
                    3, 
                    5, 
                    false
                ], 
                null
            ]
        ]
    ], 
    "from": {
        "table_units": [
            [
                "table_unit", 
                1
            ]
        ], 
        "conds": []
    }, 
    "union": null, 
    "except": null, 
    "groupBy": [
        [
            0, 
            5, 
            false
        ]
    ], 
    "limit": 1, 
    "intersect": null, 
    "where": [], 
    "having": [], 
    "select": [
        false, 
        [
            [
                0, 
                [
                    0, 
                    [
                        0, 
                        6, 
                        false
                    ], 
                    null
                ]
            ]
        ]
    ]
}
```
可以看出来以操作的关键字结构化地分割了，但是这样做的目的，以及对象中的数字、布尔值、空值的意义都不明确

而在SParC的论文[^2]中提到了，这是受Spider[^5]的启发

> Following Yu et al. (2018c), we use the exact set match metric to compute the accuracy between gold and predicted SQL answers. Instead of simply employing string match, Yu et al. (2018c) decompose predicted queries into different SQL clauses such as SELECT, WHERE, GROUP BY, and ORDER BY and compute scores for each clause using set matching separately.

这么做的原因是为了提高打分的精确度，以及对不同操作进行更细粒度的评测

> We report the following two metrics: question match, the exact set matching score over all questions, and interaction match, the exact set matching score over all interactions. The exact set matching score is 1 for each question only if all predicted SQL clauses are correct, and 1 for each interaction only if there is an exact set match for every question in the interaction.

spider也提供了[一个转换脚本](https://github.com/taoyds/spider/blob/master/process_sql.py)

## 构建数据集
上述Chase和SParC是已有工作中比较好的 ~~好像也没啥别的~~ 故采用他们提出的方法，针对项目的场景，构建自己的数据集

# 评测
使用耶鲁大学课题组与论文一同发布的[评测代码](https://github.com/taoyds/test-suite-sql-eval)，并在此基础上做一些小修改以适应需求

[^1]: [Chase 1.0: A Large-Scale and Pragmatic Chinese Dataset for Cross-Database Context-Dependent Text-to-SQL](https://xjtu-intsoft.github.io/chase/)
[^2]: [SParC 1.0: Yale & Salesforce Semantic Parsing and Text-to-SQL in Context Challenge](https://yale-lily.github.io/sparc)
[^3]: [CoSQL 1.0: A Conversational Text-to-SQL Challenge Towards Cross-Domain Natural Language Interfaces to Databases](https://yale-lily.github.io/cosql)
[^4]: SParc中有4个“0轮”的，即该条数据的interaction键对应的数组长度为0，但因为final中储存了问题和答案，所以实际也是1轮，只是缺少了意图等数据
[^5]: [Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task](https://arxiv.org/abs/1809.08887)