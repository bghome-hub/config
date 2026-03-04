
        INSERT INTO #AuditResults (
            JobID, RunID, [FILENAME], CompanyA, BatchID, JournalEntry, TrxDate, [Reference], [STATUS], [MESSAGE],
            ReversalFlag, TrxDateBasis, ReversalDate, Account, AcctDesc, Amount,
            DebitAmt, CreditAmt, NetAmt,
            gp_Journal, gp_BatchNumber, gp_Reference, gp_TrxDate, gp_PeriodID,
            gp_PeriodMonth, gp_OpenYear, gp_ReversalPeriodID, gp_ReversalPeriodMonth, gp_ReversalYear,
            gp_AccountIndex, gp_Account, gp_Description, gp_Company, gp_DebitAmount, gp_CreditAmount,
            gp_OriginatingDebitAmount, gp_OriginatingCreditAmount
        )
        SELECT 
            r.JobID, r.RunID, r.[FileName], r.CompanyA, r.BatchID, r.JournalEntry, l.TrxDate, r.[Reference], r.[Status], r.[Message],
            r.ReversalFlag, r.TrxDateBasis, r.ReversalDate, l.Account, l.AcctDesc, l.Amount,
            l.DebitAmt, l.CreditAmt, l.NetAmt,
            glh.JRNENTRY, glh.BACHNUMB, glh.REFRENCE, glh.TRXDATE, glh.PERIODID,
            fp.PeriodMonth, glh.OPENYEAR, glh.REVPRDID, fp2.PeriodMonth, glh.REVYEAR,
            act.ACTINDX, act.ACTNUMST, gld.DSCRIPTN, gld.INTERID, 
            ISNULL(gld.DEBITAMT, 0), ISNULL(gld.CRDTAMNT, 0), ISNULL(gld.ORDBTAMT, 0), ISNULL(gld.ORCRDAMT, 0)
        FROM dynamics.dbo.IA_Run r
        
        -- 1. Squash File Lines to stop duplication
        LEFT JOIN (
            SELECT RunID, Account, MAX(AcctDesc) AS AcctDesc, MIN(TrxDate) AS TrxDate, SUM(Amount) AS Amount, SUM(Debit) AS DebitAmt, SUM(Credit) AS CreditAmt, SUM(Debit - Credit) AS NetAmt
            FROM dynamics.dbo.IA_Line
            GROUP BY RunID, Account
        ) l ON l.RunID = r.RunID
        
        -- 2. GP Header
        LEFT JOIN [' + @currentCompany + N'].dbo.GL10000 glh ON glh.BACHNUMB = r.BatchID AND glh.JRNENTRY = r.JournalEntry
        
        -- 3. Link Account String to Index
        LEFT JOIN [' + @currentCompany + N'].dbo.GL00105 act ON RTRIM(l.Account) = RTRIM(act.ACTNUMST)
        
        -- 4. Squash GP Lines to stop duplication
        LEFT JOIN (
            SELECT JRNENTRY, ACTINDX, MAX(DSCRIPTN) AS DSCRIPTN, MAX(INTERID) AS INTERID, SUM(DEBITAMT) AS DEBITAMT, SUM(CRDTAMNT) AS CRDTAMNT, SUM(ORDBTAMT) AS ORDBTAMT, SUM(ORCRDAMT) AS ORCRDAMT
            FROM [' + @currentCompany + N'].dbo.GL10001
            GROUP BY JRNENTRY, ACTINDX
        ) gld ON glh.JRNENTRY = gld.JRNENTRY AND act.ACTINDX = gld.ACTINDX
        
        LEFT JOIN dynamics.dbo.ia_fiscal_period fp ON glh.PERIODID = fp.PeriodID
        LEFT JOIN dynamics.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
        WHERE r.RunID = @p_runID;
