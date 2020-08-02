[测试数据集](https://github.com/datacharmer/test_db)

### 防止全表扫描

#### 索引失效的常见场景

测试表为：

    CREATE TABLE employees (
        emp_no      INT             NOT NULL,
        birth_date  DATE            NOT NULL,
        first_name  VARCHAR(14)     NOT NULL,
        last_name   VARCHAR(16)     NOT NULL,
        gender      ENUM ('M','F')  NOT NULL,    
        hire_date   DATE            NOT NULL,
        PRIMARY KEY (emp_no)
    );

    CREATE TABLE departments (
        dept_no     CHAR(4)         NOT NULL,
        dept_name   VARCHAR(40)     NOT NULL,
        PRIMARY KEY (dept_no),
        UNIQUE  KEY (dept_name)
    );

    CREATE TABLE dept_manager (
        emp_no       INT             NOT NULL,
        dept_no      CHAR(4)         NOT NULL,
        from_date    DATE            NOT NULL,
        to_date      DATE            NOT NULL,
        FOREIGN KEY (emp_no)  REFERENCES employees (emp_no)    ON DELETE CASCADE,
        FOREIGN KEY (dept_no) REFERENCES departments (dept_no) ON DELETE CASCADE,
        PRIMARY KEY (emp_no,dept_no)
    ); 

    CREATE TABLE dept_emp (
        emp_no      INT             NOT NULL,
        dept_no     CHAR(4)         NOT NULL,
        from_date   DATE            NOT NULL,
        to_date     DATE            NOT NULL,
        FOREIGN KEY (emp_no)  REFERENCES employees   (emp_no)  ON DELETE CASCADE,
        FOREIGN KEY (dept_no) REFERENCES departments (dept_no) ON DELETE CASCADE,
        PRIMARY KEY (emp_no,dept_no)
    );

    CREATE TABLE titles (
        emp_no      INT             NOT NULL,
        title       VARCHAR(50)     NOT NULL,
        from_date   DATE            NOT NULL,
        to_date     DATE,
        FOREIGN KEY (emp_no) REFERENCES employees (emp_no) ON DELETE CASCADE,
        PRIMARY KEY (emp_no,title, from_date)
    ) 
    ; 

    CREATE TABLE salaries (
        emp_no      INT             NOT NULL,
        salary      INT             NOT NULL,
        from_date   DATE            NOT NULL,
        to_date     DATE            NOT NULL,
        FOREIGN KEY (emp_no) REFERENCES employees (emp_no) ON DELETE CASCADE,
        PRIMARY KEY (emp_no, from_date)
    )

使用索引可以防止查询数据时进行全表扫描，但是使用不当时可能造成索引失效，索引失效的场景主要有以下几种：

1. 当MYSQL的优化器认为使用全表扫描比使用索引的速度要快时，不会使用索引，但可以使用FORCE INDEX强制指定索引。

案例A：在不恰当的字段上建立索引，比如数据的基数比较小（个人认为这种情况索引不起作用是由于优化器影响的）。

首先为gender字段建立一个索引：

    alter table employees add index GENDER_INDEX(gender)
    -- 删除索引
    -- alter table employees drop index GENDER_INDEX;

使用gender字段进行排序：

    mysql> explain select *from employees order by gender;
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra          |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299556 |   100.00 | Using filesort |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
    1 row in set, 1 warning (0.00 sec)

    ## 强制使用索引
    mysql> explain select * from employees force index(GENDER_INDEX) order by gender;
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+--------+----------+-------+
    | id | select_type | table     | partitions | type  | possible_keys | key          | key_len | ref  | rows   | filtered | Extra |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+--------+----------+-------+
    |  1 | SIMPLE      | employees | NULL       | index | NULL          | GENDER_INDEX | 1       | NULL | 299556 |   100.00 | NULL  |
    +----+-------------+-----------+------------+-------+---------------+--------------+---------+------+--------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，不使用 FORCE INDEX 强制指定索引时，虽然在 gender 字段上指定的索引，但是索引并没有生效。


案例B：对某个字段使用负向查询（如 NOT、!=、<>、!<、!>、NOT IN、NOT LIKE）或者 null 判断，此时由于返回的数据在表中的占比较大，优化器也有可能让索引失效。

首先为 birth_date 字段添加索引：

    alter table employees add index BIRTH_DATE_INDEX(birth_date);
    -- 删除索引
    -- alter table employees drop index BIRTH_DATE_INDEX;

查询 birth_date 字段不为空的数据：

    mysql> explain select *from employees where birth_date is not null;
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299556 |   100.00 | NULL  |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，索引并没有生效。

注：网上流传的大部分说法是使用负向查询一定会使索引失效，这是错误的，这个只能看MYSQL优化器的选择。

修改 employees 表中 birth_date 部分数据：

    alter table employees modify column birth_date date;
    update employees set birth_date = null where birth_date not like "1955-01%";

再次查询 birth_date 字段不为空的数据：

    mysql> explain select *from employees where birth_date is not null;
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    | id | select_type | table     | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                 |
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    |  1 | SIMPLE      | employees | NULL       | range | BIRTH_DATE_INDEX | BIRTH_DATE_INDEX | 4       | NULL | 1943 |   100.00 | Using index condition |
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，此时索引并没有失效。


2. 当查询条件中使用 OR 时，并不是所有的字段都添加了索引，此时即使某些字段存在索引，索引依然会失效。

案例A：

首先为 first_name 字段添加索引：

    alter table employees add index FIRST_NAME_INDEX(first_name)
    -- 删除索引
    -- alter table employees drop index FIRST_NAME_INDEX;

使用 first_name 和 last_name 进行检索：

    mysql> explain select * from employees where first_name = "Kazuhide" or last_name = "Heyers";
    +----+-------------+-----------+------------+------+------------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys    | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+------------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | FIRST_NAME_INDEX | NULL | NULL    | NULL | 299556 |    19.00 | Using where |
    +----+-------------+-----------+------------+------+------------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

    mysql> explain select * from employees where first_name = "Kazuhide" and last_name = "Heyers";
    +----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys    | key              | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ref  | FIRST_NAME_INDEX | FIRST_NAME_INDEX | 44      | const |  234 |    10.00 | Using where |
    +----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，由于last_name字段上没有建立索引，在使用 OR 进行查询时，first_name上的索引也不会生效。

为 last_name 字段也添加索引：

    alter table employees add index LAST_NAME_INDEX(last_name)
    -- 删除索引
    -- alter table employees drop index LAST_NAME_INDEX;

使用 first_name 和 last_name 进行检索：

    mysql> explain select * from employees where first_name = "Kazuhide" or last_name = "Heyers" \G;
    *************************** 1. row ***************************
            id: 1
    select_type: SIMPLE
            table: employees
    partitions: NULL
            type: index_merge
    possible_keys: FIRST_NAME_INDEX,LAST_NAME_INDEX
            key: FIRST_NAME_INDEX,LAST_NAME_INDEX
        key_len: 44,50
            ref: NULL
            rows: 426
        filtered: 100.00
            Extra: Using union(FIRST_NAME_INDEX,LAST_NAME_INDEX); Using where
    1 row in set, 1 warning (0.00 sec)

    ERROR: 
    No query specified

此时两个字段的索引都生效了。


3. 在字段上使用内置函数，一定会导致索引失效。

案例A：将 date 类型字段转为字符串后进行比较。

首先为 birth_date 字段添加索引：

    alter table employees add index BIRTH_DATE_INDEX(birth_date);
    -- 删除索引
    -- alter table employees drop index BIRTH_DATE_INDEX;

    mysql> explain select *from employees where date_format(birth_date, "%Y-%m-%d") = "1955-01-21";
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |   100.00 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，索引并没有生效。


4. 字段的隐式转换导致索引失效

案例A：将字符串类型的字段与数字比较

对 salaries 表的 salary 字段进行修改，类型修改为 varchar：

    alter table salaries modify column salary varchar(11) NOT NULL;
    alter table salaries add index SALARY_INDEX(salary);
    -- 还原
    -- alter table salaries modify column salary int(11) NOT NULL;

对 salary 字段进行检索：

    mysql> explain select *from salaries where salary = 60117;
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    |  1 | SIMPLE      | salaries | NULL       | ALL  | SALARY_INDEX  | NULL | NULL    | NULL | 2836882 |    10.00 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    1 row in set, 3 warnings (0.00 sec)

    mysql> explain select *from salaries where salary = "60117";
    +----+-------------+----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
    | id | select_type | table    | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
    +----+-------------+----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | salaries | NULL       | ref  | SALARY_INDEX  | SALARY_INDEX | 35      | const |   65 |   100.00 | NULL  |
    +----+-------------+----------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，当 salary 字段为 varchar 时，与数字进行比较会使索引失效，此时查询某种程度上等价于

    mysql> explain select *from salaries where cast(salary as signed int) = 60117;
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    |  1 | SIMPLE      | salaries | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2838426 |   100.00 | Using where |
    +----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)


