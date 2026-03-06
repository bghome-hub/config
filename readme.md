-- Clean start
DROP TABLE IF EXISTS dbo.IC_Line;
DROP TABLE IF EXISTS dbo.IC_Run;
DROP TABLE IF EXISTS dbo.IC_Log;
DROP TABLE IF EXISTS dbo.IC_FileStage;
DROP TABLE IF EXISTS dbo.IC_File;

CREATE TABLE dbo.IC_File (
    FileID UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT (NEWID()),
    FileName NVARCHAR(260) NOT NULL,
    Status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    ErrorMessage VARCHAR(4000) NULL,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE()
);

CREATE TABLE dbo.IC_FileStage (
    Company NVARCHAR (MAX) NULL,
    Account NVARCHAR (MAX) NULL,
    AcctDesc NVARCHAR (MAX) NULL,
    Amount NVARCHAR (MAX) NULL,
    TrxDate NVARCHAR (MAX) NULL,
    Reference NVARCHAR (MAX) NULL,
    Reversal_Flag NVARCHAR(MAX) NULL
);

CREATE TABLE dbo.IC_Run (
    RunID UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT (NEWID()),
    FileID UNIQUEIDENTIFIER NOT NULL REFERENCES dbo.IC_File(FileID),
    Company VARCHAR(10) NOT NULL,
    BatchID CHAR(15) NULL,
    LineCount INT NOT NULL DEFAULT 0,
    DebitTotal NUMERIC(19,5) NOT NULL DEFAULT 0,
    CreditTotal NUMERIC(19,5) NOT NULL DEFAULT 0,
    Status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    ErrorMessage VARCHAR(MAX) NULL,
    CreatedDatetime DATETIME NOT NULL DEFAULT GETDATE()
);

CREATE TABLE dbo.IC_Line (
    LineID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    RunID UNIQUEIDENTIFIER NOT NULL REFERENCES dbo.IC_Run(RunID) ON DELETE CASCADE,
    Company VARCHAR(10) NOT NULL,
    Account VARCHAR(75) NOT NULL, 
    Amount NUMERIC(19,5) NOT NULL, 
    TrxDate DATE NOT NULL, 
    Reference VARCHAR(31) NULL,
    Reversal CHAR(1) NOT NULL
);

CREATE TABLE dbo.IC_Log (
    LogID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    TargetID UNIQUEIDENTIFIER NOT NULL,
    LogLevel VARCHAR(20) NOT NULL DEFAULT 'INFO',
    StepName VARCHAR(50) NOT NULL,
    LogMessage VARCHAR(MAX) NULL,
    LogDateTime DATETIME NOT NULL DEFAULT (GETDATE())
);

IF NOT EXISTS (SELECT * FROM sys.sequences WHERE name = 'ic_batch_seq')
    CREATE SEQUENCE [DYNAMICS].[dbo].[ic_batch_seq] AS BIGINT START WITH 1000 INCREMENT BY 1;


CREATE PROCEDURE dbo.usp_IC_Log
    @TargetID UNIQUEIDENTIFIER,
    @Step NVARCHAR(50),
    @LogLevel NVARCHAR(20),
    @LogMessage NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.IC_Log (TargetID, StepName, LogLevel, LogMessage)
    VALUES (@TargetID, @Step, UPPER(@LogLevel), @LogMessage);
END
GO

CREATE PROCEDURE dbo.usp_IC_GetNextBatchID_Seq
    @SeriesPrefix VARCHAR(8) = 'GLTX',
    @PadWidth INT = 8,
    @BatchID CHAR(15) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @n BIGINT = NEXT VALUE FOR DYNAMICS.dbo.ic_batch_seq;
    DECLARE @suffix VARCHAR(32) = RIGHT(REPLICATE('0', @PadWidth) + CAST(@n AS VARCHAR(32)), @PadWidth);
    SET @BatchID = LEFT(@SeriesPrefix + @suffix, 15);
END
GO


CREATE PROCEDURE dbo.usp_IC_ValidateFile
    @CsvPath NVARCHAR(4000),
    @FileID UNIQUEIDENTIFIER
