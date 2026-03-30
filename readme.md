CREATE PROCEDURE dbo.usp_BoA_AV_Req_RunVendorAccountVerification
(
    @TestMode bit = 1
)
AS 
BEGIN
    SET NOCOUNT ON;

    DECLARE @BatchId uniqueidentifier, 
            @FileName nvarchar(260), 
            @RowsBuffered int, 
            @VendorId nvarchar(30);

    BEGIN TRY
        -------------------------------------------------
        -- STEP 01: Detect changed vendors
        -------------------------------------------------
        DECLARE @ChangedVendors TABLE (
            VendorId NVARCHAR(30) NOT NULL,
            PayloadHash VARBINARY(32) NOT NULL
        );

        INSERT @ChangedVendors (VendorId, PayloadHash)
        EXEC dbo.usp_BoA_AV_Req_S01_FindChangedVendors;

        IF NOT EXISTS (SELECT 1 FROM @ChangedVendors) RETURN;

        -------------------------------------------------
        -- BEGIN TRANSACTION
        -------------------------------------------------
        BEGIN TRAN;

        -------------------------------------------------
        -- STEP 02: Create request rows
        -------------------------------------------------
        DECLARE vendor_cur CURSOR LOCAL FAST_FORWARD FOR
            SELECT VendorId FROM @ChangedVendors;

        OPEN vendor_cur;
        FETCH NEXT FROM vendor_cur INTO @VendorId;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            EXEC dbo.usp_BoA_AV_Req_S02_CreateVendorRequests @VendorId = @VendorId;
            FETCH NEXT FROM vendor_cur INTO @VendorId;
        END

        CLOSE vendor_cur;
        DEALLOCATE vendor_cur;

        -- Step 02b (Premature Hash Commit) has been removed.

        -------------------------------------------------
        -- STEP 03: Clean + prepare request file
        -------------------------------------------------
        EXEC dbo.usp_BoA_AV_Req_S03_WriteRows 
            @TestMode = @TestMode, 
            @BatchId = @BatchId OUTPUT, 
            @FileName = @FileName OUTPUT, 
            @RowsBuffered = @RowsBuffered OUTPUT;

        IF @BatchId IS NULL OR @RowsBuffered = 0 
        BEGIN
            COMMIT TRAN;
            RETURN;
        END

        -------------------------------------------------
        -- COMMIT TRANSACTION
        -------------------------------------------------
        -- Must commit before BCP in S04 to prevent deadlocks.
        COMMIT TRAN;

        -------------------------------------------------
        -- STEP 04: Write file via BCP
        -------------------------------------------------
        EXEC dbo.usp_BoA_AV_Req_S04_WriteFile @BatchId = @BatchId;

        -------------------------------------------------
        -- STEP 05: Commit vendor hash state
        -------------------------------------------------
        -- Only updates if S04 succeeds without throwing.
        MERGE dbo.BoA_AV_Req_VendorStaging AS tgt
        USING @ChangedVendors AS src ON tgt.VendorId = src.VendorId
        WHEN MATCHED THEN
            UPDATE SET PayloadHash = src.PayloadHash, UpdateDateTime = SYSUTCDATETIME()
        WHEN NOT MATCHED THEN
            INSERT (VendorId, PayloadHash, UpdateDateTime) 
            VALUES (src.VendorId, src.PayloadHash, SYSUTCDATETIME());

    END TRY
    BEGIN CATCH
        -- Rollback if S02 or S03 fail. S04 failure won't rollback S02/S03, 
        -- but prevents the Hash from updating, allowing clean retries.
        IF @@TRANCOUNT > 0
            ROLLBACK TRAN;

        INSERT dbo.BoA_AV_JobLog 
        (
            StepName, BatchId, FileName, Message, ErrorNumber, 
            ErrorSeverity, ErrorState, ErrorLine, ErrorProc
        )
        VALUES 
        (
            'Req_RunVendorAccountVerification', @BatchId, @FileName, 
            ERROR_MESSAGE(), ERROR_NUMBER(), ERROR_SEVERITY(), 
            ERROR_STATE(), ERROR_LINE(), ERROR_PROCEDURE()
        );

        THROW;
    END CATCH
END;
GO
