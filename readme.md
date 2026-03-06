# ==============================================================================
# 2. Build the Summary Rows
# ==============================================================================
$SummaryGroups = $Details | Group-Object Company, gp_BatchNumber, gp_Status
$SummaryHtml = ""

foreach ($Group in $SummaryGroups) {
    $Comp = $Group.Group[0].Company
    $Batch = $Group.Group[0].gp_BatchNumber
    $LineCount = $Group.Count

    # SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
    $RawStatus = $Group.Group[0].gp_Status
    $CleanStatus = if ($null -ne $RawStatus) { $RawStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

    # Strict array evaluation
    $StatusClass = if ($CleanStatus -in 'POSTED', 'UNPOSTED') { "status-ok" } else { "status-fail" }

    $SummaryHtml += "<tr>
        <td>$Comp</td>
        <td>$Batch</td>
        <td class='center'><span class='$StatusClass'>$CleanStatus</span></td>
        <td class='right'>$LineCount</td>
    </tr>`n"
}

# ==============================================================================
# 3. Build the Detail Rows (Grouped by Company with Subtotals)
# ==============================================================================
$DetailHtml = ""
$GroupedDetails = $Details | Group-Object Company

foreach ($CompGroup in $GroupedDetails) {
    $CompanyName = $CompGroup.Name
    
    $DetailHtml += "<tr class='company-header'><td colspan='9'>Company: $CompanyName</td></tr>`n"

    $SubUpDeb = 0; $SubUpCred = 0; $SubGpDeb = 0; $SubGpCred = 0; $SubVar = 0

    foreach ($Row in $CompGroup.Group) {
        $SubUpDeb  += $Row.Uploaded_Debit
        $SubUpCred += $Row.Uploaded_Credit
        $SubGpDeb  += $Row.gp_DebitAmount
        $SubGpCred += $Row.gp_CreditAmount
        $SubVar    += $Row.Net_Variance

        $UpDeb  = "{0:N2}" -f $Row.Uploaded_Debit
        $UpCred = "{0:N2}" -f $Row.Uploaded_Credit
        $GpDeb  = "{0:N2}" -f $Row.gp_DebitAmount
        $GpCred = "{0:N2}" -f $Row.gp_CreditAmount
        $Var    = "{0:N2}" -f $Row.Net_Variance

        # SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
        $RawRowStatus = $Row.gp_Status
        $CleanRowStatus = if ($null -ne $RawRowStatus) { $RawRowStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

        # Strict array evaluation
        $StatusClass = if ($CleanRowStatus -in 'POSTED', 'UNPOSTED') { "status-ok" } else { "status-fail" }
        
        $VarClass = if ($Row.Net_Variance -ne 0) { "variance-bad right" } else { "variance-good right" }

        $DetailHtml += "<tr>
            <td>$($Row.Company)</td>
            <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td>
            <td>$($Row.Account)</td>
            <td class='right'>$UpDeb</td>
            <td class='right'>$UpCred</td>
            <td class='right'>$GpDeb</td>
            <td class='right'>$GpCred</td>
            <td class='$VarClass'>$Var</td>
            <td class='center'><span class='$StatusClass'>$CleanRowStatus</span></td>
        </tr>`n"
    }

    $DetailHtml += "<tr class='subtotal-row'>
        <td colspan='3' class='right'>$CompanyName Totals:</td>
        <td class='right'>$("{0:N2}" -f $SubUpDeb)</td>
        <td class='right'>$("{0:N2}" -f $SubUpCred)</td>
        <td class='right'>$("{0:N2}" -f $SubGpDeb)</td>
        <td class='right'>$("{0:N2}" -f $SubGpCred)</td>
        <td class='right'>$("{0:N2}" -f $SubVar)</td>
        <td></td>
    </tr>`n"
}
