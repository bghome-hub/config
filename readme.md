CREATE FUNCTION dbo.fn_BoA_DigitsOnly (@s NVARCHAR(MAX))
RETURNS NVARCHAR(100)
AS
BEGIN
    IF @s IS NULL RETURN NULL;

    DECLARE @r NVARCHAR(MAX);

    ;WITH d AS
    (
        SELECT
            n  = v.number + 1,
            ch = SUBSTRING(@s, v.number + 1, 1)
        FROM master..spt_values AS v
        WHERE v.type = 'P'
          AND v.number < LEN(@s)
          AND SUBSTRING(@s, v.number + 1, 1) LIKE N'[0-9]'
    )
    SELECT @r =
    (
        SELECT d.ch
        FROM d
        ORDER BY d.n
        FOR XML PATH(''), TYPE
    ).value('.', 'nvarchar(max)');

    -- If you truly want to cap at 100 characters:
    RETURN CONVERT(nvarchar(100), @r);
END;
GO;


CREATE FUNCTION dbo.fn_BoA_SanitizeText (@s NVARCHAR(MAX))
RETURNS NVARCHAR(MAX)
AS
BEGIN
    IF @s IS NULL RETURN NULL;
    -- Normalize curly apostrophe to straight apostrophe
    SET @s = REPLACE(@s, N'’', N'''');

    DECLARE @i INT = 1, @len INT = LEN(@s), @r NVARCHAR(MAX) = N'';
    DECLARE @c NCHAR(1);

    WHILE @i <= @len
    BEGIN
        SET @c = SUBSTRING(@s, @i, 1);
        IF (@c LIKE N'[A-Za-z0-9 ''-]') SET @r += @c;  -- whitelist + space + apostrophe + hyphen
        SET @i += 1;
    END

    WHILE CHARINDEX(N'  ', @r) > 0
        SET @r = REPLACE(@r, N'  ', N' ');

    RETURN LTRIM(RTRIM(@r));
END;
GO;






/* ======================================================================
   SECTION 3 – ENQUEUE PROCEDURE (ALL Active EFTs → Outbox Pending - Comment test)
====================================================================== */
CREATE PROCEDURE dbo.usp_BoA_AV_EnqueueFromGP
    @VendorId            NVARCHAR(30) = NULL,
    @SinceModifiedUtc    DATETIME2(3) = NULL,
    @DefaultValidation   NVARCHAR(20) = N'STATUSANDOWNER',
    @Limit               INT = 20000
AS
BEGIN
    SET NOCOUNT ON;

    WITH src AS
    (
        SELECT TOP (@Limit)
            e.VendorId,
            v.VENDNAME,
            v.AddressCode,
            v.Addr1, v.Addr2, v.Addr3,
            v.City, v.State, v.PostalCode, v.Country,
            v.WorkPhone, v.HomePhone,
            v.GPModifiedDate,
            dbo.fn_BoA_DigitsOnly(e.RoutingNumber)  AS RoutingNumber,
            dbo.fn_BoA_DigitsOnly(e.AccountNumber)  AS AccountNumber,
            COALESCE(e.ValidationType, @DefaultValidation) AS ValidationType
        FROM dbo.v_GP_VendorEFT e
        LEFT JOIN dbo.v_GP_VendorPreferredAddress v ON v.VENDORID = e.VendorId
        WHERE (@VendorId IS NULL OR e.VendorId = @VendorId)
          AND (
                @SinceModifiedUtc IS NULL
             OR (v.GPModifiedDate IS NOT NULL AND v.GPModifiedDate >= @SinceModifiedUtc)
          )
        ORDER BY e.VendorId, e.AddressCode
    )
    INSERT dbo.BoA_AV_Outbox
    (
        EndToEndId, ValidationType,
        AccountNumber, RoutingNumber, AgentQualifier, AgentCountry,
        NamePrefix, NameSuffix,
        FirstName, MiddleName, LastName, BusinessName,
        Addr1, Addr2, Addr3, PostalCode, City, State, Country,
        WorkPhone, HomePhone,
        VendorId, VendorName,
        Status, FileName, CreatedUtc, LastUpdatedUtc, ErrorText
    )
    SELECT
        CONVERT(NVARCHAR(36), NEWID()),
        UPPER(s.ValidationType),
        s.AccountNumber,
        s.RoutingNumber,
        'USABA', 'US',
        NULL, NULL,
        NULL, NULL, NULL,
        dbo.fn_BoA_SanitizeText(s.VENDNAME),
        dbo.fn_BoA_SanitizeText(s.Addr1),
        dbo.fn_BoA_SanitizeText(s.Addr2),
        dbo.fn_BoA_SanitizeText(s.Addr3),
        dbo.fn_BoA_SanitizeText(s.PostalCode),
        dbo.fn_BoA_SanitizeText(s.City),
        s.State, s.Country,
        dbo.fn_BoA_DigitsOnly(s.WorkPhone),
        dbo.fn_BoA_DigitsOnly(s.HomePhone),
        s.VendorId,
        dbo.fn_BoA_SanitizeText(s.VENDNAME),
        N'Pending', NULL, SYSUTCDATETIME(), SYSUTCDATETIME(), NULL
    FROM src s
    WHERE NOT EXISTS (
        SELECT 1
        FROM dbo.BoA_AV_Outbox o
        WHERE o.VendorId      = s.VendorId
          AND o.RoutingNumber = s.RoutingNumber
          AND o.AccountNumber = s.AccountNumber
          AND o.Status IN ('Pending','ReadyForBatch','ReadyForUpload','Exported')
    );
END
GO



-- 4.2 Export Buffer → CSV using xp_cmdshell + bcp (with error handling)
CREATE PROCEDURE dbo.usp_BoA_AV_ExportBatchToFile
    @BatchId     UNIQUEIDENTIFIER = NULL OUTPUT,
    @OutFolder   NVARCHAR(4000),
    @FullPath    NVARCHAR(4000) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @FileName NVARCHAR(260);

    BEGIN TRY
        IF @BatchId IS NULL
            SELECT TOP (1) @BatchId = BatchId FROM dbo.BoA_AV_Batches WHERE Exported = 0 ORDER BY CreatedUtc DESC;

        IF @BatchId IS NULL
        BEGIN
            RAISERROR('No unexported batches found.', 11, 1);
            RETURN;
        END

        SELECT @FileName = FileName FROM dbo.BoA_AV_Batches WHERE BatchId = @BatchId;
        IF @FileName IS NULL
        BEGIN
            RAISERROR('Batch found but FileName is NULL.', 16, 1);
            RETURN;
        END

        SET @FullPath = @OutFolder + N'\' + @FileName;

        DECLARE @mkdir NVARCHAR(4000) = N'IF NOT EXIST "' + @OutFolder + N'" MKDIR "' + @OutFolder + N'"';
        EXEC master..xp_cmdshell @mkdir, NO_OUTPUT;

        DECLARE @cmd NVARCHAR(4000) = N'bcp "SELECT CsvLine FROM ' + QUOTENAME(DB_NAME()) + N'.dbo.BoA_AV_ExportBuffer WHERE BatchId=''' + CONVERT(NVARCHAR(36), @BatchId) + N''' ORDER BY LineNumber" queryout "' + @FullPath + N'" -c -T -S ' + QUOTENAME(@@SERVERNAME, '''');


        DECLARE @out TABLE(line NVARCHAR(4000));
        INSERT @out EXEC master..xp_cmdshell @cmd;

        INSERT dbo.BoA_AV_JobLog(Stepname, BatchID, FileName, Message)
        values('BCP Command', @BatchID, @Filename, @cmd)


        DECLARE @exists TABLE (FileExists INT, FileIsDir INT, ParentDirExists INT);
        INSERT @exists EXEC master..xp_fileexist @FullPath;
        IF NOT EXISTS (SELECT 1 FROM @exists WHERE FileExists = 1)
        BEGIN
            RAISERROR('BCP completed but output file not found: %s', 16, 1, @FullPath);
        END

        UPDATE dbo.BoA_AV_Batches
           SET Exported = 1,
               ExportPath = @FullPath,
               Notes = CONCAT(ISNULL(Notes,''), ' | Exported ', CONVERT(VARCHAR(19), GETUTCDATE(), 120), ' UTC')
         WHERE BatchId = @BatchId;

        UPDATE o
           SET o.Status = 'Exported',
               o.LastUpdatedUtc = SYSUTCDATETIME()
          FROM dbo.BoA_AV_Outbox o
         WHERE o.FileName = @FileName
           AND o.Status = 'ReadyForUpload';

        INSERT dbo.BoA_AV_JobLog(StepName, BatchId, FileName, Message)
        VALUES('ExportBatchToFile', @BatchId, @FileName, CONCAT('Exported to: ', @FullPath));

    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog(StepName, BatchId, FileName, Message, ErrorNumber, ErrorSeverity, ErrorState, ErrorLine, ErrorProc)
        VALUES('ExportBatchToFile', @BatchId, @FileName, ERROR_MESSAGE(), ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(), ERROR_LINE(), ERROR_PROCEDURE());
        THROW;
    END CATCH
END
GO



/* ======================================================================
   SECTION 4 – SANITIZE+BUFFER (logging) & EXPORT (bcp via xp_cmdshell)
====================================================================== */
-- 4.1 Sanitize + Log + Validate + Buffer  (builds 85-field CSV lines into ExportBuffer)
/* ======================================================================
   PROCEDURE: dbo.usp_BoA_AV_SanitizeAndBuffer
   PURPOSE  : Sanitize Outbox rows, validate, build CSV lines (85 fields),
              insert into ExportBuffer, mark Outbox rows ReadyForUpload,
              and record a batch header in BoA_AV_Batches.
   VERSION  : 2026-03-06 (Corrected)
   NOTES    :
     - TEST mode defaults to 100 row cap; PROD defaults to 50,000.
     - CSV generated is 85 columns (30 populated + 55 empty placeholders).
     - All CSV fields are COALESCE'd to '' to prevent column shifting.
     - State/Country normalized to 2-char uppercase.
====================================================================== */
CREATE PROCEDURE dbo.usp_BoA_AV_SanitizeAndBuffer
    @TestMode      BIT = 1,
    @BatchCap      INT = NULL,
    @BatchId       UNIQUEIDENTIFIER OUTPUT,
    @FileName      NVARCHAR(260)  OUTPUT,
    @RowsBuffered  INT            OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @proc SYSNAME = N'usp_BoA_AV_SanitizeAndBuffer';
    DECLARE @cap INT = COALESCE(@BatchCap, CASE WHEN @TestMode = 1 THEN 100 ELSE 50000 END);
    DECLARE @host SYSNAME = HOST_NAME();
    DECLARE @appuser SYSNAME = SUSER_SNAME();

    BEGIN TRY
        /* -------------------------------------------------------------
           1) Select a working set into a temp table
        ------------------------------------------------------------- */
        ;WITH cte AS (
            SELECT TOP (@cap) o.*
            FROM dbo.BoA_AV_Outbox o
            WHERE o.Status IN ('Pending','ReadyForBatch')
            ORDER BY o.EndToEndId
        )
        SELECT * INTO #Batch FROM cte;

        IF NOT EXISTS (SELECT 1 FROM #Batch)
        BEGIN
            SET @BatchId = NULL;
            SET @FileName = N'';
            SET @RowsBuffered = 0;

            -- (Optional) lightweight info log
            INSERT dbo.BoA_AV_JobLog (StepName, Message)
            VALUES (@proc, N'No eligible rows found (Pending/ReadyForBatch).');

            RETURN;
        END

        SET @BatchId = NEWID();

        /* -------------------------------------------------------------
           2) Log sanitize deltas (pre-update)
        ------------------------------------------------------------- */
        INSERT dbo.BoA_AV_SanitizeLog (BatchId, OutboxId, EndToEndId, FieldName, OldValue, NewValue, HostName, AppUser)
        SELECT @BatchId, o.Id, o.EndToEndId, v.FieldName, v.OldValue, v.NewValue, @host, @appuser
        FROM dbo.BoA_AV_Outbox AS o
        JOIN #Batch AS b ON b.Id = o.Id
        CROSS APPLY (VALUES
            ('FirstName',     o.FirstName,     dbo.fn_BoA_SanitizeText(o.FirstName)),
            ('MiddleName',    o.MiddleName,    dbo.fn_BoA_SanitizeText(o.MiddleName)),
            ('LastName',      o.LastName,      dbo.fn_BoA_SanitizeText(o.LastName)),
            ('BusinessName',  o.BusinessName,  dbo.fn_BoA_SanitizeText(o.BusinessName)),
            ('NamePrefix',    o.NamePrefix,    dbo.fn_BoA_SanitizeText(o.NamePrefix)),
            ('NameSuffix',    o.NameSuffix,    dbo.fn_BoA_SanitizeText(o.NameSuffix)),
            ('Addr1',         o.Addr1,         dbo.fn_BoA_SanitizeText(o.Addr1)),
            ('Addr2',         o.Addr2,         dbo.fn_BoA_SanitizeText(o.Addr2)),
            ('Addr3',         o.Addr3,         dbo.fn_BoA_SanitizeText(o.Addr3)),
            ('City',          o.City,          dbo.fn_BoA_SanitizeText(o.City)),
            ('PostalCode',    o.PostalCode,    dbo.fn_BoA_SanitizeText(o.PostalCode)),
            ('WorkPhone',     o.WorkPhone,     dbo.fn_BoA_DigitsOnly(o.WorkPhone)),
            ('HomePhone',     o.HomePhone,     dbo.fn_BoA_DigitsOnly(o.HomePhone)),
            ('AccountNumber', o.AccountNumber, dbo.fn_BoA_DigitsOnly(o.AccountNumber)),
            ('RoutingNumber', o.RoutingNumber, dbo.fn_BoA_DigitsOnly(o.RoutingNumber))
        ) AS v(FieldName, OldValue, NewValue)
        WHERE ( (v.OldValue <> v.NewValue)
             OR (v.OldValue IS NULL AND v.NewValue IS NOT NULL)
             OR (v.OldValue IS NOT NULL AND v.NewValue IS NULL) );

        /* -------------------------------------------------------------
           3) Apply sanitize + normalization
        ------------------------------------------------------------- */
        UPDATE o
           SET FirstName      = dbo.fn_BoA_SanitizeText(o.FirstName),
               MiddleName     = dbo.fn_BoA_SanitizeText(o.MiddleName),
               LastName       = dbo.fn_BoA_SanitizeText(o.LastName),
               BusinessName   = dbo.fn_BoA_SanitizeText(o.BusinessName),
               NamePrefix     = dbo.fn_BoA_SanitizeText(o.NamePrefix),
               NameSuffix     = dbo.fn_BoA_SanitizeText(o.NameSuffix),
               Addr1          = dbo.fn_BoA_SanitizeText(o.Addr1),
               Addr2          = dbo.fn_BoA_SanitizeText(o.Addr2),
               Addr3          = dbo.fn_BoA_SanitizeText(o.Addr3),
               City           = dbo.fn_BoA_SanitizeText(o.City),
               PostalCode     = dbo.fn_BoA_SanitizeText(o.PostalCode),
               WorkPhone      = dbo.fn_BoA_DigitsOnly(o.WorkPhone),
               HomePhone      = dbo.fn_BoA_DigitsOnly(o.HomePhone),
               AccountNumber  = dbo.fn_BoA_DigitsOnly(o.AccountNumber),
               RoutingNumber  = dbo.fn_BoA_DigitsOnly(o.RoutingNumber),
               State          = LEFT(UPPER(LTRIM(RTRIM(o.State))),   2),
               Country        = LEFT(UPPER(LTRIM(RTRIM(o.Country))), 2),
               LastUpdatedUtc = SYSUTCDATETIME()
          FROM dbo.BoA_AV_Outbox o
          JOIN #Batch b ON b.Id = o.Id;

        /* -------------------------------------------------------------
           4) Minimal post-scrub validations
        ------------------------------------------------------------- */
        UPDATE o SET Status='Failed', ErrorText='EndToEndId must be 8–36'
          FROM dbo.BoA_AV_Outbox o JOIN #Batch b ON b.Id = o.Id
         WHERE LEN(o.EndToEndId) NOT BETWEEN 8 AND 36;

        UPDATE o SET Status='Failed', ErrorText='ValidationType must be STATUS or STATUSANDOWNER'
          FROM dbo.BoA_AV_Outbox o JOIN #Batch b ON b.Id = o.Id
         WHERE UPPER(o.ValidationType) NOT IN ('STATUS','STATUSANDOWNER');

        UPDATE o SET Status='Failed', ErrorText='Routing must be 9 digits'
          FROM dbo.BoA_AV_Outbox o JOIN #Batch b ON b.Id = o.Id
         WHERE LEN(o.RoutingNumber) <> 9;

        UPDATE o SET Status='Failed', ErrorText='Address incomplete: need Addr1, City, State(2), PostalCode, Country(2) or leave all blank'
          FROM dbo.BoA_AV_Outbox o JOIN #Batch b ON b.Id = o.Id
         WHERE ( (o.Addr1 IS NULL AND (o.City IS NOT NULL OR o.State IS NOT NULL OR o.PostalCode IS NOT NULL OR o.Country IS NOT NULL))
              OR (o.Addr1 IS NOT NULL AND (o.City IS NULL OR o.State IS NULL OR o.PostalCode IS NULL OR o.Country IS NULL)) );

        -- Exclude failed from this batch
        DELETE b
        FROM #Batch b
        JOIN dbo.BoA_AV_Outbox o ON o.Id = b.Id
        WHERE o.Status = 'Failed';

        IF NOT EXISTS (SELECT 1 FROM #Batch)
        BEGIN
            SET @FileName = N'';
            SET @RowsBuffered = 0;

            INSERT dbo.BoA_AV_JobLog (StepName, BatchId, Message)
            VALUES (@proc, @BatchId, N'All candidates failed validation; no lines buffered.');
            RETURN;
        END

        /* -------------------------------------------------------------
           5) Build filename
        ------------------------------------------------------------- */
        DECLARE @ts NVARCHAR(20) =
            CONVERT(CHAR(8), GETUTCDATE(), 112) + N'_' + REPLACE(CONVERT(CHAR(8), GETUTCDATE(), 108), ':', '');
        DECLARE @env NVARCHAR(8) = CASE WHEN @TestMode = 1 THEN N'TEST' ELSE N'PROD' END;
        SET @FileName = @env  +N'AVREQ_' + @env + N'_' + @ts + N'.csv'; --jse 3/9/2026 added @env to first position to get the word test in the file name for BOA testing

        /* -------------------------------------------------------------
           6) Buffer CSV lines (85 columns total)
              - COALESCE all fields to '' to prevent CONCAT_WS null-skip
              - Append 55 commas → 30 populated fields + 55 blanks = 85 cols
        ------------------------------------------------------------- */
        ;WITH ready AS (
            SELECT
                COALESCE(o.EndToEndId, N'')                            AS [0],
                UPPER(COALESCE(o.ValidationType, N''))                 AS [1],
                COALESCE(o.AccountNumber, N'')                         AS [2],
                N''                                                    AS [3],
                COALESCE(o.RoutingNumber, N'')                         AS [4],
                COALESCE(o.AgentQualifier, N'USABA')                   AS [5],
                COALESCE(o.AgentCountry,  N'US')                       AS [6],
                N''                                                    AS [7],
                COALESCE(o.NamePrefix, N'')                            AS [8],
                N''                                                    AS [9],
                COALESCE(o.NameSuffix, N'')                            AS [10],
                COALESCE(o.FirstName, N'')                             AS [11],
                COALESCE(o.MiddleName, N'')                            AS [12],
                COALESCE(o.LastName, N'')                              AS [13],
                COALESCE(o.BusinessName, N'')                          AS [14],
                N'' AS [15], N'' AS [16], N'' AS [17], N'' AS [18], N'' AS [19],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.Addr1, N'')     ELSE N'' END      AS [20],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.Addr2, N'')     ELSE N'' END      AS [21],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.Addr3, N'')     ELSE N'' END      AS [22],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.PostalCode, N'') ELSE N'' END     AS [23],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.City, N'')      ELSE N'' END      AS [24],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.State, N'')     ELSE N'' END      AS [25],
                CASE WHEN o.Addr1 IS NOT NULL AND o.City IS NOT NULL AND o.State IS NOT NULL
                          AND o.PostalCode IS NOT NULL AND o.Country IS NOT NULL
                     THEN COALESCE(o.Country, N'')   ELSE N'US' END    AS [26],
                N''                                                    AS [27],
                COALESCE(o.WorkPhone, N'')                             AS [28],
                COALESCE(o.HomePhone, N'')                             AS [29]
            FROM dbo.BoA_AV_Outbox o
            JOIN #Batch b ON b.Id = o.Id
            WHERE o.Status IN ('Pending','ReadyForBatch')
        )
        INSERT dbo.BoA_AV_ExportBuffer (BatchId, LineNumber, EndToEndId, FileName, CsvLine)
        SELECT
            @BatchId,
            ROW_NUMBER() OVER (ORDER BY r.[0]),
            r.[0],
            @FileName,
            CONCAT_WS(',',
               r.[0], r.[1], r.[2], r.[3], r.[4], r.[5], r.[6], r.[7], r.[8], r.[9],
               r.[10], r.[11], r.[12], r.[13], r.[14], r.[15], r.[16], r.[17], r.[18], r.[19],
               r.[20], r.[21], r.[22], r.[23], r.[24], r.[25], r.[26], r.[27], r.[28], r.[29]
            ) + REPLICATE(',', 55)
        FROM ready r;

        /* -------------------------------------------------------------
           7) Mark Outbox rows
        ------------------------------------------------------------- */
        UPDATE o
           SET o.Status = 'ReadyForUpload',
               o.FileName = @FileName,
               o.LastUpdatedUtc = SYSUTCDATETIME()
          FROM dbo.BoA_AV_Outbox o
          JOIN #Batch b ON b.Id = o.Id
          WHERE o.Status IN ('Pending','ReadyForBatch');

        /* -------------------------------------------------------------
           8) Batch header (previously missing)
        ------------------------------------------------------------- */
        SELECT @RowsBuffered = COUNT(*)
        FROM dbo.BoA_AV_ExportBuffer
        WHERE BatchId = @BatchId;

        INSERT dbo.BoA_AV_Batches (BatchId, FileName, RowsBuffered, Notes)
        VALUES (@BatchId, @FileName, @RowsBuffered,
                CONCAT('Created ', CONVERT(VARCHAR(19), GETUTCDATE(), 120), ' UTC'));

        /* -------------------------------------------------------------
           9) Lightweight success log
        ------------------------------------------------------------- */
        INSERT dbo.BoA_AV_JobLog (StepName, BatchId, FileName, Message)
        VALUES (@proc, @BatchId, @FileName, CONCAT('Buffered rows: ', @RowsBuffered));
    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog
        (
            StepName, BatchId, FileName, Message,
            ErrorNumber, ErrorSeverity, ErrorState, ErrorLine, ErrorProc
        )
        VALUES
        (
            @proc, @BatchId, @FileName, ERROR_MESSAGE(),
            ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(), ERROR_LINE(), ERROR_PROCEDURE()
        );
        THROW;
    END CATCH
