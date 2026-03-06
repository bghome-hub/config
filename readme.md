CREATE PROCEDURE dbo.usp_IC_File_Process
(
    @CsvPath NVARCHAR(4000)
)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @FileID   UNIQUEIDENTIFIER = NEWID();
    DECLARE @RunID    UNIQUEIDENTIFIER;
    DECLARE @Company  VARCHAR(10);
    DECLARE @RevFlag  CHAR(1);
    DECLARE @FileName NVARCHAR(260);
    DECLARE @Step     VARCHAR(50);

    -- 1. Extract FileName from @CsvPath
    SET @FileName = LTRIM(RTRIM(REVERSE(LEFT(REVERSE(@CsvPath), 
                    CHARINDEX('\', REVERSE(@CsvPath) + '\') - 1))));

    -- 2. Initialize the File Record
    SET @Step = 'FILE_INIT';
    INSERT INTO dbo.IC_File (FileID, FileName, Status)
    VALUES (@FileID, @FileName, 'IN_PROGRESS');
    
    EXEC dbo.usp_IC_Log @FileID, @Step, 'Intercompany process initiated.';

    BEGIN TRY
        -- STEP: Structural Validation
        -- IC_ValidateFile handles its own @Step = 'FILE_VALIDATE'
        EXEC dbo.usp_IC_ValidateFile @CsvPath, @FileID;

        -- Capture the uniform reversal flag validated in staging
        SELECT TOP 1 @RevFlag = UPPER(LTRIM(RTRIM(Reversal_Flag))) FROM dbo.IC_FileStage;

        -- STEP: Company Identification
        SET @Step = 'COMPANY_IDENT';
        DECLARE @CompanyList TABLE (ID VARCHAR(10));
        INSERT INTO @CompanyList SELECT DISTINCT RTRIM(LTRIM(Company)) FROM dbo.IC_FileStage;

        -- 3. Main Loop: Process each company found in the file
        WHILE EXISTS (SELECT 1 FROM @CompanyList)
        BEGIN
            SELECT TOP 1 @Company = ID FROM @CompanyList;
            SET @RunID = NEWID();

            -- Create the Run record
            INSERT INTO dbo.IC_Run (RunID, FileID, CompanyID, Status, ReversalFlag)
            VALUES (@RunID, @FileID, @Company, 'PROCESSING', @RevFlag);

            BEGIN TRY
                -- STEP: Slicing (Moving data to IC_Line)
                SET @Step = 'DATA_SLICE';
                INSERT INTO dbo.IC_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
                SELECT 
                    @RunID, 
                    TRY_CONVERT(DATE, TrxDate), 
                    @Company, 
                    LEFT(Account, 129), 
                    LEFT(AcctDesc, 255), 
                    TRY_CONVERT(NUMERIC(19,5), Amount)
                FROM dbo.IC_FileStage 
                WHERE RTRIM(LTRIM(Company)) = @Company;

                -- STEP: Business Rules
                -- IC_ValidateRun handles its own @Step = 'RUN_VALIDATE'
                EXEC dbo.usp_IC_ValidateRun @RunID;

                -- STEP: GP Posting
                -- IC_PostToGP handles its own @Step = 'GP_POST'
                EXEC dbo.usp_IC_PostToGP @RunID; 

                UPDATE dbo.IC_Run SET Status = 'COMPLETED' WHERE RunID = @RunID;
                EXEC dbo.usp_IC_Log @RunID, 'BATCH_DONE', 'Company batch successfully posted.';

            END TRY
            BEGIN CATCH
                -- Handle company-specific error without crashing the whole file
                UPDATE dbo.IC_Run SET Status = 'FAILED', Message = ERROR_MESSAGE() WHERE RunID = @RunID;
                EXEC dbo.usp_IC_Log @RunID, 'BATCH_ERROR', ERROR_MESSAGE(); 
            END CATCH

            DELETE FROM @CompanyList WHERE ID = @Company;
        END

        -- Final File Completion Status
        UPDATE dbo.IC_File SET Status = 'COMPLETED' WHERE FileID = @FileID;
        EXEC dbo.usp_IC_Log @FileID, 'FILE_DONE', 'File processing complete.';

    END TRY
    BEGIN CATCH
        -- Fatal global error (e.g., Header Mismatch or Staging crash)
        UPDATE dbo.IC_File SET Status = 'FAILED', Message = ERROR_MESSAGE() WHERE FileID = @FileID;
        EXEC dbo.usp_IC_Log @FileID, 'CRITICAL_ERROR', ERROR_MESSAGE();
        THROW; 
    END CATCH
END
