
CREATE OR ALTER PROCEDURE dbo.usp_IA_Run_Audit(
    @JobID UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    /* Create Temp Table */
    DROP TABLE IF EXISTS #AuditResults;

    CREATE TABLE #AuditResults(
        AuditResultID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        JobID UNIQUEIDENTIFIER,
        runid UNIQUEIDENTIFIER, 
        [FILENAME] NVARCHAR(520),
        CompanyA VARCHAR(10),
        BatchID CHAR(15),
        JournalEntry INT,
        TrxDate DATE,
        [Reference] VARCHAR(30),
        [STATUS] VARCHAR(20),
        [MESSAGE] VARCHAR(4000),
        ReversalFlag CHAR(1),
        TrxDateBasis DATE,
        ReversalDate DATE,
        Account CHAR(129),
        AcctDesc CHAR(255),
        Amount NUMERIC(19,5),
        DebitAmt NUMERIC(19,5),
        CreditAmt NUMERIC(19,5),
        NetAmt NUMERIC(19,5),
        /* Values from GP */
        gp_Journal INT,
        gp_BatchNumber CHAR(15),
        gp_Reference CHAR(31),
        gp_TrxDate DATETIME,
        gp_ReversalFlag CHAR(1),
        gp_PeriodID SMALLINT,
        gp_PeriodMonth INT,
        gp_OpenYear SMALLINT,
        gp_ReversalPeriodID SMALLINT,
        gp_ReversalPeriodMonth INT,
        gp_ReversalYear SMALLINT,
        gp_AccountIndex INT, 
        gp_Account CHAR(129), 
        gp_Description CHAR(31),
        gp_Company CHAR(5),
        gp_DebitAmount NUMERIC(19,5),
        gp_CreditAmount NUMERIC(19,5),
        gp_OriginatingDebitAmount NUMERIC(19,5),
        gp_OriginatingCreditAmount NUMERIC(19,5)
    );

    /* Cursor Variables */
    DECLARE @currentRunID UNIQUEIDENTIFIER;
    DECLARE @currentCompany VARCHAR(20);
    DECLARE @sql NVARCHAR(MAX); 
    DECLARE @params NVARCHAR(MAX) = N'@p_runID UNIQUEIDENTIFIER';

    DECLARE run_cursor CURSOR FOR
        SELECT RunID, CompanyA
        FROM DYNAMICS.dbo.IA_Run
        WHERE JobID = @JobID;

    OPEN run_cursor;

    FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        /* Build dynamic statement that INSERTs into the temp table */
        SET @sql = N'
            INSERT INTO #AuditResults (
                JobID, runid, [FILENAME], CompanyA, BatchID, JournalEntry, TrxDate, [Reference], [STATUS], [MESSAGE],
                ReversalFlag, TrxDateBasis, ReversalDate, Account, AcctDesc, Amount,
                DebitAmt, CreditAmt, NetAmt,
                gp_Journal, gp_BatchNumber, gp_Reference, gp_TrxDate, gp_ReversalFlag, gp_PeriodID,
                gp_PeriodMonth, gp_OpenYear, gp_ReversalPeriodID, gp_ReversalPeriodMonth, gp_ReversalYear,
                gp_AccountIndex, gp_Account, gp_Description, gp_Company, gp_DebitAmount, gp_CreditAmount,
                gp_OriginatingDebitAmount, gp_OriginatingCreditAmount
            )
            SELECT 
                r.JobID,
                l.runid, 
                r.[FileName],
                r.CompanyA,
                r.BatchID,
                r.JournalEntry,
                l.TrxDate,
                r.[Reference],
                r.[Status],
                r.[Message],
                r.ReversalFlag,
                r.TrxDateBasis,
                r.ReversalDate,
                l.Account,
                l.AcctDesc,
                l.Amount,
                SUM(ISNULL(l.Debit,0)) AS DebitAmt,
                SUM(ISNULL(l.Credit,0)) AS CreditAmt,
                SUM(ISNULL(l.Debit,0) - ISNULL(l.Credit,0)) AS NetAmt,
                /* Values from GP */
                glh.JRNENTRY AS gp_Journal,
                glh.BACHNUMB AS gp_BatchNumber,
                glh.REFRENCE AS gp_Reference,
                glh.TRXDATE  AS gp_TrxDate,
                CASE 
                    WHEN glh.TRXTYPE = 0 THEN ''N''
                    WHEN glh.TRXTYPE = 1 THEN ''Y''
                END AS gp_ReversalFlag,  
                glh.PERIODID AS gp_PeriodID,
                ISNULL(fp.PeriodMonth,0) AS gp_PeriodMonth,
                glh.OPENYEAR AS gp_OpenYear,
                glh.REVPRDID AS gp_ReversalPeriodID,
                ISNULL(fp2.PeriodMonth,0) AS gp_ReversalPeriodMonth,
                glh.REVYEAR  AS gp_ReversalYear,
                gld.ACTINDX  AS gp_AccountIndex,
                act.ACTNUMST AS gp_Account,
                gld.DSCRIPTN AS gp_Description,
                gld.INTERID AS gp_Company,
                gld.DEBITAMT AS gp_DebitAmount,
                gld.CRDTAMNT AS gp_CreditAmount,
                gld.ORDBTAMT AS gp_OriginatingDebitAmount,
                gld.ORCRDAMT AS gp_OriginatingCreditAmount
            FROM dynamics.dbo.IA_Run r
            LEFT JOIN dynamics.dbo.IA_Line l ON l.RunID = r.RunID
            LEFT JOIN [' + @currentCompany + N'].dbo.GL10000 glh   ON glh.BACHNUMB = r.BatchID
            LEFT JOIN [' + @currentCompany + N'].dbo.GL10001 gld   ON glh.JRNENTRY = gld.JRNENTRY
            LEFT JOIN [' + @currentCompany + N'].dbo.GL00105 act   ON act.ACTINDX  = gld.ACTINDX
            LEFT JOIN dynamics.dbo.ia_fiscal_period fp  ON glh.PERIODID = fp.PeriodID
            LEFT JOIN dynamics.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
            WHERE r.RunID = @p_runID
            GROUP BY 
                CASE WHEN glh.TRXTYPE = 0 THEN ''N'' WHEN glh.TRXTYPE = 1 THEN ''Y'' END,
                r.JobID, l.runid, r.[FileName], r.CompanyA, r.BatchID, r.JournalEntry,
                r.[Reference], r.[Status], r.[Message], r.ReversalFlag, r.TrxDateBasis, r.ReversalDate,
                l.Amount, l.Account, l.AcctDesc, l.TrxDate,
                glh.JRNENTRY, glh.BACHNUMB, glh.REFRENCE, glh.TRXDATE,
                glh.PERIODID, glh.OPENYEAR, glh.REVPRDID, glh.REVYEAR,
                gld.ACTINDX, act.ACTNUMST, gld.DSCRIPTN, gld.INTERID,
                gld.DEBITAMT, gld.CRDTAMNT, gld.ORDBTAMT, gld.ORCRDAMT,
                fp.PeriodMonth, fp2.PeriodMonth
        ';

        EXEC sys.sp_executesql
            @stmt   = @sql,
            @params = @params,
            @p_runID = @currentRunID;

        FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;
    END

    CLOSE run_cursor;
    DEALLOCATE run_cursor;

    SELECT * FROM #AuditResults ORDER BY CompanyA, Account;

    DROP TABLE IF EXISTS #AuditResults;
END
GO






















============

