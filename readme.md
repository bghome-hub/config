CREATE PROCEDURE dbo.usp_IC_Process_Staging
    @FileID UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Co VARCHAR(10);
    DECLARE @RunID UNIQUEIDENTIFIER;
    DECLARE @BatchID CHAR(15);

    -- Loop through each company in the file
    DECLARE CoCursor CURSOR FOR SELECT DISTINCT Company FROM dbo.IC_Stage_File;
    OPEN CoCursor;
    FETCH NEXT FROM CoCursor INTO @Co;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @RunID = NEWID();
        
        -- Get BatchID from your existing source of truth
        EXEC dbo.GetNextBatchID @Company = @Co, @BatchID = @BatchID OUTPUT;

        INSERT INTO dbo.IC_Run (RunID, FileID, Company, BatchID, Status)
        VALUES (@RunID, @FileID, @Co, @BatchID, 'PENDING');

        -- Move data to Lines
        INSERT INTO dbo.IC_Line (RunID, Company, Account, Amount, TrxDate, Reference, Reversal)
        SELECT @RunID, @Co, Account, CAST(Amount AS NUMERIC(19,5)), CAST(TrxDate AS DATE), Reference, Reversal
        FROM dbo.IC_Stage_File WHERE Company = @Co;

        -- Update the Header Totals for the Report
        UPDATE dbo.IC_Run SET 
            LineCount = (SELECT COUNT(*) FROM dbo.IC_Line WHERE RunID = @RunID),
            DebitTotal = (SELECT ISNULL(SUM(Amount),0) FROM dbo.IC_Line WHERE RunID = @RunID AND Amount > 0),
            CreditTotal = (SELECT ISNULL(ABS(SUM(Amount)),0) FROM dbo.IC_Line WHERE RunID = @RunID AND Amount < 0)
        WHERE RunID = @RunID;

        FETCH NEXT FROM CoCursor INTO @Co;
    END
    CLOSE CoCursor; DEALLOCATE CoCursor;
END
