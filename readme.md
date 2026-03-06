-- ==============================================================================
        -- OUTPUT: UPLOADED (RAW) vs DYNAMICS GP (NETTED)
        -- ==============================================================================
        SELECT 
            Company,
            BatchId AS gp_BatchNumber,
            JournalEntry AS gp_Journal,
            Account,
            
            -- 1. The Uploaded Side (Raw, exactly as it was in the CSV/Staging)
            IntDebit AS Uploaded_Debit,
            IntCredit AS Uploaded_Credit,
            
            -- 2. The GP Actual Side (Netted, exactly as it appears in the GP UI)
            CASE WHEN (GpDebit - GpCredit) > 0 THEN (GpDebit - GpCredit) ELSE 0 END AS gp_DebitAmount,
            CASE WHEN (GpCredit - GpDebit) > 0 THEN (GpCredit - GpDebit) ELSE 0 END AS gp_CreditAmount,
            
            -- 3. The True Variance (Proves the raw upload mathematically matches the netted GP state)
            ((IntDebit - IntCredit) - (GpDebit - GpCredit)) AS Net_Variance,
            
            GpState AS gp_Status
        FROM #JobRecon
        ORDER BY Company, JournalEntry, Account;