ALTER PROCEDURE dbo.usp_IA_Run_Audit(
    @JobID UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    /* Create Temp Table */
    DROP TABLE IF EXISTS #AuditResults;

    CREATE TABLE #AuditResults(
        AuditResultID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        JobID UNIQUEIDENTIFIER,
        runid UNIQUEIDENTIFIER, 
        [FILENAME] NVARCHAR(520),
        CompanyA VARCHAR(10),
        BatchID CHAR(15),
        JournalEntry INT,
        TrxDate DATE,
        [Reference] VARCHAR(30),
        [STATUS] VARCHAR(20),
        [MESSAGE] VARCHAR(4000),
        ReversalFlag CHAR(1),
        TrxDateBasis DATE,
        ReversalDate DATE,
        Account CHAR(129),
        AcctDesc CHAR(255),
        Amount NUMERIC(19,5),
        DebitAmt NUMERIC(19,5),
        CreditAmt NUMERIC(19,5),
        NetAmt NUMERIC(19,5),
        /* Values from GP */
        gp_Journal INT,
        gp_BatchNumber CHAR(15),
        gp_Reference CHAR(31),
        gp_TrxDate DATETIME,
        gp_ReversalFlag CHAR(1),
        gp_PeriodID SMALLINT,
        gp_PeriodMonth INT,
        gp_OpenYear SMALLINT,
        gp_ReversalPeriodID SMALLINT,
        gp_ReversalPeriodMonth INT,
        gp_ReversalYear SMALLINT,
        gp_AccountIndex INT, 
        gp_Account CHAR(129), 
        gp_Description CHAR(31),
        gp_Company CHAR(5),
        gp_DebitAmount NUMERIC(19,5),
        gp_CreditAmount NUMERIC(19,5),
        gp_OriginatingDebitAmount NUMERIC(19,5),
        gp_OriginatingCreditAmount NUMERIC(19,5)
    );

    /* Cursor Variables */
    DECLARE @currentRunID UNIQUEIDENTIFIER;
    DECLARE @currentCompany VARCHAR(20);
    DECLARE @sql NVARCHAR(MAX); 
    DECLARE @params NVARCHAR(MAX) = N'@p_runID UNIQUEIDENTIFIER';

    DECLARE run_cursor CURSOR FOR
        SELECT RunID, CompanyA
        FROM DYNAMICS.dbo.IA_Run
        WHERE JobID = @JobID;

    OPEN run_cursor;

    FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        /* Build dynamic statement that INSERTs into the temp table */
        SET @sql = N'
            INSERT INTO #AuditResults (
                JobID, runid, [FILENAME], CompanyA, BatchID, JournalEntry, TrxDate, [Reference], [STATUS], [MESSAGE],
                ReversalFlag, TrxDateBasis, ReversalDate, Account, AcctDesc, Amount,
                DebitAmt, CreditAmt, NetAmt,
                gp_Journal, gp_BatchNumber, gp_Reference, gp_TrxDate, gp_ReversalFlag, gp_PeriodID,
                gp_PeriodMonth, gp_OpenYear, gp_ReversalPeriodID, gp_ReversalPeriodMonth, gp_ReversalYear,
                gp_AccountIndex, gp_Account, gp_Description, gp_Company, gp_DebitAmount, gp_CreditAmount,
                gp_OriginatingDebitAmount, gp_OriginatingCreditAmount
            )
            SELECT 
                r.JobID,
                l.runid, 
                r.[FileName],
                r.CompanyA,
                r.BatchID,
                r.JournalEntry,
                l.TrxDate,
                r.[Reference],
                r.[Status],
                r.[Message],
                r.ReversalFlag,
                r.TrxDateBasis,
                r.ReversalDate,
                l.Account,
                l.AcctDesc,
                l.Amount,
                SUM(ISNULL(l.Debit,0)) AS DebitAmt,
                SUM(ISNULL(l.Credit,0)) AS CreditAmt,
                SUM(ISNULL(l.Debit,0) - ISNULL(l.Credit,0)) AS NetAmt,
                /* Values from GP */
                glh.JRNENTRY AS gp_Journal,
                glh.BACHNUMB AS gp_BatchNumber,
                glh.REFRENCE AS gp_Reference,
                glh.TRXDATE  AS gp_TrxDate,
                CASE 
                    WHEN glh.TRXTYPE = 0 THEN ''N''
                    WHEN glh.TRXTYPE = 1 THEN ''Y''
                END AS gp_ReversalFlag,  
                glh.PERIODID AS gp_PeriodID,
                ISNULL(fp.PeriodMonth,0) AS gp_PeriodMonth,
                glh.OPENYEAR AS gp_OpenYear,
                glh.REVPRDID AS gp_ReversalPeriodID,
                ISNULL(fp2.PeriodMonth,0) AS gp_ReversalPeriodMonth,
                glh.REVYEAR  AS gp_ReversalYear,
                gld.ACTINDX  AS gp_AccountIndex,
                act.ACTNUMST AS gp_Account,
                gld.DSCRIPTN AS gp_Description,
                gld.INTERID AS gp_Company,
                gld.DEBITAMT AS gp_DebitAmount,
                gld.CRDTAMNT AS gp_CreditAmount,
                gld.ORDBTAMT AS gp_OriginatingDebitAmount,
                gld.ORCRDAMT AS gp_OriginatingCreditAmount
            FROM dynamics.dbo.IA_Run r
            LEFT JOIN dynamics.dbo.IA_Line l ON l.RunID = r.RunID
            LEFT JOIN [' + @currentCompany + N'].dbo.GL10000 glh   ON glh.BACHNUMB = r.BatchID
            LEFT JOIN [' + @currentCompany + N'].dbo.GL10001 gld   ON glh.JRNENTRY = gld.JRNENTRY
            LEFT JOIN [' + @currentCompany + N'].dbo.GL00105 act   ON act.ACTINDX  = gld.ACTINDX
            LEFT JOIN dynamics.dbo.ia_fiscal_period fp  ON glh.PERIODID = fp.PeriodID
            LEFT JOIN dynamics.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
            WHERE r.RunID = @p_runID
            GROUP BY 
                CASE WHEN glh.TRXTYPE = 0 THEN ''N'' WHEN glh.TRXTYPE = 1 THEN ''Y'' END,
                r.JobID, l.runid, r.[FileName], r.CompanyA, r.BatchID, r.JournalEntry,
                r.[Reference], r.[Status], r.[Message], r.ReversalFlag, r.TrxDateBasis, r.ReversalDate,
                l.Amount, l.Account, l.AcctDesc, l.TrxDate,
                glh.JRNENTRY, glh.BACHNUMB, glh.REFRENCE, glh.TRXDATE,
                glh.PERIODID, glh.OPENYEAR, glh.REVPRDID, glh.REVYEAR,
                gld.ACTINDX, act.ACTNUMST, gld.DSCRIPTN, gld.INTERID,
                gld.DEBITAMT, gld.CRDTAMNT, gld.ORDBTAMT, gld.ORCRDAMT,
                fp.PeriodMonth, fp2.PeriodMonth
            HAVING 
                ABS(SUM(ISNULL(l.Debit,0)) + SUM(ISNULL(l.Credit,0))) > 0
                OR ABS(SUM(ISNULL(l.Debit,0) - ISNULL(l.Credit,0))) > 0.00001;
        ';

        EXEC sys.sp_executesql
            @stmt   = @sql,
            @params = @params,
            @p_runID = @currentRunID;

        FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;
    END

    CLOSE run_cursor;
    DEALLOCATE run_cursor;

    SELECT * FROM #AuditResults ORDER BY CompanyA, Account;

    DROP TABLE IF EXISTS #AuditResults;
END
GO



-----------










# ----------------------------
# UTILITIES
# ----------------------------
function Ensure-Directory {
    param([Parameter(Mandatory = $true)][string]$Path)
    if (-not (Test-Path -Path $Path)) {
        New-Item -ItemType Directory -Path $Path -Force | Out-Null
    }
}

<#
function Write-Log {
    param(
        [Parameter(Mandatory = $true)][string]$Message,
        [ValidateSet("INFO", "WARN", "ERROR", "SUCCESS")][string]$Level = "INFO"
    )
    $entry = "$(Get-Date -Format 'yyyyMMdd HH:mm:ss')[$Level] - $Message"
    Ensure-Directory -Path (Split-Path -Path $Paths.Log -Parent)
    $entry | Out-File -FilePath $Paths.Log -Append -Encoding UTF8
    Write-Host $entry
}
#>


# Write to log
function Write-Log{
    param(
        [Parameter(Mandatory=$true)]
        [String]$Message,
        [switch]$IncludeHeader,
        [switch]$IncludeFooter
    )

    $TimeStamp = Get-Date -Format "yyyyMMdd HH:mm:ss"
    
    if($IncludeHeader){
        # Log Username, Clustername, NodeName
        $User = $env:USERNAME
        $Hostname = $env:COMPUTERNAME
        $Node = [System.Environment]::MachineName
        $LogEntry = "$TimeStamp | User: $User | Hostname: $Hostname | Node: $Node `n$Timestamp | $Message"
    } else {
        # Log Message Only
        $LogEntry = "$TimeStamp | $Message"
    }

    # Add linebreak
    if($IncludeFooter){
        $LogEntry += "`n"
    }

    Add-Content -Path $Paths.Log -Value $LogEntry
}




function Move-File {
    param(
        [Parameter(Mandatory = $true)][string]$Source,
        [Parameter(Mandatory = $true)][string]$DestinationDir
    )
    try {
        Ensure-Directory -Path $DestinationDir
        $target = Join-Path -Path $DestinationDir -ChildPath (Split-Path -Path $Source -Leaf)
        return Move-Item -Path $Source -Destination $target -PassThru -Force -ErrorAction Stop
    }
    catch {
        Write-Log "Failed to move file [$Source] to [$DestinationDir]. Error: $($_.Exception.Message)"
        return $null
    }
}

function Invoke-IntracompanyImportSP {
    param([Parameter(Mandatory = $true)][string]$CsvFullPath)

    $query = "EXEC $($SQLConfig.ProcName) @CsvPath = N'$CsvFullPath';"

    try {

        $SQLResult = @(Invoke-Sqlcmd -ServerInstance $SQLConfig.server -Database $SQLConfig.database -Query $query -TrustServerCertificate)
        $JobID = $SQLResult | Select-Object -ExpandProperty JobID
        if ($null -eq $JobID -or $SQLResult.Count -eq 0) {
            return "No JobID returned"
        }
        else {
            return $JobID
        }
        
    }
    catch {
        Write-Log "SQL/eConnect Error while processing [$CsvFullPath]: $($_.Exception.Message)"
        return $null
    }
}

function Get-JobResult {
    param([Parameter(Mandatory = $true)][string]$JobID)
  
    $query = @"
SELECT TOP 1 JobResult
FROM dbo.IA_Run
WHERE JobID = '$JobID'
ORDER BY StartedDT DESC;
"@
 
    try {
        
        $SQLResult = @(Invoke-Sqlcmd -ServerInstance $SQLConfig.server -Database $SQLConfig.database -Query $query -TrustServerCertificate)
        $JobResult = $SQLResult | Select-Object -ExpandProperty JobResult
        if ($null -eq $JobResult -or $SQLResult.Count -eq 0) {        
            return "No JobResult returned"
        }
        else {
            return $JobResult
        }

    }
    catch {
        Write-Log "Failed to query JobResult for JobID [$JobID]: $($_.Exception.Message)"
        return $null
    }
}


<# ================================================ #>
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [Guid]$JobId
    )

    # --- Configuration (edit for your environment) ---
    $SqlInstance = $SQLConfig.Server           # e.g. 'SQLPROD01'
    $DbName      = $SQLConfig.Database   # IA_* tables live here
    # -------------------------------------------------

    function Invoke-Db {
        param([string]$Query)
        $args = @{
            ServerInstance = $SqlInstance
            Database       = $DbName
            Query          = $Query
            ErrorAction    = 'Stop'
            TrustServerCertificate = $true
        }
       
        Invoke-Sqlcmd @args
    }

    function HtmlEncode([string]$s) {
        if ($null -eq $s) { return '' }
        return [System.Web.HttpUtility]::HtmlEncode($s)
    }

    $qRuns = @"
SELECT
    r.RunID,
    r.JobID,
    r.FileName,
    r.CompanyA,
    r.BatchID,
    r.JournalEntry,
    r.Reference,
    r.Status,
    r.Message,
    r.StartedDT,
    r.EndedDT,
    r.RowCountLine,
    r.JobResult,
    r.ReversalFlag,
    r.TrxDateBasis,
    TxDate = ISNULL(r.TrxDateBasis, (SELECT MIN(il.TrxDate) FROM [$DbName].dbo.IA_Line il WHERE il.RunID = r.RunID))
