Here is the fully rewritten S02 procedure.
I’ve structured it for readability by grouping the column alignments, brought the table variable declaration to the top, and integrated the BoA_AV_Lifecycle initialization.
I also gave you some pushback on your CTE. You had SELECT TOP 100 PERCENT coupled with an ORDER BY e.VendorId in there. Using TOP 100 PERCENT is an old SQL hack to force an ORDER BY inside a view or CTE, but since you are already filtering for a single @VendorId, sorting it does absolutely nothing except waste optimizer cycles. I ripped that out so the CTE is clean and purposeful.
CREATE PROCEDURE dbo.usp_BoA_AV_Req_S02_CreateVendorRequests
(
    @VendorId NVARCHAR(30),
    @DefaultValidation NVARCHAR(20) = N'STATUSANDOWNER'
)
AS
BEGIN
    SET NOCOUNT ON;

    /* Purpose: Create request rows in BoA_AV_Req_Outbox for a single vendor 
             that S01 identified as changed, and initialize the Lifecycle tracker.
    Rules:   - No date logic or hash logic
             - Insert-only, Idempotent via NOT EXISTS
             - One vendor at a time
    */

    ------------------------------------------------------------
    -- Guard clause
    ------------------------------------------------------------
    IF @VendorId IS NULL RETURN;

    ------------------------------------------------------------
    -- Variables
    ------------------------------------------------------------
    -- Table variable to capture the newly generated EndToEndId
    DECLARE @InsertedRows TABLE (
        EndToEndId NVARCHAR(36),
        VendorId   NVARCHAR(30)
    );

    ------------------------------------------------------------
    -- Source data for this vendor
    ------------------------------------------------------------
    ;WITH src AS (
        SELECT 
            e.VendorId,
            v.VENDNAME,
            v.Addr1,
            v.Addr2,
            v.Addr3,
            v.City,
            v.State,
            v.PostalCode,
            v.Country,
            v.WorkPhone,
            v.HomePhone,
            dbo.fn_BoA_DigitsOnly(e.RoutingNumber) AS RoutingNumber,
            dbo.fn_BoA_DigitsOnly(e.AccountNumber) AS AccountNumber,
            COALESCE(e.ValidationType, @DefaultValidation) AS ValidationType
        FROM dbo.v_GP_VendorEFT e
        JOIN dbo.v_GP_VendorPreferredAddress v ON v.VENDORID = e.VendorId
        WHERE e.VendorId = @VendorId
    )
    ------------------------------------------------------------
    -- Insert new Outbox rows only & Output the IDs
    ------------------------------------------------------------
    INSERT dbo.BoA_AV_Req_Outbox (
        EndToEndId, ValidationType, AccountNumber, RoutingNumber, AgentQualifier,
        AgentCountry, NamePrefix, NameSuffix, FirstName, MiddleName, LastName,
        BusinessName, Addr1, Addr2, Addr3, PostalCode, City, State, Country,
        WorkPhone, HomePhone, VendorId, VendorName, Status, FileName,
        CreatedUtc, LastUpdatedUtc, ErrorText
    )
    OUTPUT inserted.EndToEndId, inserted.VendorId 
      INTO @InsertedRows (EndToEndId, VendorId)
    SELECT 
        CONVERT(NVARCHAR(36), NEWID()), 
        UPPER(s.ValidationType), 
        ISNULL(s.AccountNumber, N''), 
        ISNULL(s.RoutingNumber, N''), 
        'USABA', 
        'US', 
        NULL, 
        NULL, 
        NULL, 
        NULL, 
        NULL, 
        dbo.fn_BoA_SanitizeText(s.VENDNAME), 
        dbo.fn_BoA_SanitizeText(s.Addr1), 
        dbo.fn_BoA_SanitizeText(s.Addr2), 
        dbo.fn_BoA_SanitizeText(s.Addr3), 
        dbo.fn_BoA_SanitizeText(s.PostalCode), 
        dbo.fn_BoA_SanitizeText(s.City), 
        dbo.fn_BoA_SanitizeText(s.State), 
        'US', 
        dbo.fn_BoA_DigitsOnly(s.WorkPhone), 
        dbo.fn_BoA_DigitsOnly(s.HomePhone), 
        s.VendorId, 
        dbo.fn_BoA_SanitizeText(s.VENDNAME), 
        N'Pending', 
        NULL, 
        SYSUTCDATETIME(), 
        SYSUTCDATETIME(), 
        NULL
    FROM src s
    WHERE NOT EXISTS (
        SELECT 1 
        FROM dbo.BoA_AV_Req_Outbox o 
        WHERE o.VendorId      = s.VendorId 
          AND o.RoutingNumber = s.RoutingNumber 
          AND o.AccountNumber = s.AccountNumber 
          AND o.Status IN ('Pending', 'ReadyForBatch', 'ReadyForUpload')
    );

    ------------------------------------------------------------
    -- Initialize the Lifecycle record
    ------------------------------------------------------------
    INSERT dbo.BoA_AV_Lifecycle (
        EndToEndId, 
        VendorId, 
        RequestCreatedUtc, 
        ProcessingStatus,
        CreatedUtc,
        LastUpdatedUtc
    )
    SELECT 
        EndToEndId, 
        VendorId, 
        SYSUTCDATETIME(), 
        'CREATED',
        SYSUTCDATETIME(),
        SYSUTCDATETIME()
    FROM @InsertedRows;

END;
GO

Would you like me to rewrite S04 (usp_BoA_AV_Req_S04_WriteFile) next so it updates that newly created Lifecycle record to 'SENT' after BCP drops the file?
