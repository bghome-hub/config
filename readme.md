M dynamics.dbo.ia_Run r

-- 1. PRE-AGGREGATE Source Lines by Account
LEFT JOIN (
    SELECT RunID, Account, 
           MAX(AcctDesc) AS AcctDesc, 
           MIN(TrxDate) AS TrxDate,
           SUM(Amount) AS Amount,
           SUM(Debit) AS Debit, 
           SUM(Credit) AS Credit
    FROM dynamics.dbo.IA_Line
    GROUP BY RunID, Account
) l ON l.RunID = r.RunID

-- 2. GP Header (Matching both Batch and JE safely)
LEFT JOIN [' + @currentCompany + N'].dbo.GL10000 glh   
    ON glh.BACHNUMB = r.BatchID AND glh.JRNENTRY = r.JournalEntry
    
-- 3. Account Master (Link source string account to GP integer index)
LEFT JOIN [' + @currentCompany + N'].dbo.GL00105 act   
    ON RTRIM(l.Account) = RTRIM(act.ACTNUMST)
    
-- 4. PRE-AGGREGATE GP Lines by Account to kill duplicates entirely
LEFT JOIN (
    SELECT JRNENTRY, ACTINDX, 
           MAX(DSCRIPTN) AS DSCRIPTN, 
           MAX(INTERID) AS INTERID,
           SUM(DEBITAMT) AS DEBITAMT, 
           SUM(CRDTAMNT) AS CRDTAMNT,
           SUM(ORDBTAMT) AS ORDBTAMT, 
           SUM(ORCRDAMT) AS ORCRDAMT
    FROM [' + @currentCompany + N'].dbo.GL10001
    GROUP BY JRNENTRY, ACTINDX
) gld ON glh.JRNENTRY = gld.JRNENTRY AND act.ACTINDX = gld.ACTINDX

-- 5. Fiscal Periods
LEFT JOIN dynamics.dbo.ia_fiscal_period fp ON glh.PERIODID = fp.PeriodID
LEFT JOIN dynamics.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
