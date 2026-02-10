1.  **Load the CSV**
2.  **Read the 2 distinct companies inside it**
3.  **Use those 2 companies to resolve Due To/Due From via `DYNAMICS..IC40100`** [\[gptables.prospr.biz\]](https://gptables.prospr.biz/ic40100), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing)
4.  **Build + post one balanced JE per company via eConnect**
5.  **Move the file to Archive or Error** using a SQL Agent PowerShell step (Agent supports PowerShell steps, `#NOSQLPS`, proxies, etc.) [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/powershell/sql-server/run-windows-powershell-steps-in-sql-server-agent?view=sqlserver-ps)

Below is the **complete build**: **ALL tables + ALL procs** + the **file management job step**.
*/


-- # A) CSV format (assumed)
    --CSV contains header row and these columns (Windows CRLF line endings):

    /*
    Company,Account,Account Description,Amount,TrxDate
    BF,69-000-4-461149-000,DESC,462048.73,2026-01-01
    POOL,99-000-2-461149-000,DESC,-462048.73,2026-01-01
    ...
    ```

    *   `Amount` signed (+ debit, − credit)
    *   `TrxDate` per row (we’ll use MIN(TrxDate) per company/run for posting)

    */

-- # B) One-time deployment script (ALL tables + ALL procedures)

    -- ## B1) Drop existing objects (optional reset)
    USE DYNAMICS;
    GO
    
    -- Procedures
    IF OBJECT_ID('dbo.usp_IC_File_Process', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_File_Process;
    IF OBJECT_ID('dbo.usp_IC_PostToGP', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_PostToGP;
    IF OBJECT_ID('dbo.usp_IC_BuildAutoLines', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_BuildAutoLines;
    IF OBJECT_ID('dbo.usp_IC_GetDueAccounts', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_GetDueAccounts;
    IF OBJECT_ID('dbo.usp_IC_Normalize', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_Normalize;
    IF OBJECT_ID('dbo.usp_IC_LoadCsv', 'P') IS NOT NULL DROP PROCEDURE dbo.usp_IC_LoadCsv;

    -- Tables
    IF OBJECT_ID('dbo.IC_Run', 'U') IS NOT NULL DROP TABLE dbo.IC_Run;
    IF OBJECT_ID('dbo.IC_RawCsv', 'U') IS NOT NULL DROP TABLE dbo.IC_RawCsv;
    IF OBJECT_ID('dbo.IC_Line', 'U') IS NOT NULL DROP TABLE dbo.IC_Line;
    IF OBJECT_ID('dbo.IC_AutoLine', 'U') IS NOT NULL DROP TABLE dbo.IC_AutoLine;
    GO

    SELECT * FROM dbo.IC_RawCsv irc


    -- ## B2) Tables
        -- ### 1) Run header (one row per file)
        CREATE TABLE dbo.IC_Run
        (
            RunID       uniqueidentifier NOT NULL PRIMARY KEY,
            FileName    nvarchar(260) NOT NULL,
            FilePath    nvarchar(4000) NOT NULL,
            BatchID     char(15) NOT NULL,
            Reference   varchar(30) NOT NULL,

            Status      varchar(20) NOT NULL DEFAULT 'LOADED',  -- LOADED/POSTED/FAILED
            StartedDT   datetime NOT NULL DEFAULT GETDATE(),
            EndedDT     datetime NULL,
            Message     varchar(4000) NULL
        );
        GO

        -- ### 2) Raw CSV landing table (strings only)
        CREATE TABLE dbo.IC_RawCsv
        (
            Company     varchar(20)  NULL,
            Account     varchar(129) NULL,
            AcctDesc    varchar(255) NULL,
            Amount      varchar(50)  NULL,
            TrxDate     varchar(30)  NULL
        );
        GO

        -- ### 3) Typed staging lines
        CREATE TABLE dbo.IC_Line
        (
            LineID      int IDENTITY(1,1) PRIMARY KEY,
            RunID       uniqueidentifier NOT NULL,

            TrxDate     date NOT NULL,
            CompanyID   varchar(10) NOT NULL,         -- company DB id; varchar avoids trailing-space issues
            Account     char(129) NOT NULL,
            AcctDesc    varchar(255) NULL,

            Amount      numeric(19,5) NOT NULL,       -- signed
            Debit       numeric(19,5) NOT NULL DEFAULT 0,
            Credit      numeric(19,5) NOT NULL DEFAULT 0
        );
        GO

        -- ### 4) Auto IC balancing lines (Due To/Due From)
        CREATE TABLE dbo.IC_AutoLine
        (
            AutoID       int IDENTITY(1,1) PRIMARY KEY,
            RunID        uniqueidentifier NOT NULL,
            TrxDate      date NOT NULL,

            CompanyID    varchar(10) NOT NULL,
            Account      char(129) NOT NULL,          -- Due To / Due From ACTNUMST
            Debit        numeric(19,5) NOT NULL DEFAULT 0,
            Credit       numeric(19,5) NOT NULL DEFAULT 0,
            OtherCompany varchar(10) NOT NULL
        );
        GO

        /*
            > **Why IC40100/GL00105 matter:** IC relationships and due-to/from indexes live in `IC40100` (ORCOMID/DSTCOMID and ORGFRIDX/ORGTOIDX/DSTFRIDX/DSTTOIDX).   
            > Those indexes must resolve to account numbers via `GL00105` (ACTINDX/ACTNUMST). [\[gptables.prospr.biz\]](https://gptables.prospr.biz/ic40100), [\[dyndeveloper.com\]](https://dyndeveloper.com/DynColumn.aspx?TableName=IC40100&Database=DYNAMICS) [\[gptables.prospr.biz\]](https://gptables.prospr.biz/gl00105), [\[Dynamics G...105 fields\]](http://dyndeveloper.com/DynColumn.aspx?TableName=GL00105)
        */


    -- ## B3) Stored Procedures (ALL of them)

        -- ### 1) Load CSV → Raw → IC\_Line
        DECLARE @RunID UNIQUEIDENTIFIER;
        EXEC dbo.usp_IC_LoadFromCsv @CsvPath = N'd:\ic_load\sead_2.csv',        -- nvarchar(4000)
                                    @FileName = N'sead_2',       -- nvarchar(260)
                                    @BatchID = '123',         -- char(15)
                                    @Reference = '332',       -- varchar(30)
                                    @RunID = @RunID OUTPUT -- uniqueidentifier
        
        -- Uses **BULK INSERT** with Windows CRLF row terminator. BULK INSERT options like `FIELDTERMINATOR`, `ROWTERMINATOR`, `FIRSTROW` are documented. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver17)
        CREATE OR ALTER PROCEDURE dbo.usp_IC_LoadCsv
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

            DECLARE @FileName nvarchar(260) =
                RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

            INSERT dbo.IC_Run (RunID, FileName, FilePath, BatchID, Reference)
            VALUES (@RunID, @FileName, @CsvPath, @BatchID, @Reference);

            TRUNCATE TABLE dbo.IC_RawCsv;

            DECLARE @sql nvarchar(max) = N'
        BULK INSERT DYNAMICS.dbo.IC_RawCsv
        FROM ' + QUOTENAME(@CsvPath,'''') + N'
        WITH (
            FIRSTROW = 2,
            FIELDTERMINATOR = '','',
            ROWTERMINATOR = ''0x0d0a''
        );';

            EXEC (@sql);

            INSERT dbo.IC_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
            SELECT
                @RunID,
                CONVERT(date, TrxDate),
                RTRIM(LTRIM(Company)),
                CONVERT(char(129), RTRIM(LTRIM(Account))),
                AcctDesc,
                CONVERT(numeric(19,5), REPLACE(Amount, ',', ''))
            FROM dbo.IC_RawCsv;

            DELETE dbo.IC_Line WHERE RunID=@RunID AND Amount=0;
        END
        GO

        -- ### 2) Normalize signed Amount → Debit/Credit
        CREATE OR ALTER PROCEDURE dbo.usp_IC_Normalize
        (
            @RunID uniqueidentifier
        )
        AS
        BEGIN
            SET NOCOUNT ON;

            UPDATE dbo.IC_Line
            SET
              Debit  = CASE WHEN Amount > 0 THEN Amount ELSE 0 END,
              Credit = CASE WHEN Amount < 0 THEN ABS(Amount) ELSE 0 END
            WHERE RunID=@RunID;
        END
        GO

        -- ### 3) Resolve Due To/Due From accounts between two companies

        -- This is where your earlier “NULL Account” error came from when a relationship/index couldn’t be resolved. The due-to/from indexes are stored in `IC40100`, and we resolve to `ACTNUMST` via company `GL00105`. [\[gptables.prospr.biz\]](https://gptables.prospr.biz/ic40100), [\[gptables.prospr.biz\]](https://gptables.prospr.biz/gl00105)
        CREATE OR ALTER PROCEDURE dbo.usp_IC_GetDueAccounts
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

            DECLARE
              @ORCOMID char(5), @DSTCOMID char(5),
              @ORGFRIDX int, @ORGTOIDX int, @DSTFRIDX int, @DSTTOIDX int;

            SELECT TOP 1
              @ORCOMID=ORCOMID, @DSTCOMID=DSTCOMID,
              @ORGFRIDX=ORGFRIDX, @ORGTOIDX=ORGTOIDX,
              @DSTFRIDX=DSTFRIDX, @DSTTOIDX=DSTTOIDX
            FROM DYNAMICS..IC40100
            WHERE (ORCOMID=LEFT(@CoA,5) AND DSTCOMID=LEFT(@CoB,5))
               OR (ORCOMID=LEFT(@CoB,5) AND DSTCOMID=LEFT(@CoA,5));

            DECLARE @A_ToIdx int, @A_FrIdx int, @B_ToIdx int, @B_FrIdx int;

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

            DECLARE @sql nvarchar(max);

            SET @sql = N'SELECT @X=ACTNUMST FROM ' + QUOTENAME(LEFT(@CoA,5)) + N'..GL00105 WHERE ACTINDX=@I;';
            EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@A_ToIdx, @X=@CoA_DueTo OUTPUT;
            EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@A_FrIdx, @X=@CoA_DueFrom OUTPUT;

            SET @sql = N'SELECT @X=ACTNUMST FROM ' + QUOTENAME(LEFT(@CoB,5)) + N'..GL00105 WHERE ACTINDX=@I;';
            EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@B_ToIdx, @X=@CoB_DueTo OUTPUT;
            EXEC sp_executesql @sql, N'@I int, @X char(129) OUTPUT', @I=@B_FrIdx, @X=@CoB_DueFrom OUTPUT;
        END
        GO


        -- ### 4) Build IC balancing lines **using the 2 companies in the file**
        -- Since each CSV has **only 2 companies**, we’ll explicitly pick those two and create the balancing lines directly (no cross-join of many companies).
        CREATE OR ALTER PROCEDURE dbo.usp_IC_BuildAutoLines
        (
            @RunID uniqueidentifier
        )
        AS
        BEGIN
            SET NOCOUNT ON;

            DELETE dbo.IC_AutoLine WHERE RunID=@RunID;

            DECLARE @Co1 varchar(10), @Co2 varchar(10);

            SELECT TOP 1 @Co1 = CompanyID
            FROM dbo.IC_Line
            WHERE RunID=@RunID
            ORDER BY CompanyID;

            SELECT TOP 1 @Co2 = CompanyID
            FROM dbo.IC_Line
            WHERE RunID=@RunID AND CompanyID <> @Co1
            ORDER BY CompanyID;

            DECLARE @Net1 numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Co1);
            DECLARE @Net2 numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Co2);

            DECLARE @D date = (SELECT MIN(TrxDate) FROM dbo.IC_Line WHERE RunID=@RunID);

            DECLARE @C1_DT char(129), @C1_DF char(129), @C2_DT char(129), @C2_DF char(129);
            EXEC dbo.usp_IC_GetDueAccounts
                @CoA=@Co1, @CoB=@Co2,
                @CoA_DueTo=@C1_DT OUTPUT, @CoA_DueFrom=@C1_DF OUTPUT,
                @CoB_DueTo=@C2_DT OUTPUT, @CoB_DueFrom=@C2_DF OUTPUT;


            -- after calling usp_IC_GetDueAccounts and getting @C1_DT, @C2_DF etc.
            IF (@C1_DT IS NULL OR @C1_DF IS NULL OR @C2_DT IS NULL OR @C2_DF IS NULL)
            THROW 52005, 'Due To/Due From accounts did not resolve from GL00105 (check IC40100 due-to/from indexes and company GL00105).', 1;

            DECLARE @msg NVARCHAR(1000) =  CONCAT('Due accounts resolved. C1 DueTo=',@C1_DT,' C1 DueFrom=',@C1_DF,' C2 DueTo=',@C2_DT,' C2 DueFrom=',@C2_DF)
            EXEC dbo.usp_IC_Log @RunID, 'DUE', @msg;


            -- If Co1 net positive: Co1 needs credit DueTo, Co2 needs debit DueFrom
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

        -- > This ties back to the GP concept of tracking cross-company balances via Due To/Due From. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing)
        -- ### 5) Post to GP via eConnect (one JE per company)

        --eConnect GL insertion uses stored procedures and the “lines then header” order is commonly required for correct results.
      CREATE OR ALTER PROCEDURE dbo.usp_IC_PostToGP
        (
            @RunID uniqueidentifier
        )
        AS
        BEGIN
            SET NOCOUNT ON;

            DECLARE @BatchID char(15), @Ref varchar(30);
            SELECT @BatchID=LEFT(BatchID,15), @Ref=LEFT(Reference,30)
            FROM dbo.IC_Run WHERE RunID=@RunID;

            -- Process each company in this run
            DECLARE @Company varchar(10);
            DECLARE c CURSOR LOCAL FAST_FORWARD FOR
                SELECT DISTINCT CompanyID
                FROM (
                    SELECT CompanyID FROM dbo.IC_Line     WHERE RunID=@RunID
                    UNION
                    SELECT CompanyID FROM dbo.IC_AutoLine WHERE RunID=@RunID
                ) x;

            OPEN c; FETCH NEXT FROM c INTO @Company;
            WHILE @@FETCH_STATUS=0
            BEGIN
                -- Choose date: MIN(transaction date) for the company, pass as midnight datetime
                DECLARE @TrxDate date =
                    (SELECT MIN(TrxDate) FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID=@Company);
                DECLARE @TrxDateDT datetime = DATEADD(day, DATEDIFF(day,0,@TrxDate), 0);

                -- Build the lines we will insert in this company (business + auto due-to/from)
                IF OBJECT_ID('tempdb..#Lines') IS NOT NULL DROP TABLE #Lines;
                SELECT ACTNUMST, SUM(Debit) AS Debit, SUM(Credit) AS Credit
                INTO #Lines
                FROM (
                    SELECT Account AS ACTNUMST, Debit, Credit
                    FROM dbo.IC_Line
                    WHERE RunID=@RunID AND CompanyID=@Company
                    UNION ALL
                    SELECT Account AS ACTNUMST, Debit, Credit
                    FROM dbo.IC_AutoLine
                    WHERE RunID=@RunID AND CompanyID=@Company
                ) L
                GROUP BY ACTNUMST;

                -- 🔒 Guard: company JE must balance (GP header requires balanced distributions) [2](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing)
                DECLARE @Net numeric(19,5) =
                    (SELECT ISNULL(SUM(Debit - Credit),0) FROM #Lines);
                IF ABS(@Net) > 0.00001
                DECLARE @msg NVARCHAR(200) =  CONCAT('Company=',@Company,' Net(Debit-Credit)=',@Net)
                BEGIN
                    EXEC dbo.usp_IC_Log @RunID, 'BALANCE_ERROR', @msg;
                    THROW 51020, 'Company entry is not balanced; cannot create header.', 1;
                END

                -- Post this company via eConnect, with error capture + post-check for header existence
                DECLARE @sql nvarchar(max) = N'
        USE ' + QUOTENAME(@Company) + N';
        SET NOCOUNT ON;

        DECLARE @JE char(13), @Err int=0, @ErrStr varchar(255)='''';

        -- 1) Get next JE number
        EXEC taGetNextJournalEntry
             @I_vInc_Dec=1,
             @O_vJournalEntryNumber=@JE OUTPUT,
             @O_iErrorState=@Err OUTPUT;

        IF @Err<>0
            RAISERROR(''taGetNextJournalEntry failed (%d)'',16,1,@Err);

        -- 2) Insert lines
        DECLARE @Seq int = 16384;
        DECLARE L CURSOR LOCAL FAST_FORWARD FOR
        SELECT ACTNUMST, Debit, Credit FROM #Lines;

        OPEN L;
        DECLARE @A char(129), @D numeric(19,5), @C numeric(19,5);
        FETCH NEXT FROM L INTO @A,@D,@C;

        WHILE @@FETCH_STATUS=0
        BEGIN
            SET @Err=0; SET @ErrStr='''';
            EXEC taGLTransactionLineInsert
                 @I_vBACHNUMB=@B,
                 @I_vJRNENTRY=@JE,
                 @I_vSQNCLINE=@Seq,
                 @I_vACTNUMST=@A,
                 @I_vDEBITAMT=@D,
                 @I_vCRDTAMNT=@C,
                 @I_vDSCRIPTN=@R,
                 @O_iErrorState=@Err OUTPUT,
                 @oErrString=@ErrStr OUTPUT;

            IF @Err<>0
                RAISERROR(''Line insert failed (%d) %s'',16,1,@Err,@ErrStr);

            SET @Seq=@Seq+16384;
            FETCH NEXT FROM L INTO @A,@D,@C;
        END
        CLOSE L; DEALLOCATE L;

        -- 3) Header LAST (this is what was failing)
        SET @Err=0; SET @ErrStr='''';
        EXEC taGLTransactionHeaderInsert
             @I_vBACHNUMB=@B,
             @I_vJRNENTRY=@JE,
             @I_vREFRENCE=@R,
             @I_vTRXDATE=@DT,
             @I_vTRXTYPE=0,
             @I_vSOURCDOC=''GJ'',      -- uses a valid Source Doc; GP requires a valid code in this company [2](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing)
             @O_iErrorState=@Err OUTPUT,
             @oErrString=@ErrStr OUTPUT;

        -- 4) Bubble results so outer proc can log & verify
        SELECT @JE AS JRNENTRY, @Err AS Err, @ErrStr AS ErrStr;
        ';

                DECLARE @JE_out char(13), @Err_out int, @ErrStr_out varchar(255);
                DECLARE @t TABLE (JRNENTRY char(13), Err int, ErrStr varchar(255));

                INSERT @t
                EXEC sp_executesql
                     @sql,
                     N'@B char(15), @R varchar(30), @DT datetime',
                     @B=@BatchID, @R=@Ref, @DT=@TrxDateDT;

                SELECT TOP 1 @JE_out=JRNENTRY, @Err_out=Err, @ErrStr_out=ErrStr FROM @t;

                -- Log the header outcome (so you can see SEAD/EPOOL separately)
                SET @msg = CONCAT('Company=',@Company,' JE=',COALESCE(@JE_out,''),' Err=',COALESCE(CAST(@Err_out AS varchar(10)),''),' Msg=',COALESCE(@ErrStr_out,''))
                EXEC dbo.usp_IC_Log @RunID, 'HEADER', @msg;

                -- Fail loudly if header failed
                IF @Err_out <> 0
                BEGIN
                SET @msg = CONCAT('Header insert failed in ',@Company,': ',COALESCE(@ErrStr_out,'(no message)'));
                    THROW 51010, @msg, 1;
                end
                -- Post-check: ensure a header row actually exists in this company (GL10000) for this JE/batch
                DECLARE @HeaderExists int = 0;
                DECLARE @chk nvarchar(max) = N'
                    SELECT @X = COUNT(*) FROM ' + QUOTENAME(@Company) + N'..GL10000
                    WHERE JRNENTRY=@JE AND BACHNUMB=@B;';
                DECLARE @X int;
                EXEC sp_executesql @chk,
                     N'@JE char(13), @B char(15), @X int OUTPUT',
                     @JE=@JE_out, @B=@BatchID, @X=@HeaderExists OUTPUT;

                IF ISNULL(@HeaderExists,0)=0
                BEGIN
                    SET @msg = CONCAT('Company=',@Company,' JE=',COALESCE(@JE_out,''),' not found in GL10000 after header insert.')
                    EXEC dbo.usp_IC_Log @RunID, 'HEADER_MISSING',@msg;
                    
                    SET @msg = CONCAT('GL10000 header missing in ',@Company,' after header insert.');
                    THROW 51011, @msg, 1;
                END

                -- Friendly log: totals and success for this company
                DECLARE @Deb numeric(19,5) = (SELECT SUM(Debit) FROM #Lines);
                DECLARE @Cred numeric(19,5) = (SELECT SUM(Credit) FROM #Lines);
                SET @msg = CONCAT('Company=',@Company,' JE=',@JE_out,' Deb=',@Deb,' Cred=',@Cred)
                EXEC dbo.usp_IC_Log @RunID, 'POST_OK', @msg;

                FETCH NEXT FROM c INTO @Company;
            END

            CLOSE c; DEALLOCATE c;

            UPDATE dbo.IC_Run SET Status='POSTED', EndedDT=GETDATE(), Message='OK' WHERE RunID=@RunID;
        END
        GO


        -- ### 6) Process a file end-to-end (single entry point)
        CREATE OR ALTER PROCEDURE dbo.usp_IC_File_Process
        (
            @CsvPath nvarchar(4000),
            @RunID uniqueidentifier OUTPUT
        )
        AS
        BEGIN
            SET NOCOUNT ON;

            BEGIN TRY
                DECLARE @FileName nvarchar(260) =
                    RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

                -- Generate a safe unique BatchID (15 chars) and Reference (30 chars)
                DECLARE @BatchID char(15) = LEFT(REPLACE(CONVERT(varchar(36), NEWID()), '-', ''), 15);
                DECLARE @Reference varchar(30) = LEFT(@FileName, 30);

                EXEC dbo.usp_IC_LoadCsv @CsvPath=@CsvPath, @BatchID=@BatchID, @Reference=@Reference, @RunID=@RunID OUTPUT;
                EXEC dbo.usp_IC_Normalize @RunID=@RunID;

                EXEC dbo.usp_IC_ValidateRun @RunID=@RunID;

                EXEC dbo.usp_IC_BuildAutoLines @RunID=@RunID;
                EXEC dbo.usp_IC_PostToGP @RunID=@RunID;

                UPDATE dbo.IC_Run SET Status='POSTED', EndedDT=GETDATE(), Message='OK' WHERE RunID=@RunID;
                EXEC dbo.usp_IC_Log @RunID, 'DONE', 'Posted successfully';
            END TRY
            BEGIN CATCH
                DECLARE @msg varchar(4000) = ERROR_MESSAGE();

                IF @RunID IS NOT NULL
                BEGIN
                    UPDATE dbo.IC_Run SET Status='FAILED', EndedDT=GETDATE(), Message=@msg WHERE RunID=@RunID;
                    EXEC dbo.usp_IC_Log @RunID, 'ERROR', @msg;
                END

                -- rethrow so SQL Agent sees failure and moves file to Error folder
                ;THROW;
            END CATCH
        END
        GO





        -- > If you later decide you *do* want BatchID from filename, we can swap that logic back in — but since filenames are arbitrary, this guarantees uniqueness without relying on naming.


-- # C) File management: SQL Agent PowerShell step (move/process/archive/error)
/*
    SQL Server Agent supports PowerShell job steps and the `#NOSQLPS` directive; you can run under a proxy account for file permissions. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/powershell/sql-server/run-windows-powershell-steps-in-sql-server-agent?view=sqlserver-ps)

    ### Job step script (PowerShell)
    #NOSQLPS
    Import-Module SqlServer

    $Incoming   = "\\Server\ICDrop\Incoming"
    $Processing = "\\Server\ICDrop\Processing"
    $Archive    = "\\Server\ICDrop\Archive"
    $Error      = "\\Server\ICDrop\Error"

    $SqlInstance = "YOURSQLSERVER"
    $Db = "DYNAMICS"

    # 1) Move incoming -> processing (prevents partial-file reads)
    Get-ChildItem $Incoming -Filter *.csv | ForEach-Object {
        Move-Item $_.FullName -Destination $Processing -Force
    }

    # 2) Process each file in Processing
    Get-ChildItem $Processing -Filter *.csv | ForEach-Object {

        $file  = $_.FullName
        $base  = $_.BaseName
        $stamp = Get-Date -Format "yyyyMMdd_HHmmss"

        try {
            Invoke-Sqlcmd `
              -ServerInstance $SqlInstance `
              -Database $Db `
              -QueryTimeout 0 `
              -Query "DECLARE @RunID uniqueidentifier;
                      EXEC dbo.usp_IC_File_Process @CsvPath=N'$file', @RunID=@RunID OUTPUT;
                      SELECT @RunID AS RunID;"

            Move-Item $file -Destination (Join-Path $Archive "$base`_$stamp.csv") -Force
        }
        catch {
            Move-Item $file -Destination (Join-Path $Error "$base`_$stamp.csv") -Force
            throw
        }
    }
*/


/*
    D) What this gives you (exactly)

    ✅ **They can name the file anything** — logic is driven by the **two companies inside the CSV**.  
    ✅ **One file** → one `RunID` → one pair (2 companies) assumed.  
    ✅ Each file is moved to **Archive** or **Error** after processing.  
    ✅ Uses GP’s intercompany setup table `IC40100` for due-to/from resolution.   
    ✅ Uses `BULK INSERT` for CSV ingestion and explicitly supports terminators.   
    ✅ Uses SQL Agent PowerShell step patterns supported by Microsoft. [\[gptables.prospr.biz\]](https://gptables.prospr.biz/ic40100), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing) [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver17) [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/powershell/sql-server/run-windows-powershell-steps-in-sql-server-agent?view=sqlserver-ps)
*/

SELECT * FROM dbo.IC_Run ir

/* Additions for error handling */


USE DYNAMICS;
GO

/* 1A) Add visibility columns to IC_Run (if not already present) */
IF COL_LENGTH('dbo.IC_Run','RowCountRaw') IS NULL
    ALTER TABLE dbo.IC_Run ADD RowCountRaw int NULL;
IF COL_LENGTH('dbo.IC_Run','RowCountLine') IS NULL
    ALTER TABLE dbo.IC_Run ADD RowCountLine int NULL;
IF COL_LENGTH('dbo.IC_Run','CompanyA') IS NULL
    ALTER TABLE dbo.IC_Run ADD CompanyA varchar(10) NULL;
IF COL_LENGTH('dbo.IC_Run','CompanyB') IS NULL
    ALTER TABLE dbo.IC_Run ADD CompanyB varchar(10) NULL;
IF COL_LENGTH('dbo.IC_Run','NetAmount') IS NULL
    ALTER TABLE dbo.IC_Run ADD NetAmount numeric(19,5) NULL;
GO

/* 1B) Event log table (timeline per run) */
IF OBJECT_ID('dbo.IC_RunEvent','U') IS NOT NULL DROP TABLE dbo.IC_RunEvent;
GO

CREATE TABLE dbo.IC_RunEvent
(
    EventID     int IDENTITY(1,1) PRIMARY KEY,
    RunID       uniqueidentifier NOT NULL,
    EventDT     datetime NOT NULL DEFAULT GETDATE(),
    StepName    varchar(50) NOT NULL,
    Message     varchar(4000) NOT NULL
);
GO



CREATE OR ALTER PROCEDURE dbo.usp_IC_Log
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


CREATE OR ALTER PROCEDURE dbo.usp_IC_LoadCsv
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

    DECLARE @FileName nvarchar(260) =
        RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

    INSERT dbo.IC_Run (RunID, FileName, FilePath, BatchID, Reference, Status)
    VALUES (@RunID, @FileName, @CsvPath, @BatchID, @Reference, 'LOADING');

    DECLARE @msg NVARCHAR(50) = CONCAT('Starting load: ', @FileName)
    EXEC dbo.usp_IC_Log @RunID, 'LOAD', @msg;

    TRUNCATE TABLE dbo.IC_RawCsv;

    DECLARE @sql nvarchar(max) = N'
BULK INSERT DYNAMICS.dbo.IC_RawCsv
FROM ' + QUOTENAME(@CsvPath,'''') + N'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = '','',
    ROWTERMINATOR = ''0x0d0a''
    
);';  -- Windows CSV terminator usage documented for bulk import [6](https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver17)

    EXEC (@sql);

    DECLARE @rawCount int = (SELECT COUNT(*) FROM dbo.IC_RawCsv);
    UPDATE dbo.IC_Run SET RowCountRaw=@rawCount WHERE RunID=@RunID;

    SET @msg = CONCAT('Raw rows loaded: ', @rawCount)
    EXEC dbo.usp_IC_Log @RunID, 'LOAD', @msg;

    INSERT dbo.IC_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
    SELECT
        @RunID,
        CONVERT(date, TrxDate),
        RTRIM(LTRIM(Company)),
        CONVERT(char(129), RTRIM(LTRIM(Account))),
        AcctDesc,
        CONVERT(numeric(19,5), REPLACE(Amount, ',', ''))
    FROM dbo.IC_RawCsv;

    -- remove noise zeros (optional, but helpful)
    DELETE dbo.IC_Line WHERE RunID=@RunID AND Amount=0;

    DECLARE @lineCount int = (SELECT COUNT(*) FROM dbo.IC_Line WHERE RunID=@RunID);
    UPDATE dbo.IC_Run SET RowCountLine=@lineCount WHERE RunID=@RunID;

    SET @msg = CONCAT('Staged nonzero lines: ', @lineCount)
    EXEC dbo.usp_IC_Log @RunID, 'LOAD', @msg;

    UPDATE dbo.IC_Run SET Status='LOADED' WHERE RunID=@RunID;
END
GO


CREATE OR ALTER PROCEDURE dbo.usp_IC_Normalize
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

CREATE OR ALTER PROCEDURE dbo.usp_IC_ValidateRun
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;

    -- 1) Ensure rows exist
    DECLARE @lineCount int = (SELECT COUNT(*) FROM dbo.IC_Line WHERE RunID=@RunID);
    IF @lineCount = 0
        THROW 52001, 'No staged lines found for this run (after removing zeros).', 1;

    -- 2) Detect the two companies in the file
    DECLARE @c1 varchar(10), @c2 varchar(10);
    SELECT TOP 1 @c1 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID ORDER BY CompanyID;
    SELECT TOP 1 @c2 = CompanyID FROM dbo.IC_Line WHERE RunID=@RunID AND CompanyID<>@c1 ORDER BY CompanyID;

    UPDATE dbo.IC_Run SET CompanyA=@c1, CompanyB=@c2 WHERE RunID=@RunID;

    DECLARE @msg NVARCHAR(1000) = CONCAT('Detected companies: ', @c1, ' / ', @c2)
    EXEC dbo.usp_IC_Log @RunID, 'VALIDATE', @msg;

    -- 3) Must be exactly 2 companies
    IF (SELECT COUNT(DISTINCT CompanyID) FROM dbo.IC_Line WHERE RunID=@RunID) <> 2
        THROW 52002, 'File must contain exactly 2 distinct companies.', 1;

    -- 4) File must balance overall
    DECLARE @net numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID=@RunID);
    UPDATE dbo.IC_Run SET NetAmount=@net WHERE RunID=@RunID;

    SET @msg = CONCAT('Net amount (should be 0): ', @net)
    EXEC dbo.usp_IC_Log @RunID, 'VALIDATE', @msg;

    IF ABS(@net) > 0.00001
        THROW 52003, 'File does not balance overall (SUM(Amount) <> 0).', 1;

    -- 5) Relationship must exist in IC40100 (Intercompany Account Setup) [1](https://gptables.prospr.biz/ic40100)[2](https://learn.microsoft.com/en-us/dynamics-gp/financials/intercompanyprocessing)
    IF NOT EXISTS (
        SELECT 1 FROM DYNAMICS..IC40100
        WHERE (ORCOMID=LEFT(@c1,5) AND DSTCOMID=LEFT(@c2,5))
           OR (ORCOMID=LEFT(@c2,5) AND DSTCOMID=LEFT(@c1,5))
    )
        THROW 52004, 'No intercompany relationship exists in IC40100 for these companies.', 1;

    EXEC dbo.usp_IC_Log @RunID, 'VALIDATE', 'IC40100 relationship found';
END
GO



        DECLARE @RunID UNIQUEIDENTIFIER;
        EXEC dbo.usp_IC_File_Process @CsvPath = N'd:\ic_load\sead_2.csv',        -- nvarchar(4000)
                                     @RunID = @RunID OUTPUT -- uniqueidentifier

        
  
