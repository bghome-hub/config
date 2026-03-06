CREATE TABLE dbo.IC_File
(

  FileID UNIQUEIDENTIFIER not null primary key default (newid()),
  FileName NVARCHAR(260) NOT NULL,
  FileHash varchar(64),
  TotalRuns INT NOT NULL DEFAULT 0,
  Status varchar(20) not null defauLT 'PENDING',
  ErrorMessage VARCHAR(4000) null,
  CreateDateTme DATETIME not null DEFAULT getdate()

)
CREATE Table dbo.IC_Log(
  LogID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
  TargetID  UNIQUEIDENTIFIER not null,
  LogLevel VARCHAR(10) not null default 'INFO',
  StepName Varchar(50) not null,
  LogMessage VARCHAR(MAX) null,
  LogDateTime datetime NOT NULL DEFAULT (GETDATE())
)
CREATE TABLE [dbo].[IC_Stage_File] (
    [Company]   NVARCHAR (MAX)  NULL,
    [Account]   NVARCHAR (MAX) NULL,
    [AcctDesc]  NVARCHAR (MAX) NULL,
    [Amount]    NVARCHAR (MAX)  NULL,
    [TrxDate]   NVARCHAR (MAX)  NULL,
    [Reference] NVARCHAR (MAX)  NULL,
    [Reversal_Flag] NVARCHAR(MAX) NULL
);
GO


CREATE TABLE dbo.IC_Run (
    RunID         UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT (NEWID()),
    FileID        UNIQUEIDENTIFIER NOT NULL,
    Company       VARCHAR(10)      NOT NULL,
    BatchID       CHAR(15)         NULL, -- Unique Batch ID for GP
    JournalEntry  INT              NULL, -- Captured from GP after posting
    LineCount     INT              NOT NULL DEFAULT 0,
    DebitTotal    NUMERIC(19,5)    NOT NULL DEFAULT 0,
    CreditTotal   NUMERIC(19,5)    NOT NULL DEFAULT 0,
    Status        VARCHAR(20)      NOT NULL DEFAULT 'PENDING',
    ErrorMessage  VARCHAR(MAX)     NULL,
    ProcessedDateTime   DATETIME         NULL,
    CreatedDatetime     DATETIME         NOT NULL DEFAULT (GETDATE()),
    CONSTRAINT FK_ICRun_File FOREIGN KEY (FileID) REFERENCES dbo.IC_File(FileID) 
);


