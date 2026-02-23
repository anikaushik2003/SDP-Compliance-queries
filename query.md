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
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ServiceGroupName, ServiceTreeGuid, Version, Id, Environment
| where  Timestamp between (now()-timeSpan..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend Version_Revised=iff(Version == "","V1",Version)
| extend StageName = iff(Version =~ "V1", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| summarize arg_min(Timestamp, *) by Id, Environment
| distinct  Timestamp,ProjectId,DefinitionId,PipelineUrl,Version, Id, Environment, ServiceTreeGuid, StageName;
let belowarmpipelines= TimelineRecords
| where Task contains "StratusTriggerTask"
| distinct AdoAccount,ProjectId,Id
| join kind=leftouter PipelineRecords on AdoAccount,ProjectId,Id // join between timeline records and pipeline records
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize make_set(PipelineUrl);
let abovearmMOBRpipelines = allmobrpipelines
| where not(PipelineUrl has_any (belowarmpipelines))
| distinct PipelineUrl, BuildId=tolong(Id), Environment, ServiceTreeGuid, StageName; // in 60 days, about 5,64,437 records for all runs
// Stage-level ServiceId mapping (ServiceId can be overridden per stage via templateContext.serviceTreeId)
// PipelineRecords has one row per (run, stage/environment) with the effective ServiceId for that stage
let OnlyabovearmMOBRpipelinesList = abovearmMOBRpipelines
| distinct PipelineUrl
| summarize make_set(PipelineUrl);
// Get YAML ID's for Last Run only - used to extract CURRENT stage definitions (no leftover stages)
let lastrunyaml = BuildYamlMapSnapshot // this returns 6052 rows over 60days
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where PipelineUrl has_any (OnlyabovearmMOBRpipelinesList)
| summarize arg_max(BuildId,YamlId) by OrganizationName,ProjectId,DefinitionId,PipelineUrl;
// Stage definitions from latest YAML only (avoids leftover stages from old YAMLs)
let lastRunYamlIds =
lastrunyaml
| project YamlId
| distinct YamlId
| summarize make_set(YamlId);
let stagedata=BuildYamlSnapshot
| where YamlId has_any (lastRunYamlIds)
| extend taskname = parse_json(Data).task.name
| where Type in ("stage", "job", "task")
| summarize Jobs = make_set(Index) by StageIndex = Index, StageName = tostring(parse_json(Data).stage),YamlId
| join (
    BuildYamlSnapshot
    | where YamlId has_any (lastRunYamlIds)
    | where Type == "job"
    | project JobIndex = Index, StageIndex = ParentIndex,YamlId
) on StageIndex,YamlId
| join (
    BuildYamlSnapshot
    | where YamlId has_any (lastRunYamlIds)
    | where Type == "task"
    | project TaskName =  parse_json(Data).task.name,deploymentRing = parse_json(Data).inputs.deploymentRing, namespaceRingJSON = parse_json(Data).inputs.namespaceRingJson, ParentJobIndex = ParentIndex,YamlId
) on $left.JobIndex == $right.ParentJobIndex,YamlId
| project StageName, TaskName,YamlId,deploymentRing,namespaceRingJSON
| order by StageName asc
| summarize Tasks=make_set(TaskName),DeploymentRing=make_set(deploymentRing),namespaceRingJSON=make_set(namespaceRingJSON) by StageName,YamlId;

let runStageMap = BuildYamlMapSnapshot
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| join kind=inner recentRuns on PipelineUrl, BuildId
| join kind=inner (
    BuildYamlSnapshot
    | where Type == "stage"
    | project YamlId, StageName = tostring(parse_json(Data).stage)
    | distinct YamlId, StageName
) on YamlId
| project PipelineUrl, BuildId, StageName
|distinct *;
// Enrich allPipelineRuns with the stages each run actually contains
let allPipelineRunsWithStages = allPipelineRuns
| join kind=inner runStageMap on PipelineUrl, BuildId
| project PipelineUrl, BuildId, RunTimestamp, StageName
| distinct *;
let pipelinename=BuildYamlSnapshot
| where YamlId in (recentYamlIds)
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where Index == 0
| distinct PipelineUrl,YamlId,DisplayName;
// Stage ring analysis: stages from latest YAML x matching runs
// Stage properties (ring, namespace, onboarded) come from latest YAML definition.
// Each stage is paired only with runs whose YAML actually defined that stage.
let stagering2= stagedata
| where Tasks contains "lockbox-approval-request-prod_with_onebranch" and Tasks contains "ExpressV2Internal" // RS only tasks.
| join kind=leftouter lastrunyaml on YamlId
| project PipelineUrl,StageName,Tasks,DeploymentRing,namespaceRingJSON,YamlId
| join kind=inner allPipelineRunsWithStages on PipelineUrl, StageName // only pair stages with runs that actually contain them
| project PipelineUrl,BuildId,RunTimestamp,StageName,Tasks,DeploymentRing,namespaceRingJSON,YamlId
| extend uniquerow=strcat(StageName,"--",YamlId)
| project PipelineUrl,BuildId,RunTimestamp,StageName,Tasks,DeploymentRing,namespaceRingJSON=iff(namespaceRingJSON !contains "ring",todynamic("[{Non COSMIC}]"),namespaceRingJSON),YamlId,uniquerow
| mv-expand namespaceRingJSON
| extend cosmicring = toupper(coalesce(parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].Ring,parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].ring))
| extend cosmicnamespace = coalesce(parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].Namespace,parse_json(replace_string(tostring(namespaceRingJSON),"\\",""))[0].namespace,"")
| extend runurl = strcat(tostring(split(PipelineUrl,"?")[0]),"/results?buildId=",BuildId,"&view=results")
| summarize cosmicring=strcat_array(make_set(cosmicring),","),deploymentRing=strcat_array(make_set(DeploymentRing),","),allnamespaces=strcat_array(make_set(cosmicnamespace),",") by StageName,BuildId,RunTimestamp,PipelineUrl,runurl,YamlId
| extend finaldeploymentring=iff(deploymentRing=="",cosmicring,deploymentRing)
| extend length=countof(finaldeploymentring,",")+1
| extend finaldeploymentrings=iff(length > 1, "",iff(finaldeploymentring contains "$","",finaldeploymentring))
| extend Namespace = iff(allnamespaces == "","Non-Cosmic",allnamespaces)
| project StageName,BuildId,YamlId,PipelineUrl,runurl,Ring=toupper(tostring(finaldeploymentrings)),Namespace,uniquerow=strcat(StageName,"--",YamlId),RunTimestamp
| join kind=leftouter stageServiceIds on PipelineUrl, BuildId, StageName
| join kind=leftouter ServiceTree on ServiceId
| extend ringworkload=iff(Workload has_any (ringworkloadlist),Workload,"Others")
| join kind=leftouter ringworkload on $left.ringworkload==$right.Workload
| project StageName,PipelineUrl,YamlId,runurl,Ring,Namespace,uniquerow,ServiceId, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, Workload,DevOwner,AllowedRings,BuildId,RunTimestamp
| extend Onboarded=iff(Ring=="",0,iff(AllowedRings has Ring,1,0));
let stageLevelDetails =
    stagering2
    | summarize
        Ring=any(Ring),
        Namespace=any(Namespace),
        Onboarded=any(Onboarded),
        DivisionName=any(DivisionName),
        OrganizationName=any(OrganizationName),
        ServiceGroupName=any(ServiceGroupName),
        TeamGroupName=any(TeamGroupName),
        ServiceName=any(ServiceName),
        Workload=any(Workload),
        DevOwner=any(DevOwner),
        AllowedRings=any(AllowedRings),
        RunTimestamp=any(RunTimestamp),
        runurl=any(runurl),
        uniquerow=any(uniquerow)
      by YamlId, PipelineUrl, BuildId, StageName, ServiceId;
