foreach ($Group in $SummaryGroups) {
    $FirstRow = $Group.Group[0]
    $CleanStatus = if ($null -ne $FirstRow.gp_Status) { $FirstRow.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
    $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }

    # The Indicator: Shows 'Y (MM/DD/YY)' if reversal, or 'N' if not [cite: 153, 155]
    $RevIndicator = if ($FirstRow.Reversal -eq 'Y') { 
        "Y ($($FirstRow.ReversalDate.ToString('MM/dd/yy')))" 
    } else { 
        "N" 
    }

    $SummaryHtml += "<tr> 
        <td>$($FirstRow.Company)</td> 
        <td>$($FirstRow.gp_BatchNumber)</td> 
        <td class='center'>$RevIndicator</td> 
        <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> 
        <td class='right'>$($Group.Count)</td> 
    </tr>`n"
}
