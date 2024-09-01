---
title: 在 PostgreSQL 函数中使用可选条件
date: 2024-08-31 22:35:41
updated: 2024-08-31 22:35:41
tags: [PostgreSQL]
---

为了说明问题我创建了一张员工表 `employees` 并初始化了一些数据

```sql
CREATE TABLE employees
(
    id         INTEGER, -- ID
    name       VARCHAR, -- 姓名
    age        INTEGER, -- 年龄
    department VARCHAR, -- 部门
    status     VARCHAR, -- 状态
    PRIMARY KEY (id)
);

INSERT INTO employees(id, name, age, department, status) VALUES (1, 'Bob', 18, 'HR', 'Active');
INSERT INTO employees(id, name, age, department, status) VALUES (2, 'Eve', 19, null, 'Inactive');
INSERT INTO employees(id, name, age, department, status) VALUES (3, 'Zoe', null, 'IT', null);
```

<!-- more -->

在 Java 语言中根据传入的条件获取员工，如果条件不为空则查询满足条件的员工，否则查询所有的员工，我们可能会写出类似如下的代码

```java
public List<Employee> getEmployees(String department) {
    String sql = "SELECT * FROM employees WHREE 1 = 1";
    if (department != null) {
        sql = sql + " AND department = '" + department + "'";
    }
    List<Employee> employees = jdbcTemplate.query(sql, new RowMapper<Employee>() {
        @Override
        public Employee mapRow(ResultSet rs, int rowNum) {
            Employee e = new Employee();
            e.setId(rs.getInt("id"));
            e.setName(rs.getString("name"));
            e.setAge(rs.getInt("age"));
            e.setDepartment(rs.getString("department"));
            e.setStatus(rs.getString("status"));
            return e;
        }
    });
    return employees;
}
```

那么在 SQL 函数中又该如何实现类似的功能呢？以 PostgreSQL 函数为例

```sql
CREATE OR REPLACE FUNCTION fn_get_employees(_department VARCHAR)
RETURNS SETOF employees
AS $$
DECLARE
    _sql VARCHAR;
BEGIN
    _sql = 'select * from employees where true';
    IF _department IS NOT NULL THEN
        _sql = _sql || ' AND department = ''' || _department || '''';
    END IF;
    RETURN QUERY EXECUTE _sql;
END;
$$ LANGUAGE plpgsql;
```

是否有一种办法能不拼接查询字符串而得到同样的结果呢？在 SQL 语言中我们使用 `expression IS NULL` 来表示 *expression* 为空值。因此查询还未分配部门或部门为 IT 的员工的 SQL 语句为

```sql
SELECT * FROM employees WHERE (department IS NULL OR department = 'IT');
```

我们以此为前提改写 `fn_get_employees` 函数

```sql
CREATE OR REPLACE FUNCTION fn_get_employees(_department VARCHAR)
RETURNS SETOF employees
AS $$
BEGIN
    RETURN QUERY SELECT * FROM employees WHERE (_department IS NULL OR department = _department);
END;
$$ LANGUAGE plpgsql;
```

我们使用 `(yyy IS NULL OR xxx = yyy)` 这样模式来表达当参数 `yyy` 为空时查询所有的记录，否则查询字段 `xxx` 等于 `yyy` 的记录。除此之外还可以使用 `CASE WHEN ... THEN ... ELSE ... END` 表达式来实现相同的功能（前一种方式是这种方式的简化版）

```sql
CREATE OR REPLACE FUNCTION fn_get_employees(_department VARCHAR)
RETURNS SETOF employees
AS $$
BEGIN
    RETURN QUERY SELECT * FROM employees WHERE (CASE WHEN _department IS NULL THEN TRUE ELSE department = _department END);
END;
$$ LANGUAGE plpgsql;
```

完～
