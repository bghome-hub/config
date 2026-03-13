$SQLServerConnection = New-Object System.Data.SqlClient.SqlConnection
$SQLServerConnection.ConnectionString = "Data Source=$(ESCAPE_NONE(SRVR));Initial Catalog=master;Integrated Security=SSPI;Application Name=syspolicy_purge_history"
$PolicyStoreConnection = New-Object Microsoft.SqlServer.Management.Sdk.Sfc.SqlStoreConnection($SQLServerConnection)
$PolicyStore = New-Object Microsoft.SqlServer.Management.Dmf.PolicyStore ($PolicyStoreConnection)
$PolicyStore.EraseSystemHealthPhantomRecords()