END
GO


    CREATE TABLE dbo.BoA_AV_Batches
    (
        BatchId       UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
        FileName      NVARCHAR(260) NOT NULL,
        RowsBuffered  INT           NOT NULL,
        CreatedUtc    DATETIME2(3)  NOT NULL DEFAULT SYSUTCDATETIME(),
        Exported      BIT           NOT NULL DEFAULT 0,
        ExportPath    NVARCHAR(4000) NULL,
        Notes         NVARCHAR(4000) NULL
    );
    GO;
    CREATE INDEX IX_BoA_Batches_Exported ON dbo.BoA_AV_Batches(Exported, CreatedUtc);
    GO;



        CREATE TABLE dbo.BoA_AV_ExportBuffer (
        BatchId       UNIQUEIDENTIFIER NOT NULL,
        LineNumber        INT              NOT NULL,
        EndToEndId    NVARCHAR(36)     NOT NULL,
        FileName      NVARCHAR(260)    NOT NULL,
        CsvLine       NVARCHAR(MAX)    NOT NULL,
        CreatedUtc    DATETIME2(3)     NOT NULL DEFAULT SYSUTCDATETIME(),
        CONSTRAINT PK_BoA_AV_ExportBuffer PRIMARY KEY (BatchId, LineNumber)
    );


        CREATE TABLE dbo.BoA_AV_JobLog
    (
        LogId        BIGINT IDENTITY(1,1) PRIMARY KEY,
        StepName     NVARCHAR(200) NOT NULL,
        BatchId      UNIQUEIDENTIFIER NULL,
        FileName     NVARCHAR(260) NULL,
        Message      NVARCHAR(4000) NULL,
        ErrorNumber  INT NULL,
        ErrorSeverity INT NULL,
        ErrorState   INT NULL,
        ErrorLine    INT NULL,
        ErrorProc    NVARCHAR(200) NULL,
        LoggedUtc    DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
    );



        CREATE TABLE dbo.BoA_AV_Outbox (
        Id BIGINT IDENTITY(1,1) PRIMARY KEY,
        EndToEndId      NVARCHAR(36) NOT NULL,
        ValidationType  NVARCHAR(20) NOT NULL,
        AccountNumber   NVARCHAR(17) NOT NULL,
        RoutingNumber   CHAR(9)      NOT NULL,
        AgentQualifier  CHAR(5)      NOT NULL DEFAULT 'USABA',
        AgentCountry    CHAR(2)      NOT NULL DEFAULT 'US',
        NamePrefix      NVARCHAR(4)  NULL,
        NameSuffix      NVARCHAR(4)  NULL,
        FirstName       NVARCHAR(40) NULL,
        MiddleName      NVARCHAR(40) NULL,
        LastName        NVARCHAR(40) NULL,
        BusinessName    NVARCHAR(87) NULL,
        Addr1           NVARCHAR(70) NULL,
        Addr2           NVARCHAR(70) NULL,
        Addr3           NVARCHAR(70) NULL,
        PostalCode      NVARCHAR(25) NULL,
        City            NVARCHAR(35) NULL,
        State           CHAR(2)      NULL,
        Country         CHAR(2)      NULL,
        WorkPhone       NVARCHAR(15) NULL,
        HomePhone       NVARCHAR(15) NULL,
        VendorId        NVARCHAR(30) NOT NULL,
        VendorName      NVARCHAR(100) NULL,
        Status          NVARCHAR(20) NOT NULL DEFAULT 'Pending',
        FileName        NVARCHAR(260) NULL,
        CreatedUtc      DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
        LastUpdatedUtc  DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
        ErrorText       NVARCHAR(2000) NULL
    );
    GO;
    CREATE INDEX IX_BoA_Outbox_Status ON dbo.BoA_AV_Outbox(Status);
    GO;



        CREATE TABLE dbo.BoA_AV_SanitizeLog
    (
        LogId        BIGINT IDENTITY(1,1) PRIMARY KEY,
        BatchId      UNIQUEIDENTIFIER NULL,
        OutboxId     BIGINT NOT NULL,
        EndToEndId   NVARCHAR(36) NOT NULL,
        FieldName    SYSNAME NOT NULL,
        OldValue     NVARCHAR(MAX) NULL,
        NewValue     NVARCHAR(MAX) NULL,
        LoggedUtc    DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
        HostName     SYSNAME NULL,
        AppUser      SYSNAME NULL
    );
    GO;
    CREATE INDEX IX_BoA_SanitizeLog_BatchId ON dbo.BoA_AV_SanitizeLog(BatchId);
    GO;
    CREATE INDEX IX_BoA_SanitizeLog_Outbox  ON dbo.BoA_AV_SanitizeLog(OutboxId);
    GO;
