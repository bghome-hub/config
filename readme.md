
CREATE TABLE dbo.IC_File (
    FileID      UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT (NEWID()),
    FileName    NVARCHAR(260)    NOT NULL,
    FileHash    VARCHAR(64)      NULL, -- SHA-256 fingerprint for duplicate/correction logic
    Status      VARCHAR(20)      NOT NULL DEFAULT 'PENDING',
    Message     VARCHAR(MAX)     NULL,
    CreatedDT   DATETIME         NOT NULL DEFAULT (GETDATE())
);


CREATE TABLE dbo.IC_Run (
    RunID         UNIQUEIDENTIFIER NOT NULL PRIMARY KEY DEFAULT (NEWID()),
    FileID        UNIQUEIDENTIFIER NOT NULL,
    Company       VARCHAR(10)      NOT NULL,
    BatchID       CHAR(15)         NULL, -- Unique Batch ID for GP
    JournalEntry  INT              NULL, -- Captured from GP after posting
    Status        VARCHAR(20)      NOT NULL DEFAULT 'PENDING',
    Message       VARCHAR(MAX)     NULL,
    CreatedDT     DATETIME         NOT NULL DEFAULT (GETDATE()),
    CONSTRAINT FK_ICRun_File FOREIGN KEY (FileID) REFERENCES dbo.IC_File(FileID)
);

CREATE TABLE dbo.IC_Line (
    LineID        INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    RunID         UNIQUEIDENTIFIER  NOT NULL,
    Company       VARCHAR(10)       NOT NULL,
    Account       VARCHAR(129)      NOT NULL,
    AcctDesc      VARCHAR(51)       NOT NULL, -- Length per IA_Line.sql
    Amount        NUMERIC(19,5)     NOT NULL,
    TrxDate       DATE              NOT NULL,
    Reference     VARCHAR(31)       NOT NULL, -- Length per IA_Line.sql
    ReversalFlag  CHAR(1)           NOT NULL, -- Improved (no underscore)
    ReversalDate  DATE              NULL,     -- Improved (no underscore)
    CONSTRAINT FK_ICLine_Run FOREIGN KEY (RunID) REFERENCES dbo.IC_Run(RunID)
);


CREATE TABLE dbo.IC_Log (
    LogID       INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    TargetID    UNIQUEIDENTIFIER  NOT NULL, -- Links to FileID or RunID
    StepName    VARCHAR(50)       NOT NULL, -- e.g., 'DATA_SLICE', 'RUN_VALIDATE'
    LogMessage  VARCHAR(MAX)      NULL,
    LogDT       DATETIME          NOT NULL DEFAULT (GETDATE())
);




CREATE TABLE dbo.IC_Stage_File (
    Company       NVARCHAR(MAX) NULL,
    Account       NVARCHAR(MAX) NULL,
    AcctDesc      NVARCHAR(MAX) NULL,
    Amount        NVARCHAR(MAX) NULL,
    TrxDate       NVARCHAR(MAX) NULL,
    Reference     NVARCHAR(MAX) NULL,
    ReversalFlag  NVARCHAR(MAX) NULL
);

```

---
