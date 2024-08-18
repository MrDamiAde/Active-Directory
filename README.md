# Automating User Management and Group Policies in Windows Active Directory

### Project Overview
---
This project simulates a scenario in a medium-sized company, DamiAde, undergoing a departmental reorganisation. The task involves setting up a new Active Directory environment on Windows Server, creating and organising 200 user accounts based on department and role, and applying appropriate Group Policies for security and access control. PowerShell scripting is used for automating the bulk user creation process and validating the configuration.

### Tools and Technologies

- VirtualBox
- Windows Server 2019
- Active Directory Domain Services (AD DS)
- PowerShell
- Group Policy Management Console (GPMC)

### Step 1. Preparation

#### 1. Organisational Unit (OU) creation

I first created 5 organisational units of different departments under the company domain: IT, HR, Marketing, Sales, and Finance.


![Screenshot 2024-08-18 114511](https://github.com/user-attachments/assets/d13d101d-a5b3-4d08-9808-f0ee8a1d0d12)

#### 2. 200 Users Creation

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



### Step 2. Automating User Creation with PowerShell

I used the below powershell script that reads the CSV file created above and use that data to create accounts on Active Directory:

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

I have now succesfuly created 500 user accounts on active directry using the data from the CSV file. I also automatically set all apsswords as "P@ssw0rd" and set used the ```ChangePasswordAtLogon $true``` code for each user to change their password upon loging in.


![Screenshot 2024-08-18 184520](https://github.com/user-attachments/assets/427b245a-1b2b-475e-a1a7-fb0972a57899)

I used ```Import-Module ActiveDirectory
Get-ADUser -Filter * -Properties Name, SamAccountName, UserPrincipalName | Select-Object Name, SamAccountName, UserPrincipalName``` to verify that the users were created on pwoershell too.


![Screenshot 2024-08-18 185907](https://github.com/user-attachments/assets/cfc09923-3693-4769-8762-df9bad59cd10)


### Step 3: Managing Group Memberships

I will now automate the process of adding new users to the appropriate groups based on their departments. For example, users in "IT" OU should be added to the "ITGroup" polocy.

I used Powershell to create the groups:
``` Powershell
# Import the Active Directory module
Import-Module ActiveDirectory

# Define group names
$groups = @("FinanceGroup", "ITGroup", "HRGroup", "MarketingGroup", "SalesGroup")

# Define the OU path where groups will be created
$ouPath = "OU=CompanyUsers,DC=DamiAde,DC=com"

foreach ($group in $groups) {
    # Check if the group already exists
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


Verifying User Creation

I will now verify that all 200 users have been successfully uploaded on Entra ID using the below PowerShell script:

```PowerShell
Get-MgUser -Filter "UserType eq 'Member'"
```

The screenshot below confirms the success:

![Screenshot 2024-08-17 133209](https://github.com/user-attachments/assets/b78bec12-3715-4f87-8702-a043d2232074)


### Results

1. Successfully created 1000 Users in Entra ID
2. Verified the creation's success using Powershell

   


### References

1. https://1000randomnames.com/
2. https://learn.microsoft.com/en-us/entra/identity/users/users-bulk-add
