-- ==============================================================================
-- 1. Sequence for Batch Number Generation
-- ==============================================================================
IF NOT EXISTS (SELECT * FROM sys.sequences WHERE name = 'IC_Batch_Seq')
BEGIN
    CREATE SEQUENCE dbo.IC_Batch_Seq
        START WITH 1
        INCREMENT BY 1
        NO CACHE;
END
GO

-- ==============================================================================
-- 2. Job Level: Master Control Record
-- ==============================================================================
CREATE TABLE dbo.IC_Import_Job (
    JobID UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT NEWID(),
    FileName NVARCHAR(260) NOT NULL,
    FilePath NVARCHAR(4000) NOT NULL,
    JobStatus VARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, VALIDATED, FAILED, POSTED
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE()
);
GO

-- ==============================================================================
-- 3. Staging Level: 100% VARCHAR(MAX) Landing Zone
-- ==============================================================================
-- Note: ReversalFlag is kept as VARCHAR to catch multi-character garbage data
CREATE TABLE dbo.IC_Staging_Raw (
    StagingID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobID UNIQUEIDENTIFIER NOT NULL,
    Company VARCHAR(MAX) NULL,
    Account VARCHAR(MAX) NULL,
    AcctDesc VARCHAR(MAX) NULL,
    Amount VARCHAR(MAX) NULL,
    TrxDate VARCHAR(MAX) NULL,
    Reference VARCHAR(MAX) NULL,
    ReversalFlag VARCHAR(MAX) NULL,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),
    
    CONSTRAINT FK_IC_Staging_Job FOREIGN KEY (JobID) 
        REFERENCES dbo.IC_Import_Job(JobID) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 4. Company Header: The Grouping & Payload Guard
-- ==============================================================================
CREATE TABLE dbo.IC_Company_Header (
    HeaderID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobID UNIQUEIDENTIFIER NOT NULL,
    CompanyID VARCHAR(10) NOT NULL,
    BatchID CHAR(15) NULL,           -- Generated from IC_Batch_Seq during validation
    JournalEntry INT NULL,           -- Populated by eConnect taGetNextJournalEntry
    ReversalFlag CHAR(1) NOT NULL,
    TrxDateBasis DATE NOT NULL,      -- The single transaction date for this batch
    ReversalDate DATE NULL,          -- Calculated 1st of next month if ReversalFlag = 'Y'
    PayloadHash VARBINARY(32) NULL,  -- The Duplicate Guard (hash of lines for this company)
    HeaderStatus VARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, SKIPPED, FAILED, VALIDATED, POSTED
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),

    CONSTRAINT FK_IC_Header_Job FOREIGN KEY (JobID) 
        REFERENCES dbo.IC_Import_Job(JobID) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 5. Detail Line: Strongly Typed Data for eConnect
-- ==============================================================================
CREATE TABLE dbo.IC_Detail_Line (
    LineID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    HeaderID INT NOT NULL,
    JobID UNIQUEIDENTIFIER NOT NULL,
    Account CHAR(129) NOT NULL,
    AcctDesc VARCHAR(255) NULL,
    TrxDate DATE NOT NULL,
    Reference VARCHAR(30) NULL,
    DebitAmount NUMERIC(19,5) NOT NULL DEFAULT 0,
    CreditAmount NUMERIC(19,5) NOT NULL DEFAULT 0,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),

    CONSTRAINT FK_IC_Line_Header FOREIGN KEY (HeaderID) 
        REFERENCES dbo.IC_Company_Header(HeaderID) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 6. System Log: Robust Tiered Logging
-- ==============================================================================
CREATE TABLE dbo.IC_System_Log (
    LogID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobID UNIQUEIDENTIFIER NULL,     -- Nullable: In case an error happens before JobID is generated
    HeaderID INT NULL,               -- Nullable: Associates the error to a specific company batch if applicable
    LogLevel VARCHAR(10) NOT NULL,   -- INFO, WARN, ERROR, FATAL
    StepName VARCHAR(50) NOT NULL,
    LogMessage VARCHAR(MAX) NOT NULL,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE()
);
GO

-- Create an index to make querying logs for a specific job instant
CREATE NONCLUSTERED INDEX IX_IC_System_Log_JobID ON dbo.IC_System_Log(JobID);
GO