create PROCEDURE dbo.usp_IC_ValidateFile(
  @CsvPath NVARCHAR(4000),
  @FileID UNIQUEIDENTIFIER

)
AS  
BEGIN 
  
  SET NOCOUNT ON;
  DECLARE @ErrorMsg varchar(4000);
  DECLARE @Step nvarchar(50) = 'VALIDATE_FILE'

  /* Define expected headers */
  DECLARE @ExpectedHeader NVARCHAR(MAX) =	'Company,Account,Account Description,Amount,Date,Reference,Reversal';
	DECLARE @ActualHeader NVARCHAR(MAX);

	DROP TABLE IF EXISTS #HeaderCheck;
	CREATE TABLE #HeaderCheck (RawRow VARCHAR(MAX));

  /* Load first row into temp table */
	SET @sql = N'BULK INSERT #HeaderCheck FROM ' + QUOTENAME(@CsvPath,'''') + N' WITH (FIRSTROW = 1, LASTROW = 1, ROWTERMINATOR = ''0x0d0a'');';
	EXEC(@sql)

  /* Check headers */
	SELECT TOP 1 @ActualHeader = LTRIM(RTRIM(RawRow)) FROM #HeaderCheck;
	IF @ActualHeader <> @ExpectedHeader
	BEGIN
		SET @ErrorMsg = CONCAT('Header Mismatch. Found: ', ISNULL(@ActualHeader,'NoHeader'));
		EXEC dbo.usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;throw 54001, @ErrorMsg, 1;
  END

  /* Headers passed, Load file into Staging */
  TRUNCATE TABLE dbo.IC_FileStage;

	SET @sql = N'BULK INSERT dbo.IC_FileStage FROM ' + QUOTENAME(@CsvPath,'''') + N' WITH (FIRSTROW = 2, FIELDTERMINATOR = '','', ROWTERMINATOR = ''0x0d0a'', FIELDQUOTE''"'');';
	EXEC(@sql)

  /*  Validate Amount Format */
  IF EXISTS(SELECT 1 FROM dbo.IC_FileStage WHERE TRY_CONVERT(numeric(19,5), Amount) IS NULL)
  BEGIN
    SET @ErrorMsg = 'File Rejected: Amount column contains invalid characters.';
    exec usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;Throw 54002, @ErrorMsg, 1
  END

  /* Validate Reversal Values */
  IF EXISTS(SELECT 1 from dbo.IC_FileStage WHERE UPPER(LTRIM(RTRIM(Reversal_Flag))) NOT IN ('Y','N'))
  BEGIN 
      SET @ErrorMsg = 'File Rejected: Reversal Flag must be Y or N.';
    exec usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;Throw 54003, @ErrorMsg, 1
  END

  /* Validate consistent Reversal Vales */
  IF (SELECT COUNT(DISTINCT UPPER(LTRIM(RTRIM(Reversal_Flag)))) from dbo.IC_FileStage) <> 1 
  BEGIN
      SET @ErrorMsg = 'File Rejected: Reversal Flag must be the same for each line.';
    exec usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;Throw 54004, @ErrorMsg, 1
  END

  /* File Validation complete*/
  EXEC dbo.usp_IC_Log @FileId, @Step, "INFO", "File Structure and data format verified."

END

CREATE PROCEDURE dbo.usp_IC_ValidateRun(
  @RunID UNIQUEIDENTIFIER
)
AS 
BEGIN

  SET NOCOUNT ON;
  DECLARE @ErrorMsg varchar(4000);
  DECLARE @Step varchar(50) = 'VALIDATE_DATA'
   
  DECLARE @Company Varchar(10);
  Declare @TrxDate DATE
  declare @RevFlag bit = 0
  declare @IsOpen bit = 0

/* Check for existence of data */
  IF NOT EXISTS(SELECT 1 FROM dbo.IC_Line WHERE RunID = @RunID)
  BEGIN
		SET @ErrorMsg = 'Accounting Error: No transaction lines found for this run';
		EXEC dbo.usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;THROW 54010, @ErrorMsg, 1;
  END

  /* Get this Company's info */
  select 
    @Company = Company,
    @TrxDate = (SELECT MIN(TrxDate) FROM dbo.IC_Line WHERE RunID = @RunID),
    @RevFlag = Reversal_Flag
    from dbo.IC_Line where runid = @runid
 


  /* Verify Company exists */
  /* TODO */

  /* Verify that the run has only one company */
  IF (SELECT COUNT(DISTINCT CompanyID) FROM dbo.IC_Line WHERE RunID = @RunID) <> 1
  BEGIN
		SET @ErrorMsg = 'Accounting Error: A single run cannot contain multiple companies';
		EXEC dbo.usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;THROW 54011, @ErrorMsg, 1;
  END

  /* Verify net amount = 0 */
  DECLARE @net numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IC_Line WHERE RunID = @RunID);
  IF ABS(@Net) > 0.00001
  BEGIN
		SET @ErrorMsg = CONCAT('Accounting Error: Transaction is out of balance. Net Amount: ', @net);
		EXEC dbo.usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
    ;THROW 54012, @ErrorMsg, 1;
  END

  /* Verify same reversal date for all lines */
  IF EXISTS(SELECT 1 FROM dbo.IC_Run WHERE RunID = @RunID AND Reversal_Flag = 'Y')
  BEGIN 
    IF (SELECT COUNT(DISTINCT TrxDate) FROM dbo.IC_Line WHERE RunID = @RunID) <> 1
    BEGIN
      SET @ErrorMsg = 'Accounting Error: Reversal runs must have exactly one unique Transaction Date';
      EXEC dbo.usp_IC_Log @FileID, @Step, "ERROR", @ErrorMsg
      ;THROW 54013, @ErrorMsg, 1;
    END
  END

  /* Verify period is open  */
  if @RevFlag = 'N'
  BEGIN 
    DECLARE @sql NVARCHAR(max) = N'
      SELECT @o_Open = 1 FROM ' + QUOTENAME(@company) + N'.dbo.SY40100 WHERE @p_TrxDate between PERIODDT and PERDENDT AND CLOSED = 0 AND SERIES = 2 and PERIODID <> 0';
    EXEC sp_executesql @SQL, N'@p_TrxDate date, @o_Open BIT OUTPUT', @p_TrxDate = @TrxDate, @o_Open = @IsOpen OUTPUT

    IF ISNULL(@IsOpen,0) = 0
    BEGIN
      SET @ErrorMsg = CONCAT('Accounting Error: Date ', convert(varchar(10), @TrxDate, 120), ' is not an OPEN Financial Period for ', @company);
      exec dbo.usp_IC_Log @RunID, @Step, @ErrorMsg
      ;THROW 54014, @ErrorMsg, 1;
    END
  END
  
  /* Verify Account Exists */
  /* TODO */


  EXEC dbo.usp_IC_Log @RunID, @Step, "INFO", 'Data validation completed successfully'

END


