# Automating User Management and Group Policies in Windows Active Directory

### Project Overview
---
This project simulates a scenario in a medium-sized company, DamiAde, undergoing a departmental reorganisation. The task involves setting up a new Active Directory environment on Windows Server, creating and organising 200 user accounts based on department and role, and applying appropriate Group Policies for security and access control. PowerShell scripting is used for automating the bulk user creation process and validating the configuration.

### Tools and Technologies

- VirtualBox - used to run Windows 2022 
- Windows Server 2022
- Active Directory Domain Services (AD DS)
- PowerShell
- Group Policy Management Console (GPMC)

### Step 1. Preparation: Dummy Data Creation

I created a CSV file with 200 rows of dummy data using the below PowerShell script:

```Powershell
$firstNames = @("John", "Jane", "Michael", "Emily", "Daniel", "Sarah", "David", "Laura", "Chris", "Jessica",
                "Emma", "Liam", "Olivia", "Noah", "Ava", "Sophia", "Isabella", "Mason", "Logan", "Ethan", 
                "James", "Alexander", "Jacob", "Elijah", "Matthew", "Benjamin", "Lucas", "Henry", "Sebastian", 
                "Jack", "Oliver", "William", "Charlotte", "Amelia", "Harper", "Ella", "Abigail", "Zoe", "Chloe")

$lastNames = @("Smith", "Johnson", "Williams", "Brown", "Jones", "Miller", "Davis", "García", "Rodriguez", 
               "Martínez", "Hernandez", "Lopez", "Gonzalez", "Wilson", "Anderson", "Thomas", "Taylor", 
               "Moore", "Jackson", "Martin", "Lee", "Perez", "Thompson", "White", "Harris", "Sanchez", 
               "Clark", "Lewis", "Robinson", "Walker", "Hall", "Young", "King", "Wright", "Scott", "Green")

$recordCount = 200
$outputFile = "C:\Users\Administrator\Desktop\dummydata.csv"

$data = @()
$departments = @("HR", "IT", "Marketing", "Finance", "Sales")
$usedNames = @()

for ($i = 1; $i -le $recordCount; $i++) {
    $attempt = 0
    do {
        $firstName = $firstNames[(Get-Random -Minimum 0 -Maximum ($firstNames.Length - 1))]
        $lastName = $lastNames[(Get-Random -Minimum 0 -Maximum ($lastNames.Length - 1))]
        $fullName = "$firstName $lastName"
        $attempt++
        if ($attempt -gt 1000) {
            Write-Host "Unable to generate a unique name after 1000 attempts." -ForegroundColor Red
            exit
        }
    } until ($usedNames -notcontains $fullName)
    
    $usedNames += $fullName
    
    $email = "user$i@example.com"
    $username = "user$i"
    $phoneNumber = "+1234567890"
    $department = $departments[(Get-Random -Minimum 0 -Maximum ($departments.Length - 1))]
    $title = "Title$((Get-Random -Minimum 1 -Maximum 5))"
    
    $data += [pscustomobject]@{
        FirstName    = $firstName
        LastName     = $lastName
        Email        = $email
        Username     = $username
        PhoneNumber  = $phoneNumber
        Department   = $department
        Title        = $title
    }
}

$data | Export-Csv -Path $outputFile -NoTypeInformation



```

I then used the below code to confirm that the file content was correctly created:

```Powershell
$csvFile = "C:\Users\Administrator\Desktop\dummydata.csv"

if (Test-Path $csvFile) {
    $data = Import-Csv -Path $csvFile
    $data | Format-Table -AutoSize
} else {
    Write-Output "File not found: $csvFile"
}


```