5. 隐式字符编码转换导致索引失效

对表进行关联查询的过程中，由于关联字段的编码不同，比较过程中，MYSQLK可能会对字段的编码进行了隐式转换，导致索引失效。

参考网上的说法，测试没有成功。


6. 不恰当的使用 like 通配符导致索引失效

当使用 % 开头进行查询时，会使索引失效。

案例A：

首先为 first_name 字段添加索引：

    alter table employees add index FIRST_NAME_INDEX(first_name)
    -- 删除索引
    -- alter table employees drop index FIRST_NAME_INDEX;

对 first_name 字段进行检索：

    mysql> explain select *from employees where first_name like "%atricio";
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |    11.11 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

    mysql> explain select *from employees where first_name like "%atrici%";
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |    11.11 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.01 sec)

    mysql> explain select *from employees where first_name like "Patrici%";
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    | id | select_type | table     | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                 |
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    |  1 | SIMPLE      | employees | NULL       | range | FIRST_NAME_INDEX | FIRST_NAME_INDEX | 44      | NULL |  452 |   100.00 | Using index condition |
    +----+-------------+-----------+------------+-------+------------------+------------------+---------+------+------+----------+-----------------------+
    1 row in set, 1 warning (0.00 sec)


观察结果可知，使用 % 开头会使索引失效，使用 % 结尾则不一定会。


