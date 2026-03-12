foreach ($Group in $SummaryGroups) {
    $Row = $Group.Group[0]
    $CleanStatus = if ($null -ne $Row.gp_Status) { $Row.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
    
    # Simple check for the reversal text
    $RevText = if ($Row.Reversal -eq 'Y') { "Yes ($($Row.ReversalDate.ToShortDateString()))" } else { "No" }
    
    $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
    
    # Add the $RevText into a new <td>
    $SummaryHtml += "<tr> <td>$($Row.Company)</td> <td>$($Row.gp_BatchNumber)</td> <td class='center'>$RevText</td> <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> <td class='right'>$($Group.Count)</td> </tr>`n"
}


<thead>
    <tr>
        <th>Company</th>
        <th>Batch ID</th>
        <th class="center">Reversal?</th> <th class="center">Status</th>
        <th class="right">Total Uploaded Lines</th>
    </tr>
</thead>
