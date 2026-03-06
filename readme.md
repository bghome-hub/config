/* Grouping & Totals */
        .company-header { background-color: #2c3e50; color: #ffffff; font-weight: bold; font-size: 14px; }
        .company-header td { padding: 10px; border-bottom: none; }
        .subtotal-row { background-color: #ecf0f1; font-weight: bold; }
        .subtotal-row td { border-top: 2px solid #bdc3c7; }




        step 3
        # 3. Build the Detail Rows (Grouped by Company with Subtotals)
$DetailHtml = ""

# Group the raw SQL results by Company
$GroupedDetails = $Details | Group-Object Company

foreach ($CompGroup in $GroupedDetails) {
    $CompanyName = $CompGroup.Name
    
    # Inject the Company Delineation Header
    $DetailHtml += "<tr class='company-header'><td colspan='9'>Company: $CompanyName</td></tr>`n"

    # Initialize Math Variables for the Subtotal
    $SubUpDeb = 0; $SubUpCred = 0; $SubGpDeb = 0; $SubGpCred = 0; $SubVar = 0

    foreach ($Row in $CompGroup.Group) {
        # Accumulate the totals
        $SubUpDeb  += $Row.Uploaded_Debit
        $SubUpCred += $Row.Uploaded_Credit
        $SubGpDeb  += $Row.gp_DebitAmount
        $SubGpCred += $Row.gp_CreditAmount
        $SubVar    += $Row.Net_Variance

        # Format as Money
        $UpDeb  = "{0:N2}" -f $Row.Uploaded_Debit
        $UpCred = "{0:N2}" -f $Row.Uploaded_Credit
        $GpDeb  = "{0:N2}" -f $Row.gp_DebitAmount
        $GpCred = "{0:N2}" -f $Row.gp_CreditAmount
        $Var    = "{0:N2}" -f $Row.Net_Variance

        # Exact String Match for Status (The Bug Fix)
        $StatusClass = if ($Row.gp_Status -eq "POSTED" -or $Row.gp_Status -eq "UNPOSTED") { "status-ok" } else { "status-fail" }
        
        # Variance Highlighting
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
            <td class='center'><span class='$StatusClass'>$($Row.gp_Status)</span></td>
        </tr>`n"
    }

    # Inject the Subtotal Footer for this Company
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
