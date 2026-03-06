CREATE PROCEDURE dbo.usp_IC_Process_File
    @CsvPath NVARCHAR(4000)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @FileID UNIQUEIDENTIFIER = NEWID(); -- We create it here.
    DECLARE @FileName NVARCHAR(260) = REVERSE(LEFT(REVERSE(@CsvPath), CHARINDEX('\', REVERSE(@CsvPath)) - 1));
    DECLARE @Step NVARCHAR(50) = 'FILE_PROCESS';

    -- 1. Create the master record immediately
    INSERT INTO dbo.IC_File (FileID, FileName, Status)
    VALUES (@FileID, @FileName, 'PROCESSING');

    BEGIN TRY
        -- STEP 1: GATEKEEPER (CSV -> Staging)
        -- Uses the @FileID we just generated
        EXEC dbo.usp_IC_ValidateFile @CsvPath = @CsvPath, @FileID = @FileID;

        -- STEP 2: DIVIDER (Staging -> Runs/Lines)
        EXEC dbo.usp_IC_Process_Staging @FileID = @FileID;

        -- STEP 3: ACCOUNTANT (Validate the individual Company Runs)
        DECLARE @RunID UNIQUEIDENTIFIER;
        DECLARE RunCursor CURSOR FOR 
            SELECT RunID FROM dbo.IC_Run WHERE FileID = @FileID;

        OPEN RunCursor;
        FETCH NEXT FROM RunCursor INTO @RunID;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            BEGIN TRY
                EXEC dbo.usp_IC_ValidateRun @RunID = @RunID;
                UPDATE dbo.IC_Run SET Status = 'VALIDATED' WHERE RunID = @RunID;
            END TRY
            BEGIN CATCH
                UPDATE dbo.IC_Run SET Status = 'FAILED', ErrorMessage = ERROR_MESSAGE() WHERE RunID = @RunID;
                EXEC dbo.usp_IC_Log @RunID, 'VALIDATE_RUN', 'ERROR', ERROR_MESSAGE();
            END CATCH

            FETCH NEXT FROM RunCursor INTO @RunID;
        END
        CLOSE RunCursor; DEALLOCATE RunCursor;

        -- Finalize
        UPDATE dbo.IC_File SET Status = 'COMPLETED' WHERE FileID = @FileID;
        EXEC dbo.usp_IC_Log @FileID, @Step, 'INFO', 'Process completed for ' + @FileName;

    END TRY
    BEGIN CATCH
        DECLARE @ErrMsg NVARCHAR(MAX) = ERROR_MESSAGE();
        UPDATE dbo.IC_File SET Status = 'ERROR', ErrorMessage = @ErrMsg WHERE FileID = @FileID;
        EXEC dbo.usp_IC_Log @FileID, @Step, 'ERROR', @ErrMsg;
    END CATCH

    -- Return the ID to the caller in case they need it for the report
    SELECT @FileID AS GeneratedFileID;
END
