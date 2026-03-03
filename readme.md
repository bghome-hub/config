
<!DOCTYPE html>
<html>
<head>
    <style>
        /* Overall Page Styling */
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            font-size: 13px; 
            color: #333333; 
            background-color: #f4f7f6;
            margin: 20px;
        }
        
        /* Headers */
        h2 { 
            color: #2c3e50; 
            border-bottom: 2px solid #3498db; 
            padding-bottom: 5px;
            margin-top: 30px;
        }
        h3 { 
            color: #34495e; 
            background-color: #e8ecf1;
            padding: 8px;
            border-left: 4px solid #2980b9;
            margin-top: 25px;
            margin-bottom: 0;
        }

        /* Table Styling */
        table { 
            border-collapse: collapse; 
            width: 100%; 
            margin-bottom: 20px; 
            background-color: #ffffff;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        th, td { 
            border: 1px solid #dddddd; 
            padding: 10px 12px; 
            text-align: left; 
        }
        th { 
            background-color: #2c3e50; 
            color: #ffffff; 
            font-weight: 600; 
            text-transform: uppercase;
            font-size: 12px;
            letter-spacing: 0.5px;
        }
        
        /* Sub-headers for the side-by-side GP vs Import */
        th.sub-header {
            background-color: #34495e;
            color: #ecf0f1;
        }

        /* Row Hover and Zebra Striping */
        tr:nth-child(even) { background-color: #f9f9f9; }
        tr:hover { background-color: #f1f5f8; }

        /* Status Pills */
        .status-success { 
            color: #155724; 
            background-color: #d4edda; 
            border: 1px solid #c3e6cb;
            padding: 3px 8px;
            border-radius: 12px;
            font-weight: bold;
            display: inline-block;
        }
        .status-error { 
            color: #721c24; 
            background-color: #f8d7da; 
            border: 1px solid #f5c6cb;
            padding: 3px 8px;
            border-radius: 12px;
            font-weight: bold;
            display: inline-block;
        }

        /* Number Formatting */
        .amt { 
            text-align: right; 
            font-family: 'Consolas', 'Courier New', monospace;
            font-weight: 600;
        }
    </style>
</head>
<body>
    <h2>Dynamics GP Import Job Summary</h2>
    <table>
        <thead>
            <tr>
                <th>Company</th>
                <th>Batch ID</th>
                <th>Journal Entry</th>
                <th>Status</th>
                <th class="amt">Import Net</th>
                <th class="amt">GP Net</th>
                <th>Message</th>
            </tr>
        </thead>
        <tbody>
            {{SummaryRows}}
        </tbody>
    </table>

    <h2>Import vs. GL Detail Lines</h2>
    <!-- The PowerShell script will inject the detailed tables here -->
    {{DetailTables}}
</body>
</html>

