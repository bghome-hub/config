/* eConnect Dynamic SQL */
        DECLARE @sql nvarchar(max) = N'
        USE ' + QUOTENAME(@Company) + N';
        SET NOCOUNT ON;
        DECLARE @JE_String char(13), @JE_Int int, @Err int=0, @ErrStr varchar(255)='''';
        
        /* 1. Get Next JE (Returns a String) */
        EXEC taGetNextJournalEntry 
            @I_vInc_Dec = 1,
            @O_vJournalEntryNumber = @JE_String OUTPUT, 
            @O_iErrorState = @Err OUTPUT;
            
        IF @Err<>0 RAISERROR(''taGetNextJournalEntry failed (%d)'',16,1,@Err);

        /* Convert to INT for the Transaction Procs */
        SET @JE_Int = CAST(@JE_String AS int);

        /* 2. Insert Lines */
        DECLARE @Seq numeric(19,5) = 16384;
        DECLARE L CURSOR LOCAL FAST_FORWARD FOR SELECT ACTNUMST, Debit, Credit FROM #Lines;
        OPEN L;
        DECLARE @A char(129), @D numeric(19,5), @C numeric(19,5);
        FETCH NEXT FROM L INTO @A,@D,@C;
        
        WHILE @@FETCH_STATUS=0 BEGIN
            EXEC taGLTransactionLineInsert 
                @I_vBACHNUMB = @B,
                @I_vJRNENTRY = @JE_Int,
                @I_vSQNCLINE = @Seq,
                @I_vACTNUMST = @A,
                @I_vDEBITAMT = @D,
                @I_vCRDTAMNT = @C,
                @I_vDSCRIPTN = @R,
                @O_iErrorState = @Err OUTPUT, 
                @oErrString = @ErrStr OUTPUT;
                
            IF @Err<>0 RAISERROR(''Line insert failed (%d) %s'',16,1,@Err,@ErrStr);
            
            SET @Seq = @Seq + 16384;
            FETCH NEXT FROM L INTO @A,@D,@C;
        END
        CLOSE L; DEALLOCATE L;

        /* 3. Insert Header (Last) */
        EXEC taGLTransactionHeaderInsert 
            @I_vBACHNUMB = @B,
            @I_vJRNENTRY = @JE_Int,
            @I_vREFRENCE = @R,
            @I_vTRXDATE = @DT,
            @I_vTRXTYPE = 0,
            @I_vSOURCDOC = ''GJ'',
            @O_iErrorState = @Err OUTPUT, 
            @oErrString = @ErrStr OUTPUT;
            
        IF @Err<>0 RAISERROR(''Header insert failed (%d) %s'',16,1,@Err,@ErrStr);

        /* Return Results */
        SELECT CAST(@JE_Int AS char(13)) AS JRNENTRY, @Err AS Err, @ErrStr AS ErrStr;
        ';
