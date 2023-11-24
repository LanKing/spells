# Fill table with random data

Detects table structure and fills all columns with random values in user's defined range. Make as many rows as user choosed.

_@todo: Что можно доработать: для Float полей генерация дробного числа, для строкового - текстового хеша, для bool - двоичного варианта, также возможность указания статических данных для отдельных колонок._


## Call example (with table creating)

```mysql
CREATE TABLE IF NOT EXISTS `performance_testing` ( 
  `a` VARCHAR(255) NULL DEFAULT 'NULL' ,
  `b` VARCHAR(255) NULL DEFAULT 'NULL' ,
  `c` VARCHAR(255) NULL DEFAULT 'NULL' ,
  `d` VARCHAR(255) NULL DEFAULT 'NULL' 
);

TRUNCATE TABLE `performance_testing`;
CALL SeedTableWithRandomValues('performance_testing', 10, 1, 100);
```


## Seed procedure

```mysql
DROP PROCEDURE IF EXISTS SeedTableWithRandomValues;

DELIMITER //

CREATE PROCEDURE SeedTableWithRandomValues(
    IN TableName VARCHAR(255),
    IN NumRows INT,
    IN MinVal INT,
    IN MaxVal INT
)
BEGIN
    DECLARE i INT DEFAULT 1;
    
    DECLARE columns_count INT; 
    DECLARE columns_create_list VARCHAR(1000);
    
    DECLARE sql_query VARCHAR(5000);
    DECLARE fill_row_sql VARCHAR(5000);

    
    -- Get source table colums information
    SET columns_create_list = '';
    
    SELECT 
        COUNT(*),
        GROUP_CONCAT(
            DISTINCT
            CASE
                WHEN data_type IN ('INT', 'DECIMAL') THEN 
                    CONCAT(column_name, ' ', data_type)
                ELSE
                    CONCAT(column_name, ' VARCHAR(255)')
            END
        , '' SEPARATOR ', '),
        
        GROUP_CONCAT(
            CASE
                WHEN data_type IN ('INT', 'DECIMAL') THEN 
                    CONCAT('CAST(FLOOR(RAND() * (', MaxVal - MinVal + 1, ') + ', MinVal, ') AS INT)')
                ELSE
                    CONCAT('CAST(FLOOR(RAND() * (', MaxVal - MinVal + 1, ') + ', MinVal, ') AS CHAR)')
            END
            
        , '' SEPARATOR ', ')
    INTO columns_count, columns_create_list, fill_row_sql
    FROM information_schema.columns
    WHERE table_name = TableName;
    
    
    -- Re/Create temporary table for new values
    DROP TEMPORARY TABLE IF EXISTS tmpSeedTableWithRandomValuesCache;
    SET sql_query = CONCAT('CREATE TEMPORARY TABLE tmpSeedTableWithRandomValuesCache (', columns_create_list, ') ENGINE=MEMORY');
    PREPARE stmt FROM sql_query;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    

    -- Fill temporary table with random values
    SET sql_query = CONCAT('INSERT INTO tmpSeedTableWithRandomValuesCache VALUES (', fill_row_sql, ')');

    SET i = 1;
    WHILE i <= NumRows DO
        PREPARE stmt FROM sql_query;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
        SET i = i + 1;
    END WHILE;
    
    -- Insert data to destination table
    SET sql_query = CONCAT('INSERT INTO ', TableName, ' SELECT * FROM tmpSeedTableWithRandomValuesCache');
    PREPARE stmt FROM sql_query;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    DROP TEMPORARY TABLE IF EXISTS tmpSeedTableWithRandomValuesCache;

    -- Results output
    SELECT CONCAT('Table `', TableName, '` was seeded by ', NumRows, ' rows with random values (from ', MinVal, ' to ', MaxVal, ')') AS Result;

END //

DELIMITER ;
```
