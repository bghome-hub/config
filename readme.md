            DECLARE @sql nvarchar(MAX) = N'BULK INSERT #RawStaging
            FROM ' + QUOTENAME(@CsvPath, '''') + N'
            WITH (
                FIRSTROW = 1, 
                FIELDTERMINATOR = '','',
                ROWTERMINATOR = ''\n'',
                FIELDQUOTE = ''"''
            );';

			
