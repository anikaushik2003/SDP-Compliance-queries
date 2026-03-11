set notruncation;
set norequesttimeout;
set servertimeout = 1h;
set maxmemoryconsumptionperiterator=32212254720; // 30GB
let timeSpan = 60d;
// Exempted Pipelines List
let exemptedPipelines = dynamic([
    "o365exchange_959adb23-f323-4d52-8203-ff34e5cbeefa_51671",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36193",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35003",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36303",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36344",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35235",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35514",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35486",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35638",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35639",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35940",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35941",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_34999"
]);
// Workload Info
let sglist = dynamic(["IC3","Yammer","TAOS","O365 FAST","Substrate Platform","Mesh","Microsoft Search Assistants & Intelligence (MSAI)","O365 Enterprise Cloud","Microsoft Teams"]);
let orglist = dynamic(["WebXT","OPG","OneDrive/SharePoint","Data Security and Privacy"]);
let ServiceTree = ServiceTreeHierarchySnapshot
| where Level == "Service" and ServiceId != ""
| project DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, ServiceId
| extend Workload = iif(ServiceGroupName in (sglist),ServiceGroupName,iif(OrganizationName in (orglist),OrganizationName,iif(OrganizationName has "W+D","W+D",OrganizationName)))
| distinct ServiceId, Workload,DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName
| join kind = leftouter (ServiceTreeSnapshot
| distinct ServiceId,DevOwner) on ServiceId
| distinct ServiceId,Workload,DevOwner,DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName
| distinct ServiceId,Workload=iff(Workload == "Skype","M365 Core -IC3",iff(Workload == "Microsoft Teams","CAP - Microsoft Teams",Workload)),DevOwner, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName;
// Ring Workload Info
let ringworkload = datatable(Workload:string,AllowedRings:dynamic)[
"M365 Core - IC3", dynamic(["NPE","TDF", "SDF", "MSIT", "GENERAL", "Legacy_TDF", "Early", "Deferred"]),
"Others", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"])
];
let ringworkloadlist=ringworkload
| summarize make_set(Workload);
// Get All Above-ARM MOBR Pipelines
let allmobrpipelines = PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount, ProjectId, DefinitionId, ServiceGroupName, ServiceTreeGuid, Version, Id, Environment, ProjectName
| where  Timestamp between (now()-timeSpan..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend Version_Revised=iff(Version == "","V1",Version)
| extend StageName = iff(Version =~ "V1", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| summarize arg_min(Timestamp, *) by Id, Environment
| distinct  AdoAccount, ProjectName, ProjectId,DefinitionId,PipelineUrl,Version, Id, Environment, ServiceTreeGuid, StageName, ServiceGroupName;
let belowarmpipelines= TimelineRecords
| where Task contains "StratusTriggerTask"
| distinct AdoAccount,ProjectId,Id
| join kind=leftouter PipelineRecords on AdoAccount,ProjectId,Id // join between timeline records and pipeline records
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize make_set(PipelineUrl);
let abovearmMOBRpipelines = allmobrpipelines
| where not(PipelineUrl has_any (belowarmpipelines))
| extend UniqueStageId = strcat(tolower(AdoAccount),"|", tolower(ProjectName),"|",tostring(DefinitionId),"|", Id, "|", StageName)
| distinct PipelineUrl, BuildId=tolong(Id), ServiceTreeGuid, StageName, ServiceGroupName, UniqueStageId, AdoAccount, ProjectId, ProjectName;
abovearmMOBRpipelines