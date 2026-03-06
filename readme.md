CREATE TABLE dbo.IC_CompanyHeader (
    HeaderId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    JobId UNIQUEIDENTIFIER NOT NULL,
    Company VARCHAR(20) NOT NULL,
    FileSection NVARCHAR(300) NOT NULL, 
    BatchId CHAR(15) NULL,           
    JournalEntry INT NULL,           
    ReversalFlag CHAR(1) NOT NULL,
    
    -- The actual transaction date from the file
    TrxDate DATE NOT NULL,           
    
    -- The calculated end-of-month date (only populated if ReversalFlag = 'Y')
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
