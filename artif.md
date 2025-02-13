Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Artifactory details
$artifactoryUrl = "https://your-artifactory-instance.com"
$apiEndpoint = "$artifactoryUrl/api/repositories"

# Create Form
$form = New-Object System.Windows.Forms.Form
$form.Text = "Artifactory File Uploader"
$form.Size = New-Object System.Drawing.Size(500,400)
$form.StartPosition = "CenterScreen"

# Username Label and Textbox
$lblUser = New-Object System.Windows.Forms.Label
$lblUser.Text = "Username:"
$lblUser.Location = New-Object System.Drawing.Point(20,20)
$form.Controls.Add($lblUser)

$txtUser = New-Object System.Windows.Forms.TextBox
$txtUser.Location = New-Object System.Drawing.Point(120,18)
$txtUser.Width = 200
$form.Controls.Add($txtUser)

# Password Label and Textbox
$lblPass = New-Object System.Windows.Forms.Label
$lblPass.Text = "Password:"
$lblPass.Location = New-Object System.Drawing.Point(20,50)
$form.Controls.Add($lblPass)

$txtPass = New-Object System.Windows.Forms.TextBox
$txtPass.Location = New-Object System.Drawing.Point(120,48)
$txtPass.Width = 200
$txtPass.PasswordChar = '*'
$form.Controls.Add($txtPass)

# Login Button
$btnLogin = New-Object System.Windows.Forms.Button
$btnLogin.Text = "Login"
$btnLogin.Location = New-Object System.Drawing.Point(340, 35)
$form.Controls.Add($btnLogin)

# Repository Dropdown
$lblRepo = New-Object System.Windows.Forms.Label
$lblRepo.Text = "Select Destination:"
$lblRepo.Location = New-Object System.Drawing.Point(20,90)
$form.Controls.Add($lblRepo)

$cmbRepo = New-Object System.Windows.Forms.ComboBox
$cmbRepo.Location = New-Object System.Drawing.Point(120,88)
$cmbRepo.Width = 250
$form.Controls.Add($cmbRepo)

# File Selection Button
$lblFile = New-Object System.Windows.Forms.Label
$lblFile.Text = "Select File:"
$lblFile.Location = New-Object System.Drawing.Point(20,130)
$form.Controls.Add($lblFile)

$txtFilePath = New-Object System.Windows.Forms.TextBox
$txtFilePath.Location = New-Object System.Drawing.Point(120,128)
$txtFilePath.Width = 250
$form.Controls.Add($txtFilePath)

$btnBrowse = New-Object System.Windows.Forms.Button
$btnBrowse.Text = "Browse"
$btnBrowse.Location = New-Object System.Drawing.Point(380,126)
$form.Controls.Add($btnBrowse)

# User Selection Dropdown
$lblUploader = New-Object System.Windows.Forms.Label
$lblUploader.Text = "Upload As:"
$lblUploader.Location = New-Object System.Drawing.Point(20,170)
$form.Controls.Add($lblUploader)

$cmbUploader = New-Object System.Windows.Forms.ComboBox
$cmbUploader.Location = New-Object System.Drawing.Point(120,168)
$cmbUploader.Width = 250
$cmbUploader.Items.AddRange(@("User1", "User2", "User3")) # Modify with actual users
$form.Controls.Add($cmbUploader)

# Upload Button
$btnUpload = New-Object System.Windows.Forms.Button
$btnUpload.Text = "Upload"
$btnUpload.Location = New-Object System.Drawing.Point(200, 220)
$form.Controls.Add($btnUpload)

# Function to get repositories from Artifactory
function Get-ArtifactoryRepositories {
    param (
        [string]$user,
        [string]$password
    )
    $pair = "$user`:$password"
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($pair)
    $base64 = [System.Convert]::ToBase64String($bytes)
    $headers = @{Authorization = "Basic $base64"}

    try {
        $response = Invoke-RestMethod -Uri $apiEndpoint -Headers $headers -Method Get
        return $response | ForEach-Object { $_.key }
    } catch {
        [System.Windows.Forms.MessageBox]::Show("Error retrieving repositories: $_", "Error", "OK", "Error")
        return @()
    }
}

# Login Button Click Event
$btnLogin.Add_Click({
    $repositories = Get-ArtifactoryRepositories -user $txtUser.Text -password $txtPass.Text
    $cmbRepo.Items.Clear()
    $cmbRepo.Items.AddRange($repositories)
})

# Browse Button Click Event
$btnBrowse.Add_Click({
    $openFileDialog = New-Object System.Windows.Forms.OpenFileDialog
    if ($openFileDialog.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {
        $txtFilePath.Text = $openFileDialog.FileName
    }
})

# Upload Function
function Upload-ToArtifactory {
    param (
        [string]$filePath,
        [string]$repo,
        [string]$user,
        [string]$password
    )
    if (-not (Test-Path $filePath)) {
        [System.Windows.Forms.MessageBox]::Show("File not found!", "Error", "OK", "Error")
        return
    }

    $fileName = [System.IO.Path]::GetFileName($filePath)
    $destinationUrl = "$artifactoryUrl/$repo/$fileName"

    $pair = "$user`:$password"
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($pair)
    $base64 = [System.Convert]::ToBase64String($bytes)
    $headers = @{Authorization = "Basic $base64"}

    try {
        Invoke-RestMethod -Uri $destinationUrl -Headers $headers -Method Put -InFile $filePath -ContentType "application/octet-stream"
        [System.Windows.Forms.MessageBox]::Show("Upload Successful!", "Success", "OK", "Information")
    } catch {
        [System.Windows.Forms.MessageBox]::Show("Upload Failed: $_", "Error", "OK", "Error")
    }
}

# Upload Button Click Event
$btnUpload.Add_Click({
    if (-not $txtFilePath.Text -or -not $cmbRepo.SelectedItem) {
        [System.Windows.Forms.MessageBox]::Show("Please select a file and destination!", "Warning", "OK", "Warning")
        return
    }
    Upload-ToArtifactory -filePath $txtFilePath.Text -repo $cmbRepo.SelectedItem -user $txtUser.Text -password $txtPass.Text
})

# Run the Form
$form.ShowDialog()