FROM [$DbName].dbo.IA_Run r
WHERE r.JobID = '$JobId'
ORDER BY r.StartedDT ASC;
"@

    try {
        $runs = Invoke-Db -Query $qRuns
    } catch {
        $err = $_.Exception.Message
        $subject = "Dynamics Import Job (Job ${JobId}) - ERROR"
        $body = "<html><body><pre>$([System.Web.HttpUtility]::HtmlEncode($err))</pre></body></html>"
        return [pscustomobject]@{ Subject = $subject; Body = $body }
    }

    if (-not $runs -or $runs.Count -eq 0) {
        $subject = "Dynamics Import Job (Job ${JobId}) - 0 Runs, Posted = 0, Failed = 0"
        $body = @"
<!doctype html>
<html><head><meta http-equiv='Content-Type' content='text/html; charset=utf-8'></head>
<body style="font-family:Segoe UI,Arial,sans-serif;color:#111;padding:12px">
  <h2 style="margin:0 0 8px 0">Dynamics Import Summary</h2>
  <div>Filename: <em>(none)</em></div>
  <div>Result: <strong>ERROR</strong></div>
  <p style="margin-top:12px">No runs exist for JobID <code>${JobId}</code> in <strong>$DbName</strong>.</p>
</body></html>
"@
        return [pscustomobject]@{ Subject = $subject; Body = $body }
    }

    # -------------------------
    # Derive job-level stats
    # -------------------------
    $runsCount   = $runs.Count
    $postedCount = ($runs | Where-Object { $_.Status -eq 'POSTED' }).Count
    $failedCount = ($runs | Where-Object { $_.Status -eq 'FAILED' }).Count

    $jobResult = ($runs | Select-Object -First 1).JobResult
    if ([string]::IsNullOrWhiteSpace($jobResult)) {
        if     ($postedCount -eq $runsCount) { $jobResult = 'SUCCESS' }
        elseif ($failedCount -eq $runsCount) { $jobResult = 'ERROR' }
        elseif ($failedCount -gt 0 -and $postedCount -gt 0) { $jobResult = 'PARTIAL' }
        else { $jobResult = 'IN-PROGRESS' }
    }

    $filename = $runs | Select-Object -ExpandProperty FileName -Unique

    # -------------------------
    # Load per-run line items
    # -------------------------
    # We’ll build a map: RunID -> DataTable of aggregated lines by Account
    $linesByRun = @{}
    foreach ($r in $runs) {
        $rid = $r.RunID
        $qLines = @"
SELECT
    il.Account,
    SUM(ISNULL(il.Debit,0))  AS DebitAmt,
    SUM(ISNULL(il.Credit,0)) AS CreditAmt,
    SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0)) AS NetAmt
FROM [$DbName].dbo.IA_Line il
WHERE il.RunID = '$rid'
GROUP BY il.Account
HAVING ABS(SUM(ISNULL(il.Debit,0)) + SUM(ISNULL(il.Credit,0))) > 0
   OR ABS(SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0))) > 0.00001
ORDER BY il.Account;
"@
        try {
            $dt = Invoke-Db -Query $qLines
        } catch {
            $dt = @()
        }
        $linesByRun[$rid] = $dt
    }

    # -------------------------
    # Build HTML (with stylesheet)
    # -------------------------
    $subject = "Dynamics Import Job [${fileName}] - ${runsCount} Runs, Posted = ${postedCount} , Failed = ${failedCount}"

    $sb = New-Object System.Text.StringBuilder
    [void]$sb.AppendLine("<!doctype html>")
    [void]$sb.AppendLine("<html>")
    [void]$sb.AppendLine("<head>")
    [void]$sb.AppendLine("<meta http-equiv='Content-Type' content='text/html; charset=utf-8'/>")
    [void]$sb.AppendLine("<style type='text/css'>")
    [void]$sb.AppendLine("body{font-family:Segoe UI,Arial,sans-serif;color:#111;margin:0;padding:12px}")
    [void]$sb.AppendLine(".h{margin:0 0 8px 0}")
    [void]$sb.AppendLine(".p{margin:2px 0}")
    [void]$sb.AppendLine(".badge{display:inline-block;padding:2px 6px;border-radius:10px;font-weight:600;font-size:11px}")
    [void]$sb.AppendLine(".b-success{background:#e6f4ea;color:#0b6e2f}.b-error{background:#fdecea;color:#b42318}.b-partial{background:#fff8e1;color:#8a5b00}.b-progress{background:#eef3ff;color:#1a4fb5}")
    [void]$sb.AppendLine("table{border-collapse:collapse;border-spacing:0;mso-table-lspace:0pt;mso-table-rspace:0}")
    [void]$sb.AppendLine("table.summary{width:100%;table-layout:fixed;border:1px solid #d9d9d9;font-size:11px;line-height:1.1;margin:8px 0 12px 0}")
    [void]$sb.AppendLine("table.summary th{background:#f3f3f3;padding:2px 6px;text-align:left;font-weight:600;border:0;border-bottom:1px solid #d9d9d9;white-space:nowrap}")
    [void]$sb.AppendLine("table.summary td{padding:2px 6px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;border:1px solid transparent}")
    [void]$sb.AppendLine(".num{text-align:right}")
    [void]$sb.AppendLine(".w-co{width:14%}.w-bt{width:14%}.w-je{width:8%}.w-rf{width:24%}.w-dt{width:10%}.w-cnt{width:7%}.w-st{width:11%}.w-rv{width:6%}.w-err{width:6%}")
    [void]$sb.AppendLine("tr.posted td{background:#e6f4ea;border-color:#e6f4ea}")
    [void]$sb.AppendLine("tr.failed td{background:#fdecea;border-color:#fdecea}")
    [void]$sb.AppendLine("tr.loaded td{background:#fff8e1;border-color:#fff8e1}")
    [void]$sb.AppendLine("tr.other td{background:#eef3ff;border-color:#eef3ff}")
    [void]$sb.AppendLine(".co{margin-top:12px}")
    [void]$sb.AppendLine(".co h3{margin:8px 0 6px 0}")
    [void]$sb.AppendLine(".meta{margin:0 0 6px 0;font-size:12px}")
    [void]$sb.AppendLine("table.lines{width:100%;table-layout:fixed;border:1px solid #e0e0e0;font-size:11px;line-height:1.1;margin:2px 0 10px 0}")
    [void]$sb.AppendLine("table.lines th{background:#f7f7f7;padding:2px 6px;text-align:left;font-weight:600;border:0;border-bottom:1px solid #e0e0e0}")
    [void]$sb.AppendLine("table.lines td{padding:2px 6px;border-top:1px solid #f0f0f0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}")
    [void]$sb.AppendLine(".foot{margin-top:14px;font-size:10px;color:#666;border-top:1px solid #ddd;padding-top:6px;font-family:Consolas,monospace}")
    [void]$sb.AppendLine("</style>")
    [void]$sb.AppendLine("</head>")
    [void]$sb.AppendLine("<body>")

    # Header
    [void]$sb.AppendLine("<h2 class='h'>Dynamics Import Summary</h2>")
    [void]$sb.AppendLine("<div class='p'>Filename: <code>$(HtmlEncode $fileDisplay)</code></div>")

    $badgeClass = if ($jobResult -eq 'SUCCESS') { 'b-success' } elseif ($jobResult -eq 'ERROR') { 'b-error' } elseif ($jobResult -eq 'PARTIAL') { 'b-partial' } else { 'b-progress' }
    [void]$sb.AppendLine("<div class='p'>Result: <span class='badge $badgeClass'>$(HtmlEncode $jobResult)</span></div>")
    [void]$sb.AppendLine("<div class='p'>Job ID: <span class='badge $badgeClass'>$(HtmlEncode $JobId)</span></div>")

    # Overall summary table
    [void]$sb.AppendLine("<table class='summary' cellspacing='0' cellpadding='0'>")
    [void]$sb.AppendLine("<thead><tr>")
    [void]$sb.AppendLine("<th class='w-co'>Company</th>")
    [void]$sb.AppendLine("<th class='w-bt'>Batch</th>")
    [void]$sb.AppendLine("<th class='w-je'>JE</th>")
    [void]$sb.AppendLine("<th class='w-rf'>Reference</th>")
    [void]$sb.AppendLine("<th class='w-dt'>TrxDate</th>")
    [void]$sb.AppendLine("<th class='w-cnt'>#Rows</th>")
    [void]$sb.AppendLine("<th class='w-st'>RunStatus</th>")
    [void]$sb.AppendLine("<th class='w-rv'>Reversal</th>")
    [void]$sb.AppendLine("<th class='w-err'>Error</th>")
    [void]$sb.AppendLine("</tr></thead><tbody>")

    foreach ($r in $runs) {
        $rowClass = switch (([string]$r.Status).ToUpperInvariant()) {
            'POSTED' { 'posted' }
            'FAILED' { 'failed' }
            'LOADED' { 'loaded' }
            default  { 'other' }
        }
        $company  = HtmlEncode $r.CompanyA
        $batch    = HtmlEncode $r.BatchID
        $je       = if ($r.JournalEntry) { HtmlEncode ([string]$r.JournalEntry) } else { '<none>' }
        $ref      = HtmlEncode $r.Reference
        $txDate   = if ($r.TxDate) { ([DateTime]$r.TxDate).ToString('yyyy-MM-dd') } else { '' }
        $rowsOut  = if ($r.RowCountLine -ne $null) { [int]$r.RowCountLine } else { 0 }
        $status   = HtmlEncode $r.Status
        $revYN    = if ($r.ReversalFlag -eq 'Y') { 'Y' } elseif ($r.ReversalFlag -eq 'N') { 'N' } else { '' }
        $errMsg   = if ($r.Status -eq 'FAILED' -and $r.Message) { HtmlEncode $r.Message } else { '' }

        [void]$sb.AppendLine("<tr class='$rowClass'>")
        [void]$sb.AppendLine("<td class='w-co'>$company</td>")
        [void]$sb.AppendLine("<td class='w-bt'><code>$batch</code></td>")
        [void]$sb.AppendLine("<td class='w-je num'>$je</td>")
        [void]$sb.AppendLine("<td class='w-rf'>$ref</td>")
        [void]$sb.AppendLine("<td class='w-dt'>$txDate</td>")
        [void]$sb.AppendLine("<td class='w-cnt num'>$rowsOut</td>")
        [void]$sb.AppendLine("<td class='w-st'>$status</td>")
        [void]$sb.AppendLine("<td class='w-rv'>$revYN</td>")
        [void]$sb.AppendLine("<td class='w-err'>$errMsg</td>")
        [void]$sb.AppendLine("</tr>")
    }

    [void]$sb.AppendLine("</tbody></table>")

    # --- Line break / separator ---
    [void]$sb.AppendLine("<div style='height:8px'></div>")

    # -------------------------
    # Per-company detail blocks
    # -------------------------
    foreach ($r in $runs) {
        $company  = [string]$r.CompanyA
        $jeStr    = if ($r.JournalEntry) { [string]$r.JournalEntry } else { '<none>' }
        $txDate   = if ($r.TxDate) { ([DateTime]$r.TxDate).ToString('yyyy-MM-dd') } else { '' }
        $batch    = [string]$r.BatchID
        $ref      = [string]$r.Reference
        $runid    = [guid]$r.runid

        [void]$sb.AppendLine("<div class='co'>")
        [void]$sb.AppendLine("<h3>$(HtmlEncode $company)</h3>")
        
        <# #>

        # --- 2-column metadata table (no borders) ---
        # --- 2-column LABEL | VALUE table (no borders) ---
        [void]$sb.AppendLine("<table role='presentation' cellspacing='0' cellpadding='0' style='border-collapse:collapse;table-layout:fixed;width:60%;font-size:12px'>")

        [void]$sb.AppendLine("<tr>")
        [void]$sb.AppendLine("  <td style='padding:2px 8px 2px 0;vertical-align:top;width:10%;white-space:nowrap'><strong>Journal Entry</strong></td>")
        [void]$sb.AppendLine("  <td style='padding:2px 0 2px 8px;vertical-align:top;width:90%'>$(HtmlEncode $jeStr)</td>")
        [void]$sb.AppendLine("</tr>")

        [void]$sb.AppendLine("<tr>")
        [void]$sb.AppendLine("  <td style='padding:2px 8px 2px 0;vertical-align:top;white-space:nowrap'><strong>TxDate</strong></td>")
        [void]$sb.AppendLine("  <td style='padding:2px 0 2px 8px;vertical-align:top'>$(HtmlEncode $txDate)</td>")
        [void]$sb.AppendLine("</tr>")

        [void]$sb.AppendLine("<tr>")
        [void]$sb.AppendLine("  <td style='padding:2px 8px 2px 0;vertical-align:top;white-space:nowrap'><strong>BatchID</strong></td>")
        [void]$sb.AppendLine("  <td style='padding:2px 0 2px 8px;vertical-align:top'><code>$(HtmlEncode $batch)</code></td>")
        [void]$sb.AppendLine("</tr>")

        [void]$sb.AppendLine("<tr>")
        [void]$sb.AppendLine("  <td style='padding:2px 8px 2px 0;vertical-align:top;white-space:nowrap'><strong>Reference</strong></td>")
        [void]$sb.AppendLine("  <td style='padding:2px 0 2px 8px;vertical-align:top;white-space:normal;word-break:break-word'>$(HtmlEncode $ref)</td>")
        [void]$sb.AppendLine("</tr>")

        [void]$sb.AppendLine("<tr>")
        [void]$sb.AppendLine("  <td style='padding:2px 8px 2px 0;vertical-align:top;white-space:nowrap'><strong>JobID</strong></td>")
        [void]$sb.AppendLine("  <td style='padding:2px 0 2px 8px;vertical-align:top;white-space:normal;word-break:break-word'>$(HtmlEncode $runid)</td>")
        [void]$sb.AppendLine("</tr>")

        [void]$sb.AppendLine("</table>")


        $dt = $linesByRun[$($r.RunID)]
        if ($dt -and $dt.Count -gt 0) {
            [void]$sb.AppendLine("<table class='lines' cellspacing='0' cellpadding='0'>")
            [void]$sb.AppendLine("<thead><tr>")
                [void]$sb.AppendLine("<th style='width:46%'>Account</th>")
                [void]$sb.AppendLine("<th class='num';style='width:18%;text-align:right'>Debit</th>")
                [void]$sb.AppendLine("<th class='num';style='width:18%;text-align:right'>Credit</th>")
                [void]$sb.AppendLine("<th class='num';style='width:18%;text-align:right'>Total</th>")
            [void]$sb.AppendLine("</tr></thead><tbody>")

            $sumD = 0.0; $sumC = 0.0; $sumN = 0.0
            foreach ($row in $dt) {
                $acct = HtmlEncode ([string]$row.Account)
                $d = [decimal]$row.DebitAmt
                $c = [decimal]$row.CreditAmt
                $n = [decimal]$row.NetAmt
                $sumD += $d; $sumC += $c; $sumN += $n
                $dOut = ('{0:N2}' -f $d); $cOut = ('{0:N2}' -f $c); $nOut = ('{0:N2}' -f $n)
                [void]$sb.AppendLine("<tr><td><code>$acct</code></td><td class='num'>$dOut</td><td class='num'>$cOut</td><td class='num'>$nOut</td></tr>")
            }

            $sd = ('{0:N2}' -f $sumD); $sc = ('{0:N2}' -f $sumC); $sn = ('{0:N2}' -f $sumN)
            [void]$sb.AppendLine("<tr><td style='font-weight:600'>Totals</td><td class='num' style='font-weight:600'>$sd</td><td class='num' style='font-weight:600'>$sc</td><td class='num' style='font-weight:600'>$sn</td></tr>")
            [void]$sb.AppendLine("</tbody></table>")
        } else {
            [void]$sb.AppendLine("<div class='p' style='color:#666'>No line items.</div>")
        }

        [void]$sb.AppendLine("</div>")
    }

    # Footer: JobID + RunIDs
    [void]$sb.AppendLine("<div class='foot'>SQL Debug<code></code></div>")
    [void]$sb.AppendLine("<div class='foot'>SELECT * FROM dynamics.dbo.IA_Run r JOIN dynamics.dbo.IA_RunEvent re ON r.RunID = re.RunID WHERE r.JobID = '$jobid'</div>")
    #foreach ($r in $runs) {
    #    [void]$sb.AppendLine("<div class='foot'><code>$($r.RunID)</code></div>")
    #}

    [void]$sb.AppendLine("</body></html>")

    [pscustomobject]@{
        Subject = $subject
        Body    = $sb.ToString()
    }
}


