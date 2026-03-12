foreach ($Group in $SummaryGroups) {
    $FirstRow = $Group.Group[0]
    $CleanStatus = if ($null -ne $FirstRow.gp_Status) { $FirstRow.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
    
    # Logic for Reversal Column display
    $ReversalDisplay = "No"
    if ($FirstRow.Reversal -eq 'Y') {
        $RevDateFormatted = Get-Date $FirstRow.ReversalDate -Format "MM/dd/yyyy"
        $ReversalDisplay = "Yes ($RevDateFormatted)"
    }

    $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
    
    $SummaryHtml += "<tr> 
        <td>$($FirstRow.Company)</td> 
        <td>$($FirstRow.gp_BatchNumber)</td> 
        <td class='center'>$ReversalDisplay</td> 
        <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> 
        <td class='right'>$($Group.Count)</td> 
    </tr>`n"
}



<h3>Execution Summary</h3>
<table>
    <thead>
        <tr>
            <th>Company</th>
            <th>Batch ID</th>
            <th class="center">Reversal (Date)</th>
            <th class="center">Status</th>
            <th class="right">Total Uploaded Lines</th>
        </tr>
    </thead>
    <tbody>
        {{SummaryRows}}
    </tbody>
</table>
