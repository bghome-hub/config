CREATE PROCEDURE dbo.usp_IC_Import_Master
    @CsvPath NVARCHAR(4000),
    @FileName NVARCHAR(260)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @FileID UNIQUEIDENTIFIER = NEWID();
    DECLARE @Step NVARCHAR(50) = 'MASTER_PROCESS';

    -- 1. Initialize the File Record
    INSERT INTO dbo.IC_File (FileID, FileName, Status)
    VALUES (@FileID, @FileName, 'PROCESSING');

    BEGIN TRY
        -- STEP 1: VALIDATE FILE (Bulk Insert & Format Check)
        EXEC dbo.usp_IC_ValidateFile @CsvPath = @CsvPath, @FileID = @FileID;

        -- STEP 2: PROCESS STAGING (Split into Company Runs & Lines)
        EXEC dbo.usp_IC_Process_Staging @FileID = @FileID;

        -- STEP 3: VALIDATE RUNS (Accounting Checks)
        -- Loop through the runs we just created and validate each one
        DECLARE @RunID UNIQUEIDENTIFIER;
        DECLARE RunCursor CURSOR FOR 
            SELECT RunID FROM dbo.IC_Run WHERE FileID = @FileID;

        OPEN RunCursor;
        FETCH NEXT FROM RunCursor INTO @RunID;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- We wrap each run validation in its own try/catch 
            -- so one bad batch doesn't kill the whole report.
            BEGIN TRY
                EXEC dbo.usp_IC_ValidateRun @RunID = @RunID;
                
                UPDATE dbo.IC_Run SET Status = 'VALIDATED' WHERE RunID = @RunID;
            END TRY
            BEGIN CATCH
                UPDATE dbo.IC_Run 
                SET Status = 'FAILED', 
                    ErrorMessage = ERROR_MESSAGE() 
                WHERE RunID = @RunID;
            END CATCH

            FETCH NEXT FROM RunCursor INTO @RunID;
        END
        CLOSE RunCursor; DEALLOCATE RunCursor;

        -- Final File Status
        UPDATE dbo.IC_File 
        SET Status = 'COMPLETED' 
        WHERE FileID = @FileID;

        EXEC dbo.usp_IC_Log @FileID, @Step, 'INFO', 'Import process completed.';

    END TRY
    BEGIN CATCH
        -- This catches major failures (like Step 1 Header mismatches)
        DECLARE @TopLevelError NVARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_File 
        SET Status = 'ERROR', ErrorMessage = @TopLevelError 
        WHERE FileID = @FileID;

        EXEC dbo.usp_IC_Log @FileID, @Step, 'ERROR', @TopLevelError;
    END CATCH
END