![Screenshot 2024-08-18 183423](https://github.com/user-attachments/assets/da91e45e-2cdc-45d7-9d77-5805071f4641)



### Step 2. Automating Active Directory User Creation with PowerShell

I used the below Powershell script that reads the dummy CSV file created previously and uses that data to create accounts on Active Directory:

```Powershell

Import-Module ActiveDirectory

$csvFile = "C:\Users\Administrator\Desktop\dummydata.csv"
$userData = Import-Csv -Path $csvFile

foreach ($user in $userData) {
    $firstName = $user.FirstName
    $lastName = $user.LastName
    $username = $user.Username
    $email = $user.Email
    $department = $user.Department
    $title = $user.Title
    $ou = "OU=CompanyUsers,DC=damiade,DC=com"

    New-ADUser `
        -Name "$firstName $lastName" `
        -GivenName $firstName `
        -Surname $lastName `
        -UserPrincipalName "$username@damiade.com" `
        -SamAccountName $username `
        -EmailAddress $email `
        -Path $ou `
        -Department $department `
        -Title $title `
        -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) `
        -Enabled $true `
        -ChangePasswordAtLogon $true
}

```

I have now successfully created 200 user accounts on Active Directory using the data from the CSV file. I also automatically set all passwords as "P@ssw0rd" and then used the ```ChangePasswordAtLogon $true``` command to ensure every user changes their password upon logging in.


![Screenshot 2024-08-18 184520](https://github.com/user-attachments/assets/427b245a-1b2b-475e-a1a7-fb0972a57899)

I used ```Import-Module ActiveDirectory
Get-ADUser -Filter * -Properties Name, SamAccountName, UserPrincipalName | Select-Object Name, SamAccountName, UserPrincipalName``` to verify that the users were created on PowerShell too.


![Screenshot 2024-08-18 185907](https://github.com/user-attachments/assets/cfc09923-3693-4769-8762-df9bad59cd10)


### Step 3: Managing Group Memberships

After manually creating 5 OU for 5 departments (IT, HR, Marking, Sales, Finance), I  will now automate the process of adding new users to the appropriate groups based on their departments. For example, users in "IT" OU should be added to the "ITGroup".

I used Powershell to create the groups:
``` Powershell
Import-Module ActiveDirectory

$groups = @("FinanceGroup", "ITGroup", "HRGroup", "MarketingGroup", "SalesGroup")

$ouPath = "OU=CompanyUsers,DC=DamiAde,DC=com"

foreach ($group in $groups) {
    if (-not (Get-ADGroup -Filter { Name -eq $group } -SearchBase $ouPath -ErrorAction SilentlyContinue)) {
        # Create the group
        New-ADGroup -Name $group -Path $ouPath -GroupScope Global -PassThru | Out-Null
        Write-Host "Group '$group' created in OU '$ouPath'."
    } else {
        Write-Host "Group '$group' already exists in OU '$ouPath'."
    }
}
```


![Screenshot 2024-08-18 191012](https://github.com/user-attachments/assets/6b997002-00c5-4448-b88b-9db3d219fb86)


### Step 4. Verifying User Creation

I will now verify that all 200 users have been successfully added to the correct group using the below script:

```PowerShell
Import-Module ActiveDirectory

$groups = @{
    "Finance" = "FinanceGroup"
    "IT" = "ITGroup"
    "HR" = "HRGroup"
    "Marketing" = "MarketingGroup"
    "Sales" = "SalesGroup"
}

$csvFile = "C:\Users\Administrator\Desktop\dummydata.csv"
$userData = Import-Csv -Path $csvFile

foreach ($user in $userData) {
    $username = $user.Username
    $department = $user.Department
    $group = $groups[$department]

    if ($group) {
        $isMember = Get-ADGroupMember -Identity $group -Recursive | Where-Object { $_.SamAccountName -eq $username }

        if ($isMember) {
            Write-Host "User '$username' is a member of the '$group' group."
        } else {
            Write-Host "User '$username' is NOT a member of the '$group' group."
        }
    } else {
        Write-Host "No group found for department '$department' for user '$username'."
    }
}

```

The screenshot below confirms its success:

![Screenshot 2024-08-18 191615](https://github.com/user-attachments/assets/2bb1285e-2ede-4bb3-a639-e00b45b627c9)


### Step 5. Implementing Group Policies

I will now implement a strong password GPO to ensure all users and groups in the 'CompanyUsers' OU are abiding by the safety standards of password setting.

```Powershell
New-GPO -Name "Strong Password Policy" -Comment "Enforces strong password requirements."
Set-GPRegistryValue -Name "Strong Password Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\Safer\CodeIdentifiers" -ValueName "MinimumPasswordLength" -Type DWord -Value 12
Set-GPRegistryValue -Name "Strong Password Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\Safer\CodeIdentifiers" -ValueName "MaximumPasswordAge" -Type DWord -Value 60
New-GPLink -Name "Strong Password Policy" -Target "OU=CompanyUsers,DC=DamiAde,DC=com"
```
I manually confirmed that the right OU was linked to the GPO:

![Screenshot 2024-08-18 195021](https://github.com/user-attachments/assets/122a6905-528d-4cdb-869d-78fc1b2b6887)



### Results

1. Successfully created 200 AD accounts using Powershell
2. Created 5 groups for 5 departments using Powershell
3. Created a GPO using PowerShell and successfully linked it to the CompanyUsers OU
   

### Reflection

While PowerShell is useful, there are some areas where manually checking could have saved me more time than finding the right scripts. For example, I struggled to create a script to confirm if the GPO was linked to the CompanyUsers OU but could have just manually checked as I ultimately did.
   


### References
1. ChatGBT for scripts
2. https://www.youtube.com/watch?v=db0TV_Dters 
   

