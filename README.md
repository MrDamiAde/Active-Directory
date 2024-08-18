# Automating User Management and Group Policies in Windows Active Directory

### Project Overview
---
This project simulates a scenario in a medium-sized company undergoing a departmental reorganisation. The task involves setting up a new Active Directory environment on Windows Server, creating and organising 500 user accounts based on department and role, and applying appropriate Group Policies for security and access control. PowerShell scripting is used for automating the bulk user creation process and validating the configuration.

### Tools and Technologies

- Windows Server 2019/2022
- Active Directory Domain Services (AD DS)
- PowerShell
- Group Policy Management Console (GPMC)

### Preparation

#### 1. Organisational Unit (OU) creation

I first created 5 organisational units of different departments under the company domain: IT, HR, Marketing, Sales, and Finance.


![Screenshot 2024-08-18 114511](https://github.com/user-attachments/assets/d13d101d-a5b3-4d08-9808-f0ee8a1d0d12)

#### 2. 

I created a CSV file with 500 rows of dummy data using the below PowerShell script:

```Powershell
$recordCount = 500
$outputFile = "C:\path\to\your\dummydata.csv"
$data = @()
$departments = @("HR", "IT", "Marketing", "Finance", "Sales")

for ($i = 1; $i -le $recordCount; $i++) {
    $firstName = "FirstName$i"
    $lastName = "LastName$i"
    $email = "user$i@example.com"
    $username = "user$i"
    $phoneNumber = "+1234567890"
    $department = $departments[(Get-Random -Minimum 0 -Maximum 4)]
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
$csvFile = "C:\path\to\your\dummydata.csv"

$data = Import-Csv -Path $csvFile

$data | Format-Table -AutoSize


```

### 2. Bulk Uploading Using Bulk Operations

I used the bulk operation tool to import the 1000 users into Entra ID.

https://github.com/user-attachments/assets/abf2d241-efa8-473e-b3d0-310e7f469aec



### 3. User Analysis using Powershell

I will now verify that all 1000 users have been successfully uploaded on Entra ID using the below PowerShell script:

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