```

### Step 2: The Final PowerShell Script
This PowerShell script maps **exactly** to the columns your `usp_IA_Run_Audit` procedure outputs (`DebitAmt`, `NetAmt`, `gp_DebitAmount`, `TrxDate`). 

Delete the old function in your PowerShell file and paste this:

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param([Parameter(Mandatory)][Guid]$JobId)

    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    try { $auditData = Invoke-Db -Query $query } catch { return "<h3 style='color:red;'>Error retrieving data.</h3>" }
    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    $firstGlobal = $auditData

    # --- PART 1: SUMMARY TABLE ---
    $font       = "font-family: 'Segoe UI', Arial, sans-serif; font-size: 11px; color: #000;"
    $thSumStyle = "border-bottom: 1px solid #c8d8c8; padding: 6px 10px; background-color: #e2f0d9; text-align: left; font-weight: bold;"
    $tdSumBase  = "padding: 6px 10px; text-align: left;"
    
    $html = "<div style='$font'>"
    $html += "<div style='margin-bottom: 2px;'>Filename: <strong>$($firstGlobal.FILENAME)</strong></div>"
    $html += "<div style='margin-bottom: 15px;'>Job ID: <strong>$JobId</strong></div>"

    $html += "<table style='border-collapse: collapse; width: 100%; $font'>"
    $html += "<thead><tr><th style='$thSumStyle'>Company</th><th style='$thSumStyle'>Batch</th><th style='$thSumStyle'>JE</th><th style='$thSumStyle'>Reference</th><th style='$thSumStyle'>TrxDate</th><th style='$thSumStyle'>#Rows</th><th style='$thSumStyle'>RunStatus</th><th style='$thSumStyle'>Reversal</th><th style='$thSumStyle'>Error</th></tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    $rowIndex = 0

    foreach ($run in $runs) {
        $first = $run.Group
        $bgColor = if ($rowIndex % 2 -eq 0) { "#ffffff" } else { "#f4f9f4" }
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }
        
        $html += "<tr style='background-color: $bgColor;'>"
        $html += "<td style='$tdSumBase'>$($first.CompanyA)</td><td style='$tdSumBase'>$($first.BatchID)</td><td style='$tdSumBase'>$($first.JournalEntry)</td><td style='$tdSumBase'>$($first.Reference)</td><td style='$tdSumBase'>$trxDateStr</td><td style='$tdSumBase'>$($run.Group.Count)</td><td style='$tdSumBase'>$($first.STATUS)</td><td style='$tdSumBase'>$($first.ReversalFlag)</td><td style='$tdSumBase'>$($first.MESSAGE)</td>"
        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # --- PART 2: THE SIDE-BY-SIDE DETAIL TABLES ---
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        $first = $run.Group
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }

        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr><th style='$thDet' rowspan='2'>Account</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th></tr><tr><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        $accounts = $run.Group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            $gpNet   = $gpDeb - $gpCred

            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr><td style='$tdDet'>$($acc.Name)</td><td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $impCred)</td><td style='$tdAmt'>$("{0:N2}" -f $impNet)</td><td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td><td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $html += "<tr><td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td><td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}


if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    $firstGlobal = $auditData

    # --- PART 1: SUMMARY TABLE ---
    $font       = "font-family: 'Segoe UI', Arial, sans-serif; font-size: 11px; color: #000;"
    $thSumStyle = "border-bottom: 1px solid #c8d8c8; padding: 6px 10px; background-color: #e2f0d9; text-align: left; font-weight: bold;"
    $tdSumBase  = "padding: 6px 10px; text-align: left;"
    
    $html = "<div style='$font'>"
    $html += "<div style='margin-bottom: 2px;'>Filename: <strong>$($firstGlobal.FILENAME)</strong></div>"
    $html += "<div style='margin-bottom: 15px;'>Job ID: <strong>$JobId</strong></div>"

    $html += "<table style='border-collapse: collapse; width: 100%; $font'>"
    $html += "<thead><tr><th style='$thSumStyle'>Company</th><th style='$thSumStyle'>Batch</th><th style='$thSumStyle'>JE</th><th style='$thSumStyle'>Reference</th><th style='$thSumStyle'>TrxDate</th><th style='$thSumStyle'>#Rows</th><th style='$thSumStyle'>RunStatus</th><th style='$thSumStyle'>Reversal</th><th style='$thSumStyle'>Error</th></tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    $rowIndex = 0

    foreach ($run in $runs) {
        $first = $run.Group
        $bgColor = if ($rowIndex % 2 -eq 0) { "#ffffff" } else { "#f4f9f4" }
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }
        
        $html += "<tr style='background-color: $bgColor;'>"
        $html += "<td style='$tdSumBase'>$($first.CompanyA)</td><td style='$tdSumBase'>$($first.BatchID)</td><td style='$tdSumBase'>$($first.JournalEntry)</td><td style='$tdSumBase'>$($first.Reference)</td><td style='$tdSumBase'>$trxDateStr</td><td style='$tdSumBase'>$($run.Group.Count)</td><td style='$tdSumBase'>$($first.STATUS)</td><td style='$tdSumBase'>$($first.ReversalFlag)</td><td style='$tdSumBase'>$($first.MESSAGE)</td>"
        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # --- PART 2: THE SIDE-BY-SIDE DETAIL TABLES ---
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        $first = $run.Group
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }

        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr><th style='$thDet' rowspan='2'>Account</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th></tr><tr><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        $accounts = $run.Group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            $gpNet   = $gpDeb - $gpCred

            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr><td style='$tdDet'>$($acc.Name)</td><td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $impCred)</td><td style='$tdAmt'>$("{0:N2}" -f $impNet)</td><td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td><td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $html += "<tr><td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td><td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}
```

