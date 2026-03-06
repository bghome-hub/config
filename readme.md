CREATE PROCEDURE dbo.usp_IC_Log
    @TargetID   UNIQUEIDENTIFIER,
    @Step       NVARCHAR(50),
    @LogLevel   NVARCHAR(20), -- 'INFO', 'WARNING', 'ERROR'
    @LogMessage NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    -- We use a simple insert. 
    -- If the Log table doesn't exist yet, we should create it first.
    INSERT INTO dbo.IC_Log (
        TargetID, 
        Step, 
        LogLevel, 
        LogMessage, 
        LogDateTime
    )
    VALUES (
        @TargetID, 
        @Step, 
        UPPER(@LogLevel), 
        @LogMessage, 
        GETDATE()
    );
END
