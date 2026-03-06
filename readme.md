.status-pending { color: #856404; background-color: #fff3cd; font-weight: bold; padding: 3px 6px; border-radius: 3px; display: inline-block;}


# SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
    $RawStatus = $Group.Group[0].gp_Status
    $CleanStatus = if ($null -ne $RawStatus) { $RawStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

    # Strict Switch Evaluation
    switch ($CleanStatus) {
        'POSTED'   { $StatusClass = 'status-ok'; break }
        'UNPOSTED' { $StatusClass = 'status-pending'; break }
        default    { $StatusClass = 'status-fail'; break }
    }

    # SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
        $RawRowStatus = $Row.gp_Status
        $CleanRowStatus = if ($null -ne $RawRowStatus) { $RawRowStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

        # Strict Switch Evaluation
        switch ($CleanRowStatus) {
            'POSTED'   { $StatusClass = 'status-ok'; break }
            'UNPOSTED' { $StatusClass = 'status-pending'; break }
            default    { $StatusClass = 'status-fail'; break }
        }
