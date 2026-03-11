set notruncation;
set norequesttimeout;
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
let ServiceTree = cluster("servicetreepublic.westus").database("Shared").table("DataStudio_ServiceTree_Hierarchy_Snapshot")
| where Level == "Service" and ServiceId != ""
| project DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, ServiceId
| extend Workload = iif(ServiceGroupName in (sglist),ServiceGroupName,iif(OrganizationName in (orglist),OrganizationName,iif(OrganizationName has "W+D","W+D",OrganizationName)))
| distinct ServiceId, Workload,DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName
| join kind = leftouter (cluster("servicetreepublic.westus").database("Shared").table("DataStudio_ServiceTree_ServiceCommonMetadata_Snapshot")
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
let allmobrpipelines = cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ServiceGroupName, ServiceTreeGuid, Version
| where  Timestamp between (now()-60d..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend Version_Revised=iff(Version == "","V1",Version)
| distinct  AdoAccount,ProjectId,DefinitionId,PipelineUrl,Version;
let belowarmpipelines= cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').TimelineRecords
| where Task contains "StratusTriggerTask"
| distinct AdoAccount,ProjectId,Id
| join kind=leftouter cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').PipelineRecords on AdoAccount,ProjectId,Id // join between timeline records and pipeline records
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize make_set(PipelineUrl);
let abovearmMOBRpipelines=allmobrpipelines
| where not(PipelineUrl has_any (belowarmpipelines))
| summarize make_set(PipelineUrl);
let abovearmmobrpipelineslastrun = cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ServiceGroupName, ServiceTreeGuid, Version,Id
| where  Timestamp between (now()-60d..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend Version_Revised=iff(Version == "","V1",Version)
| summarize arg_max(Id,ServiceTreeGuid) by PipelineUrl
| where not(PipelineUrl has_any (belowarmpipelines))
| distinct PipelineUrl,ServiceId=ServiceTreeGuid;
// Get YAML ID's for Last Run of All MOBR Pipelines
let lastrunyaml = cluster('1es').database('AzureDevOps').BuildYamlMap
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where PipelineUrl has_any (abovearmMOBRpipelines)
| summarize arg_max(BuildId,YamlId) by OrganizationName,ProjectId,DefinitionId,PipelineUrl;
let lastrunyamllist = cluster('1es').database('AzureDevOps').BuildYamlMap
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where PipelineUrl has_any (abovearmMOBRpipelines)
| summarize arg_max(BuildId,YamlId) by OrganizationName,ProjectId,DefinitionId, PipelineUrl
| summarize make_set(YamlId);
let stagedata=cluster('1es').database('AzureDevOps').BuildYaml
| where YamlId has_any (lastrunyamllist) // 6112 pipelines (before abovearm last run filter is applied)
| extend taskname = parse_json(Data).task.name
| where Type in ("stage", "job", "task")
| summarize Jobs = make_set(Index) by StageIndex = Index, StageName = tostring(parse_json(Data).stage),YamlId
| join (
    cluster('1es').database('AzureDevOps').BuildYaml
    | where YamlId has_any (lastrunyamllist)
    | where Type == "job"
    | project JobIndex = Index, StageIndex = ParentIndex,YamlId
) on StageIndex,YamlId
| join (
    cluster('1es').database('AzureDevOps').BuildYaml
    | where YamlId has_any (lastrunyamllist)
    | where Type == "task"
    | project TaskName =  parse_json(Data).task.name,deploymentRing = parse_json(Data).inputs.deploymentRing, namespaceRingJSON = parse_json(Data).inputs.namespaceRingJson, ParentJobIndex = ParentIndex,YamlId
) on $left.JobIndex == $right.ParentJobIndex,YamlId
| project StageName, TaskName,YamlId,deploymentRing,namespaceRingJSON
| order by StageName asc
| summarize Tasks=make_set(TaskName),DeploymentRing=make_set(deploymentRing),namespaceRingJSON=make_set(namespaceRingJSON) by StageName,YamlId;
let pipelinename=cluster('1es').database('AzureDevOps').BuildYaml
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where Index == 0
| distinct PipelineUrl,YamlId,DisplayName;
let stagering2= stagedata
| where Tasks contains "lockbox-approval-request-prod_with_onebranch" and Tasks contains "ExpressV2Internal" // RS only tasks.
| join kind=leftouter lastrunyaml on YamlId
| project PipelineUrl,BuildId,StageName,Tasks,DeploymentRing,namespaceRingJSON,YamlId
| extend uniquerow=strcat(StageName,"--",YamlId) // don't we lose job and task level data here?
| project PipelineUrl,BuildId,StageName,Tasks,DeploymentRing,namespaceRingJSON=iff(namespaceRingJSON !contains "ring",todynamic("[{Non COSMIC}]"),namespaceRingJSON),YamlId,uniquerow
| mv-expand namespaceRingJSON
| extend cosmicring = toupper(coalesce(parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].Ring,parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].ring))
| extend cosmicnamespace = coalesce(parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].Namespace,parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].namespace,"")
| extend runurl = strcat(tostring(split(PipelineUrl,"?")[0]),"/results?buildId=",BuildId,"&view=results")
| summarize cosmicring=make_set(cosmicring),deploymentRing=make_set(DeploymentRing),allnamespaces=make_set(cosmicnamespace) by StageName,BuildId,PipelineUrl,runurl,YamlId
| extend finaldeploymentring=iff(tostring(deploymentRing)=="[]",cosmicring,deploymentRing)
| extend length=array_length(finaldeploymentring)
| extend finaldeploymentrings=iff(length > 1, "",iff(finaldeploymentring contains "$","",finaldeploymentring[0]))
| extend Namespace = iff(array_length(allnamespaces) == 0 or tostring(allnamespaces[0]) == "","Non-Cosmic",strcat_array(allnamespaces, ","))
| project StageName,BuildId,YamlId,PipelineUrl,runurl,Ring=toupper(tostring(finaldeploymentrings)),Namespace,uniquerow=strcat(StageName,"--",YamlId)
| join kind=leftouter (abovearmmobrpipelineslastrun) on PipelineUrl
| join kind=leftouter ServiceTree on ServiceId
| extend ringworkload=iff(Workload has_any (ringworkloadlist),Workload,"Others")
| join kind=leftouter ringworkload on $left.ringworkload==$right.Workload
| project StageName,PipelineUrl,YamlId,runurl,Ring,Namespace,uniquerow,ServiceId, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, Workload,DevOwner,AllowedRings
| extend Onboarded=iff(Ring=="",0,iff(AllowedRings has Ring,1,0)); //fix allowed rings logic
let stageLevelDetails = stagering2;
let stagering = stagering2
| summarize Total_Lockbox_Ev2Classic_Stages=dcount(StageName), OnboardedStages=sum(Onboarded),NotOnboardedStages=make_set_if(StageName,Onboarded == 0) by PipelineUrl,YamlId,runurl,ServiceId,Workload,DevOwner; // 743 services, 3853 pipelines
let trainsetpipelines= cluster('1es').database('AzureDevOps').BuildYaml
| where YamlId has_any (lastrunyamllist)
| where Data contains "trainset"
| project OrganizationName,ProjectId,DefinitionId,YamlId,Data,EtlIngestDate
| extend trainset = parse_json(Data).variables
| mv-expand trainset
| where trainset contains "trainset"
| extend trainsetid=split(trainset.value,"\"")[3]
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| distinct PipelineUrl,trainsetid=tostring(trainsetid),YamlId;
let cloudpipelines= cluster('1es').database('AzureDevOps').BuildYaml
| where YamlId has_any (lastrunyamllist)
| extend cloud = parse_json(Data).inputs.cloud
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| distinct PipelineUrl,cloud=tostring(cloud), YamlId;
let deploymenttypepipelines =cluster('1es').database('AzureDevOps').BuildYaml
| where YamlId has_any (lastrunyamllist)
| extend deploymentType =iff( isempty(tostring(parse_json(Data).inputs.deploymentType)), "Normal", tostring(parse_json(Data).inputs.deploymentType))
| extend PipelineUrl = strcat( "https://dev.azure.com/", tolower(OrganizationName), "/", ProjectId, "/_build?definitionId=", DefinitionId, "&_a=summary" )
| distinct PipelineUrl, deploymentType, YamlId;
let ingestdateandpipelinename= cluster('1es').database('AzureDevOps').BuildYaml
| where YamlId has_any (lastrunyamllist)
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize IngestDate=max(EtlIngestDate) by YamlId,PipelineUrl;
//Services using RA
let raservices=stagedata
| where Tasks contains "Ev2RARollout"
| distinct StageName, YamlId
| join kind=leftouter lastrunyaml on YamlId
| distinct PipelineUrl,YamlId
| join kind=leftouter (cluster('1es').database('1ESPTInsights').AllADOYamlReleasePipelines_X_marcklim
| distinct PipelineUrl,ServiceId) on PipelineUrl
| distinct PipelineUrl,YamlId,ServiceId
| where ServiceId != ""
| summarize make_set(ServiceId);
let pipelineInfo = cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ProjectName, ServiceId = ServiceTreeGuid, Version,Id
| where  Timestamp between (now()-60d..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize arg_max(Id, *) by PipelineUrl, ServiceId;
 // 7532 
// ---- Service classification (Cosmic + EV2 type) ----
let servicemetadata =
    stagedata
    | where Tasks contains "lockbox-approval-request-prod_with_onebranch"
      and (Tasks contains "ExpressV2Internal" or Tasks contains "Ev2RARollout")
    // normalize missing ring info EXACTLY like source of truth
    | extend normalizedRing =
        iff(tostring(namespaceRingJSON) !contains "ring",
            "[{Non COSMIC}]",
            tostring(namespaceRingJSON))
    // reintroduce stage truth
    | extend isCOSMICStage =
        iff(normalizedRing == "[{Non COSMIC}]", 0, 1)
    | summarize
        TotalStages = dcount(StageName),
        CosmicStages = sum(isCOSMICStage),
        ClassicStages = dcountif(StageName, Tasks contains "ExpressV2Internal"),
        RAStages = dcountif(StageName, Tasks contains "Ev2RARollout")
      by YamlId;
let inscopeservices =
    servicemetadata
    | join kind=leftouter lastrunyaml on YamlId
    | join kind=leftouter abovearmmobrpipelineslastrun on PipelineUrl
    | summarize
        TotalPipelines = dcount(PipelineUrl),
        TotalStages = sum(TotalStages),
        TotalClassicStages = sum(ClassicStages),
        TotalRAStages = sum(RAStages),
        ClassicPipelines = dcountif(PipelineUrl, ClassicStages > 0),
        RAPipelines = dcountif(PipelineUrl, RAStages > 0),
        CosmicPipelines = dcountif(PipelineUrl, CosmicStages > 0)
      by ServiceId
    | extend
        isCosmicService =
            iff(CosmicPipelines > 0,
                "Cosmic Service",
                "Entirely Non-Cosmic"),
        ev2Type =
            iff(TotalPipelines == RAPipelines,
                "RA Only",
                iff(TotalPipelines == ClassicPipelines,
                    "Classic Only",
                    "Hybrid")),
        // Service-level EV2 classification based on stages
        Ev2ServiceType =
            iff(TotalRAStages == 0 and TotalClassicStages > 0,
                "RS Only",  // No RA stages at all, only Classic = RS Only
                iff(TotalRAStages > 0 and TotalClassicStages == 0,
                    "RA Only",  // Only RA stages, no Classic = RA Only
                    iff(TotalRAStages > 0 and TotalClassicStages > 0,
                        "Hybrid Services",  // Both RA and Classic stages = Hybrid Services
                        "Unknown"  // Neither (shouldn't happen for inscope services)
                    )
                )
            );
let resolved= cluster('s360prodro').database('service360db').GetResolvedActionItems
| extend KPIID = parse_json(CustomDimensions).Ingestion_KpiId
| extend PipelineURL = parse_json(CustomDimensions).PipelineUrl
| where KPIID=="7a9c1355-805d-4fb2-89a5-8c0a82483465"
| extend active=0;
let active=cluster('s360prodro').database('service360db').GetActiveActionItems
| extend KPIID = parse_json(CustomDimensions).Ingestion_KpiId
| extend PipelineURL = parse_json(CustomDimensions).PipelineUrl
| where KPIID=="7a9c1355-805d-4fb2-89a5-8c0a82483465"
| extend active=1;
let all=union resolved,active
| extend uniquekey=parse_json(CustomDimensions).S360_UniqueKey;
let allQEIpipelines=all
| distinct PipelineURL=tostring(PipelineURL)
| extend QEIWave8 = "Wave 8", PipelineUrl=tostring(PipelineURL);
stagering
| join kind=leftouter trainsetpipelines on YamlId,PipelineUrl
| join kind=leftouter cloudpipelines on YamlId,PipelineUrl
| join kind=leftouter deploymenttypepipelines on YamlId,PipelineUrl
| join kind=leftouter pipelinename on PipelineUrl, YamlId
| join kind=leftouter ingestdateandpipelinename on YamlId,PipelineUrl
| join kind=leftouter stageLevelDetails  on YamlId, PipelineUrl
| join kind=leftouter pipelineInfo on ServiceId, PipelineUrl
| join kind=leftouter inscopeservices on ServiceId
| join kind=leftouter allQEIpipelines on PipelineUrl
| extend ev2Type = tostring(ev2Type), isCosmicService = tostring(isCosmicService), Ev2ServiceType = tostring(Ev2ServiceType)
| extend PipelineOnboarded= iff(Total_Lockbox_Ev2Classic_Stages==OnboardedStages and trainsetid != "",1,0)
// | extend IsQEIWave8 = iff( PipelineOnboarded == 0 and ( Workload in ("M365 Core - IC3", "CAP - Microsoft Teams") or ( ev2Type == "Classic Only" and Workload !in ("M365 Core - IC3", "CAP - Microsoft Teams") and isCosmicService == "Cosmic Service" ) ), 1, 0 )
| extend IsQEIWave8 = iff( QEIWave8 == "Wave 8", 1, 0 )
| extend QEIWave = iff(IsQEIWave8 == 1, "Wave 8", "Not Wave 8"), OnboardingStatus = iff(PipelineOnboarded == 1, "Onboarded", "Not Onboarded")
| extend UniqueStageId = strcat(tolower(AdoAccount),"|", tolower(ProjectName),"|",tostring(DefinitionId),"|", Id, "|", StageName)
| extend PipelineKey = strcat(tolower(AdoAccount), "_", tolower(tostring(ProjectId)), "_", tostring(DefinitionId))
| extend exempted_pipelines = iff(PipelineKey in (exemptedPipelines), 1, 0)
| where Workload != "CAP - OneDrive/SharePoint"
// | where not(ServiceId has_any(raservices))
| project Timestamp, StageName, runurl, Workload, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, DevOwner,
        AdoAccount, ProjectId, ProjectName, DefinitionId, Id,
        ServiceId,
        PipelineOnboarded, trainsetid, cloud, deploymentType, Ring, Namespace,
        PipelineUrl, PipelineName = DisplayName, UniqueStageId, Onboarded, QEIWave, isCosmicService, Ev2ServiceType, exempted_pipelines
| summarize arg_max(Id, *) by PipelineUrl, ServiceId, Id, StageName;