No more confusion. Save your SQL, run the test, and you are done./
        Account CHAR(129),
        AcctDesc VARCHAR(255),
        Amount NUMERIC(19,5),
        DebitAmt NUMERIC(19,5),
        CreditAmt NUMERIC(19,5),
        NetAmt NUMERIC(19,5),
        
        /* GP-Specific Fields (prefixed with gp_) */
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
        gp_Account CHAR(129),
        gp_AccountIndex INT,   -- Added per 2.png and critique
        gp_Description CHAR(31),
        gp_Company CHAR(5),
        gp_DebitAmount NUMERIC(19,5),
        gp_CreditAmount NUMERIC(19,5),
        gp_OriginatingDebitAmount NUMERIC(19,5),
        gp_OriginatingCreditAmount NUMERIC(19,5)
    );

    -- 2. Cursor Orchestration
    DECLARE @currentRunID UNIQUEIDENTIFIER;
    DECLARE @currentCompany VARCHAR(20);
    DECLARE @currentJE INT;
    DECLARE @currentBatch CHAR(15);
    DECLARE @sql NVARCHAR(MAX);
    DECLARE @params NVARCHAR(MAX);

    -- Iterate through each Run assigned to the JobID
    DECLARE run_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT RunID, CompanyA, JournalEntry, BatchID
    FROM DYNAMICS.dbo.IA_Run
    WHERE JobID = @JobID;

    OPEN run_cursor;

    FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany, @currentJE, @currentBatch;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        BEGIN TRY
            /* 3. Dynamic SQL Construction with Pre-Aggregation */
            -- Optimization: The subquery for GL10001 (gld) is filtered by @p_JE and @p_Batch 
            -- inside the scope to prevent performance degradation on large work tables.
            
            SET @sql = N'
            INSERT INTO #AuditResults (
                JobID, runid, [FILENAME], CompanyA, BatchID, JournalEntry, TrxDate, [Reference], [STATUS], [MESSAGE], 
                ReversalFlag, ReversalDate, TrxDateBasis, Account, AcctDesc, Amount, DebitAmt, CreditAmt, NetAmt,
                gp_Journal, gp_BatchNumber, gp_Reference, gp_TrxDate, gp_ReversalFlag, 
                gp_PeriodID, gp_PeriodMonth, gp_OpenYear, 
                gp_ReversalPeriodID, gp_ReversalPeriodMonth, gp_ReversalYear,
                gp_Account, gp_AccountIndex, gp_Description, gp_Company, 
                gp_DebitAmount, gp_CreditAmount, gp_OriginatingDebitAmount, gp_OriginatingCreditAmount
            )
            SELECT 
                r.JobID, r.RunID, r.[FileName], r.CompanyA, r.BatchID, r.JournalEntry, r.TrxDate, r.[Reference], r.[Status], r.[Message],
                r.ReversalFlag, r.ReversalDate, r.TrxDateBasis, il.Account, il.AcctDesc, il.SumAmount, il.SumDebit, il.SumCredit, (il.SumDebit - il.SumCredit),
                gh.JRNENTRY, gh.BACHNUMB, gh.REFRENCE, gh.TRXDATE,
                CASE WHEN gh.TRXTYPE = 0 THEN ''N'' WHEN gh.TRXTYPE = 1 THEN ''Y'' ELSE '''' END,
                gh.PERIODID, fp.PeriodMonth, gh.OPENYEAR,
                gh.REVPRDID, fp2.PeriodMonth, gh.REVYEAR,
                act.ACTNUMST, gld.ACTINDX, gld.DSCRIPTN, gld.INTERID,
                gld.SumDebit, gld.SumCredit, gld.SumOrigDebit, gld.SumOrigCredit
            FROM DYNAMICS.dbo.IA_Run r
            -- IA_Line Pre-Aggregation (Summing by Run/Account to prevent M:M duplicates)
            INNER JOIN (
                SELECT 
                    RunID, Account, AcctDesc, 
                    SUM(Amount) as SumAmount, 
                    SUM(Debit) as SumDebit, 
                    SUM(Credit) as SumCredit
                FROM DYNAMICS.dbo.IA_Line
                WHERE RunID = @p_runID
                GROUP BY RunID, Account, AcctDesc
            ) il ON r.RunID = il.RunID
            -- GP Work Header Join
            LEFT JOIN ' + QUOTENAME(@currentCompany) + '.dbo.GL10000 gh ON gh.JRNENTRY = r.JournalEntry AND gh.BACHNUMB = r.BatchID
            -- GP Work Detail Pre-Aggregation (Filtered by current JE/Batch for performance)
            LEFT JOIN (
                SELECT 
                    JRNENTRY, BACHNUMB, ACTINDX, DSCRIPTN, INTERID,
                    SUM(DEBITAMT) as SumDebit, 
                    SUM(CRDTAMNT) as SumCredit,
                    SUM(ORDBTAMT) as SumOrigDebit,
                    SUM(ORCRDAMT) as SumOrigCredit
                FROM ' + QUOTENAME(@currentCompany) + '.dbo.GL10001
                WHERE JRNENTRY = @p_JE AND BACHNUMB = @p_Batch
                GROUP BY JRNENTRY, BACHNUMB, ACTINDX, DSCRIPTN, INTERID
            ) gld ON gld.JRNENTRY = gh.JRNENTRY AND gld.BACHNUMB = gh.BACHNUMB
            -- GP Account Master resolution
            LEFT JOIN ' + QUOTENAME(@currentCompany) + '.dbo.GL00105 act ON act.ACTINDX = gld.ACTINDX
            -- Fiscal Period Month resolution from DYNAMICS
            LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp ON gh.PERIODID = fp.PeriodID
            LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp2 ON gh.REVPRDID = fp2.PeriodID
            WHERE r.RunID = @p_runID;';

            SET @params = N'@p_runID UNIQUEIDENTIFIER, @p_JE INT, @p_Batch CHAR(15)';

            /* 4. Execution */
            EXEC sys.sp_executesql @sql, @params, 
                @p_runID = @currentRunID, 
                @p_JE = @currentJE, 
                @p_Batch = @currentBatch;

        END TRY
        BEGIN CATCH
            PRINT 'Error processing RunID: ' + CAST(@currentRunID AS VARCHAR(50)) + ' in company ' + @currentCompany;
            PRINT ERROR_MESSAGE();
        END CATCH

        FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany, @currentJE, @currentBatch;
    END

    CLOSE run_cursor;
    DEALLOCATE run_cursor;

    -- Final Output: Ordered by Company and Account as requested.
    SELECT * 
    FROM #AuditResults 
    ORDER BY CompanyA, Account;

END
GO



T-SQL Implementation Script: dbo.usp_IA_Run_Audit

CREATE OR ALTER PROCEDURE DBO.USP_IA_RUN_AUDIT (
    @JOBID UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    /* INITIALIZE TEMPORARY STORAGE FOR AGGREGATED AUDIT RESULTS */
    CREATE TABLE #AUDITRESULTS (
        AUDITRESULTID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        JOBID UNIQUEIDENTIFIER,
        RUNID UNIQUEIDENTIFIER,
        [FILENAME] NVARCHAR(520),
        COMPANYA VARCHAR(10),
        BATCHID CHAR(15),
        JOURNALENTRY INT,
        TRXDATE DATE,
        [REFERENCE] VARCHAR(30),
        [STATUS] VARCHAR(20),
        [MESSAGE] VARCHAR(4000),
        REVERSALFLAG CHAR(1),
        REVERSALDATE DATE,
        TRXDATEBASIS DATE,
        ACCOUNT CHAR(129),
        ACCTDESC CHAR(255),
        AMOUNT NUMERIC(19,5),
        DEBITAMT NUMERIC(19,5),
        CREDITAMT NUMERIC(19,5),
        NETAMT NUMERIC(19,5),
        GP_JOURNAL INT,
        GP_BATCHNUMBER CHAR(15),
        GP_REFERENCE CHAR(31),
        GP_TRXDATE DATETIME,
        GP_REVERSALFLAG CHAR(1),
        GP_PERIODID SMALLINT,
        GP_PERIODMONTH INT,
        GP_OPENYEAR SMALLINT,
        GP_REVERSALPERIODID SMALLINT,
        GP_REVERSALPERIODMONTH INT,
        GP_REVERSALYEAR SMALLINT,
        GP_ACCOUNTINDEX INT,
        GP_ACCOUNT CHAR(129),
        GP_DESCRIPTION CHAR(31),
        GP_COMPANY CHAR(5),
        GP_DEBITAMOUNT NUMERIC(19,5),
        GP_CREDITAMOUNT NUMERIC(19,5),
        GP_ORIGINATINGDEBITAMOUNT NUMERIC(19,5),
        GP_ORIGINATINGCREDITAMOUNT NUMERIC(19,5)
    );

    DECLARE @CURRENTRUNID UNIQUEIDENTIFIER;
    DECLARE @CURRENTCOMPANY VARCHAR(20);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @PARAMS NVARCHAR(MAX) = N'@P_RUNID UNIQUEIDENTIFIER';

    /* DEFINE CURSOR TO ITERATE THROUGH ALL COMPANY DATA RUNS ASSOCIATED WITH THE JOB */
    DECLARE RUN_CURSOR CURSOR FOR
    SELECT RUNID, COMPANYA
    FROM DYNAMICS.DBO.IA_RUN
    WHERE JOBID = @JOBID;

    OPEN RUN_CURSOR;
    FETCH NEXT FROM RUN_CURSOR INTO @CURRENTRUNID, @CURRENTCOMPANY;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        /* CONSTRUCT DYNAMIC SQL TO RECONCILE IA DATA WITH DYNAMICS GP WORK TABLES */
        SET @SQL = N'
        INSERT INTO #AUDITRESULTS (
            JOBID, RUNID, [FILENAME], COMPANYA, BATCHID, JOURNALENTRY, TRXDATE, [REFERENCE], [STATUS], [MESSAGE],
            REVERSALFLAG, REVERSALDATE, TRXDATEBASIS, ACCOUNT, ACCTDESC, AMOUNT, DEBITAMT, CREDITAMT, NETAMT,
            GP_JOURNAL, GP_BATCHNUMBER, GP_REFERENCE, GP_TRXDATE, GP_REVERSALFLAG, GP_PERIODID,
            GP_PERIODMONTH, GP_OPENYEAR, GP_REVERSALPERIODID, GP_REVERSALPERIODMONTH, GP_REVERSALYEAR,
            GP_ACCOUNTINDEX, GP_ACCOUNT, GP_DESCRIPTION, GP_COMPANY, GP_DEBITAMOUNT, GP_CREDITAMOUNT,
            GP_ORIGINATINGDEBITAMOUNT, GP_ORIGINATINGCREDITAMOUNT
        )
        WITH IA_AGGR AS (
            /* SUBQUERY A: AGGREGATE IA_LINE BY RUN AND ACCOUNT TO PREVENT JOIN DUPLICATION */
            SELECT
                RUNID,
                ACCOUNT,
                MAX(ACCTDESC) AS ACCTDESC,
                SUM(ISNULL(AMOUNT, 0)) AS AMOUNT,
                SUM(ISNULL(DEBIT, 0)) AS DEBITAMT,
                SUM(ISNULL(CREDIT, 0)) AS CREDITAMT
            FROM DYNAMICS.DBO.IA_LINE
            WHERE RUNID = @P_RUNID
            GROUP BY RUNID, ACCOUNT
        ),
        GP_AGGR AS (
            /* SUBQUERY B: AGGREGATE GP TRANSACTION LINES JOINED WITH ACCOUNT MASTER */
            SELECT
                GL1.BACHNUMB,
                GL1.JRNENTRY,
                ACT.ACTNUMST,
                MAX(GL1.ACTINDX) AS ACTINDX,
                SUM(GL1.DEBITAMT) AS GP_DEBITAMOUNT,
                SUM(GL1.CRDTAMNT) AS GP_CREDITAMOUNT,
                SUM(GL1.ORDBTAMT) AS GP_ORIGINATINGDEBITAMOUNT,
                SUM(GL1.ORCTAMNT) AS GP_ORIGINATINGCREDITAMOUNT
            FROM ' + QUOTENAME(@CURRENTCOMPANY) + N'.' + QUOTENAME('DBO') + N'.' + QUOTENAME('GL10001') + N' GL1
            INNER JOIN ' + QUOTENAME(@CURRENTCOMPANY) + N'.' + QUOTENAME('DBO') + N'.' + QUOTENAME('GL00105') + N' ACT ON GL1.ACTINDX = ACT.ACTINDX
            GROUP BY GL1.BACHNUMB, GL1.JRNENTRY, ACT.ACTNUMST
        )
        SELECT
            R.JOBID, R.RUNID, R.[FILENAME], R.COMPANYA, R.BATCHID, R.JOURNALENTRY, R.TRXDATE, R.[REFERENCE], R.[STATUS], R.[MESSAGE],
            R.REVERSALFLAG, R.REVERSALDATE, R.TRXDATEBASIS, IA.ACCOUNT, IA.ACCTDESC, IA.AMOUNT, IA.DEBITAMT, IA.CREDITAMT,
            (IA.DEBITAMT - IA.CREDITAMT) AS NETAMT,
            GLH.JRNENTRY, GLH.BACHNUMB, GLH.REFRENCE, GLH.TRXDATE,
            CASE WHEN GLH.TRXTYPE = 0 THEN ''N'' WHEN GLH.TRXTYPE = 1 THEN ''Y'' END AS GP_REVERSALFLAG,
            GLH.PERIODID, FP1.PERIODMONTH, GLH.OPENYEAR, GLH.REVPRDID, FP2.PERIODMONTH, GLH.REVYEAR,
            GPD.ACTINDX, GPD.ACTNUMST, GLH.DSCRIPTN, GLH.INTERID,
            GPD.GP_DEBITAMOUNT, GPD.GP_CREDITAMOUNT, GPD.GP_ORIGINATINGDEBITAMOUNT, GPD.GP_ORIGINATINGCREDITAMOUNT
        FROM DYNAMICS.DBO.IA_RUN R
        INNER JOIN IA_AGGR IA ON R.RUNID = IA.RUNID
        LEFT JOIN ' + QUOTENAME(@CURRENTCOMPANY) + N'.' + QUOTENAME('DBO') + N'.' + QUOTENAME('GL10000') + N' GLH 
            ON R.BATCHID = GLH.BACHNUMB 
            AND R.JOURNALENTRY = GLH.JRNENTRY
        LEFT JOIN GP_AGGR GPD 
            ON GLH.BACHNUMB = GPD.BACHNUMB 
            AND GLH.JRNENTRY = GPD.JRNENTRY 
            AND IA.ACCOUNT = GPD.ACTNUMST
        LEFT JOIN DYNAMICS.DBO.IA_FISCAL_PERIOD FP1 
            ON GLH.PERIODID = FP1.PERIODID
        LEFT JOIN DYNAMICS.DBO.IA_FISCAL_PERIOD FP2 
            ON GLH.REVPRDID = FP2.PERIODID
        WHERE R.RUNID = @P_RUNID;';

        EXEC SYS.SP_EXECUTESQL @SQL, @PARAMS, @P_RUNID = @CURRENTRUNID;

        FETCH NEXT FROM RUN_CURSOR INTO @CURRENTRUNID, @CURRENTCOMPANY;
    END

    CLOSE RUN_CURSOR;
    DEALLOCATE RUN_CURSOR;

    /* RETURN CONSOLIDATED AUDIT RESULTS TO THE CALLER */
    SELECT * FROM #AUDITRESULTS;

END
GO




SQL Script: Finalized Audit Procedure dbo.usp_IA_Run_Audit

/********************************************************************************
PROCEDURE:  dbo.usp_IA_Run_Audit
AUTHOR:     Senior SQL Database Architect
PURPOSE:    Compares Intracompany (IA) staging data with Great Plains (GP) 
            Work tables (GL10000/GL10001) to verify data integrity post-posting.
            This procedure identifies discrepancies by aggregating line items 
            before joining to eliminate many-to-many duplication issues.
********************************************************************************/

CREATE OR ALTER PROCEDURE dbo.usp_IA_Run_Audit (
    @JobID UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    -- 1. Audit Results Temporary Table Construction
    -- Schema aligned with Source Context 1.png and IA_Line definitions
    IF OBJECT_ID('tempdb..#AuditResults') IS NOT NULL
        DROP TABLE #AuditResults;

    CREATE TABLE #AuditResults (
        AuditResultID               INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        JobID                       UNIQUEIDENTIFIER,
        runid                       UNIQUEIDENTIFIER,
        [FILENAME]                  NVARCHAR(520),
        CompanyA                    VARCHAR(10),
        BatchID                     CHAR(15),
        JournalEntry                INT,
        TrxDate                     DATE,
        [Reference]                 VARCHAR(30),
        [STATUS]                    VARCHAR(20),
        [MESSAGE]                   VARCHAR(4000),
        ReversalFlag                CHAR(1),
        TrxDateBasis                DATE,
        ReversalDate                DATE,
        Account                     CHAR(129),
        AcctDesc                    VARCHAR(255),
        Amount                      NUMERIC(19, 5),
        DebitAmt                    NUMERIC(19, 5),
        CreditAmt                   NUMERIC(19, 5),
        NetAmt                      NUMERIC(19, 5),
        /* GP Values */
        gp_Journal                  INT,
        gp_BatchNumber              CHAR(15),
        gp_Reference                CHAR(31),
        gp_TrxDate                  DATETIME,
        gp_ReversalFlag             CHAR(1),
        gp_PeriodID                 SMALLINT,
        gp_PeriodMonth              INT,
        gp_OpenYear                 SMALLINT,
        gp_ReversalPeriodID         SMALLINT,
        gp_ReversalPeriodMonth      INT,
        gp_ReversalYear             SMALLINT,
        gp_AccountIndex             INT,
        gp_Account                  CHAR(129),
        gp_Description              CHAR(31),
        gp_Company                  CHAR(5),
        gp_DebitAmount              NUMERIC(19, 5),
        gp_CreditAmount             NUMERIC(19, 5),
        gp_OriginatingDebitAmount   NUMERIC(19, 5),
        gp_OriginatingCreditAmount  NUMERIC(19, 5)
    );

    -- 2. Iteration Logic: Cursor Initialization
    DECLARE @currentRunID UNIQUEIDENTIFIER;
    DECLARE @currentCompany VARCHAR(20);
    DECLARE @sql NVARCHAR(MAX);
    DECLARE @params NVARCHAR(MAX) = N'@runid UNIQUEIDENTIFIER';

    DECLARE run_cursor CURSOR LOCAL FORWARD_ONLY READ_ONLY FOR
    SELECT RunID, CompanyA
    FROM DYNAMICS.dbo.IA_Run
    WHERE JobID = @JobID;

    OPEN run_cursor;
    FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- 3. Dynamic SQL Core Logic
        -- Implements Pre-Aggregation subqueries to ensure account-level accuracy
        -- Join Chain: IA_Line -> GL00100 (Account Master) -> GL10001 (GP Lines)
        SET @sql = N'
        INSERT INTO #AuditResults (
            JobID, runid, [FILENAME], CompanyA, BatchID, JournalEntry, TrxDate, [Reference], [STATUS], [MESSAGE], 
            ReversalFlag, TrxDateBasis, ReversalDate, Account, AcctDesc, Amount, DebitAmt, CreditAmt, NetAmt,
            gp_Journal, gp_BatchNumber, gp_Reference, gp_TrxDate, gp_ReversalFlag, 
            gp_PeriodID, gp_PeriodMonth, gp_OpenYear, 
            gp_ReversalPeriodID, gp_ReversalPeriodMonth, gp_ReversalYear,
            gp_AccountIndex, gp_Account, gp_Description, gp_Company, 
            gp_DebitAmount, gp_CreditAmount, gp_OriginatingDebitAmount, gp_OriginatingCreditAmount
        )
        SELECT 
            r.JobID, 
            r.RunID, 
            r.[FileName], 
            r.CompanyA, 
            r.BatchID, 
            r.JournalEntry, 
            r.TrxDate, 
            r.[Reference], 
            r.[Status], 
            r.[Message],
            r.ReversalFlag, 
            r.TrxDateBasis, 
            r.ReversalDate, 
            il.Account, 
            il.AcctDesc, 
            il.Amount, 
            il.DebitAmt, 
            il.CreditAmt, 
            (il.DebitAmt - il.CreditAmt) AS NetAmt,
            glh.JRNENTRY, 
            glh.BACHNUMB, 
            glh.REFRENCE, 
            glh.TRXDATE, 
            CASE WHEN glh.TRXTYPE = 0 THEN ''N'' WHEN glh.TRXTYPE = 1 THEN ''Y'' ELSE NULL END,
            glh.PERIODID, 
            fp.PeriodMonth, 
            glh.OPENYEAR,
            glh.REVPRDID, 
            fp2.PeriodMonth, 
            glh.REVYEAR,
            gld.ACTINDX, 
            act.ACTNUMST, 
            act.DSCRIPTN, 
            gld.INTERID,
            gld.DEBITAMT, 
            gld.CRDTAMNT, 
            gld.ORDBTAMT, 
            gld.ORCRDAMT
        FROM DYNAMICS.dbo.IA_Run r
        INNER JOIN (
            SELECT 
                RunID, 
                Account, 
                MAX(AcctDesc) AS AcctDesc, 
                SUM(Amount) AS Amount, 
                SUM(Debit) AS DebitAmt, 
                SUM(Credit) AS CreditAmt
            FROM DYNAMICS.dbo.IA_Line
            WHERE RunID = @runid
            GROUP BY RunID, Account
        ) il ON r.RunID = il.RunID
        LEFT JOIN ' + QUOTENAME(@currentCompany) + N'.dbo.GL10000 glh ON glh.BACHNUMB = r.BatchID
        LEFT JOIN ' + QUOTENAME(@currentCompany) + N'.dbo.GL00100 act ON il.Account = act.ACTNUMST
        LEFT JOIN (
            SELECT 
                JRNENTRY, 
                ACTINDX, 
                MAX(INTERID) AS INTERID,
                SUM(DEBITAMT) AS DEBITAMT, 
                SUM(CRDTAMNT) AS CRDTAMNT, 
                SUM(ORDBTAMT) AS ORDBTAMT, 
                SUM(ORCRDAMT) AS ORCRDAMT
            FROM ' + QUOTENAME(@currentCompany) + N'.dbo.GL10001
            GROUP BY JRNENTRY, ACTINDX
        ) gld ON glh.JRNENTRY = gld.JRNENTRY AND act.ACTINDX = gld.ACTINDX
        LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp ON glh.PERIODID = fp.PeriodID
        LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
        WHERE r.RunID = @runid;';

        -- 4. Execution
        EXEC sys.sp_executesql @sql, @params, @runid = @currentRunID;

        FETCH NEXT FROM run_cursor INTO @currentRunID, @currentCompany;
    END

    -- 5. Cleanup and Final Output
    CLOSE run_cursor;
    DEALLOCATE run_cursor;

    -- Return Audit Results to caller
    SELECT * FROM #AuditResults ORDER BY CompanyA, BatchID, Account;

    DROP TABLE #AuditResults;

END
GO
