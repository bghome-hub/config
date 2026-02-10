USE DYNAMICS;
GO

/* =============================================
   1. CLEANUP & RESET
   ============================================= */
-- Procedures
IF OBJECT_ID('dbo.usp_IC_File_Process', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_File_Process;
IF OBJECT_ID('dbo.usp_IC_PostToGP', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_PostToGP;
IF OBJECT_ID('dbo.usp_IC_BuildAutoLines', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_BuildAutoLines;
IF OBJECT_ID('dbo.usp_IC_GetDueAccounts', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_GetDueAccounts;
IF OBJECT_ID('dbo.usp_IC_ValidateRun', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_ValidateRun;
IF OBJECT_ID('dbo.usp_IC_Normalize', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_Normalize;
IF OBJECT_ID('dbo.usp_IC_LoadCsv', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_LoadCsv;
IF OBJECT_ID('dbo.usp_IC_Log', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_Log;

-- Tables
IF OBJECT_ID('dbo.IC_RunEvent', 'U') IS NOT NULL DROP TABLE dbo.IC_RunEvent;
IF OBJECT_ID('dbo.IC_AutoLine', 'U') IS NOT NULL DROP TABLE dbo.IC_AutoLine;
IF OBJECT_ID('dbo.IC_Line', 'U') IS NOT NULL DROP TABLE dbo.IC_Line;
IF OBJECT_ID('dbo.IC_RawCsv', 'U') IS NOT NULL DROP TABLE dbo.IC_RawCsv;
IF OBJECT_ID('dbo.IC_Run', 'U') IS NOT NULL DROP TABLE dbo.IC_Run;
GO

/* =============================================
   2. TABLES
   ============================================= */

-- 2.1 Run Header (Merged with later ALTER additions)
CREATE TABLE dbo.IC_Run
(
    RunID           uniqueidentifier NOT NULL PRIMARY KEY,
    FileName        nvarchar(260) NOT NULL,
    FilePath        nvarchar(4000) NOT NULL,
    BatchID         char(15) NOT NULL,
    Reference       varchar(30) NOT NULL,
    Status          varchar(20) NOT NULL DEFAULT 'LOADED', -- LOADED/POSTED/FAILED
    StartedDT       datetime NOT NULL DEFAULT GETDATE(),
    EndedDT         datetime NULL,
    Message         varchar(4000) NULL,
    -- Enhanced Reporting Columns
    RowCountRaw     int NULL,
    RowCountLine    int NULL,
    CompanyA        varchar(10) NULL,
    CompanyB        varchar(10) NULL,
    NetAmount       numeric(19,5) NULL
);
GO

-- 2.2 Logging Table
CREATE TABLE dbo.IC_RunEvent
(
    EventID     int IDENTITY(1,1) PRIMARY KEY,
    RunID       uniqueidentifier NOT NULL,
    EventDT     datetime NOT NULL DEFAULT GETDATE(),
    StepName    varchar(50) NOT NULL,
    Message     varchar(4000) NOT NULL
);
GO

-- 2.3 Raw CSV Staging (Strings only for BULK INSERT)
CREATE TABLE dbo.IC_RawCsv
(
    Company     varchar(20)  NULL,
    Account     varchar(129) NULL,
    AcctDesc    varchar(255) NULL,
    Amount      varchar(50)  NULL,
    TrxDate     varchar(30)  NULL
);
GO

-- 2.4 Typed Staging Lines
CREATE TABLE dbo.IC_Line
(
    LineID      int IDENTITY(1,1) PRIMARY KEY,
    RunID       uniqueidentifier NOT NULL,
    TrxDate     date NOT NULL,
    CompanyID   varchar(10) NOT NULL,
    Account     char(129) NOT NULL,
    AcctDesc    varchar(255) NULL,
    Amount      numeric(19,5) NOT NULL,
    Debit       numeric(19,5) NOT NULL DEFAULT 0,
    Credit      numeric(19,5) NOT NULL DEFAULT 0
);
GO

-- 2.5 Auto-Balancing Lines (Due To/From)
CREATE TABLE dbo.IC_AutoLine
(
    AutoID       int IDENTITY(1,1) PRIMARY KEY,
    RunID        uniqueidentifier NOT NULL,
    TrxDate      date NOT NULL,
    CompanyID    varchar(10) NOT NULL,
    Account      char(129) NOT NULL,
    Debit        numeric(19,5) NOT NULL DEFAULT 0,
    Credit       numeric(19,5) NOT NULL DEFAULT 0,
    OtherCompany varchar(10) NOT NULL
);
GO

/* =============================================
   3. STORED PROCEDURES
   ============================================= */

-- 3.1 Logger Helper
CREATE PROCEDURE dbo.usp_IC_Log
(
    @RunID uniqueidentifier,
    @StepName varchar(50),
    @Message varchar(4000)
)
AS
BEGIN
    INSERT dbo.IC_RunEvent (RunID, StepName, Message)
    VALUES (@RunID, @StepName, @Message);
END
GO

-- 3.2 Load CSV (Bulk Insert)
CREATE PROCEDURE dbo.usp_IC_LoadCsv
(
    @CsvPath   nvarchar(4000),
    @BatchID   char(15),
    @Reference varchar(30),
    @RunID     uniqueidentifier OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;
    SET @RunID = NEWID();

    DECLARE @FileName nvarchar(260) = RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);
    
    INSERT dbo.IC_Run (RunID, FileName, FilePath, BatchID, Reference, Status)
    VALUES (@RunID, @FileName, @CsvPath, @BatchID, @Reference, 'LOADING');

    DECLARE @msg NVARCHAR(200) = CONCAT('Starting load: ', @FileName);
    EXEC dbo.usp_IC_Log @RunID, 'LOAD', @msg;

    -- Clear staging
    TRUNCATE TABLE dbo.IC_RawCsv;

    -- Dynamic SQL for BULK INSERT
    DECLARE @sql nvarchar(max) = N'
    BULK INSERT DYNAMICS.dbo.IC_RawCsv
    FROM ' + QUOTENAME(@CsvPath,'''') + N'
    WITH (
        FIRSTROW = 2,
        FIELDTERMINATOR = '','',
        ROWTERMINATOR = ''0x0d0a''
    );';
    EXEC (@sql);

    -- Log Raw Count
    DECLARE @rawCount int = (SELECT COUNT(*) FROM dbo.IC_RawCsv);
    UPDATE dbo.IC_Run SET RowCountRaw=@rawCount WHERE RunID=@RunID;
    
    -- Transform to Typed Table
    INSERT dbo.IC_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
    SELECT
        @RunID,
        CONVERT(date, TrxDate),
        RTRIM(LTRIM(Company)),
        CONVERT(char(129), RTRIM(LTRIM(Account))),
        AcctDesc,
        CONVERT(numeric(19,5), REPLACE(Amount, ',', ''))
    FROM dbo.IC_RawCsv;

    -- Cleanup zero rows
    DELETE dbo.IC_Line WHERE RunID=@RunID AND Amount=0;

    -- Log Line Count
    DECLARE @lineCount int = (SELECT COUNT(*) FROM dbo.IC_Line WHERE RunID=@RunID);
    UPDATE dbo.IC_Run SET RowCountLine=@lineCount, Status='LOADED' WHERE RunID=@RunID;
    
    SET @msg = CONCAT('Staged nonzero lines: ', @lineCount);
    EXEC dbo.usp_IC_Log @RunID, 'LOAD', @msg;
END
GO

-- 3.3 Normalize (Signed Amount -> Debit/Credit)
CREATE PROCEDURE dbo.usp_IC_Normalize
(
    @RunID uniqueidentifier
)
AS
BEGIN
    UPDATE dbo.IC_Line
    SET
      Debit  = CASE WHEN Amount > 0 THEN Amount ELSE 0 END,
      Credit = CASE WHEN Amount < 0 THEN ABS(Amount) ELSE 0 END
    WHERE RunID=@RunID;
    
    EXEC dbo.usp_IC_Log @RunID, 'NORMALIZE', 'Converted Amount into Debit/Credit';
END
GO

-- 3.4 Validate Run (Business Rules)
CREATE PROCEDURE dbo.usp_IC_ValidateRun
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Rule 1: Rows must exist
    IF (SELECT COUNT(*) FROM dbo.IC_Line WHERE RunID=@RunID) = 0
        THROW 52001, 'No staged lines found for this run (after removing zeros).', 1;

    -- Rule 2: Exactly 2 distinct companies
    DECLARE @c1 varchar(10), @c2 varchar(10);
    SELECT TOP 1 @c1 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID ORDER BY CompanyID;
    SELECT TOP 1 @c2 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID<>@c1 ORDER BY CompanyID;
    
    UPDATE dbo.IC_Run SET CompanyA=@c1, CompanyB=@c2 WHERE RunID=@RunID;

    IF (SELECT COUNT(DISTINCT CompanyID) FROM dbo.IC_Line WHERE RunID=@RunID) <> 2
        THROW 52002, 'File must contain exactly 2 distinct companies.', 1;

    -- Rule 3: File must balance overall
    DECLARE @net numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID=@RunID);
    UPDATE dbo.IC_Run SET NetAmount=@net WHERE RunID=@RunID;
    
    IF ABS(@net) > 0.00001
        THROW 52003, 'File does not balance overall (SUM(Amount) <> 0).', 1;

    -- Rule 4: IC Relationship must exist in IC40100
    IF NOT EXISTS (
        SELECT 1 FROM DYNAMICS..IC40100
        WHERE (ORCOMID=LEFT(@c1,5) AND DSTCOMID=LEFT(@c2,5))
           OR (ORCOMID=LEFT(@c2,5) AND DSTCOMID=LEFT(@c1,5))
    )
        THROW 52004, 'No intercompany relationship exists in IC40100 for these companies.', 1;

    DECLARE @msg NVARCHAR(1000) = CONCAT('Validated: ', @c1, ' / ', @c2, ' Net: ', @net);
    EXEC dbo.usp_IC_Log @RunID, 'VALIDATE', @msg;
END
GO

-- 3.5 Helper: Get Due To/From Accounts from GP Logic
CREATE PROCEDURE dbo.usp_IC_GetDueAccounts
(
    @CoA varchar(10),
    @CoB varchar(10),
    @CoA_DueTo   char(129) OUTPUT,
    @CoA_DueFrom char(129) OUTPUT,
    @CoB_DueTo   char(129) OUTPUT,
    @CoB_DueFrom char(129) OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @ORCOMID char(5), @DSTCOMID char(5),
            @ORGFRIDX int, @ORGTOIDX int, @DSTFRIDX int, @DSTTOIDX int;

    -- Retrieve relationship indexes from IC40100
    SELECT TOP 1
      @ORCOMID=ORCOMID, @DSTCOMID=DSTCOMID,
      @ORGFRIDX=ORGFRIDX, @ORGTOIDX=ORGTOIDX,
      @DSTFRIDX=DSTFRIDX, @DSTTOIDX=DSTTOIDX
    FROM DYNAMICS..IC40100
    WHERE (ORCOMID=LEFT(@CoA,5) AND DSTCOMID=LEFT(@CoB,5))
       OR (ORCOMID=LEFT(@CoB,5) AND DSTCOMID=LEFT(@CoA,5));

    DECLARE @A_ToIdx int, @A_FrIdx int, @B_ToIdx int, @B_FrIdx int;

    -- Map indexes based on directionality
    IF (@ORCOMID=LEFT(@CoA,5) AND @DSTCOMID=LEFT(@CoB,5))
    BEGIN
        SET @A_FrIdx=@ORGFRIDX; SET @A_ToIdx=@ORGTOIDX;
        SET @B_FrIdx=@DSTFRIDX; SET @B_ToIdx=@DSTTOIDX;
    END
    ELSE
    BEGIN
        SET @A_FrIdx=@DSTFRIDX; SET @A_ToIdx=@DSTTOIDX;
        SET @B_FrIdx=@ORGFRIDX; SET @B_ToIdx=@ORGTOIDX;
    END

    -- Resolve Indexes to Strings (ACTNUMST) via GL00105
    DECLARE @sql nvarchar(max);
    SET @sql = N'SELECT @X=ACTNUMST FROM ' + QUOTENAME(LEFT(@CoA,5)) + N'..GL00105 WHERE ACTINDX=@I;';
    EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@A_ToIdx, @X=@CoA_DueTo OUTPUT;
    EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@A_FrIdx, @X=@CoA_DueFrom OUTPUT;

    SET @sql = N'SELECT @X=ACTNUMST FROM ' + QUOTENAME(LEFT(@CoB,5)) + N'..GL00105 WHERE ACTINDX=@I;';
    EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@B_ToIdx, @X=@CoB_DueTo OUTPUT;
    EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@B_FrIdx, @X=@CoB_DueFrom OUTPUT;
END
GO

-- 3.6 Build Auto-Balancing Lines
CREATE PROCEDURE dbo.usp_IC_BuildAutoLines
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;
    DELETE dbo.IC_AutoLine WHERE RunID=@RunID;

    DECLARE @Co1 varchar(10), @Co2 varchar(10);
    SELECT TOP 1 @Co1 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID ORDER BY CompanyID;
    SELECT TOP 1 @Co2 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID <> @Co1 ORDER BY CompanyID;

    DECLARE @Net1 numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Co1);
    DECLARE @D date = (SELECT MIN(TrxDate) FROM dbo.IC_Line WHERE RunID=@RunID);

    DECLARE @C1_DT char(129), @C1_DF char(129), @C2_DT char(129), @C2_DF char(129);
    
    EXEC dbo.usp_IC_GetDueAccounts
        @CoA=@Co1, @CoB=@Co2,
        @CoA_DueTo=@C1_DT OUTPUT, @CoA_DueFrom=@C1_DF OUTPUT,
        @CoB_DueTo=@C2_DT OUTPUT, @CoB_DueFrom=@C2_DF OUTPUT;

    IF (@C1_DT IS NULL OR @C1_DF IS NULL OR @C2_DT IS NULL OR @C2_DF IS NULL)
        THROW 52005, 'Due To/Due From accounts did not resolve from GL00105.', 1;

    EXEC dbo.usp_IC_Log @RunID, 'DUE', 'Resolved DueTo/DueFrom accounts successfully.';

    -- Logic: If Co1 is Net Positive, it needs a Credit to Due To; Co2 needs Debit to Due From.
    IF @Net1 > 0
    BEGIN
        INSERT dbo.IC_AutoLine VALUES (@RunID,@D,@Co1,@C1_DT,0,@Net1,@Co2);
        INSERT dbo.IC_AutoLine VALUES (@RunID,@D,@Co2,@C2_DF,@Net1,0,@Co1);
    END
    ELSE IF @Net1 < 0
    BEGIN
        DECLARE @X numeric(19,5) = ABS(@Net1);
        INSERT dbo.IC_AutoLine VALUES (@RunID,@D,@Co1,@C1_DF,@X,0,@Co2);
        INSERT dbo.IC_AutoLine VALUES (@RunID,@D,@Co2,@C2_DT,0,@X,@Co1);
    END
END
GO

-- 3.7 Post to GP (eConnect Wrapper)
CREATE PROCEDURE dbo.usp_IC_PostToGP
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @BatchID char(15), @Ref varchar(30);
    SELECT @BatchID=LEFT(BatchID,15), @Ref=LEFT(Reference,30) FROM dbo.IC_Run WHERE RunID=@RunID;

    -- Cursor through distinct companies in the run
    DECLARE @Company varchar(10);
    DECLARE c CURSOR LOCAL FAST_FORWARD FOR
        SELECT DISTINCT CompanyID FROM (
            SELECT CompanyID FROM dbo.IC_Line WHERE RunID=@RunID
            UNION
            SELECT CompanyID FROM dbo.IC_AutoLine WHERE RunID=@RunID
        ) x;

    OPEN c; FETCH NEXT FROM c INTO @Company;
    WHILE @@FETCH_STATUS=0
    BEGIN
        DECLARE @TrxDate date = (SELECT MIN(TrxDate) FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Company);
        DECLARE @TrxDateDT datetime = DATEADD(day, DATEDIFF(day,0,@TrxDate), 0);

        -- Prepare Lines
        IF OBJECT_ID('tempdb..#Lines') IS NOT NULL DROP TABLE #Lines;
        SELECT ACTNUMST, SUM(Debit) AS Debit, SUM(Credit) AS Credit
        INTO #Lines
        FROM (
            SELECT Account AS ACTNUMST, Debit, Credit FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Company
            UNION ALL
            SELECT Account AS ACTNUMST, Debit, Credit FROM dbo.IC_AutoLine WHERE RunID=@RunID AND CompanyID=@Company
        ) L
        GROUP BY ACTNUMST;

        -- Guard: Must Balance
        DECLARE @Net numeric(19,5) = (SELECT ISNULL(SUM(Debit - Credit),0) FROM #Lines);
        IF ABS(@Net) > 0.00001
        BEGIN
             DECLARE @errMsg NVARCHAR(200) = CONCAT('Company=',@Company,' Net(Debit-Credit)=',@Net);
             EXEC dbo.usp_IC_Log @RunID, 'BALANCE_ERROR', @errMsg;
             THROW 51020, 'Company entry is not balanced; cannot create header.', 1;
        END

        -- eConnect Dynamic SQL
        DECLARE @sql nvarchar(max) = N'
        USE ' + QUOTENAME(@Company) + N';
        SET NOCOUNT ON;
        DECLARE @JE char(13), @Err int=0, @ErrStr varchar(255)='''';
        
        -- 1. Get Next JE
        EXEC taGetNextJournalEntry 1, @JE OUTPUT, @Err OUTPUT;
        IF @Err<>0 RAISERROR(''taGetNextJournalEntry failed (%d)'',16,1,@Err);

        -- 2. Insert Lines
        DECLARE @Seq int = 16384;
        DECLARE L CURSOR LOCAL FAST_FORWARD FOR SELECT ACTNUMST, Debit, Credit FROM #Lines;
        OPEN L;
        DECLARE @A char(129), @D numeric(19,5), @C numeric(19,5);
        FETCH NEXT FROM L INTO @A,@D,@C;
        WHILE @@FETCH_STATUS=0 BEGIN
            EXEC taGLTransactionLineInsert @B, @JE, @Seq, @A, @D, @C, @R, @O_iErrorState=@Err OUTPUT, @oErrString=@ErrStr OUTPUT;
            IF @Err<>0 RAISERROR(''Line insert failed (%d) %s'',16,1,@Err,@ErrStr);
            SET @Seq=@Seq+16384;
            FETCH NEXT FROM L INTO @A,@D,@C;
        END
        CLOSE L; DEALLOCATE L;

        -- 3. Insert Header (Last)
        EXEC taGLTransactionHeaderInsert @B, @JE, @R, @DT, 0, ''GJ'', @O_iErrorState=@Err OUTPUT, @oErrString=@ErrStr OUTPUT;
        IF @Err<>0 RAISERROR(''Header insert failed (%d) %s'',16,1,@Err,@ErrStr);

        -- Return Results
        SELECT @JE AS JRNENTRY, @Err AS Err, @ErrStr AS ErrStr;
        ';

        DECLARE @JE_out char(13), @Err_out int, @ErrStr_out varchar(255);
        DECLARE @t TABLE (JRNENTRY char(13), Err int, ErrStr varchar(255));

        INSERT @t
        EXEC sp_executesql @sql,
             N'@B char(15), @R varchar(30), @DT datetime',
             @B=@BatchID, @R=@Ref, @DT=@TrxDateDT;

        SELECT TOP 1 @JE_out=JRNENTRY, @Err_out=Err, @ErrStr_out=ErrStr FROM @t;

        -- Verify Header Existence
        DECLARE @HeaderExists int = 0;
        DECLARE @chk nvarchar(max) = N'SELECT @X = COUNT(*) FROM ' + QUOTENAME(@Company) + N'..GL10000 WHERE JRNENTRY=@JE AND BACHNUMB=@B;';
        EXEC sp_executesql @chk, N'@JE char(13), @B char(15), @X int OUTPUT', @JE=@JE_out, @B=@BatchID, @X=@HeaderExists OUTPUT;

        IF ISNULL(@HeaderExists,0)=0
        BEGIN
            EXEC dbo.usp_IC_Log @RunID, 'HEADER_MISSING', 'Header missing in GL10000 after insert.';
            THROW 51011, 'GL10000 header missing after insert.', 1;
        END

        DECLARE @msg NVARCHAR(200) = CONCAT('Company=',@Company,' JE=',@JE_out,' Posted Successfully');
        EXEC dbo.usp_IC_Log @RunID, 'POST_OK', @msg;

        FETCH NEXT FROM c INTO @Company;
    END
    CLOSE c; DEALLOCATE c;

    UPDATE dbo.IC_Run SET Status='POSTED', EndedDT=GETDATE(), Message='OK' WHERE RunID=@RunID;
END
GO

-- 3.8 Master Wrapper
CREATE PROCEDURE dbo.usp_IC_File_Process
(
    @CsvPath nvarchar(4000),
    @RunID uniqueidentifier OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        -- Generate IDs
        DECLARE @BatchID char(15) = LEFT(REPLACE(CONVERT(varchar(36), NEWID()), '-', ''), 15);
        DECLARE @FileName nvarchar(260) = RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);
        DECLARE @Reference varchar(30) = LEFT(@FileName, 30);

        -- Execute Steps
        EXEC dbo.usp_IC_LoadCsv @CsvPath=@CsvPath, @BatchID=@BatchID, @Reference=@Reference, @RunID=@RunID OUTPUT;
        EXEC dbo.usp_IC_Normalize @RunID=@RunID;
        EXEC dbo.usp_IC_ValidateRun @RunID=@RunID;
        EXEC dbo.usp_IC_BuildAutoLines @RunID=@RunID;
        EXEC dbo.usp_IC_PostToGP @RunID=@RunID;

        EXEC dbo.usp_IC_Log @RunID, 'DONE', 'Process completed successfully.';
    END TRY
    BEGIN CATCH
        DECLARE @msg varchar(4000) = ERROR_MESSAGE();
        IF @RunID IS NOT NULL
        BEGIN
            UPDATE dbo.IC_Run SET Status='FAILED', EndedDT=GETDATE(), Message=@msg WHERE RunID=@RunID;
            EXEC dbo.usp_IC_Log @RunID, 'ERROR', @msg;
        END
        ;THROW;
    END CATCH
END
GO
