## 一种基于存储过程分表建表的的方法

本文讲述一种基于存储过程和mysql的的分表建表方法。相关背景如下：
```
在云容器时代，应用容器化因其隔离性及部署迁移的灵活性等特征越来越成为主流，而容器调度编排利器kubernetes也深受广大程序员们的喜爱，kubernetes中的资源基于namespace隔离，例如deployment、job、cronJob等。以此为背景，假设要设计数据库表存储deployment相关信息，且考虑到后续数据量要做分表存储。
```
通过上述背景及相关知识，我们可以得到以下几点要求：
```
1.namespace与deployment是一对多的关系。
2.deployment表中至少要包含namespace和deployment名称两个字段；
3.deployment表要分表设计，分为32张表（32张表仅为示例数据，不必对该数字做深究）；
```
前面两点都很好实现，数据库表中设计namespace和name两个字段就可以解决，第三点需要分析以下几个问题：
#### 1.基于什么分表？

kubernetes中查询deployment资源时需要namespace，产品功能的定义也是基于namespace对deployment对数据库做增删改查，因此可以考虑根据namespace来对deployment分表，不同namespace下的deployment数据存储在不同的表中，根据namespace按一定规则（如hash）确定表名，确保同一namespace下的deployment存储在同一张表中。试想如果是根据deployment name来分表，产品上要查询某一命名空间下所有deployment的功能将无法实现（除非加另外的关系表）。

#### 2.表名如何设计？

上边说到可以根据namespace分表，并对namespace做相关变换如hash，得到表名。但是如果真的用普通的hash得到一个字符串会不会有什么问题呢？首先，普通hash不好保证hash后只有32个结果，其次，如果表名是deployment_xuhytjrmmjpa, deployment_yhkptsxqwsrbd...从后续运维的角度上来说会很不方便。因此考虑表名结构为“deployment_”+“数字”，如deployment_0，deployment_1，... deployment_31。

#### 3.程序中表名如何获取？

以go语言为例，可以这么来实现：
```
package main

import (
    "hash/crc32"
    "strconv"
)
const divisionNum = 32 //分表数量

func crcHash(src string) string {
    sum := crc32.ChecksumIEEE([]byte(src))
    return strconv.Itoa(int(sum % divisionNum))
}

func GetTableName(baseName, arg string) string {
    return baseName + "_" + crc32Hash(arg)
}

func main() {
    const baseTableName = "deployment"
    const namespace = "default"
    fmt.Println(GetTableName(baseTableName, namespace))
}
```

#### 4.建表sql如何写（基于mysql）？

如果创建一张表，建表sql就是CREATE TABLE deployment (...)，但是deployment有32张表，写32个CREATE TABLE吗？后续如果再增加9张其它表也要分表，那写320条建表sql吗？考虑到这些原因，可以考虑用存储过程来建表，如下：
```
-- 分表建表存储过程
DELIMITER $$
DROP PROCEDURE IF EXISTS create_table;
CREATE PROCEDURE create_table(IN tableName VARCHAR(32), IN tableInfo VARCHAR(2048))
BEGIN
    -- 分成32张表
    DECLARE `@divisionNum` INT DEFAULT 32;
    DECLARE `@i` INT DEFAULT 0;

    WHILE `@i` < `@divisionNum` DO
        -- 拼接表名，表名与序号中间用'_'连接，如table_0, table1
        SET @newTableName = CONCAT(tableName, '_', `@i`);
        -- 拼接建表语句
        SET @createTableSql = CONCAT('CREATE TABLE IF NOT EXISTS', @newTableName, tableInfo);
        -- 执行建表语句
        PREPARE stmt FROM @createTableSql;
        EXECUTE stmt;

        SET `@i` = `@i`+1;
    END WHILE;
END; $$
DELIMITER ;

-- 创建deployment表存储过程
DELIMITER $$
DROP PROCEDURE IF EXISTS create_deployment_table;
CREATE PROCEDURE create_deployment_table()
BEGIN
    SET @tableName = 'deployment';
    SET @tableInfo = '(
        `id` INT(11) UNSIGNED NOT NULL AUTO_INCREMENT,
        `namespace` VARCHAR(64) NOT NULL COMMENT "命名空间namespace",
        `name` VARCHAR(64) NOT NULL COMMENT "deployment名称",
        `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT "创建时间",
        PRIMARY KEY (`id`)
    ) ENGINE=INNODB DEFAULT CHARSET=UTF8MB4 COMMENT="deployment表";';

    CALL create_table(@tableName, @tableInfo);
END; $$
DELIMITER ;

-- 调用创建deployment表的存储过程
CALL create_deployment_table();
```

### 总结
在数据库分表的时候，如何分表，即根据什么信息分表是最为重要的，因为这影响到后续功能的实现；在代码中可以考虑用crc的方式做表名拼接；而在建表的时候，存储过程是一种比较优雅的实现方式。