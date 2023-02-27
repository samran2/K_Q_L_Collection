# K_Q_L_Collection
K_Q_L_Collection  Collection of the best KQL Queries

ASR
---------------------------------------------------------------------------------------------------------------------------------------------------
ASR Single Rule Audits
---------------------------------------------------------------------------------------------------------------------------------------------------
// uncomment one line at a time to look at hits within a specific category
DeviceEvents
//| where ActionType =~ "AsrAdobeReaderChildProcessAudited"
//| where ActionType =~ "AsrExecutableEmailContentAudited"
//| where ActionType =~ "AsrExecutableOfficeContentAudited"
//| where ActionType =~ "AsrLsassCredentialTheftAudited"
//| where ActionType =~ "AsrObfuscatedScriptAudited"
//| where ActionType =~ "AsrOfficeChildProcessAudited"
//| where ActionType =~ "AsrOfficeCommAppChildProcessAudited"
//| where ActionType =~ "AsrOfficeMacroWin32ApiCallsAudited"
//| where ActionType =~ "AsrOfficeProcessInjectionAudited"
//| where ActionType =~ "AsrPersistenceThroughWmiAudited"
//| where ActionType =~ "AsrPsexecWmiChildProcessAudited"
//| where ActionType =~ "AsrRansomwareAudited"
//| where ActionType =~ "AsrScriptExecutableDownloadAudited"
//| where ActionType =~ "AsrUntrustedExecutableAudited"
//| where ActionType =~ "AsrUntrustedUsbProcessAudited"
//| where ActionType =~ "AsrVulnerableSignedDriverAudited"
//| where ActionType =~ "ControlledFolderAccessViolationAudited"
| project DeviceName, FileName, FolderPath, InitiatingProcessFileName, InitiatingProcessFolderPath, ProcessCommandLine

---------------------------------------------------------------------------------------------------------------------------------------------------
ASR Summary
---------------------------------------------------------------------------------------------------------------------------------------------------
// This query was updated from https://github.com/Azure/Azure-Sentinel/tree/master/Hunting%20Queries/Microsoft%20365%20Defender/Protection%20events/ExploitGuardAsrDescriptions.yaml
let AsrDescriptionTable = datatable(RuleDescription:string, RuleGuid:string)
[
"Block abuse of exploited vulnerable signed drivers","56a863a9-875e-4185-98a7-b882c64b5ce5",
"Block executable content from email client and webmail","be9ba2d9-53ea-4cdc-84e5-9b1eeee46550",
"Block Office applications from creating child processes","d4f940ab-401b-4efc-aadc-ad5f3c50688a",
"Block Office applications from creating executable content","3b576869-a4ec-4529-8536-b80a7769e899",
"Block Office applications from injecting code into other processes","75668c1f-73b5-4cf0-bb93-3ecf5cb7cc84",
"Block JavaScript or VBScript from launching downloaded executable content","d3e037e1-3eb8-44c8-a917-57927947596d",
"Block execution of potentially obfuscated scripts","5beb7efe-fd9a-4556-801d-275e5ffc04cc",
"Block Win32 API calls from Office macro","92e97fa1-2edf-4476-bdd6-9dd0b4dddc7b",
"Block executable files from running unless they meet a prevalence, age, or trusted list criteria","01443614-cd74-433a-b99e-2ecdc07bfc25",
"Use advanced protection against ransomware","c1db55ab-c21a-4637-bb3f-a12568109d35",
"Block credential stealing from the Windows local security authority subsystem (lsass.exe)","9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2",
"Block process creations originating from PSExec and WMI commands","d1e49aac-8f56-4280-b9ba-993a6d77406c",
"Block untrusted and unsigned processes that run from USB","b2b3f03d-6a65-4f7b-a9c7-1c7ef74a9ba4",
"Block Office communication applications from creating child processes (available for beta testing)","26190899-1602-49e8-8b27-eb1d0a1ce869",
"Block Adobe Reader from creating child processes","7674ba52-37eb-4a4f-a9a1-f0f9a1619a2c",
"Block persistence through WMI event subscription","e6db77e5-3df2-4cf1-b95a-636979351e5b",
];
DeviceEvents
| where ActionType startswith "Asr"
| extend RuleGuid = tolower(tostring(parsejson(AdditionalFields).RuleId))
| extend IsAudit = parse_json(AdditionalFields).IsAudit
| project DeviceName, RuleGuid, DeviceId, IsAudit
| join kind = fullouter (
    AsrDescriptionTable
    | project RuleGuid = tolower(RuleGuid), RuleDescription
) on RuleGuid
| summarize MachinesWithAuditEvents = dcountif(DeviceId,IsAudit==1), MachinesWithBlockEvents = dcountif(DeviceId, IsAudit==0), AllEvents=iff(count()==1, 0, count()) by RuleDescription, RuleGuid=RuleGuid1
| sort by MachinesWithBlockEvents, MachinesWithAuditEvents, AllEvents
---------------------------------------------------------------------------------------------------------------------------------------------------
ASR Top Files
---------------------------------------------------------------------------------------------------------------------------------------------------
DeviceEvents
| where ActionType startswith 'Asr'
//     or ActionType startswith 'ControlledFolderAccessViolation'
    and ActionType endswith 'Audited'
| summarize Count = count() by ActionType, FileName, FolderPath
| sort by Count
---------------------------------------------------------------------------------------------------------------------------------------------------
ASR Top Rules
---------------------------------------------------------------------------------------------------------------------------------------------------
// top categories with audit hits
DeviceEvents
| where ActionType startswith 'Asr'
     or ActionType startswith 'ControlledFolderAccessViolation'
    and ActionType endswith 'Audited'
| summarize Count = count() by ActionType
| sort by Count

---------------------------------------------------------------------------------------------------------------------------------------------------
ASR Top Users
---------------------------------------------------------------------------------------------------------------------------------------------------
DeviceEvents
| where ActionType startswith 'Asr'
    and ActionType endswith 'Audited'
| summarize Count = count(), Categories = make_set(ActionType) by DeviceName, OnPremSid=AccountSid
| join IdentityInfo on OnPremSid
| project DeviceName, FullName=strcat(GivenName, " ", Surname), JobTitle, Count, Categories
| distinct DeviceName, FullName, JobTitle, Count, tostring(Categories)
| sort by Count
