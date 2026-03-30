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
    SELECT o.* INTO #Batch 
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
    -- 2) Structural Validation ONLY (Protecting the pipeline)
    ------------------------------------------------------------
    -- We must have a valid EndToEndId to map the bank's response back to our system.
    UPDATE o SET Status='Failed', ErrorText='EndToEndId length invalid'
    FROM dbo.BoA_AV_Req_Outbox o JOIN #Batch b ON b.OutboxID = o.OutboxID
    WHERE LEN(o.EndToEndId) NOT BETWEEN 8 AND 36;

    -- Remove failed rows from the current batch processing scope
    DELETE b FROM #Batch b
    JOIN dbo.BoA_AV_Req_Outbox o ON o.OutboxID = b.OutboxID
    WHERE o.Status = 'Failed';

    IF NOT EXISTS (SELECT 1 FROM #Batch)
    BEGIN
        SET @BatchId = NULL;
        SET @RowsBuffered = 0;
        RETURN;
    END

    ------------------------------------------------------------
    -- 3) Build file name
    ------------------------------------------------------------
    DECLARE @ts NVARCHAR(20) = CONVERT(CHAR(8), SYSUTCDATETIME(), 112) + '_' + REPLACE(CONVERT(CHAR(8), SYSUTCDATETIME(), 108), ':', '');
    SET @FileName = CASE WHEN @TestMode = 1 THEN 'TEST_' ELSE '' END + 'AVREQ_' + @ts + '.csv';

    ------------------------------------------------------------
    -- 4) Build CSV rows (85 columns)
    ------------------------------------------------------------
    -- Stripping commas inline to prevent column shifting.
    -- All other GP data is sent raw. If it's bad, the bank rejects it.
    INSERT dbo.BoA_AV_Req_ExportBuffer (BatchId, LineNumber, EndToEndId, FileName, CsvLine)
    SELECT 
        @BatchId, 
        ROW_NUMBER() OVER (ORDER BY o.EndToEndId), 
        o.EndToEndId, 
        @FileName,
        CONCAT_WS(',',
            o.EndToEndId,
            UPPER(o.ValidationType),
            REPLACE(o.AccountNumber, ',', ''),
            '', 
            REPLACE(o.RoutingNumber, ',', ''),
            'USABA',
            'US',
            '', 
            REPLACE(o.NamePrefix, ',', ''),
            '', 
            REPLACE(o.NameSuffix, ',', ''),
            REPLACE(o.FirstName, ',', ''),
            REPLACE(o.MiddleName, ',', ''),
            REPLACE(o.LastName, ',', ''),
            REPLACE(o.BusinessName, ',', ''),
            '', '', '', '', '', 
            REPLACE(o.Addr1, ',', ''),
            REPLACE(o.Addr2, ',', ''),
            REPLACE(o.Addr3, ',', ''),
            REPLACE(o.PostalCode, ',', ''),
            REPLACE(o.City, ',', ''),
            REPLACE(o.State, ',', ''),
            REPLACE(o.Country, ',', ''),
            '', 
            REPLACE(o.WorkPhone, ',', ''),
            REPLACE(o.HomePhone, ',', '')
        ) + REPLICATE(',', 55)
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID;

    ------------------------------------------------------------
    -- 5) Finalize batch + Outbox
    ------------------------------------------------------------
    SELECT @RowsBuffered = COUNT(*) 
    FROM dbo.BoA_AV_Req_ExportBuffer 
    WHERE BatchId = @BatchId;

    INSERT dbo.BoA_AV_Req_Batches (BatchId, FileName, RowsBuffered)
    VALUES (@BatchId, @FileName, @RowsBuffered);

    UPDATE o SET 
        o.Status = 'ReadyForUpload',
        o.FileName = @FileName,
        o.LastUpdatedUtc = SYSUTCDATETIME()
    FROM dbo.BoA_AV_Req_Outbox o
    JOIN #Batch b ON b.OutboxID = o.OutboxID;

END;
GO