<# ================================================ #>

function Send-JobEmail {
    param(
        [Parameter(Mandatory = $true)][string]$Subject,
        [Parameter(Mandatory = $true)][string]$Body
    )

    try {
    
        Send-MailMessage -SmtpServer $EmailConfig.Server -from $EmailConfig.From -To $EmailConfig.To -Subject $Subject -Body $Body -BodyAsHtml
        return $true

    }
    catch {
        Write-Log "Email send failed: $($_.Exception.Message)" 
        return $false
    }
}

# ----------------------------
# MAIN
# ----------------------------
function Invoke-Main {

    $Paths.Keys | ForEach-Object {
        if ($_ -ne 'Log') { Ensure-Directory -Path $Paths[$_] }
    }

    $files = @(Get-ChildItem -Path $Paths.In -Filter *.csv -File -ErrorAction SilentlyContinue)
    if ($files.Count -eq 0) { return }

    Write-Log "-- Import Begin --" -IncludeHeader

    foreach ($file in $files) {
        Write-Log "Found file: $($file.Name)" 

        $working = Move-File -Source $file.FullName -DestinationDir $Paths.Proc
        if ($null -eq $working) {
            Write-Log "Skipping: file locked/in use or failed to move. File: $($file.Name)"
            continue
        }

        try {
            Write-Log "Starting SQL import for: $($working.FullName)" 

            $jobId = Invoke-IntracompanyImportSP -CsvFullPath $working.FullName
            if (-not $jobId) {
                Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                Write-Log "No JobID returned. Moved to error: $($file.Name)" 
                continue
            }

            $jobResult = Get-JobResult -JobID $jobId
            if (-not $jobResult) {
                Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                Write-Log "JobResult not found for JobID [$jobId]. Moved to error: $($file.Name)"
                continue
            }

            switch ($jobResult.ToUpperInvariant()) {
                'SUCCESS' {
                    $archived = Move-File -Source $working.FullName -DestinationDir $Paths.Archive
                    if ($null -ne $archived) {
                        Write-Log "Success: $($file.Name) → archived to $($Paths.Archive)"
                    }
                }
                'PARTIAL' {
                    $partialDir = if ($Paths.ContainsKey('Partial')) { $Paths.Partial } else { $Paths.ErrorDir }
                    Move-File -Source $working.FullName -DestinationDir $partialDir | Out-Null
                    Write-Log "Partial result for JobID [$jobId]. Moved to: $partialDir"
                }
                default {
                    Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                    Write-Log "Error result for JobID [$jobId]. Moved to error: $($file.Name)" 
                }
            }

        }
        catch {
            Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
            Write-Log "Unhandled failure for $($file.Name): $($_.Exception.Message)"
        }
    }

    Write-Log "Composing Email"
    $subject = $null
    
    $JobDetails = New-DynamicsImportJobEmailContent -JobId $jobId
    $Subject    = $JobDetails.Subject
    $Body       = $JobDetails.Body

    Send-JobEmail -Subject $Subject -Body $Body
    
    write-log "Email Sent"
    Write-Log "-- Import End --" -IncludeFooter
}

Function Invoke-Debug {
    param(
        [Parameter(Mandatory = $false)][string]$JobID = '2990B5FC-6DE7-43A6-9958-CE0E3EAB62DC',
        [Parameter(Mandatory = $false)][bool]$SendEmail = 0
    )

    write-host $JobID

    $JobDetails = New-DynamicsImportJobEmailContent -JobId $jobId
    $Subject    = $JobDetails.Subject
    $Body       = $JobDetails.Body

    Send-JobEmail -Subject $Subject -Body $Body

}

#Invoke-Main
$JobID = 'ffda0da6-cae8-487c-b297-0628ba209fcf'
Invoke-Debug -JobID $JobID




CREATE TABLE [dbo].[IA_FileStage] (
    [Company]   VARCHAR (20)  NULL,
    [Account]   VARCHAR (129) NULL,
    [AcctDesc]  VARCHAR (255) NULL,
    [Amount]    VARCHAR (50)  NULL,
    [TrxDate]   VARCHAR (30)  NULL,
    [Reference] VARCHAR (31)  NULL,
    [Reversal_Flag]   CHAR(1) NULL
);
GO

CREATE TABLE [dbo].[IA_Line] (
    [LineID]    INT              IDENTITY (1, 1) NOT NULL,
    [RunID]     UNIQUEIDENTIFIER NOT NULL,
    [TrxDate]   DATE             NOT NULL,
    [CompanyID] VARCHAR (10)     NOT NULL,
    [Account]   CHAR (129)       NOT NULL,
    [AcctDesc]  VARCHAR (255)    NULL,
    [Amount]    NUMERIC (19, 5)  NOT NULL,
    [Debit]     NUMERIC (19, 5)  DEFAULT ((0)) NOT NULL,
    [Credit]    NUMERIC (19, 5)  DEFAULT ((0)) NOT NULL,
    PRIMARY KEY CLUSTERED ([LineID] ASC)
);
GO

CREATE TABLE [dbo].[IA_RawCsv] (
    [Company]   VARCHAR (20)  NULL,
    [Account]   VARCHAR (129) NULL,
    [AcctDesc]  VARCHAR (255) NULL,
    [Amount]    VARCHAR (50)  NULL,
    [TrxDate]   VARCHAR (30)  NULL,
    [Reference] VARCHAR (31)  NULL,
    [Reversal_Flag] CHAR(1) NULL
);
GO

