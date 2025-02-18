# Lab Submission

**Note:** There is a difference between data and information. The procedure, function, and view must all make an effort to process the data into information.

The procedure and function must include an IF statement or any other similar control flow function.

## The Procedure

DELIMITER //

CREATE PROCEDURE CalculateMishandlingStats(IN reportYear INT)
BEGIN
    DECLARE meanRate DECIMAL(10,2);
    DECLARE stdDevRate DECIMAL(10,2);
    DECLARE totalMonths INT;
    
    -- Step 1: Calculate mean mishandling rate
    SELECT AVG((mishandledBaggage / totalBaggage) * 100)
    INTO meanRate
    FROM (
        SELECT MONTH(checked_in_date) AS Month,
               COUNT(*) AS totalBaggage,
               SUM(CASE WHEN status = 'Mishandled' THEN 1 ELSE 0 END) AS mishandledBaggage
        FROM baggage
        WHERE YEAR(checked_in_date) = reportYear
        GROUP BY Month
    ) AS MonthlyData;

    -- Step 2: Calculate standard deviation of mishandling rate
    SELECT COUNT(*) INTO totalMonths FROM (
        SELECT DISTINCT MONTH(checked_in_date) FROM baggage WHERE YEAR(checked_in_date) = reportYear
    ) AS MonthCount;

    SELECT SQRT(SUM(POWER(((mishandledBaggage / totalBaggage) * 100) - meanRate, 2)) / totalMonths)
    INTO stdDevRate
    FROM (
        SELECT MONTH(checked_in_date) AS Month,
               COUNT(*) AS totalBaggage,
               SUM(CASE WHEN status = 'Mishandled' THEN 1 ELSE 0 END) AS mishandledBaggage
        FROM baggage
        WHERE YEAR(checked_in_date) = reportYear
        GROUP BY Month
    ) AS MonthlyData;

    -- Output results
    SELECT reportYear AS Year, meanRate AS Mean_Mishandling_Rate, stdDevRate AS Mishandling_StdDev;
END //

DELIMITER ;


## The Function

DELIMITER //

CREATE FUNCTION GetMishandlingZScore(month INT, year INT)
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE meanRate DECIMAL(10,2);
    DECLARE stdDevRate DECIMAL(10,2);
    DECLARE monthRate DECIMAL(10,2);
    DECLARE zScore DECIMAL(10,2);

    -- Get mean and standard deviation from the stored procedure calculation
    SELECT AVG((mishandledBaggage / totalBaggage) * 100),
           SQRT(SUM(POWER(((mishandledBaggage / totalBaggage) * 100) - meanRate, 2)) / COUNT(*))
    INTO meanRate, stdDevRate
    FROM (
        SELECT MONTH(checked_in_date) AS Month,
               COUNT(*) AS totalBaggage,
               SUM(CASE WHEN status = 'Mishandled' THEN 1 ELSE 0 END) AS mishandledBaggage
        FROM baggage
        WHERE YEAR(checked_in_date) = year
        GROUP BY Month
    ) AS MonthlyData;

    -- Get the mishandling rate for the given month
    SELECT (SUM(CASE WHEN status = 'Mishandled' THEN 1 ELSE 0 END) / COUNT(*)) * 100
    INTO monthRate
    FROM baggage
    WHERE MONTH(checked_in_date) = month AND YEAR(checked_in_date) = year;

    -- Compute Z-score
    IF stdDevRate > 0 THEN
        SET zScore = (monthRate - meanRate) / stdDevRate;
    ELSE
        SET zScore = 0;
    END IF;

    RETURN zScore;
END //

DELIMITER ;


## The View that makes use of the Function

CREATE VIEW BaggageMishandlingAnalysis AS
SELECT 
    b.Month, 
    b.Year, 
    b.Mishandling_Rate,
    (LAG(b.Mishandling_Rate, 1) OVER (ORDER BY b.Year, b.Month) +
     b.Mishandling_Rate +
     LEAD(b.Mishandling_Rate, 1) OVER (ORDER BY b.Year, b.Month)) / 3 AS Moving_Average_3Months,
    GetMishandlingZScore(b.Month, b.Year) AS Z_Score
FROM (
    SELECT MONTH(checked_in_date) AS Month, YEAR(checked_in_date) AS Year,
           (SUM(CASE WHEN status = 'Mishandled' THEN 1 ELSE 0 END) / COUNT(*)) * 100 AS Mishandling_Rate
    FROM baggage
    GROUP BY Year, Month
) AS b;


## A User to Access only the View
CREATE USER 'analytics_user'@'localhost' IDENTIFIED BY 'securepassword';

GRANT SELECT ON BaggageMishandlingAnalysis TO 'analytics_user'@'localhost';

FLUSH PRIVILEGES;

