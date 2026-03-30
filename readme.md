CREATE PROCEDURE dbo.usp_BoA_AV_Req_RunVendorAccountVerification
(
    @TestMode bit = 1
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE
        @BatchId     uniqueidentifier,
        @FileName    nvarchar(260),
        @RowsBuffered int,
        @VendorId    nvarchar(30);

    BEGIN TRY

        -------------------------------------------------
        -- STEP 01: Detect changed vendors
        -------------------------------------------------
        DECLARE @ChangedVendors TABLE
        (
            VendorId    NVARCHAR(30)    NOT NULL,
            PayloadHash VARBINARY(32)   NOT NULL
        );

        INSERT @ChangedVendors (VendorId, PayloadHash)
        EXEC dbo.usp_BoA_AV_Req_S01_FindChangedVendors;

        IF NOT EXISTS (SELECT 1 FROM @ChangedVendors)
        RETURN;

        -------------------------------------------------
        -- STEP 02: Create request rows
        -------------------------------------------------
        DECLARE vendor_cur CURSOR LOCAL FAST_FORWARD FOR
            SELECT VendorId FROM @ChangedVendors;

        OPEN vendor_cur;
        FETCH NEXT FROM vendor_cur INTO @VendorId;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            EXEC dbo.usp_BoA_AV_Req_S02_CreateVendorRequests
                @VendorId = @VendorId;

            FETCH NEXT FROM vendor_cur INTO @VendorId;
        END

        CLOSE vendor_cur;
        DEALLOCATE vendor_cur;


        -------------------------------------------------
        -- STEP 02b: Commit vendor hash state
        -------------------------------------------------
        MERGE dbo.BoA_AV_Req_VendorStaging AS tgt
        USING @ChangedVendors AS src
            ON tgt.VendorId = src.VendorId
        WHEN MATCHED THEN
            UPDATE
            SET PayloadHash    = src.PayloadHash,
                UpdateDateTime = SYSUTCDATETIME()
        WHEN NOT MATCHED THEN
            INSERT (VendorId, PayloadHash, UpdateDateTime)
            VALUES (src.VendorId, src.PayloadHash, SYSUTCDATETIME());


        -------------------------------------------------
        -- STEP 03: Clean + prepare request file
        -------------------------------------------------
        EXEC dbo.usp_BoA_AV_Req_S03_WriteRows
            @TestMode     = @TestMode,
            @BatchId      = @BatchId OUTPUT,
            @FileName     = @FileName OUTPUT,
            @RowsBuffered = @RowsBuffered OUTPUT;

        IF @BatchId IS NULL OR @RowsBuffered = 0
            RETURN;

        -------------------------------------------------
        -- STEP 04: Write file
        -------------------------------------------------

        EXEC dbo.usp_BoA_AV_Req_S04_WriteFile
            @BatchId = @BatchId; 

        MERGE dbo.BoA_AV_Req_VendorStaging AS tgt
        USING @ChangedVendors AS src
            ON tgt.VendorId = src.VendorId
        WHEN MATCHED THEN
            UPDATE
            SET PayloadHash    = src.PayloadHash,
                UpdateDateTime = SYSUTCDATETIME()
        WHEN NOT MATCHED THEN
            INSERT (VendorId, PayloadHash, UpdateDateTime)
            VALUES (src.VendorId, src.PayloadHash, SYSUTCDATETIME());

    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog
        (
            StepName,
            BatchId,
            FileName,
            Message,
            ErrorNumber,
            ErrorSeverity,
            ErrorState,
            ErrorLine,
            ErrorProc
        )
        VALUES
        (
            'Req_RunVendorAccountVerification',
            @BatchId,
            @FileName,
            ERROR_MESSAGE(),
            ERROR_NUMBER(),
            ERROR_SEVERITY(),
            ERROR_STATE(),
            ERROR_LINE(),
            ERROR_PROCEDURE()
        );
        THROW;
    END CATCH
END;
GO
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S01_FindChangedVendors
AS
BEGIN
    SET NOCOUNT ON;

    /*
        Purpose:
            Identify vendors whose EFT-related payload has changed
            since the last successful request file was written.

        Output:
            VendorId
            PayloadHash (SHA2_256)

        Notes:
            - Hash comparison is the ONLY change indicator
            - No timestamps
            - No inserts / updates
            - No temp tables
            - Safe for INSERT ... EXEC
    */

    ;WITH CurrentState AS
    (
        SELECT
            e.VendorId,
            PayloadHash =
                HASHBYTES
                (
                    'SHA2_256',
                    CONVERT
                    (
                        NVARCHAR(MAX),
                        (
                            SELECT
                                e.VendorId,
                                dbo.fn_BoA_DigitsOnly(e.RoutingNumber)     AS RoutingNumber,
                                dbo.fn_BoA_DigitsOnly(e.AccountNumber)     AS AccountNumber,
                                v.VENDNAME                                AS VendorName,
                                v.Addr1,
                                v.Addr2,
                                v.Addr3,
                                v.City,
                                v.State,
                                v.PostalCode,
                                v.Country,
                                dbo.fn_BoA_DigitsOnly(v.WorkPhone)         AS WorkPhone,
                                dbo.fn_BoA_DigitsOnly(v.HomePhone)         AS HomePhone
                            FOR JSON PATH,
                                WITHOUT_ARRAY_WRAPPER,
                                INCLUDE_NULL_VALUES
                        )
                    )
                )
        FROM dbo.v_GP_VendorEFT e
        JOIN dbo.v_GP_VendorPreferredAddress v
            ON v.VENDORID = e.VendorId
    )
    SELECT
        c.VendorId,
        c.PayloadHash
    FROM CurrentState c
    LEFT JOIN dbo.BoA_AV_Req_VendorStaging s
        ON s.VendorId = c.VendorId
    WHERE
        s.VendorId IS NULL          -- new vendor
        OR s.PayloadHash <> c.PayloadHash;  -- changed vendor
END;
GO
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S02_CreateVendorRequests
(
    @VendorId NVARCHAR(30),                      -- REQUIRED
    @DefaultValidation NVARCHAR(20) = N'STATUSANDOWNER'
)
AS
BEGIN
    SET NOCOUNT ON;

    /*
        Purpose:
            Create request rows in BoA_AV_Req_Outbox for a single vendor
            that has already been identified as changed by S01.

        Rules:
            - No date logic
            - No hash logic
            - Insert-only
            - Idempotent via NOT EXISTS
            - One vendor at a time
    */

    ------------------------------------------------------------
    -- Guard clause
    ------------------------------------------------------------
    IF @VendorId IS NULL
        RETURN;

    ------------------------------------------------------------
    -- Source data for this vendor
    ------------------------------------------------------------
    ;WITH src AS
    (
        SELECT TOP 100 PERCENT
            e.VendorId,
            v.VENDNAME,
            v.Addr1,
            v.Addr2,
            v.Addr3,
            v.City,
            v.State,
            v.PostalCode,
            v.Country,
            v.WorkPhone,
            v.HomePhone,
            dbo.fn_BoA_DigitsOnly(e.RoutingNumber) AS RoutingNumber,
            dbo.fn_BoA_DigitsOnly(e.AccountNumber) AS AccountNumber,
            COALESCE(e.ValidationType, @DefaultValidation) AS ValidationType
        FROM dbo.v_GP_VendorEFT e
        JOIN dbo.v_GP_VendorPreferredAddress v
            ON v.VENDORID = e.VendorId
        WHERE e.VendorId = @VendorId
        ORDER BY e.VendorId
    )
    ------------------------------------------------------------
    -- Insert new Outbox rows only
    ------------------------------------------------------------
    INSERT dbo.BoA_AV_Req_Outbox
    (
        EndToEndId,
        ValidationType,
        AccountNumber,
        RoutingNumber,
        AgentQualifier,
        AgentCountry,
        NamePrefix,
        NameSuffix,
        FirstName,
        MiddleName,
        LastName,
        BusinessName,
        Addr1,
        Addr2,
        Addr3,
        PostalCode,
        City,
        State,
        Country,
        WorkPhone,
        HomePhone,
        VendorId,
        VendorName,
        Status,
        FileName,
        CreatedUtc,
        LastUpdatedUtc,
        ErrorText
    )
    SELECT
        CONVERT(NVARCHAR(36), NEWID()),
        UPPER(s.ValidationType),
        ISNULL(s.AccountNumber, N''),
        ISNULL(s.RoutingNumber, N''),
        'USABA',
        'US',
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        dbo.fn_BoA_SanitizeText(s.VENDNAME),
        dbo.fn_BoA_SanitizeText(s.Addr1),
        dbo.fn_BoA_SanitizeText(s.Addr2),
        dbo.fn_BoA_SanitizeText(s.Addr3),
        dbo.fn_BoA_SanitizeText(s.PostalCode),
        dbo.fn_BoA_SanitizeText(s.City),
        dbo.fn_BoA_SanitizeText(s.State),
        'US',
        dbo.fn_BoA_DigitsOnly(s.WorkPhone),
        dbo.fn_BoA_DigitsOnly(s.HomePhone),
        s.VendorId,
        dbo.fn_BoA_SanitizeText(s.VENDNAME),
        N'Pending',
        NULL,
        SYSUTCDATETIME(),
        SYSUTCDATETIME(),
        NULL
    FROM src s
    WHERE NOT EXISTS
    (
        SELECT 1
        FROM dbo.BoA_AV_Req_Outbox o
        WHERE
            o.VendorId      = s.VendorId
            AND o.RoutingNumber = s.RoutingNumber
            AND o.AccountNumber = s.AccountNumber
            AND o.Status IN ('Pending','ReadyForBatch','ReadyForUpload')
    );
END;
GO
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S03_WriteRows
(
    @TestMode BIT = 1,
    @BatchId UNIQUEIDENTIFIER OUTPUT,
    @FileName NVARCHAR(260) OUTPUT,
    @RowsBuffered INT OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @proc SYSNAME = N'usp_BoA_AV_Req_S03_WriteRows';

    ------------------------------------------------------------
    -- 1) Select eligible Outbox rows
    ------------------------------------------------------------
    SELECT o.*
    INTO #Batch
    FROM dbo.BoA_AV_Req_Outbox o
    WHERE o.Status IN ('Pending', 'ReadyForBatch')
    ORDER BY o.EndToEndId;

    IF NOT EXISTS (SELECT 1 FROM #Batch)
    BEGIN
        SET @BatchId = NULL;
        SET @FileName = NULL;
        SET @RowsBuffered = 0;
        RETURN;
    END

    SET @BatchId = NEWID();

    ------------------------------------------------------------
    -- 2) Sanitize + normalize in-place
    ------------------------------------------------------------
    UPDATE o
    SET
        FirstName     = dbo.fn_BoA_SanitizeText(o.FirstName),
        MiddleName    = dbo.fn_BoA_SanitizeText(o.MiddleName),
        LastName      = dbo.fn_BoA_SanitizeText(o.LastName),
        BusinessName  = dbo.fn_BoA_SanitizeText(o.BusinessName),
        NamePrefix    = dbo.fn_BoA_SanitizeText(o.NamePrefix),
        NameSuffix    = dbo.fn_BoA_SanitizeText(o.NameSuffix),
        Addr1         = dbo.fn_BoA_SanitizeText(o.Addr1),
        Addr2         = dbo.fn_BoA_SanitizeText(o.Addr2),
        Addr3         = dbo.fn_BoA_SanitizeText(o.Addr3),
        City          = dbo.fn_BoA_SanitizeText(o.City),
        PostalCode    = dbo.fn_BoA_SanitizeText(o.PostalCode),
        WorkPhone     = dbo.fn_BoA_DigitsOnly(o.WorkPhone),
        HomePhone     = dbo.fn_BoA_DigitsOnly(o.HomePhone),
        AccountNumber = dbo.fn_BoA_DigitsOnly(o.AccountNumber),
        RoutingNumber = dbo.fn_BoA_DigitsOnly(o.RoutingNumber),
        State         = LEFT(UPPER(LTRIM(RTRIM(o.State))), 2),
        Country       = LEFT(UPPER(LTRIM(RTRIM(o.Country))), 2),
        LastUpdatedUtc = SYSUTCDATETIME()
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID;

    ------------------------------------------------------------
    -- 3) Validation (hard failures only)
    ------------------------------------------------------------
    UPDATE o SET Status='Failed', ErrorText='EndToEndId length invalid'
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID
    WHERE LEN(o.EndToEndId) NOT BETWEEN 8 AND 36;

    UPDATE o SET Status='Failed', ErrorText='Routing number must be 9 digits'
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID
    WHERE LEN(o.RoutingNumber) <> 9;

    UPDATE o SET Status='Failed', ErrorText='Account number missing'
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID
    WHERE ISNULL(LTRIM(RTRIM(o.AccountNumber)), '') = '';

    DELETE b
    FROM #Batch b
    JOIN dbo.BoA_AV_Req_Outbox o ON o.OutboxID = b.OutboxID
    WHERE o.Status = 'Failed';

    IF NOT EXISTS (SELECT 1 FROM #Batch)
    BEGIN
        SET @BatchId = NULL;
        SET @RowsBuffered = 0;
        RETURN;
    END

    ------------------------------------------------------------
    -- 4) Build file name
    ------------------------------------------------------------
    DECLARE @ts NVARCHAR(20) =
        CONVERT(CHAR(8), SYSUTCDATETIME(), 112) + '_' +
        REPLACE(CONVERT(CHAR(8), SYSUTCDATETIME(), 108), ':', '');

    SET @FileName =
        CASE WHEN @TestMode = 1 THEN 'TEST_' ELSE '' END +
        'AVREQ_' + @ts + '.csv';

    ------------------------------------------------------------
    -- 5) Build CSV rows (85 columns)
    ------------------------------------------------------------
    INSERT dbo.BoA_AV_Req_ExportBuffer
    (
        BatchId,
        LineNumber,
        EndToEndId,
        FileName,
        CsvLine
    )
    SELECT
        @BatchId,
        ROW_NUMBER() OVER (ORDER BY o.EndToEndId),
        o.EndToEndId,
        @FileName,
        CONCAT_WS(',',
            o.EndToEndId,
            UPPER(o.ValidationType),
            o.AccountNumber,
            '',
            o.RoutingNumber,
            'USABA',
            'US',
            '',
            o.NamePrefix,
            '',
            o.NameSuffix,
            o.FirstName,
            o.MiddleName,
            o.LastName,
            o.BusinessName,
            '', '', '', '', '',
            o.Addr1,
            o.Addr2,
            o.Addr3,
            o.PostalCode,
            o.City,
            o.State,
            o.Country,
            '',
            o.WorkPhone,
            o.HomePhone
        ) + REPLICATE(',', 55)
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID;

    ------------------------------------------------------------
    -- 6) Finalize batch + Outbox
    ------------------------------------------------------------
    SELECT @RowsBuffered = COUNT(*)
    FROM dbo.BoA_AV_Req_ExportBuffer
    WHERE BatchId = @BatchId;

    INSERT dbo.BoA_AV_Req_Batches
    (
        BatchId,
        FileName,
        RowsBuffered
    )
    VALUES
    (
        @BatchId,
        @FileName,
        @RowsBuffered
    );

    UPDATE o
    SET
        o.Status = 'ReadyForUpload',
        o.FileName = @FileName,
        o.LastUpdatedUtc = SYSUTCDATETIME()
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID;
END;
GO
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S04_WriteFile
(
    @BatchId     UNIQUEIDENTIFIER,
    @Environment NVARCHAR(10) = N'PREPROD',
    @FullPath    NVARCHAR(4000) OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE
        @OutFolder NVARCHAR(4000),
        @FileName  NVARCHAR(260),
        @cmd       NVARCHAR(4000),
        @cmdSql    NVARCHAR(4000),
        @proc      SYSNAME = N'usp_BoA_AV_Req_S04_WriteFile';

    BEGIN TRY
        ------------------------------------------------------------
        -- 1) Validate batch
        ------------------------------------------------------------
        SELECT @FileName = FileName
        FROM dbo.BoA_AV_Req_Batches
        WHERE BatchId = @BatchId
          AND Exported = 0;

        IF @FileName IS NULL
        BEGIN
            RAISERROR('No unexported batch found for supplied BatchId.',16,1);
            RETURN;
        END

        ------------------------------------------------------------
        -- 2) Resolve output folder
        ------------------------------------------------------------
        SET @OutFolder =
            CASE
                WHEN @Environment = 'PROD'
                    THEN N'\\VLOBPRODFS\PRODUCTION_PASS\'
                ELSE
                    N'\\VLOBPREPRODFS\PREPROD_PASS\'
            END +
            N'GP2018Data\FileTransfer\boa_av\sent_files';

        SET @FullPath = @OutFolder + N'\' + @FileName;

        ------------------------------------------------------------
        -- 3) Write CSV via BCP
        ------------------------------------------------------------
        SET @cmd =
            N'bcp "SELECT CsvLine FROM ' +
            QUOTENAME(DB_NAME()) +
            N'.dbo.BoA_AV_Req_ExportBuffer ' +
            N'WHERE BatchId = ''' +
            CONVERT(NVARCHAR(36), @BatchId) +
            N''' ORDER BY LineNumber" ' +
            N'queryout "' + @FullPath + N'" -c -T -S ' + @@SERVERNAME;

        SET @cmdSql = N'EXEC master..xp_cmdshell @c';

        EXEC sp_executesql
            @cmdSql,
            N'@c NVARCHAR(4000)',
            @c = @cmd;

        ------------------------------------------------------------
        -- 4) Verify file existence
        ------------------------------------------------------------
        DECLARE @exists TABLE
        (
            FileExists        INT,
            FileIsDirectory   INT,
            ParentDirExists   INT
        );

        INSERT @exists
        EXEC master..xp_fileexist @FullPath;

        IF NOT EXISTS (SELECT 1 FROM @exists WHERE FileExists = 1)
        BEGIN
            RAISERROR('BCP reported success but file not found: %s',16,1,@FullPath);
            RETURN;
        END

        ------------------------------------------------------------
        -- 5) Mark batch + rows exported
        ------------------------------------------------------------
        UPDATE dbo.BoA_AV_Req_Batches
        SET
            Exported   = 1,
            ExportPath = @FullPath,
            Notes      = CONCAT(
                            ISNULL(Notes,''),
                            ' Exported ',
                            CONVERT(VARCHAR(19), SYSUTCDATETIME(), 120),
                            ' UTC'
                          )
        WHERE BatchId = @BatchId;

        UPDATE o
        SET
            o.Status = 'Exported',
            o.LastUpdatedUtc = SYSUTCDATETIME()
        FROM dbo.BoA_AV_Req_Outbox o
        WHERE o.FileName = @FileName
          AND o.Status   = 'ReadyForUpload';

        ------------------------------------------------------------
        -- 6) Notify
        ------------------------------------------------------------
        EXEC dbo.usp_BoA_AV_Req_S05_SendEmail
            @BatchId     = @BatchId,
            @FileName    = @FileName,
            @ExportPath  = @FullPath,
            @Status      = 'SUCCESS';
    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog
        (
            StepName,
            BatchId,
            FileName,
            Message,
            ErrorNumber,
            ErrorSeverity,
            ErrorState,
            ErrorLine,
            ErrorProc
        )
        VALUES
        (
            @proc,
            @BatchId,
            @FileName,
            ERROR_MESSAGE(),
            ERROR_NUMBER(),
            ERROR_SEVERITY(),
            ERROR_STATE(),
            ERROR_LINE(),
            ERROR_PROCEDURE()
        );

        SET @FileName = ISNULL(@FileName,'UNKNOWN')
        DECLARE @err_msg  nvarchar(max) =  ERROR_MESSAGE()

        EXEC dbo.usp_BoA_AV_Req_S05_SendEmail
            @BatchId     = @BatchId,
            @FileName    = @FileName,
            @ExportPath  = NULL,
            @Status      = 'FAILED',
            @ErrorMessage = @err_msg;

        THROW;
    END CATCH
END;
GO
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S05_SendEmail
(
    @BatchId       UNIQUEIDENTIFIER,
    @FileName      NVARCHAR(260),
    @ExportPath    NVARCHAR(4000),
    @Status        NVARCHAR(20),   -- SUCCESS | FAILED
    @ErrorMessage  NVARCHAR(4000) = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE
        @subject    NVARCHAR(255),
        @body       NVARCHAR(MAX),
        @rowdetails NVARCHAR(MAX) = N'';

    ------------------------------------------------------------
    -- 1) Build row-level summary (read-only)
    ------------------------------------------------------------
    SELECT
        @rowdetails =
            @rowdetails +
            RIGHT(SPACE(15) + o.VendorId, 15) + ' ' +
            LEFT(o.VendorName + SPACE(30), 30) + ' ' +
            o.Status + CHAR(13) + CHAR(10)
    FROM dbo.BoA_AV_Req_Outbox o
    WHERE o.FileName = @FileName
    ORDER BY o.VendorId;

    IF @rowdetails = N''
        SET @rowdetails = N'(No rows found)';

    ------------------------------------------------------------
    -- 2) Subject
    ------------------------------------------------------------
    SET @subject =
        CONCAT('[BoA AV] ', @Status, ' - ', @FileName);

    ------------------------------------------------------------
    -- 3) Body
    ------------------------------------------------------------
    SET @body =
        N'BoA Account Verification Request File' + CHAR(13) + CHAR(10) +
        N'------------------------------------' + CHAR(13) + CHAR(10) +
        N'Status      : ' + @Status + CHAR(13) + CHAR(10) +
        N'Batch ID    : ' + CONVERT(NVARCHAR(36), @BatchId) + CHAR(13) + CHAR(10) +
        N'File Name   : ' + @FileName + CHAR(13) + CHAR(10) +
        N'Export Path : ' + COALESCE(@ExportPath, N'N/A') + CHAR(13) + CHAR(10) +
        N'UTC Time    : ' + CONVERT(NVARCHAR(19), SYSUTCDATETIME(), 120) +
        CHAR(13) + CHAR(10) + CHAR(13) + CHAR(10) +
        N'VendorId        VendorName                    Status' + CHAR(13) + CHAR(10) +
        N'--------------- ------------------------------ ----------' + CHAR(13) + CHAR(10) +
        @rowdetails;

    IF @ErrorMessage IS NOT NULL
        SET @body +=
            CHAR(13) + CHAR(10) +
            N'Error:' + CHAR(13) + CHAR(10) +
            @ErrorMessage;

    ------------------------------------------------------------
    -- 4) Send email
    ------------------------------------------------------------
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'AccountVerification',
        @recipients   = 'ben.gregory@trsga.com;jeff.ellard@trsga.com',
        @subject      = @subject,
        @body         = @body;
END;
GO
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
    SET @s = REPLACE(@s, N'â€™', N'''');

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
    CREATE TABLE dbo.BoA_AV_JobLog
    (
        JobLogID     INT IDENTITY(1,1) PRIMARY KEY,
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
CREATE TABLE dbo.BoA_AV_Lifecycle
(
    -- Identity
    LifeCycleID          INT IDENTITY(1,1) PRIMARY KEY,
    EndToEndId           NVARCHAR(36) NOT NULL,
    VendorId             NVARCHAR(30) NOT NULL,

    -- Request-side anchors
    RequestBatchId       UNIQUEIDENTIFIER NULL,
    RequestFileName      NVARCHAR(260) NULL,
    RequestCreatedUtc    DATETIME2(3) NOT NULL,
    FileSentUtc          DATETIME2(3) NULL,

    -- Response-side anchors
    ResponseRequestId    NVARCHAR(36) NULL,  -- BoA-generated Request ID
    ResponseFileName     NVARCHAR(260) NULL,
    ResponseReceivedUtc  DATETIME2(3) NULL,

    -- Processing lifecycle (system-driven)
    ProcessingStatus     NVARCHAR(30) NOT NULL,
    -- Examples:
    -- CREATED, SENT, AWAITING_RESPONSE, RESPONSE_RECEIVED, COMPLETED, TERMINAL_ERROR

    -- Business result (derived)
    ResultStatus         NVARCHAR(30) NULL,
    -- Examples:
    -- VALIDATED, REJECTED, CONDITIONAL, DATA_UNAVAILABLE, TECHNICAL_FAILURE

    ResultReasonCode     NVARCHAR(35) NULL,
    ResultReasonDesc     NVARCHAR(255) NULL,

    -- Confidence / scoring (summary only)
    OwnerMatchScore      INT NULL,
    OwnerMatchCategory   CHAR(1) NULL, -- Y / C / N / U

    -- Audit
    CreatedUtc           DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
    LastUpdatedUtc       DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE TABLE dbo.BoA_AV_LU_AccountStatus
(
    AccountStatusCode        NVARCHAR(35) NOT NULL PRIMARY KEY,
    AccountStatusDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_AccountStatusReason
(
    ReasonCode        NVARCHAR(35) NOT NULL PRIMARY KEY,
    ReasonDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_OwnerMatch
(
    MatchCode        CHAR(1) NOT NULL PRIMARY KEY,
    MatchDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_OwnerReason
(
    ReasonCode        NVARCHAR(35) NOT NULL PRIMARY KEY,
    ReasonDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_RecordStatus
(
    StatusCode        NVARCHAR(30) NOT NULL PRIMARY KEY,
    StatusDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_RejectionReason
(
    RejectionCode        NVARCHAR(35) NOT NULL PRIMARY KEY,
    RejectionDescription NVARCHAR(255) NOT NULL
);
CREATE TABLE dbo.BoA_AV_LU_RequestStatus
(
    StatusCode        NVARCHAR(30) NOT NULL PRIMARY KEY,
    StatusDescription NVARCHAR(255) NOT NULL
);
    CREATE TABLE dbo.BoA_AV_Req_Batches
    (
        BatchID       UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
        FileName      NVARCHAR(260) NOT NULL,
        RowsBuffered  INT           NOT NULL,
        CreatedUtc    DATETIME2(3)  NOT NULL DEFAULT SYSUTCDATETIME(),
        Exported      BIT           NOT NULL DEFAULT 0,
        ExportPath    NVARCHAR(4000) NULL,
        Notes         NVARCHAR(4000) NULL
    );
    GO;
    CREATE TABLE dbo.BoA_AV_Req_ExportBuffer (
        BatchId       UNIQUEIDENTIFIER NOT NULL,
        LineNumber        INT              NOT NULL,
        EndToEndId    NVARCHAR(36)     NOT NULL,
        FileName      NVARCHAR(260)    NOT NULL,
        CsvLine       NVARCHAR(MAX)    NOT NULL,
        CreatedUtc    DATETIME2(3)     NOT NULL DEFAULT SYSUTCDATETIME(),
        CONSTRAINT PK_BoA_AV_Req_ExportBuffer PRIMARY KEY (BatchId, LineNumber)
    );
    CREATE TABLE dbo.BoA_AV_Req_Outbox (
        OutboxID        INT IDENTITY(1,1) PRIMARY KEY,
        EndToEndId      NVARCHAR(36) NOT NULL,
        ValidationType  NVARCHAR(20) NOT NULL,
        AccountNumber   NVARCHAR(17) NULL,
        RoutingNumber   CHAR(9)      NULL,
        AgentQualifier  CHAR(5)      NOT NULL DEFAULT 'USABA',
        AgentCountry    CHAR(2)      NOT NULL DEFAULT 'US',
        NamePrefix      NVARCHAR(255)  NULL,
        NameSuffix      NVARCHAR(255)  NULL,
        FirstName       NVARCHAR(255) NULL,
        MiddleName      NVARCHAR(255) NULL,
        LastName        NVARCHAR(255) NULL,
        BusinessName    NVARCHAR(255) NULL,
        Addr1           NVARCHAR(255) NULL,
        Addr2           NVARCHAR(255) NULL,
        Addr3           NVARCHAR(255) NULL,
        PostalCode      NVARCHAR(255) NULL,
        City            NVARCHAR(255) NULL,
        State           NVARCHAR(255) NULL,
        Country         NVARCHAR(255) NULL,
        WorkPhone       NVARCHAR(255) NULL,
        HomePhone       NVARCHAR(255) NULL,
        VendorId        NVARCHAR(30) NOT NULL,
        VendorName      NVARCHAR(255) NULL,
        Status          NVARCHAR(20) NOT NULL DEFAULT 'Pending',
        FileName        NVARCHAR(260) NULL,
        CreatedUtc      DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
        LastUpdatedUtc  DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
        ErrorText       NVARCHAR(2000) NULL
    );
    GO;
    CREATE TABLE dbo.BoA_AV_Req_SanitizeLog
    (
        SanitizeLogID        INT IDENTITY(1,1) PRIMARY KEY,
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
CREATE TABLE [dbo].[BoA_AV_Req_VendorStaging]
(
    VendorId        NVARCHAR(30) NOT NULL PRIMARY KEY,
    PayloadHash     VARBINARY(32) NOT NULL,   -- SHA2_256
    UpdateDateTime      DATETIME2(3) NOT NULL
);
CREATE TABLE dbo.BoA_AV_Resp_Exceptions
(

    RespErrorId INT IDENTITY(1,1) PRIMARY KEY,
    SourceFileName NVARCHAR(260) NULL,
    SourceRowNumber INT NULL,
    EndToEndId NVARCHAR(36) NULL,
    ErrorStage NVARCHAR(50) NOT NULL, -- DISCOVER / STAGE / APPLY
    ErrorMessage NVARCHAR(4000) NOT NULL,
    LoggedUtc DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()

)
CREATE TABLE dbo.BoA_AV_Resp_Raw
(
    RespRawId           BIGINT IDENTITY PRIMARY KEY,
    FileName            NVARCHAR(260) NOT NULL,
    LineNumber          INT NOT NULL,

    -- CSV positions (string only)
    F00_EndToEndId      NVARCHAR(36) NULL,
    F01_RequestId       NVARCHAR(36) NULL,
    F02_RecordId        NVARCHAR(36) NULL,
    F03_RequestStatus   NVARCHAR(30) NULL,
    F04_RecordStatus    NVARCHAR(30) NULL,
    F05_RejectCode      NVARCHAR(35) NULL,
    F06_RejectDesc      NVARCHAR(255) NULL,
    F08_AccStatReasonCd NVARCHAR(35) NULL,
    F09_AccStatReasonDs NVARCHAR(255) NULL,
    F11_OwnerReasonCd   NVARCHAR(35) NULL,
    F12_OwnerReasonDesc NVARCHAR(255) NULL,
    F16_OwnerScore      INT NULL,
    F20_AccountStatus   NVARCHAR(35) NULL,
    F21_AccountStatusDs NVARCHAR(255) NULL,

    -- Match flags
    F63_PrefixMatch     CHAR(1) NULL,
    F64_SuffixMatch     CHAR(1) NULL,
    F65_FNMatch         CHAR(1) NULL,
    F66_MNMatch         CHAR(1) NULL,
    F67_LNMatch         CHAR(1) NULL,
    F68_BNMatch         CHAR(1) NULL,
    F70_AddressMatch    CHAR(1) NULL,
    F71_PostalMatch     CHAR(1) NULL,
    F72_CityMatch       CHAR(1) NULL,
    F73_StateMatch      CHAR(1) NULL,

    LoadedUtc           DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE TABLE dbo.BoA_AV_Resp_Rows
(
    RespRowId BIGINT IDENTITY(1,1) PRIMARY KEY,
    EndToEndId NVARCHAR(36) NOT NULL,
    BankRequestId NVARCHAR(36) NULL,
    BankRecordId  NVARCHAR(36) NULL,
    RequestStatus NVARCHAR(30) NULL,
    RecordStatus  NVARCHAR(30) NULL,
    RejectionReasonCode NVARCHAR(35) NULL,
    RejectionReasonDesc NVARCHAR(255) NULL,
    AccountStatusCode NVARCHAR(35) NULL,
    AccountStatusDesc NVARCHAR(255) NULL,
    AccountStatusAdditionalInfo NVARCHAR(35) NULL,
    OwnerReasonCode NVARCHAR(35) NULL,
    OwnerReasonDesc NVARCHAR(255) NULL,
    OwnerReasonAdditionalInfo NVARCHAR(35) NULL,
    OwnerMatchScore INT NULL,
    OwnerMatchCategory CHAR(1) NULL,
    PrefixMatch CHAR(1) NULL,
    SuffixMatch CHAR(1) NULL,
    FirstNameMatch CHAR(1) NULL,
    MiddleNameMatch CHAR(1) NULL,
    LastNameMatch CHAR(1) NULL,
    BusinessNameMatch CHAR(1) NULL,
    AddressMatch CHAR(1) NULL,
    PostalCodeMatch CHAR(1) NULL,
    CityMatch CHAR(1) NULL,
    StateMatch CHAR(1) NULL,
    WorkPhoneMatch CHAR(1) NULL,
    HomePhoneMatch CHAR(1) NULL,
    FullNameMatch CHAR(1) NULL,
    SourceFileName NVARCHAR(260) NULL,
    SourceRowNumber INT NULL,
    ReceivedUtc DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE VIEW dbo.v_GP_VendorEFT
AS
SELECT
    s.VENDORID             AS VendorId,
    s.ADRSCODE             AS AddressCode,
    s.EFTTransitRoutingNo  AS RoutingNumber,
    s.EFTBankAcct          AS AccountNumber,
    CAST(NULL AS NVARCHAR(20)) AS ValidationType
FROM dbo.SY06000 AS s
WHERE ISNULL(s.INACTIVE, 0) = 0
  AND s.EFTTransitRoutingNo IS NOT NULL
  AND s.EFTBankAcct         IS NOT NULL
CREATE VIEW dbo.v_GP_VendorPreferredAddress
AS
WITH Choice AS
(
    SELECT
        V.VENDORID,
        PreferredADRSCODE = COALESCE(M.ADRSCODE, F.ADRSCODE)
    FROM dbo.PM00200 AS V
    OUTER APPLY
    (
        -- Prefer 'MAIN' if it exists for the vendor
        SELECT TOP (1) A1.ADRSCODE
        FROM dbo.PM00300 AS A1
        WHERE A1.VENDORID = V.VENDORID
          AND A1.ADRSCODE = 'MAIN'
    ) AS M
    OUTER APPLY
    (
        -- Fallback: first address code by lexicographic order
        SELECT TOP (1) A2.ADRSCODE
        FROM dbo.PM00300 AS A2
        WHERE A2.VENDORID = V.VENDORID
        ORDER BY A2.ADRSCODE
    ) AS F
)
SELECT
    V.VENDORID,
    V.VENDNAME,
    C.PreferredADRSCODE       AS AddressCode,
    A.ADDRESS1                AS Addr1,
    A.ADDRESS2                AS Addr2,
    A.ADDRESS3                AS Addr3,
    A.CITY,
    A.STATE,
    A.ZIPCODE                 AS PostalCode,
    A.COUNTRY                 AS Country,
    A.PHNUMBR1                AS WorkPhone,
    A.PHNUMBR2                AS HomePhone,
    CASE
        WHEN A.DEX_ROW_TS IS NULL OR V.DEX_ROW_TS >= A.DEX_ROW_TS
            THEN V.DEX_ROW_TS
        ELSE A.DEX_ROW_TS
    END                       AS GPModifiedDate
FROM dbo.PM00200 AS V
LEFT JOIN Choice  AS C
    ON C.VENDORID = V.VENDORID
LEFT JOIN PM00300 AS A
    ON A.VENDORID = V.VENDORID
   AND A.ADRSCODE = C.PreferredADRSCODE;
GO