CREATE TABLE dbo.IA_Run
(
    -- Keys
    RunID           UNIQUEIDENTIFIER NOT NULL,
    JobID           UNIQUEIDENTIFIER NULL,        -- groups all company runs from the same file
    PRIMARY KEY CLUSTERED (RunID ASC),

    -- File context
    FileName        NVARCHAR(260)    NOT NULL,
    FileSection     NVARCHAR(260)    NOT NULL,
    FilePath        NVARCHAR(4000)   NOT NULL,

    -- Posting context
    BatchID         CHAR(15)         NOT NULL,
    Reference       VARCHAR(30)      NOT NULL,
    JournalEntry    int null,

    -- Lifecycle
    Status          VARCHAR(20)      NOT NULL DEFAULT ('LOADED'),  -- LOADING | LOADED | POSTED | FAILED
    StartedDT       DATETIME         NOT NULL DEFAULT (GETDATE()),
    EndedDT         DATETIME         NULL,
    Message         VARCHAR(4000)    NULL,

    -- Counts & metrics
    RowCountRaw     INT              NULL,
    RowCountLine    INT              NULL,
    CompanyA        VARCHAR(10)      NULL,
    NetAmount       NUMERIC(19,5)    NULL,

    -- Reversal Definitions
    ReversalFlag  CHAR(1)  NULL,
    TrxDateBasis  DATE     NULL,
    ReversalDate  DATE     NULL,

    -- Content identity
    FileSignature   VARBINARY(32)    NULL,

    -- <<< SIMPLE JOB-LEVEL RESULT FOR POWERSHELL >>>
    -- SQL will set this once per JobID after all company runs complete:
    -- SUCCESS | ERROR | PARTIAL (PowerShell only checks this to move the file)
    JobResult       VARCHAR(20)      NULL
);
GO
CREATE TABLE [dbo].[IA_RunEvent] (
    [EventID]  INT              IDENTITY (1, 1) NOT NULL,
    [RunID]    UNIQUEIDENTIFIER NOT NULL,
    [EventDT]  DATETIME         DEFAULT (getdate()) NOT NULL,
    [StepName] VARCHAR (50)     NOT NULL,
    [Message]  VARCHAR (4000)   NOT NULL,
    PRIMARY KEY CLUSTERED ([EventID] ASC)
);
GO









