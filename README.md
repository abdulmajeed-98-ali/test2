# ==========================================
# USER INPUT
# ==========================================

$RootPath = Read-Host "Enter the Root Share Path (Example: \\server\share)"
$LogPath  = Read-Host "Enter the Log Folder Path (Example: C:\Logs)"

if (!(Test-Path $RootPath)) {
    Write-Host "Root path not found."
    exit
}

if (!(Test-Path $LogPath)) {
    New-Item -ItemType Directory -Path $LogPath | Out-Null
}

$TimeStamp = Get-Date -Format "yyyyMMdd_HHmmss"

$CsvReport = Join-Path $LogPath "PermissionReport_$TimeStamp.csv"
$LogFile   = Join-Path $LogPath "PermissionLog_$TimeStamp.log"
$EmptyFile = Join-Path $LogPath "EmptyFolders_$TimeStamp.txt"

# ==========================================
# LOG FUNCTION
# ==========================================

function Write-Log {
    param ($Message)
    $Time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$Time - $Message" | Out-File -Append $LogFile
    Write-Host "$Time - $Message"
}

Write-Log "Starting scan on $RootPath"

# ==========================================
# STEP 1 – GET ALL FOLDERS
# ==========================================

$AllFolders = Get-ChildItem $RootPath -Recurse -Directory

Write-Log "Total folders found: $($AllFolders.Count)"

# ==========================================
# STEP 2 – IDENTIFY FOLDERS CONTAINING FILES
# ==========================================

$FoldersWithFiles = New-Object System.Collections.Generic.HashSet[string]

$AllFiles = Get-ChildItem $RootPath -Recurse -File

foreach ($file in $AllFiles) {

    $current = $file.Directory.FullName

    # Mark parent and all ancestors up to root
    while ($current -and $current.StartsWith($RootPath)) {
        $FoldersWithFiles.Add($current) | Out-Null
        $current = Split-Path $current -Parent
    }
}

Write-Log "Folders containing files detected: $($FoldersWithFiles.Count)"

# ==========================================
# STEP 3 – BREAK INHERITANCE ON DATA FOLDERS
# ==========================================

foreach ($folder in $FoldersWithFiles) {

    Write-Log "Breaking inheritance on data folder: $folder"

    # Disable inheritance but keep existing permissions
    icacls "$folder" /inheritance:d | Out-Null
}

# ==========================================
# STEP 4 – IDENTIFY EMPTY FOLDERS
# ==========================================

$EmptyFolders = $AllFolders | Where-Object {
    -not $FoldersWithFiles.Contains($_.FullName)
}

Write-Log "Empty folders detected: $($EmptyFolders.Count)"

# ==========================================
# STEP 5 – APPLY EVERYONE FULL CONTROL
# ==========================================

$Results = @()

foreach ($folder in $EmptyFolders) {

    $folder.FullName | Out-File -Append $EmptyFile

    Write-Log "Applying Everyone permission on empty folder: $($folder.FullName)"

    # Add Everyone FullControl safely
    icacls "$($folder.FullName)" /grant Everyone:(OI)(CI)F | Out-Null

    # Validation check
    $check = icacls "$($folder.FullName)"

    if ($check -match "Everyone") {
        $Status = "Success"
        Write-Log "SUCCESS: Everyone applied on $($folder.FullName)"
    }
    else {
        $Status = "Failed"
        Write-Log "FAILED: Everyone not found on $($folder.FullName)"
    }

    $Results += [PSCustomObject]@{
        FolderPath        = $folder.FullName
        PermissionApplied = "Everyone - FullControl"
        Status            = $Status
        TimeStamp         = Get-Date
    }
}

# ==========================================
# EXPORT REPORT
# ==========================================

$Results | Export-Csv -NoTypeInformation $CsvReport

Write-Log "Completed successfully."
Write-Host "Completed. Reports saved to $LogPath"