7. 对含有索引的字段进行 +，-，*，/ 等运算，一定会使索引失效

案例A：对 emp_no 字段进行检索

    mysql> explain select *from employees where emp_no + 1 = 10002;
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |   100.00 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

    mysql> explain select *from employees where emp_no = 10002 - 1;
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
    | id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | employees | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，当在 = 的左边进行字段的运算会导致索引失效，优化时可以将运算放在 = 的右边进行。


8. 联合索引没有遵循最左匹配原则，一定会使索引失效。

案例A：

对 first_name 和 last_name 建立联合索引，此时没有单个字段索引。

    alter table employees add index NAME_INDEX(first_name, last_name);
    -- alter table employees drop index NAME_INDEX;

对 last_name 进行检索：

    mysql> explain select *from employees where first_name = "Saniya";
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    | id | select_type | table     | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | employees | NULL       | ref  | NAME_INDEX    | NAME_INDEX | 44      | const |  257 |   100.00 | NULL  |
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

    mysql> explain select *from employees where first_name = "Saniya" and last_name = "Kalloufi";
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------------+------+----------+-------+
    | id | select_type | table     | partitions | type | possible_keys | key        | key_len | ref         | rows | filtered | Extra |
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------------+------+----------+-------+
    |  1 | SIMPLE      | employees | NULL       | ref  | NAME_INDEX    | NAME_INDEX | 94      | const,const |    1 |   100.00 | NULL  |
    +----+-------------+-----------+------------+------+---------------+------------+---------+-------------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

    mysql> explain select *from employees where last_name = "Kalloufi";
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |    10.00 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)

观察结果可知，直接对 last_name 字段进行检索，索引是不会生效的。

注：参照网上流传的部分说法，上述查询使用 last_name = "Kalloufi" and first_name = "Saniya" 时索引会失效，这是错误的，MYSQL会对SQL进行优化，此时依然会按照最左匹配原则进行检索，与书写的顺序并没有之间关联。

    
9. 使用了 IGNORE INDEX 关键字忽略索引。

案例A：

首先为 first_name 字段添加索引：

    alter table employees add index FIRST_NAME_INDEX(first_name)
    -- 删除索引
    -- alter table employees drop index FIRST_NAME_INDEX;

对 first_name 字段进行检索并忽略 FIRST_NAME_INDEX 索引：

    mysql> explain select *from employees IGNORE INDEX(FIRST_NAME_INDEX) where first_name = "Saniya";
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    | id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    |  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299147 |    10.00 | Using where |
    +----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)