// Aggregate to pipeline-run level: one row per pipeline per run
let stagering = stagering2
| summarize Total_Lockbox_Ev2Classic_Stages=dcount(StageName), OnboardedStages=sum(Onboarded),NotOnboardedStages=make_set_if(StageName,Onboarded == 0) by PipelineUrl,YamlId,runurl,ServiceId,Workload,DevOwner,BuildId,RunTimestamp
| summarize arg_min(RunTimestamp, *) by YamlId, PipelineUrl, BuildId, ServiceId;
let trainsetpipelines= BuildYamlSnapshot
| where YamlId in (recentYamlIds)
| where Data contains "trainset"
| project OrganizationName,ProjectId,DefinitionId,YamlId,Data,EtlIngestDate
| extend trainset = parse_json(Data).variables
| mv-expand trainset
| where trainset contains "trainset"
| extend trainsetid=split(trainset.value,"\"")[3]
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| distinct PipelineUrl,trainsetid=tostring(trainsetid),YamlId;
let cloudpipelines= BuildYamlSnapshot
| where YamlId in (recentYamlIds)
| where Type == "stage"
| extend StageName = tostring(parse_json(Data).stage)
| extend cloud = tostring(parse_json(Data).inputs.cloud)
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| project PipelineUrl, YamlId, StageName, cloud;
let deploymenttypepipelines =BuildYamlSnapshot
| where YamlId in (recentYamlIds)
| extend deploymentType = iff(isempty(tostring(parse_json(Data).inputs.deploymentType)), "Normal", tostring(parse_json(Data).inputs.deploymentType))
| extend PipelineUrl = strcat( "https://dev.azure.com/", tolower(OrganizationName), "/", ProjectId, "/_build?definitionId=", DefinitionId, "&_a=summary" )
| extend deploymentTypePriority = case(deploymentType == "GlobalOutage", 1, deploymentType == "Emergency", 2, 3)
| summarize arg_min(deploymentTypePriority, deploymentType) by PipelineUrl, YamlId
| project PipelineUrl, YamlId, deploymentType;
let ingestdateandpipelinename= BuildYamlSnapshot
| where YamlId in (recentYamlIds)
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
// Pipeline + run metadata for all runs in last 60 days (filtered to MOBR pipelines only)
let pipelineInfo = PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ProjectName, ServiceId = ServiceTreeGuid, Version,Id=tolong(Id)
| where  Timestamp between (now()-timeSpan..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| join kind=inner abovearmMOBRpipelines on PipelineUrl
| summarize arg_max(Timestamp, *) by Id, PipelineUrl;
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
    | join kind=leftouter (stageServiceIds | distinct PipelineUrl, BuildId, ServiceId) on PipelineUrl, BuildId
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
stageLevelDetails
// Enrich with pipeline-run aggregates (many:1 from stage to run)
| join kind=leftouter stagering on YamlId, PipelineUrl, BuildId, ServiceId
| project-away YamlId1, PipelineUrl1, BuildId1, ServiceId1, runurl1, Workload1, DevOwner1, RunTimestamp1
// Enrich with pipeline-level metadata
| join kind=leftouter trainsetpipelines on YamlId, PipelineUrl
| project-away YamlId1, PipelineUrl1
| join kind=leftouter pipelinename on PipelineUrl, YamlId
| project-away YamlId1, PipelineUrl1
| join kind=leftouter ingestdateandpipelinename on YamlId, PipelineUrl
| project-away YamlId1, PipelineUrl1
| join kind=leftouter deploymenttypepipelines on YamlId, PipelineUrl
| project-away YamlId1, PipelineUrl1
// Enrich with stage-level cloud
| join kind=leftouter cloudpipelines on YamlId, PipelineUrl, StageName
| project-away YamlId1, PipelineUrl1, StageName1
// Enrich with run-level pipeline metadata (no ServiceId in key — run metadata is service-agnostic)
| join kind=leftouter pipelineInfo on PipelineUrl, $left.BuildId == $right.Id
| project-away PipelineUrl1, ServiceId1
// Enrich with service classification
| join kind=leftouter inscopeservices on ServiceId
| project-away ServiceId1
| extend ev2Type = tostring(ev2Type), isCosmicService = tostring(isCosmicService), Ev2ServiceType = tostring(Ev2ServiceType)
| extend PipelineOnboarded= iff(Total_Lockbox_Ev2Classic_Stages==OnboardedStages and trainsetid != "",1,0)
// | extend IsQEIWave8 = iff( PipelineOnboarded == 0 and ( Workload in ("M365 Core - IC3", "CAP - Microsoft Teams") or ( ev2Type == "Classic Only" and Workload !in ("M365 Core - IC3", "CAP - Microsoft Teams") and isCosmicService == "Cosmic Service" ) ), 1, 0 )
| extend OnboardingStatus = iff(PipelineOnboarded == 1, "Onboarded", "Not Onboarded")
| extend UniqueStageId = strcat(tolower(AdoAccount),"|", tolower(ProjectName),"|",tostring(DefinitionId),"|", BuildId, "|", StageName)
| extend PipelineKey = strcat(tolower(AdoAccount), "_", tolower(tostring(ProjectId)), "_", tostring(DefinitionId))
| extend exempted_pipelines = iff(PipelineKey in (exemptedPipelines), 1, 0)
| where Workload != "CAP - OneDrive/SharePoint"
// | where not(ServiceId has_any(raservices))
| project Timestamp=RunTimestamp, StageName, runurl, Workload, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, DevOwner,
        AdoAccount, ProjectId, ProjectName, DefinitionId, Id=BuildId,
        ServiceId,
        PipelineOnboarded, trainsetid, cloud, deploymentType, Ring, Namespace,
        PipelineUrl, PipelineName = DisplayName, UniqueStageId, Onboarded, isCosmicService, Ev2ServiceType, exempted_pipelines
| distinct *;
