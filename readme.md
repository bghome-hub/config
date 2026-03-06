
-- ==============================================================================
-- 0. CLEANUP (For testing/redeployment)
-- ==============================================================================
DROP TABLE IF EXISTS dbo.IC_SystemLog;
DROP TABLE IF EXISTS dbo.IC_DetailLine;
DROP TABLE IF EXISTS dbo.IC_CompanyHeader;
DROP TABLE IF EXISTS dbo.IC_StagingRaw;
DROP TABLE IF EXISTS dbo.IC_ImportJob;
DROP SEQUENCE IF EXISTS dbo.IC_BatchSeq;
GO

-- ==============================================================================
-- 1. SEQUENCE: eConnect Batch Number Generation
-- ==============================================================================
CREATE SEQUENCE dbo.IC_BatchSeq
    START WITH 1
    INCREMENT BY 1
    NO CACHE;
GO

-- ==============================================================================
-- 2. IC_ImportJob: Master Control Record
-- ==============================================================================
CREATE TABLE dbo.IC_ImportJob (
    JobId UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT NEWID(),
    FileName NVARCHAR(260) NOT NULL,
    FilePath NVARCHAR(4000) NOT NULL,
    JobStatus VARCHAR(20) NOT NULL DEFAULT 'PENDING', 
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE()
);
GO

-- ==============================================================================
-- 3. IC_StagingRaw: 100% VARCHAR(MAX) Landing Zone
-- ==============================================================================
CREATE TABLE dbo.IC_StagingRaw (
    StagingId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobId UNIQUEIDENTIFIER NOT NULL,
    Company VARCHAR(MAX) NULL,
    Account VARCHAR(MAX) NULL,
    AcctDesc VARCHAR(MAX) NULL,
    Amount VARCHAR(MAX) NULL,
    TrxDate VARCHAR(MAX) NULL,
    Reference VARCHAR(MAX) NULL,
    ReversalFlag VARCHAR(MAX) NULL,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),
    
    CONSTRAINT FK_ICStagingRaw_Job FOREIGN KEY (JobId) 
        REFERENCES dbo.IC_ImportJob(JobId) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 4. IC_CompanyHeader: The Grouping & Duplicate Guard
-- ==============================================================================
CREATE TABLE dbo.IC_CompanyHeader (
    HeaderId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobId UNIQUEIDENTIFIER NOT NULL,
    Company VARCHAR(20) NOT NULL,
    FileSection NVARCHAR(300) NOT NULL, 
    BatchId CHAR(15) NULL,           
    JournalEntry INT NULL,           
    ReversalFlag CHAR(1) NOT NULL,
    TrxDate DATE NOT NULL,      
    ReversalDate DATE NULL,          
    ContentHash VARBINARY(32) NULL,  
    LineCount INT NULL,              
    TotalAmount NUMERIC(19,5) NULL,  
    HeaderStatus VARCHAR(20) NOT NULL DEFAULT 'PENDING', 
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),

    CONSTRAINT FK_ICCompanyHeader_Job FOREIGN KEY (JobId) 
        REFERENCES dbo.IC_ImportJob(JobId) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 5. IC_DetailLine: Strongly Typed Data for eConnect
-- ==============================================================================
CREATE TABLE dbo.IC_DetailLine (
    LineId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    HeaderId INT NOT NULL,
    JobId UNIQUEIDENTIFIER NOT NULL,
    Account CHAR(129) NOT NULL,
    AcctDesc VARCHAR(255) NULL,
    TrxDate DATE NOT NULL,
    Reference VARCHAR(30) NULL,
    DebitAmount NUMERIC(19,5) NOT NULL DEFAULT 0,
    CreditAmount NUMERIC(19,5) NOT NULL DEFAULT 0,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE(),

    CONSTRAINT FK_ICDetailLine_Header FOREIGN KEY (HeaderId) 
        REFERENCES dbo.IC_CompanyHeader(HeaderId) ON DELETE CASCADE
);
GO

-- ==============================================================================
-- 6. IC_SystemLog: Centralized DBA Logging
-- ==============================================================================
CREATE TABLE dbo.IC_SystemLog (
    LogId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobId UNIQUEIDENTIFIER NULL,     
    HeaderId INT NULL,               
    LogLevel VARCHAR(10) NOT NULL,   
    StepName VARCHAR(50) NOT NULL,
    LogMessage VARCHAR(MAX) NOT NULL,
    CreateDateTime DATETIME NOT NULL DEFAULT GETDATE()
);
GO

-- Index to make querying logs for a specific job instant
CREATE NONCLUSTERED INDEX IX_ICSystemLog_JobId ON dbo.IC_SystemLog(JobId);
GO

```