AS  
BEGIN 
    SET NOCOUNT ON;
    DECLARE @ErrorMsg VARCHAR(4000), @Step NVARCHAR(50) = 'VALIDATE_FILE', @sql NVARCHAR(MAX);
    DECLARE @ExpectedHeader NVARCHAR(MAX) = 'Company,Account,AcctDesc,Amount,Date,Reference,Reversal';
    DECLARE @ActualHeader NVARCHAR(MAX);

    DROP TABLE IF EXISTS #HeaderCheck;
    CREATE TABLE #HeaderCheck (RawRow VARCHAR(MAX));

    SET @sql = N'BULK INSERT #HeaderCheck FROM ' + QUOTENAME(@CsvPath,'''') + N' WITH (FIRSTROW = 1, LASTROW = 1, ROWTERMINATOR = ''0x0d0a'');';
    EXEC(@sql);

    SELECT TOP 1 @ActualHeader = LTRIM(RTRIM(RawRow)) FROM #HeaderCheck;
    IF @ActualHeader <> @ExpectedHeader
    BEGIN
        SET @ErrorMsg = CONCAT('Header Mismatch. Found: ', ISNULL(@ActualHeader,'NoHeader'));
        THROW 54001, @ErrorMsg, 1;
    END

    TRUNCATE TABLE dbo.IC_FileStage;
    SET @sql = N'BULK INSERT dbo.IC_FileStage FROM ' + QUOTENAME(@CsvPath,'''') + N' WITH (FIRSTROW = 2, FIELDTERMINATOR = '','', ROWTERMINATOR = ''0x0d0a'', FIELDQUOTE=''"'');';
    EXEC(@sql);

    IF EXISTS(SELECT 1 FROM dbo.IC_FileStage WHERE TRY_CONVERT(NUMERIC(19,5), Amount) IS NULL)
        THROW 54002, 'File Rejected: Amount column contains invalid characters.', 1;

    IF EXISTS(SELECT 1 FROM dbo.IC_FileStage WHERE UPPER(LTRIM(RTRIM(Reversal_Flag))) NOT IN ('Y','N'))
        THROW 54003, 'File Rejected: Reversal Flag must be Y or N.', 1;
END
GO

CREATE PROCEDURE dbo.usp_IC_Process_Staging
    @FileID UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Co VARCHAR(10), @RunID UNIQUEIDENTIFIER, @BatchID CHAR(15);

    DECLARE CoCursor CURSOR FOR SELECT DISTINCT Company FROM dbo.IC_FileStage;
    OPEN CoCursor; FETCH NEXT FROM CoCursor INTO @Co;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @RunID = NEWID();
        EXEC dbo.usp_IC_GetNextBatchID_Seq @BatchID = @BatchID OUTPUT;

        INSERT INTO dbo.IC_Run (RunID, FileID, Company, BatchID, Status)
        VALUES (@RunID, @FileID, @Co, @BatchID, 'PENDING');

        INSERT INTO dbo.IC_Line (RunID, Company, Account, Amount, TrxDate, Reference, Reversal)
        SELECT @RunID, Company, Account, CAST(Amount AS NUMERIC(19,5)), CAST(TrxDate AS DATE), Reference, LEFT(Reversal_Flag, 1)
        FROM dbo.IC_FileStage WHERE Company = @Co;

        UPDATE R SET LineCount = L.Cnt, DebitTotal = L.Dr, CreditTotal = L.Cr
        FROM dbo.IC_Run R CROSS APPLY (
            SELECT COUNT(*) AS Cnt, SUM(CASE WHEN Amount > 0 THEN Amount ELSE 0 END) AS Dr, ABS(SUM(CASE WHEN Amount < 0 THEN Amount ELSE 0 END)) AS Cr
            FROM dbo.IC_Line WHERE RunID = @RunID
        ) L WHERE R.RunID = @RunID;

        FETCH NEXT FROM CoCursor INTO @Co;
    END
    CLOSE CoCursor; DEALLOCATE CoCursor;
END
GO

CREATE PROCEDURE dbo.usp_IC_ValidateRun
    @RunID UNIQUEIDENTIFIER
AS 
BEGIN
    SET NOCOUNT ON;
    DECLARE @ErrorMsg VARCHAR(4000), @Step VARCHAR(50) = 'VALIDATE_DATA';
    DECLARE @Company VARCHAR(10), @TrxDate DATE, @RevFlag CHAR(1), @IsOpen BIT = 0, @sql NVARCHAR(MAX);

    SELECT @Company = Company, @TrxDate = TrxDate, @RevFlag = Reversal FROM dbo.IC_Line WHERE RunID = @RunID;

    DECLARE @net NUMERIC(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID = @RunID);
    IF ABS(@net) > 0.00001
        THROW 54012, 'Accounting Error: Transaction is out of balance.', 1;

    SET @sql = N'SELECT @o_Open = 1 FROM ' + QUOTENAME(@Company) + N'.dbo.SY40100 WHERE @p_TrxDate BETWEEN PERIODDT AND PERDENDT AND CLOSED = 0 AND SERIES = 2 AND PERIODID <> 0';
    EXEC sp_executesql @sql, N'@p_TrxDate DATE, @o_Open BIT OUTPUT', @p_TrxDate = @TrxDate, @o_Open = @IsOpen OUTPUT;

    IF ISNULL(@IsOpen, 0) = 0
        THROW 54014, 'Accounting Error: Financial Period is closed.', 1;
END
GO



CREATE PROCEDURE dbo.usp_IC_Process_File
    @CsvPath NVARCHAR(4000)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @FileID UNIQUEIDENTIFIER = NEWID(), @Step NVARCHAR(50) = 'FILE_PROCESS', @ErrorMsg NVARCHAR(MAX);
    DECLARE @FileName NVARCHAR(260) = REVERSE(LEFT(REVERSE(@CsvPath), CHARINDEX('\', REVERSE(@CsvPath) + '\') - 1));

    INSERT INTO dbo.IC_File (FileID, FileName, Status) VALUES (@FileID, @FileName, 'PROCESSING');

    BEGIN TRY
        EXEC dbo.usp_IC_ValidateFile @CsvPath = @CsvPath, @FileID = @FileID;
        EXEC dbo.usp_IC_Process_Staging @FileID = @FileID;

        DECLARE @RunID UNIQUEIDENTIFIER;
        DECLARE RunCursor CURSOR FOR SELECT RunID FROM dbo.IC_Run WHERE FileID = @FileID;
        OPEN RunCursor; FETCH NEXT FROM RunCursor INTO @RunID;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            BEGIN TRY
                EXEC dbo.usp_IC_ValidateRun @RunID = @RunID;
                UPDATE dbo.IC_Run SET Status = 'VALIDATED' WHERE RunID = @RunID;
            END TRY
            BEGIN CATCH
                SET @ErrorMsg = ERROR_MESSAGE();
                UPDATE dbo.IC_Run SET Status = 'FAILED', ErrorMessage = @ErrorMsg WHERE RunID = @RunID;
                EXEC dbo.usp_IC_Log @RunID, 'VALIDATE_RUN', 'ERROR', @ErrorMsg;
            END CATCH
            FETCH NEXT FROM RunCursor INTO @RunID;
        END
        CLOSE RunCursor; DEALLOCATE RunCursor;

        UPDATE dbo.IC_File SET Status = 'COMPLETED' WHERE FileID = @FileID;
        SET @ErrorMsg = CONCAT('Process completed for ', @FileName);
        EXEC dbo.usp_IC_Log @FileID, @Step, 'INFO', @ErrorMsg;
    END TRY
    BEGIN CATCH
        SET @ErrorMsg = ERROR_MESSAGE();
        UPDATE dbo.IC_File SET Status = 'ERROR', ErrorMessage = @ErrorMsg WHERE FileID = @FileID;
        EXEC dbo.usp_IC_Log @FileID, @Step, 'ERROR', @ErrorMsg;
    END CATCH

    SELECT @FileID AS GeneratedFileID;
END
GO
