CREATE PROCEDURE dbo.usp_BoA_AV_Req_S05_SendEmail
(
    @BatchId UNIQUEIDENTIFIER,
    @FileName NVARCHAR(260),
    @ExportPath NVARCHAR(4000),
    @Status NVARCHAR(20), -- SUCCESS | FAILED
    @ErrorMessage NVARCHAR(4000) = NULL
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @subject NVARCHAR(255), 
            @body NVARCHAR(MAX), 
            @rowdetails NVARCHAR(MAX) = N'';

    ------------------------------------------------------------
    -- 1) Build row-level summary (read-only)
    ------------------------------------------------------------
    -- ISNULL added to VendorName to prevent the entire string from becoming NULL
    SELECT @rowdetails = @rowdetails + 
           RIGHT(SPACE(15) + o.VendorId, 15) + ' ' + 
           LEFT(ISNULL(o.VendorName, 'UNKNOWN') + SPACE(30), 30) + ' ' + 
           o.Status + CHAR(13) + CHAR(10)
    FROM dbo.BoA_AV_Req_Outbox o
    WHERE o.FileName = @FileName
    ORDER BY o.VendorId;

    IF @rowdetails = N'' 
        SET @rowdetails = N'(No rows found)';

    ------------------------------------------------------------
    -- 2) Subject
    ------------------------------------------------------------
    SET @subject = CONCAT('[BoA AV] ', @Status, ' - ', @FileName);

    ------------------------------------------------------------
    -- 3) Body
    ------------------------------------------------------------
    SET @body = N'BoA Account Verification Request File' + CHAR(13) + CHAR(10) +
                N'------------------------------------' + CHAR(13) + CHAR(10) +
                N'Status      : ' + @Status + CHAR(13) + CHAR(10) +
                N'Batch ID    : ' + CONVERT(NVARCHAR(36), @BatchId) + CHAR(13) + CHAR(10) +
                N'File Name   : ' + @FileName + CHAR(13) + CHAR(10) +
                N'Export Path : ' + COALESCE(@ExportPath, N'N/A') + CHAR(13) + CHAR(10) +
                N'UTC Time    : ' + CONVERT(NVARCHAR(19), SYSUTCDATETIME(), 120) + CHAR(13) + CHAR(10) +
                CHAR(13) + CHAR(10) +
                N'VendorId        VendorName                     Status' + CHAR(13) + CHAR(10) +
                N'--------------- ------------------------------ ----------' + CHAR(13) + CHAR(10) +
                @rowdetails;

    IF @ErrorMessage IS NOT NULL
        SET @body += CHAR(13) + CHAR(10) + N'Error:' + CHAR(13) + CHAR(10) + @ErrorMessage;

    ------------------------------------------------------------
    -- 4) Send email
    ------------------------------------------------------------
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'AccountVerification',
        @recipients = 'ben.gregory@trsga.com;jeff.ellard@trsga.com',
        @subject = @subject,
        @body = @body;
END;
GO
