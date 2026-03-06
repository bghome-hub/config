-- ==============================================================================
        -- OUTPUT: FORMATTED EXACTLY LIKE THE DYNAMICS GP UI
        -- ==============================================================================
        SELECT 
            Company,
            BatchId AS gp_BatchNumber,
            JournalEntry AS gp_Journal,
            Account,
            
            -- 1. The Uploaded Side (UI-Netted)
            CASE WHEN (IntDebit - IntCredit) > 0 THEN (IntDebit - IntCredit) ELSE 0 END AS Uploaded_Debit,
            CASE WHEN (IntCredit - IntDebit) > 0 THEN (IntCredit - IntDebit) ELSE 0 END AS Uploaded_Credit,
            
            -- 2. The GP Actual Side (UI-Netted)
            CASE WHEN (GpDebit - GpCredit) > 0 THEN (GpDebit - GpCredit) ELSE 0 END AS gp_DebitAmount,
            CASE WHEN (GpCredit - GpDebit) > 0 THEN (GpCredit - GpDebit) ELSE 0 END AS gp_CreditAmount,
            
            -- 3. The True Variance
            ((IntDebit - IntCredit) - (GpDebit - GpCredit)) AS Net_Variance,
            
            GpState AS gp_Status
        FROM #JobRecon
        ORDER BY Company, JournalEntry, Account;