/* 3) Signature on IA_RawCsv */
CREATE   PROCEDURE dbo.usp_IA_ComputeRawCsvSig
(
    @RunID  uniqueidentifier,
    @OutSig varbinary(32) OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT EXISTS (SELECT 1 FROM dbo.IA_RawCsv)
    BEGIN
        DECLARE @msg varchar(200) = 'ComputeRawCsvSig: IA_RawCsv is empty; did BULK INSERT load any rows?';
        THROW 53100, @msg, 1;
    END


    -- Determine whether this run is reversal (from RawCsv)
    DECLARE @isRev BIT =
    CASE
        WHEN EXISTS (SELECT 1 FROM dbo.IA_RawCsv WHERE UPPER(LTRIM(RTRIM(Reversal_Flag))) = 'Y') THEN 1
        ELSE 0
    END;

    DECLARE @payload nvarchar(max)

    IF @isRev = 1
        BEGIN
            -- Include the flag and a derived per-row ReversalDate (first day next month)
            -- TRY_CONVERT for TrxDate as RawCsv stores it as VARCHAR.
            SELECT @payload =
            (
                SELECT
                    Company,
                    Account,
                    AcctDesc,
                    Amount,
                    TrxDate,
                    Reference,
                    Reversal_Flag = UPPER(LTRIM(RTRIM(Reversal_Flag))),
                    ReversalDateDerived = CONVERT(char(10),
                        DATEADD(DAY, 1, EOMONTH(TRY_CONVERT(date, TrxDate))), 120)
                FROM dbo.IA_RawCsv
                ORDER BY Company, Account, TrxDate, AcctDesc, Amount, Reference, UPPER(LTRIM(RTRIM(Reversal_Flag)))
                FOR JSON PATH, INCLUDE_NULL_VALUES
            );
        END
        ELSE
        BEGIN
            -- Original payload (no reversal fields)
            SELECT @payload =
            (
                SELECT Company, Account, AcctDesc, Amount, TrxDate, Reference
                FROM dbo.IA_RawCsv
                ORDER BY Company, Account, TrxDate, AcctDesc, Amount, Reference
                FOR JSON PATH, INCLUDE_NULL_VALUES
            );
        END

    SET @OutSig = HASHBYTES('SHA2_256', CAST(@payload AS varbinary(max)));

    UPDATE dbo.IA_Run
    SET FileSignature = @OutSig
    WHERE RunID = @RunID;


    DECLARE @hex varchar(66) = CONVERT(varchar(66), @OutSig, 1);
    DECLARE @Message VARCHAR(4000) =
        CASE WHEN @isRev=1
             THEN CONCAT('RawCsvSig=', @hex, ' (includes Reversal_Flag + derived ReversalDate)')
             ELSE CONCAT('RawCsvSig=', @hex)
        END;

    EXEC dbo.usp_IA_Log @RunID, 'SIG', @Message;

END
GO


/* 4) Duplicate guard (per FileName + signature) */
CREATE   PROCEDURE dbo.usp_IA_DuplicateGuard
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @FileName nvarchar(260), @Sig varbinary(32), @lockName nvarchar(500), @lockResult int;

    SELECT @FileName = FileName, @Sig = FileSignature
    FROM dbo.IA_Run WHERE RunID = @RunID;

    IF @Sig IS NULL
    BEGIN
        DECLARE @msgNull varchar(200) = 'DuplicateGuard: FileSignature is NULL (compute after load).';
        THROW 53110, @msgNull, 1;
    END

    SET @lockName = CONCAT('IA_DUP_', @FileName, '_', CONVERT(varchar(66), @Sig, 1));

    EXEC @lockResult = sp_getapplock
        @Resource   = @lockName,
        @LockMode   = 'Exclusive',
        @LockOwner  = 'Session',
        @LockTimeout= 10000;

    IF @lockResult < 0
    BEGIN
        DECLARE @msgLock varchar(200) = 'DuplicateGuard: Unable to acquire lock; try again.';
        THROW 53112, @msgLock, 1;
    END

    IF EXISTS
    (
        SELECT 1
        FROM dbo.IA_Run
        WHERE FileName = @FileName
          AND FileSignature = @Sig
          AND RunID <> @RunID
          AND Status IN ('LOADED','POSTED')
    )
    BEGIN
        DECLARE @Message VARCHAR(4000) = CONCAT('Duplicate detected for FileName=', @FileName, ' with same normalized content.');
        EXEC dbo.usp_IA_Log @RunID, 'DUPCHK', @Message;

        DECLARE @msgDup varchar(200) = 'Duplicate file+content detected; rename the file to re-run.';
        THROW 53111, @msgDup, 1;
    END

    EXEC dbo.usp_IA_Log @RunID, 'DUPCHK', 'No prior imports found with same FileName + normalized content.';
END
GO

CREATE PROCEDURE dbo.usp_IA_File_Process
(
    @CsvPath NVARCHAR(4000),
    @RunID UNIQUEIDENTIFIER OUTPUT,
    @UseStaged BIT = 0,
    @FileNameSuffix NVARCHAR(64) = NULL,
    @BatchPrefix VARCHAR(8) = 'GLTX',
    @JobID UNIQUEIDENTIFIER = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        DECLARE @FileName NVARCHAR(260) =
            RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

        DECLARE @BatchID CHAR(15);
        EXEC DYNAMICS.dbo.usp_IA_GetNextBatchID_Seq @SeriesPrefix=@BatchPrefix, @BatchID=@BatchID OUTPUT;

        IF (@UseStaged = 0)
        BEGIN
            EXEC dbo.usp_IA_LoadCsv
                @CsvPath=@CsvPath,
                @BatchID=@BatchID,
                @RunID=@RunID OUTPUT,
                @JobID=@JobID;
        END
        ELSE
        BEGIN
            SET @RunID = NEWID();

            DECLARE @FileSection NVARCHAR(260) =
                LEFT(@FileName + ISNULL(@FileNameSuffix, N''), 260);

            INSERT dbo.IA_Run (RunID, JobID, FileName, FileSection, FilePath, BatchID, Reference, Status)
            VALUES (@RunID, @JobID, @FileName, @FileSection, @CsvPath, @BatchID, '', 'LOADING');

            EXEC dbo.usp_IA_Log @RunID, 'LOAD', 'Using pre-staged IA_RawCsv (by-company run).';

            DECLARE @RawSig VARBINARY(32);
            EXEC dbo.usp_IA_ComputeRawCsvSig @RunID=@RunID, @OutSig=@RawSig OUTPUT;

            UPDATE dbo.IA_Run
            SET RowCountRaw = (SELECT COUNT(*) FROM dbo.IA_RawCsv)
            WHERE RunID=@RunID;

            INSERT dbo.IA_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
            SELECT
                @RunID,
                TRY_CONVERT(date, TrxDate),
                RTRIM(LTRIM(Company)),
                CONVERT(char(129), RTRIM(LTRIM(Account))),
                AcctDesc,
                TRY_CONVERT(numeric(19,5), REPLACE(Amount, ',', ''))
            FROM dbo.IA_RawCsv;

            DELETE dbo.IA_Line
            WHERE RunID=@RunID
              AND (Amount=0 OR TrxDate IS NULL OR CompanyID='' OR Account='');

            UPDATE dbo.IA_Run
            SET RowCountLine = (SELECT COUNT(*) FROM dbo.IA_Line WHERE RunID=@RunID)
            WHERE RunID=@RunID;

            DECLARE @Ref VARCHAR(30) =
            (
                SELECT TOP (1) LEFT(RTRIM(LTRIM(Reference)), 30)
                FROM dbo.IA_RawCsv
                WHERE Reference IS NOT NULL AND LTRIM(RTRIM(Reference)) <> ''
            );
            IF (@Ref IS NULL OR @Ref = '')
                SET @Ref = LEFT(@FileName, 30);

            UPDATE dbo.IA_Run SET Reference=@Ref, Status='LOADED' WHERE RunID=@RunID;
        END

        EXEC dbo.usp_IA_Normalize      @RunID=@RunID;
        EXEC dbo.usp_IA_DuplicateGuard @RunID=@RunID;
        EXEC dbo.usp_IA_ValidateRun    @RunID=@RunID;
        EXEC dbo.usp_IA_PostToGP       @RunID=@RunID;

        UPDATE dbo.IA_Run SET Status='POSTED', EndedDT=GETDATE(), Message='OK' WHERE RunID=@RunID;
        EXEC dbo.usp_IA_Log @RunID, 'DONE', 'Posted successfully';
    END TRY
    BEGIN CATCH
        DECLARE @msg VARCHAR(4000) = ERROR_MESSAGE();
        IF @RunID IS NOT NULL
        BEGIN
            UPDATE dbo.IA_Run SET Status='FAILED', EndedDT=GETDATE(), Message=@msg WHERE RunID=@RunID;
            EXEC dbo.usp_IA_Log @RunID, 'ERROR', @msg;
        END
        ;THROW;
    END CATCH
END
GO
CREATE PROCEDURE dbo.usp_IA_File_Process_ByCompany
(
    @CsvPath NVARCHAR(4000)
)
AS
BEGIN
    SET NOCOUNT ON;


    DECLARE @JobID UNIQUEIDENTIFIER = NEWID();
    DECLARE @sql NVARCHAR(MAX);
    DECLARE @FileName NVARCHAR(260);

    -- Extract filename safely
    IF CHARINDEX('\', @CsvPath) > 0
        SET @FileName = RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);
    ELSE
        SET @FileName = @CsvPath;

    -- Start clean (shared staging)
    TRUNCATE TABLE dbo.IA_FileStage;

    -- BULK load entire file to stage (shared for all companies in this file)
    BEGIN TRY
        SET @sql = N'
            BULK INSERT DYNAMICS.dbo.IA_FileStage
            FROM ' + QUOTENAME(@CsvPath,'''') + N'
            WITH (
                FIRSTROW       = 2,
                FIELDTERMINATOR= '','',
                ROWTERMINATOR  = ''0x0d0a'',
                FIELDQUOTE     = ''"'',
                TABLOCK
            );';
        EXEC (@sql);
    END TRY
    BEGIN CATCH
        -- Record a failed run tied to this JobID so the job can be evaluated by the caller
        DECLARE @bulkErr VARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @tmpRun UNIQUEIDENTIFIER = NEWID();

        INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status, Message)
        VALUES (
            @tmpRun, @JobID,
            @FileName,
            @CsvPath,
            'BULKFAIL', '', 'FAILED', @bulkErr
        );

        EXEC dbo.usp_IA_Log @tmpRun, 'ERROR', @bulkErr;

        -- Set overall JobResult immediately (ERROR) and return only JobID
        UPDATE dbo.IA_Run SET JobResult = 'ERROR' WHERE JobID = @JobID;

        SELECT @JobID AS JobID;
        RETURN;
    END CATCH;

    IF NOT EXISTS (SELECT 1 FROM dbo.IA_FileStage)
    BEGIN
        -- Ensure at least one run row exists for the job to be visible
        DECLARE @tmpRun2 UNIQUEIDENTIFIER = NEWID();
        INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status, Message)
        VALUES (@tmpRun2, @JobID, @FileName, @CsvPath, 'EMPTY', '', 'FAILED', 'No rows loaded from CSV.');

        -- Set overall JobResult immediately (ERROR) and return only JobID
        UPDATE dbo.IA_Run SET JobResult = 'ERROR' WHERE JobID = @JobID;

        SELECT @JobID AS JobID;
        RETURN;
    END

    
----------------------------------------------------------------
    -- NEW: Enforce file-level uniformity of Reversal_Flag (Y or N)
    ----------------------------------------------------------------
    ;WITH s AS
    (
        SELECT DISTINCT UPPER(LTRIM(RTRIM(Reversal_Flag))) AS F
        FROM dbo.IA_FileStage
    )
    SELECT * INTO #Flags FROM s;

    -- Reject NULL/blank/invalid and mixtures
    IF EXISTS (SELECT 1 FROM #Flags WHERE F NOT IN ('Y','N'))
    BEGIN
        DECLARE @tmpRunBad UNIQUEIDENTIFIER = NEWID();
        INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status, Message)
        VALUES (@tmpRunBad, @JobID, @FileName, @CsvPath, 'BADFLAG', '', 'FAILED', 'Reversal_Flag must be Y or N for all rows.');
        UPDATE dbo.IA_Run SET JobResult = 'ERROR' WHERE JobID = @JobID;
        SELECT @JobID AS JobID;
        RETURN;
    END

    DECLARE @flagCount INT = (SELECT COUNT(*) FROM #Flags);
    IF @flagCount <> 1
    BEGIN
        DECLARE @tmpRunMix UNIQUEIDENTIFIER = NEWID();
        INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status, Message)
        VALUES (@tmpRunMix, @JobID, @FileName, @CsvPath, 'MIXFLAG', '', 'FAILED', 'File contains mixed Reversal_Flag values; file must be all Y or all N.');
        UPDATE dbo.IA_Run SET JobResult = 'ERROR' WHERE JobID = @JobID;
        SELECT @JobID AS JobID;
        RETURN;
    END

    DECLARE @FileReversalFlag CHAR(1) = (SELECT TOP 1 F FROM #Flags);

    -- Build company list from staged rows
    DECLARE @Companies TABLE (Company VARCHAR(20) PRIMARY KEY);
    INSERT @Companies(Company)
    SELECT DISTINCT RTRIM(LTRIM(Company))
    FROM dbo.IA_FileStage
    WHERE Company IS NOT NULL
      AND LTRIM(RTRIM(Company)) <> '';

    IF NOT EXISTS (SELECT 1 FROM @Companies)
    BEGIN
        DECLARE @tmpRun3 UNIQUEIDENTIFIER = NEWID();
        INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status, Message)
        VALUES (@tmpRun3, @JobID, @FileName, @CsvPath, 'NOCOMP', '', 'FAILED', 'No valid Company values found in CSV.');

        -- Set overall JobResult immediately (ERROR) and return only JobID
        UPDATE dbo.IA_Run SET JobResult = 'ERROR' WHERE JobID = @JobID;

        SELECT @JobID AS JobID;
        RETURN;
    END

    DECLARE @Co    VARCHAR(20);
    DECLARE @RunID UNIQUEIDENTIFIER;

    WHILE EXISTS (SELECT 1 FROM @Companies)
    BEGIN
        SELECT TOP(1) @Co = Company FROM @Companies ORDER BY Company;

        BEGIN TRY
            TRUNCATE TABLE dbo.IA_RawCsv;

            INSERT dbo.IA_RawCsv (Company, Account, AcctDesc, Amount, TrxDate, Reference, Reversal_Flag)
            SELECT Company, Account, AcctDesc, Amount, TrxDate, Reference, UPPER(RTRIM(LTRIM(Reversal_Flag)))
            FROM dbo.IA_FileStage
            WHERE RTRIM(LTRIM(Company)) = @Co;

            IF NOT EXISTS (SELECT 1 FROM dbo.IA_RawCsv)
                THROW 53301, 'Company mapped to zero rows.', 1;

            DECLARE @suffix NVARCHAR(64) = CONCAT(' | Co=', @Co);

            -- Process this company using the pre-staged rows, passing the shared JobID
            EXEC dbo.usp_IA_File_Process
                @CsvPath        = @CsvPath,
                @RunID          = @RunID OUTPUT,
                @UseStaged      = 1,
                @FileNameSuffix = @suffix,
                @BatchPrefix    = 'GLTX',
                @JobID          = @JobID;
        END TRY
        BEGIN CATCH
            -- Error already logged inside usp_IA_File_Process; continue to next company
        END CATCH;

        DELETE FROM @Companies WHERE Company = @Co;
        SET @Co = NULL;
        SET @RunID = NULL;
    END

    --------------------------------------------------------------------
    -- Compute overall JobResult and stamp it onto IA_Run for this JobID
    --------------------------------------------------------------------
    DECLARE @rc INT, @rcPosted INT, @rcFailed INT, @jobResult VARCHAR(20);

    SELECT
        @rc       = COUNT(*),
        @rcPosted = SUM(CASE WHEN Status = 'POSTED' THEN 1 ELSE 0 END),
        @rcFailed = SUM(CASE WHEN Status = 'FAILED' THEN 1 ELSE 0 END)
    FROM dbo.IA_Run
    WHERE JobID = @JobID;

    SET @jobResult =
        CASE
            WHEN ISNULL(@rc,0) = 0 THEN 'ERROR'                 -- nothing created = treat as error
            WHEN @rcPosted = @rc THEN 'SUCCESS'                 -- all posted
            WHEN @rcPosted > 0 AND @rcFailed > 0 THEN 'PARTIAL' -- mix
            WHEN @rcFailed > 0 AND @rcPosted = 0 THEN 'ERROR'   -- all failed
            ELSE 'ERROR'
        END;

    UPDATE dbo.IA_Run
    SET JobResult = @jobResult
    WHERE JobID = @JobID;

    -- OUTPUT: only JobID
    SELECT @JobID AS JobID;
END
GO

CREATE   PROCEDURE dbo.usp_IA_GetNextBatchID_Seq
(
    @SeriesPrefix varchar(8) = 'GLTX',  -- adjust prefix if needed
    @PadWidth     int        = 8,       -- ICTX + 8 digits -> 12 chars total
    @BatchID      CHAR(15)   OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @n bigint = NEXT VALUE FOR DYNAMICS.dbo.ia_batch_seq;
    DECLARE @suffix varchar(32) = RIGHT(REPLICATE('0', @PadWidth) + CAST(@n AS varchar(32)), @PadWidth);
    DECLARE @candidate varchar(15) = @SeriesPrefix + @suffix;

    IF LEN(@candidate) > 15
		THROW 52041, 'Generated batch ID exceeds 15 characters.', 1;

    SET @BatchID = @candidate;
END
GO

CREATE PROCEDURE dbo.usp_IA_LoadCsv
(
    @CsvPath   NVARCHAR(4000),
    @BatchID   CHAR(15),
    @RunID     UNIQUEIDENTIFIER OUTPUT,
    @JobID     UNIQUEIDENTIFIER = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    SET @RunID = NEWID();

    DECLARE @FileName NVARCHAR(260) = RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

    INSERT dbo.IA_Run (RunID, JobID, FileName, FilePath, BatchID, Reference, Status)
    VALUES (@RunID, @JobID, @FileName, @CsvPath, @BatchID, '', 'LOADING');

    DECLARE @Message NVARCHAR(1000) = CONCAT('Starting load: ', @FileName);
    EXEC dbo.usp_IA_Log @RunID, 'LOAD', @Message;

    TRUNCATE TABLE dbo.IA_RawCsv;

    DECLARE @sql NVARCHAR(MAX) = N'
        BULK INSERT DYNAMICS.dbo.IA_RawCsv
        FROM ' + QUOTENAME(@CsvPath,'''') + N'
        WITH (
            FIRSTROW       = 2,
            FIELDTERMINATOR= '','',
            ROWTERMINATOR  = ''0x0d0a'',
            FIELDQUOTE     = ''"''      -- allows commas in descriptions if quoted
        );';
    EXEC (@sql);

    DECLARE @RawSig VARBINARY(32);
    EXEC dbo.usp_IA_ComputeRawCsvSig @RunID=@RunID, @OutSig=@RawSig OUTPUT;

    DECLARE @rawCount INT = (SELECT COUNT(*) FROM dbo.IA_RawCsv);
    UPDATE dbo.IA_Run SET RowCountRaw=@rawCount WHERE RunID=@RunID;

    SET @Message = CONCAT('Raw rows loaded: ', @rawCount);
    EXEC dbo.usp_IA_Log @RunID, 'LOAD', @Message;

    INSERT dbo.IA_Line (RunID, TrxDate, CompanyID, Account, AcctDesc, Amount)
    SELECT
        @RunID,
        TRY_CONVERT(date, TrxDate),
        RTRIM(LTRIM(Company)),
        CONVERT(char(129), RTRIM(LTRIM(Account))),
        AcctDesc,
        TRY_CONVERT(numeric(19,5), REPLACE(Amount, ',', ''))
    FROM dbo.IA_RawCsv;

    DELETE dbo.IA_Line
    WHERE RunID=@RunID
      AND (Amount=0 OR TrxDate IS NULL OR CompanyID='' OR Account='');

    DECLARE @lineCount INT = (SELECT COUNT(*) FROM dbo.IA_Line WHERE RunID=@RunID);
    UPDATE dbo.IA_Run SET RowCountLine=@lineCount WHERE RunID=@RunID;

    SET @Message = CONCAT('Staged nonzero lines: ', @lineCount);
    EXEC dbo.usp_IA_Log @RunID, 'LOAD', @Message;

    DECLARE @ReferenceFromCsv VARCHAR(30) =
    (
        SELECT TOP (1) LEFT(RTRIM(LTRIM(Reference)), 30)
        FROM dbo.IA_RawCsv
        WHERE Reference IS NOT NULL AND LTRIM(RTRIM(Reference)) <> ''
    );
    IF (@ReferenceFromCsv IS NULL OR @ReferenceFromCsv = '')
        SET @ReferenceFromCsv = LEFT(@FileName, 30);

    UPDATE dbo.IA_Run SET Reference=@ReferenceFromCsv, Status='LOADED' WHERE RunID=@RunID;
END


/* 2) Logging */
CREATE   PROCEDURE dbo.usp_IA_Log
(
    @RunID uniqueidentifier,
    @StepName varchar(50),
    @Message varchar(4000)
)
AS
BEGIN
    INSERT dbo.IA_RunEvent (RunID, StepName, Message)
    VALUES (@RunID, @StepName, @Message);
END
GO


/* 6) Normalize Amount â†’ Debit/Credit */
CREATE   PROCEDURE dbo.usp_IA_Normalize
(
    @RunID uniqueidentifier
)
AS
BEGIN
    UPDATE dbo.IA_Line
    SET
      Debit  = CASE WHEN Amount > 0 THEN Amount ELSE 0 END,
      Credit = CASE WHEN Amount < 0 THEN ABS(Amount) ELSE 0 END
    WHERE RunID=@RunID;

    EXEC dbo.usp_IA_Log @RunID, 'NORMALIZE', 'Converted Amount into Debit/Credit';
END
GO


/* 8) Post to GP (single company; net by account; eConnect lines then header) */
CREATE   PROCEDURE dbo.usp_IA_PostToGP
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @BatchID char(15), @Ref varchar(30), @Company varchar(10), @RevDate date, @IsReversal bit;
    SELECT @BatchID = BatchID, @Ref = Reference, @Company = CompanyA, @RevDate = ReversalDate
    FROM dbo.IA_Run WHERE RunID = @RunID;

   SET @IsReversal = CASE WHEN @RevDate IS NULL THEN 0 ELSE 1 END;

    DROP TABLE IF EXISTS #Lines;

    SELECT
        ACTNUMST = Account,
        Net      = SUM(ISNULL(Debit,0) - ISNULL(Credit,0))
    INTO #Lines
    FROM dbo.IA_Line
    WHERE RunID = @RunID AND CompanyID = @Company
    GROUP BY Account
    HAVING ABS(SUM(ISNULL(Debit,0) - ISNULL(Credit,0))) > 0.00001;

    DECLARE @TrxDate DATE   = (SELECT MIN(TrxDate) FROM dbo.IA_Line WHERE RunID = @RunID AND CompanyID = @Company);
    DECLARE @TrxDT   DATETIME = DATEADD(day, DATEDIFF(day, 0, @TrxDate), 0);

    DECLARE @sql nvarchar(max) = N'
    USE ' + QUOTENAME(LEFT(@Company,5)) + N';
    SET NOCOUNT ON;

    DECLARE @B   char(15)         = @pB;
    DECLARE @R   varchar(30)      = @pR;
    DECLARE @DT  datetime         = @pDT;
    DECLARE @Run uniqueidentifier = @pRun;
    DECLARE @Rev date             = @pRev;
    DECLARE @IsRev bit            = @pIsRev;

    DECLARE @log_message varchar(4000);
    DECLARE @JE char(13), @Err int = 0, @ErrStr varchar(255) = '''';

    IF NOT EXISTS (SELECT 1 FROM #Lines)
    BEGIN
        SET @log_message = ''No non-zero lines after netting; nothing to post.'';
        EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''POST'', @log_message;
        RETURN;
    END

    EXEC taGetNextJournalEntry
        @I_vInc_Dec = 1,
        @O_vJournalEntryNumber = @JE OUTPUT,
        @O_iErrorState = @Err OUTPUT;

    IF (@Err <> 0 OR @JE IS NULL)
    BEGIN
        SET @log_message = CONCAT(''taGetNextJournalEntry failed. ErrState='', @Err);
        EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''ERROR'', @log_message;
        RAISERROR(''Failed to get next journal entry.'', 16, 1);
        RETURN;
    END

    SET @log_message = CONCAT(''JE obtained: '', @JE);
    EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''POST'', @log_message;

    DECLARE @Seq int = 16384;

    DECLARE L CURSOR LOCAL FAST_FORWARD FOR
        SELECT
            ACTNUMST,
            CASE WHEN Net > 0 THEN Net  ELSE 0 END AS D_pass,
            CASE WHEN Net < 0 THEN -Net ELSE 0 END AS C_pass
        FROM #Lines;

    OPEN L;
    DECLARE @A char(129), @D_pass numeric(19,5), @C_pass numeric(19,5);
    FETCH NEXT FROM L INTO @A, @D_pass, @C_pass;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @Err = 0; SET @ErrStr = '''';

        EXEC taGLTransactionLineInsert
            @I_vBACHNUMB = @B,
            @I_vJRNENTRY = @JE,
            @I_vSQNCLINE = @Seq,
            @I_vACTNUMST = @A,
            @I_vDEBITAMT = @D_pass,
            @I_vCRDTAMNT = @C_pass,
            @I_vDOCDATE  = @DT,
            @I_vDSCRIPTN = @R,
            @O_iErrorState = @Err OUTPUT,
            @oErrString   = @ErrStr OUTPUT;

        IF (@Err <> 0 OR ISNULL(@ErrStr,'''') <> '''')
        BEGIN
            SET @log_message = CONCAT(
                ''Line insert failed. ACTNUMST='', @A,
                '' Debit='', CONVERT(varchar(50), @D_pass),
                '' Credit='', CONVERT(varchar(50), @C_pass),
                '' ErrState='', @Err,
                '' ErrListRaw=['', @ErrStr, '']''
            );
            EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''ERROR'', @log_message;

            DECLARE @errText nvarchar(1024) =
                CONCAT(''eConnect line insert failed. ErrState='', CONVERT(varchar(20), @Err),
                       ''. ErrListRaw=['', @ErrStr, '']'');
            RAISERROR(@errText, 16, 1);
            CLOSE L; DEALLOCATE L;
            RETURN;
        END

        SET @Seq = @Seq + 16384;
        FETCH NEXT FROM L INTO @A, @D_pass, @C_pass;
    END

    CLOSE L; DEALLOCATE L;

    SET @Err = 0; SET @ErrStr = '''';
    
    IF @IsRev = 1
    BEGIN
        EXEC taGLTransactionHeaderInsert
            @I_vBACHNUMB = @B,
            @I_vJRNENTRY = @JE,
            @I_vREFRENCE = @R,
            @I_vTRXDATE  = @DT,
            @I_vTRXTYPE  = 1,
            @I_vSOURCDOC = ''GJ'',
            @I_vRVRSNGDT = @Rev,
            @O_iErrorState = @Err OUTPUT,
            @oErrString   = @ErrStr OUTPUT;

        IF (@Err <> 0 OR ISNULL(@ErrStr,'''') <> '''')
        BEGIN
            SET @log_message = CONCAT(
                ''Header insert (reversing) failed. JE='', @JE,
                ''. ErrState='', @Err, '' ErrListRaw=['', @ErrStr, '']''
            );
            EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''ERROR'', @log_message;

            DECLARE @errHead nvarchar(1024) =
                CONCAT(''eConnect header insert (reversing) failed. JE='', @JE,
                       ''. ErrState='', CONVERT(varchar(20), @Err),
                       ''. ErrListRaw=['', @ErrStr, '']'');
            RAISERROR(@errHead, 16, 1);
            RETURN;
        END

        SET @log_message = CONCAT(''Header created (reversing) for JE='', @JE, '' batch='', @B,
                                  '' ReversalDate='', CONVERT(varchar(10), @Rev, 120));
        EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''POST'', @log_message;
    END
    ELSE
    BEGIN
        EXEC taGLTransactionHeaderInsert
            @I_vBACHNUMB = @B,
            @I_vJRNENTRY = @JE,
            @I_vREFRENCE = @R,
            @I_vTRXDATE  = @DT,
            @I_vTRXTYPE  = 0,
            @I_vSOURCDOC = ''GJ'',
            @O_iErrorState = @Err OUTPUT,
            @oErrString   = @ErrStr OUTPUT;

        IF (@Err <> 0 OR ISNULL(@ErrStr,'''') <> '''')
        BEGIN
            SET @log_message = CONCAT(
                ''Header insert failed. JE='', @JE,
                ''. ErrState='', @Err, '' ErrListRaw=['', @ErrStr, '']''
            );
            EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''ERROR'', @log_message;

            DECLARE @errHead2 nvarchar(1024) =
                CONCAT(''eConnect header insert failed. JE='', @JE,
                       ''. ErrState='', CONVERT(varchar(20), @Err),
                       ''. ErrListRaw=['', @ErrStr, '']'');
            RAISERROR(@errHead2, 16, 1);
            RETURN;
        END

        UPDATE DYNAMICS.dbo.IA_Run SET JournalEntry = @JE WHERE RunID = @Run;

        SET @log_message = CONCAT(''Header created (normal) for JE='', @JE, '' batch='', @B);
        EXEC DYNAMICS.dbo.usp_IA_Log @Run, ''POST'', @log_message;
    END
    ';

    EXEC sp_executesql @sql,
        N'@pB char(15), @pR varchar(30), @pDT datetime, @pRun uniqueidentifier, @pRev date, @pIsRev bit',
        @pB = @BatchID, @pR = @Ref, @pDT = @TrxDT, @pRun = @RunID, @pRev = @RevDate, @pIsRev = @IsReversal;
END
GO


/* 11) Run Audit (single-company) */
CREATE   PROCEDURE dbo.usp_IA_Run_Audit
(
      @RunID    uniqueidentifier = NULL
    , @BatchID  char(15)         = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    IF (@RunID IS NULL AND @BatchID IS NULL)
    BEGIN
        RAISERROR('Provide either @RunID or @BatchID.', 16, 1);
        RETURN;
    END

    IF (@RunID IS NULL AND @BatchID IS NOT NULL)
    BEGIN
        SELECT TOP (1) @RunID = RunID
        FROM dbo.IA_Run
        WHERE BatchID = @BatchID
        ORDER BY StartedDT DESC;

        IF (@RunID IS NULL)
        BEGIN
            RAISERROR('No IA_Run found for the provided @BatchID.', 16, 1);
            RETURN;
        END
    END

    IF NOT EXISTS (SELECT 1 FROM dbo.IA_Run WHERE RunID = @RunID)
    BEGIN
        RAISERROR('RunID not found in IA_Run.', 16, 1);
        RETURN;
    END

    SELECT 
        'ia_run' AS table_name, 
        r.RunID, r.FileName, r.FilePath, r.BatchID, r.Reference, r.Status,
        r.StartedDT, r.EndedDT, r.Message,
        r.RowCountRaw, r.RowCountLine, r.CompanyA, r.NetAmount,
        NormalizedSigHex = CONVERT(varchar(66), r.FileSignature, 1)
    FROM dbo.IA_Run r WHERE r.RunID = @RunID;

    SELECT 'ia_runevent' AS table_name, e.EventDT, e.StepName, e.Message
    FROM dbo.IA_RunEvent e
    WHERE e.RunID = @RunID
    ORDER BY e.EventDT ASC, e.EventID ASC;

    SELECT
        'ia_line' AS table_name,
        il.CompanyID,
        TotalAmount   = SUM(il.Debit - il.Credit),
        TotalDebits   = SUM(il.Debit),
        TotalCredits  = SUM(il.Credit),
        FirstTrxDate  = MIN(il.TrxDate),
        LastTrxDate   = MAX(il.TrxDate),
        AccountsCount = COUNT(DISTINCT il.Account),
        LinesCount    = COUNT_BIG(1)
    FROM dbo.IA_Line il
    WHERE il.RunID = @RunID
    GROUP BY il.CompanyID
    ORDER BY il.CompanyID;

    -- Net by account (non-zero only)
    SELECT
        '#lines' AS table_name,
        il.CompanyID,
        il.Account AS ACTNUMST,
        Net = SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0))
    FROM dbo.IA_Line il
    WHERE il.RunID = @RunID
    GROUP BY il.CompanyID, il.Account
    HAVING ABS(SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0))) > 0.00001
    ORDER BY il.CompanyID, il.Account;

    -- Company GL work tables for this BatchID
    DECLARE @Batch char(15), @CoA varchar(10), @DBA sysname;
    SELECT @Batch = BatchID, @CoA = CompanyA FROM dbo.IA_Run WHERE RunID = @RunID;
    SET @DBA = LEFT(@CoA, 5);

    IF OBJECT_ID('tempdb..#GL10000') IS NOT NULL DROP TABLE #GL10000;
    IF OBJECT_ID('tempdb..#GL10001') IS NOT NULL DROP TABLE #GL10001;

    CREATE TABLE #GL10000
    (
        CompanyDB sysname NOT NULL, JRNENTRY int NULL, BACHNUMB char(15) NULL,
        REFRENCE char(30) NULL, TRXDATE datetime NULL, SERIES smallint NULL, SOURCDOC char(4) NULL
    );

    CREATE TABLE #GL10001
    (
        CompanyDB sysname NOT NULL, JRNENTRY int NULL, SQNCLINE int NULL,
        ACTNUMST char(129) NULL, DEBITAMT numeric(19,5) NULL, CRDTAMNT numeric(19,5) NULL
    );

    DECLARE @sql nvarchar(max), @parm nvarchar(100) = N'@B char(15)';

    IF @DBA IS NOT NULL AND DB_ID(@DBA) IS NOT NULL
    BEGIN
        SET @sql = N'
            INSERT INTO #GL10000 (CompanyDB, JRNENTRY, BACHNUMB, REFRENCE, TRXDATE, SERIES, SOURCDOC)
            SELECT DB_NAME(), JRNENTRY, BACHNUMB, REFRENCE, TRXDATE, SERIES, SOURCDOC
            FROM ' + QUOTENAME(@DBA) + N'..GL10000 WHERE BACHNUMB = @B;

            INSERT INTO #GL10001 (CompanyDB, JRNENTRY, SQNCLINE, ACTNUMST, DEBITAMT, CRDTAMNT)
            SELECT DB_NAME(), JRNENTRY, SQNCLINE, ACTNUMST, DEBITAMT, CRDTAMNT
            FROM ' + QUOTENAME(@DBA) + N'..GL10001 WHERE BACHNUMB = @B;
        ';
        EXEC sp_executesql @sql, @parm, @B = @Batch;
    END

    SELECT * FROM #GL10000 ORDER BY CompanyDB, JRNENTRY DESC;
    SELECT * FROM #GL10001 ORDER BY CompanyDB, JRNENTRY DESC, SQNCLINE;
END
GO


/* 7) Validate (strict: exactly 1 company; net must be 0) */
CREATE   PROCEDURE dbo.usp_IA_ValidateRun
(
    @RunID uniqueidentifier
)
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT EXISTS (SELECT 1 FROM dbo.IA_Line WHERE RunID=@RunID)
    BEGIN
        DECLARE @msg1 varchar(200) = 'No staged lines found for this run (after removing zeros).';
        THROW 53201, @msg1, 1;
    END

    DECLARE @c1 varchar(10);
    SELECT TOP 1 @c1 = CompanyID FROM dbo.IA_Line WHERE RunID=@RunID ORDER BY CompanyID;

    IF (SELECT COUNT(DISTINCT CompanyID) FROM dbo.IA_Line WHERE RunID=@RunID) <> 1
    BEGIN
        DECLARE @msg2 varchar(200) = 'INTRA file must contain exactly one company per run.';
        THROW 53202, @msg2, 1;
    END

    UPDATE dbo.IA_Run SET CompanyA=@c1 WHERE RunID=@RunID;

    DECLARE @Message VARCHAR(4000) = CONCAT('Detected company: ', @c1);
    EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', @Message;

    DECLARE @net numeric(19,5) = (SELECT SUM(Amount) FROM dbo.IA_Line WHERE RunID=@RunID);
    UPDATE dbo.IA_Run SET NetAmount=@net WHERE RunID=@RunID;

    SET @Message = CONCAT('Net amount: ', @net);
    EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', @Message;

    IF ABS(@net) > 0.00001
    BEGIN
        DECLARE @msg3 varchar(200) = 'File does not balance overall (SUM(Amount) <> 0).';
        THROW 53203, @msg3, 1;
    END
    EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', 'Balanced=YES';


-- NEW: Resolve ReversalFlag from RawCsv content for this run
    DECLARE @flag CHAR(1);

    ;WITH f AS
    (
        SELECT DISTINCT UPPER(LTRIM(RTRIM(Reversal_Flag))) AS F
        FROM dbo.IA_RawCsv
    )
    SELECT @flag =
        CASE WHEN EXISTS (SELECT 1 FROM f WHERE F='Y') THEN 'Y'
             WHEN EXISTS (SELECT 1 FROM f WHERE F='N') THEN 'N'
             ELSE NULL
        END;

    -- In ByCompany path, RawCsv was pre-sliced; in single-file path, RawCsv contains only this runâ€™s rows now.

    IF @flag NOT IN ('Y','N')
    BEGIN
        -- Default to 'N' if entirely missing (for backward compatibility),
        -- or you can fail strictly. Using strict per your requirement:
        ;THROW 53220, 'Reversal_Flag must be Y or N for this run.', 1;
    END

    UPDATE dbo.IA_Run SET ReversalFlag = @flag WHERE RunID=@RunID;
    SET @Message = CONCAT('ReversalFlag = ', @flag)
    EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', @Message;

    IF (@flag = 'Y')
    BEGIN
        -- NEW: Enforce single TrxDate within this company/run
        DECLARE @dtCount INT = (SELECT COUNT(DISTINCT TrxDate) FROM dbo.IA_Line WHERE RunID=@RunID);
        IF @dtCount <> 1
        BEGIN
            ;THROW 53221, 'Reversal file requires a single transaction date within the company run.', 1;
        END

        DECLARE @basis DATE = (SELECT MIN(TrxDate) FROM dbo.IA_Line WHERE RunID=@RunID); -- all same by rule
        DECLARE @rev   DATE = DATEADD(DAY, 1, EOMONTH(@basis));

        UPDATE dbo.IA_Run
        SET TrxDateBasis = @basis,
            ReversalDate = @rev
        WHERE RunID=@RunID;

        SET @Message = CONCAT('Reversal: TrxDateBasis=', CONVERT(varchar(10), @basis, 120),
                   ' ReversalDate=', CONVERT(varchar(10), @rev, 120),
                   ' (1st day of next month)')
        EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', @Message;
    END
    ELSE
    BEGIN
        -- Normal mode; clear basis/reversal if set
        UPDATE dbo.IA_Run
        SET TrxDateBasis = NULL,
            ReversalDate = NULL
        WHERE RunID=@RunID;

        EXEC dbo.usp_IA_Log @RunID, 'VALIDATE', 'Normal (non-reversing) run.';
    END


END
GO

