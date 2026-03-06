
        INSERT INTO #JobReport (
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
            il.TrxDate, 
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
            gld.DSCRIPTN, 
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
				TrxDate,
                MAX(AcctDesc) AS AcctDesc, 
                SUM(Amount) AS Amount, 
                SUM(Debit) AS DebitAmt, 
                SUM(Credit) AS CreditAmt
            FROM DYNAMICS.dbo.IA_Line
            WHERE RunID = @runid
            GROUP BY RunID, Account, TrxDate
        ) il ON r.RunID = il.RunID
        LEFT JOIN ' + QUOTENAME(@currentCompany) + N'.dbo.GL10000 glh ON glh.BACHNUMB = r.BatchID
        LEFT JOIN ' + QUOTENAME(@currentCompany) + N'.dbo.GL00105 act ON il.Account = act.ACTNUMST
        LEFT JOIN (
            SELECT 
                JRNENTRY, 
                ACTINDX, 
				DSCRIPTN,
                MAX(INTERID) AS INTERID,
                SUM(DEBITAMT) AS DEBITAMT, 
                SUM(CRDTAMNT) AS CRDTAMNT, 
                SUM(ORDBTAMT) AS ORDBTAMT, 
                SUM(ORCRDAMT) AS ORCRDAMT
            FROM ' + QUOTENAME(@currentCompany) + N'.dbo.GL10001
            GROUP BY JRNENTRY, ACTINDX, DSCRIPTN
        ) gld ON glh.JRNENTRY = gld.JRNENTRY AND act.ACTINDX = gld.ACTINDX
        LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp ON glh.PERIODID = fp.PeriodID
        LEFT JOIN DYNAMICS.dbo.ia_fiscal_period fp2 ON glh.REVPRDID = fp2.PeriodID
        WHERE r.RunID = @runid;';
