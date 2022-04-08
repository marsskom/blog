---
layout: post
title: Load CSV into MySQL
---

# Loading CSV into MySQL

> Disclaimer: All matches in the task and code with real cases are random!

## Table Of Contents

1. [Preconditions](#preconditions)
2. [The Task](#the-task)
3. [The MySQL schema](#the-mysql-schema)
4. [The Solutions](#the-solutions)


## Preconditions

A couple weeks ago I got the simple task.

Long story short, I **failed** 🔥 because I was concentrated 🤔 on the PHP code *more* instead on an **essence** of the task.

Of course, I'm not agree with a couple remarks but it doesn't matter now. I made conclusions for myself and it is the most important.

But enough for that. Let's go to the task 🎂.

## The Task

So, let's image you have a huge **csv** 📑 file with data. And you need to put the whole data into MySQL **table** 🪗.

Very simple, isn't it?

OK. Therefore, let's add a couple conditions.

- 🕐 Don't need to validate the data;
- 🕙 `Employee` has many `Accounts`, `Account` has many `Products`;
- 🕥 `Employee` has unique key;
- 🕚 `Account` has composite key that contains columns: `Employee ID`, `Account Date`, `Account Number`;
- 🕦 Products does not have any unique keys so having duplicates is OK;
- 🕛 Optimization (less memory, CPU etc) is on priority;

The **csv** file example :godmode: :

```csv
Employee ID,Employee Name,Account Date,Account Number,Product Name,Count,Product Code,Amount
188334196455537411,Employee #188334196455537411,2022-04-01,318120263190184361531275138,Bread,2,7834345,12.90
25038386429445678,Employee #25038386429445678,2022-04-04,127729800343966008685923545,Milk,1,1263445,8.45
25038386429445678,Employee #25038386429445678,2022-04-04,127729800343966008685923545,Cereal,1,601263425,25.16
1130797777898009571,Employee #1130797777898009571,2022-01-02,790532412213810154522128591,Ice Cream,10,5681,21.5
4521656741639988127,Employee #4521656741639988127,2022-02-03,137305493681158320196407475,Chicken,1,781469,32.52
...
```

Or in table format 😅 :

| Employee ID         | Employee Name                 | Account Date | Account Number              | Product Name | Count | Product Code | Amount |
|---------------------|-------------------------------|--------------|-----------------------------|--------------|-------|--------------|--------|
| 188334196455537411  | Employee #188334196455537411  | 2022-04-01   | 318120263190184361531275138 | Bread        | 2     | 7834345      | 12.90  |
| 25038386429445678   | Employee #25038386429445678   | 2022-04-04   | 127729800343966008685923545 | Milk         | 1     | 1263445      | 8.45   |
| 25038386429445678   | Employee #25038386429445678   | 2022-04-04   | 127729800343966008685923545 | Cereal       | 1     | 601263425    | 25.16  |
| 1130797777898009571 | Employee #1130797777898009571 | 2022-01-02   | 790532412213810154522128591 | Ice Cream    | 10    | 5681         | 21.5   |
| 4521656741639988127 | Employee #4521656741639988127 | 2022-02-03   | 137305493681158320196407475 | Chicken      | 1     | 781469       | 32.52  |


Will do you use the PHP? Maybe. 🙅‍♂️ *Or not?*


## The MySQL schema

Let's start from **schema** creation.

### Employees Table 👷

It is simple:

```sql
CREATE TABLE IF NOT EXISTS `employee` (
  `id` BIGINT(21) UNSIGNED NOT NULL,
  `name` VARCHAR(255),
  PRIMARY KEY (`id`)
) ENGINE = INNODB;
```

The `id` column is `BIGINT` type because all values is less than 20 digits.

### Accounts Table 🧻

```sql
CREATE TABLE IF NOT EXISTS `account` (
  `employee_id` BIGINT(21) UNSIGNED NOT NULL,
  `date` DATE NOT NULL,
  `number` CHAR(27) NOT NULL,
  PRIMARY KEY (`employee_id`, `date`, `number`),
  FOREIGN KEY (`employee_id`) REFERENCES `employee` (`id`) ON UPDATE CASCADE ON DELETE CASCADE
) ENGINE = INNODB;
```

Account `number` always contains `27` chars.

### Products Table 🥖

```sql
CREATE TABLE IF NOT EXISTS `product` (
  `employee_id` BIGINT(21) UNSIGNED NOT NULL,
  `account_date` DATE NOT NULL,
  `account_number` CHAR(27) NOT NULL,
  `code` INT(11) NOT NULL,
  `name` VARCHAR(255),
  `count` INT(11) NOT NULL,
  `amount` DECIMAL(20, 2),
  FOREIGN KEY (`employee_id`, `account_date`, `account_number`) REFERENCES `account` (`employee_id`, `date`, `number`) ON UPDATE CASCADE ON DELETE CASCADE
) ENGINE = INNODB;
```

The `count` is always integer.

Maybe you have a question why I use `Account` composite key as foreign key inside `Product` table. It is because the task is concluded in **how to more efficiently load file into database**.

Data is not so big for duplication in two tables, I think. Moreover you is free to use `INSERT IGNORE` (batch mode) without `SELECT` for account id inside and don't store primary key in PHP variable (remember about memory).

## The Solutions

🧗‍♀️

### Pure PHP 🧰

One of the condition was **"to use pure PHP"**. This point is about it approach.

OK, we have [PDO][PDO] and [Generators][Generators].

How to read file with Generators:

```php
/**
 * {@inheritdoc}
 * @throws CannotOpenFileException
 */
public function read(): Generator
{
    $this->file = fopen($this->pathToFile, "rb");

    if (!$this->file) {
        throw new CannotOpenFileException(
            sprintf(
                'Cannot open file "%s"',
                $this->pathToFile
            )
        );
    }

    while (false !== ($lineData = fgetcsv($this->file))) {
        yield $lineData;
    }

    $this->close();
}
```

Then you should assembly the big `INSERT` query thinking about query length (execute query after 500 read lines or something similar) and execute whole query until the end of the **csv** file.

### Pure MySQL 🪛

What will be when I say to you that you can use **MySQL** only ❓

Yeah, it is more cooler, doesn't it ❓

#### Data As Is 💆

Let's start from **schema** again and create table that contains the columns exactly the same as in the **csv** file:

```sql
CREATE TABLE IF NOT EXISTS `file_data` (
  `Employee ID` BIGINT(21) UNSIGNED NOT NULL,
  `Employee Name` VARCHAR(255),
  `Account Date` DATE NOT NULL,
  `Account Number` CHAR(27) NOT NULL,
  `Product Code` INT(11) NOT NULL,
  `Product Name` VARCHAR(255),
  `Count` INT(11) NOT NULL,
  `Amount` DECIMAL(20, 2)
) ENGINE = INNODB;
```

Because of the condition **use PHP** we only need this code:

```php
$filename = __DIR__ . DIRECTORY_SEPARATOR . 'data.csv';

// Clear old data from table.
$pdo->exec("TRUNCATE `file_data`;");

// Load from file.
$pdo->exec("
    LOAD DATA LOCAL INFILE '$filename'
    INTO TABLE `file_data`
    FIELDS TERMINATED BY ',' ENCLOSED BY '\"'
    LINES TERMINATED BY '\n'
    IGNORE 1 ROWS;
");
```

Definitely, you should create PDO object with `PDO::MYSQL_ATTR_LOCAL_INFILE => true` options:

```php
new PDO(
    "mysql:host=$host;dbname=$name",
    $user,
    $password,
    [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_EMULATE_PREPARES   => false,
        PDO::MYSQL_ATTR_LOCAL_INFILE => true,
    ]
);
```

> Of course, you can run `LOAD DATA LOCAL INFILE` command under the **MySQL** client as well. But here we work with PHP.

As you can guess it is not enough. The code above just fill the `file_data` table, but we need data in our tables.

OK 👍

Let's create [the trigger][MySQL-Triggers] 🥱 :

```sql
CREATE TRIGGER `file_data_trigger`
    AFTER INSERT
    ON `file_data`
    FOR EACH ROW
BEGIN
    INSERT IGNORE INTO `employee`(`id`, `name`) VALUES (new.`Employee ID`, new.`Employee Name`);

    INSERT IGNORE INTO `account`(`employee_id`, `date`, `number`)
    VALUES (new.`Employee ID`, new.`Account Date`, new.`Account Number`);
    
    INSERT IGNORE INTO `product` (
      `employee_id`,
      `account_date`,
      `account_number`,
      `code`,
      `name`,
      `count`,
      `amount`
    ) VALUES (
      new.`Employee ID`,
      new.`Account Date`,
      new.`Account Number`,
      new.`Product Code`,
      new.`Product Name`,
      new.`Count`,
      new.`Amount`
    );
END;
```

That's why our script truncate `file_data` table before execution so always keeps table clean ⚛️ .

#### `Account` Primary Key 🛫

In the other hand, if you add the primary key into `Account` table:

```sql
CREATE TABLE IF NOT EXISTS `account` (
  `id` BIGINT(21) UNSIGNED NOT NULL AUTO_INCREMENT,
  `employee_id` BIGINT(21) UNSIGNED NOT NULL,
  `date` DATE NOT NULL,
  `number` CHAR(27) NOT NULL,
  PRIMARY KEY (`id`),
  FOREIGN KEY (`employee_id`) REFERENCES `employee` (`id`) ON UPDATE CASCADE ON DELETE CASCADE
) ENGINE = INNODB;
```

And update the `Product` table:

```sql
CREATE TABLE IF NOT EXISTS `product` (
  `account_id` BIGINT(21) UNSIGNED NOT NULL,
  `code` INT(11) NOT NULL,
  `name` VARCHAR(255),
  `count` INT(11) NOT NULL,
  `amount` DECIMAL(20, 2),
  FOREIGN KEY (`account_id`) REFERENCES `account` (`id`) ON UPDATE CASCADE ON DELETE CASCADE
) ENGINE = INNODB;
```

Therefore just update the trigger:

```sql
CREATE TRIGGER `file_data_trigger`
    AFTER INSERT
    ON `file_data`
    FOR EACH ROW
BEGIN
    INSERT IGNORE INTO `employee`(`id`, `name`) VALUES (new.`Employee ID`, new.`Employee Name`);

    INSERT IGNORE INTO `account`(`employee_id`, `date`, `number`)
    VALUES (new.`Employee ID`, new.`Account Date`, new.`Account Number`);
    
    INSERT IGNORE INTO `product` (
      `account_id`,
      `code`,
      `name`,
      `count`,
      `amount`
    ) VALUES (
      (
          SELECT `account`.`id`
          FROM `account`
          WHERE `account`.`employee_id` = new.`Employee ID`
            AND `account`.`date` = new.`Account Date`
            AND `account`.`number` = new.`Account Number`
          LIMIT 1
      ),
      new.`Product Code`,
      new.`Product Name`,
      new.`Count`,
      new.`Amount`
    );
END;
```

💣 Interesting solution that's why I decided keep it in my blog. 💣





[PDO]: https://www.php.net/manual/en/book.pdo.php
[Generators]: https://www.php.net/manual/en/language.generators.php
[MySQL-Triggers]: https://dev.mysql.com/doc/refman/8.0/en/triggers.html
