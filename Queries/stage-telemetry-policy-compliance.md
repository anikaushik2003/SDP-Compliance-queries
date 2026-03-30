set notruncation;
set norequesttimeout;
set servertimeout = 1h;
set maxmemoryconsumptionperiterator=32212254720;
let timeSpan = 60d;
// Workload Info
let sglist = dynamic(["IC3","Yammer","TAOS","O365 FAST","Substrate Platform","Mesh","Microsoft Search Assistants & Intelligence (MSAI)","O365 Enterprise Cloud","Microsoft Teams"]);
let orglist = dynamic(["WebXT","OPG","OneDrive/SharePoint","Data Security and Privacy"]);
let ServiceTree = ServiceTreeHierarchySnapshot
| where Level == "Service" and ServiceId != ""
| project DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, ServiceId
| extend Workload = iif(ServiceGroupName in (sglist),ServiceGroupName,iif(OrganizationName in (orglist),OrganizationName,iif(OrganizationName has "W+D","W+D",OrganizationName)))
| distinct ServiceId, Workload, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName
| join kind = leftouter (ServiceTreeSnapshot
| distinct ServiceId, DevOwner) on ServiceId
| distinct ServiceId, Workload=iff(Workload == "Skype","M365 Core -IC3",iff(Workload == "Microsoft Teams","CAP - Microsoft Teams",Workload)), DevOwner, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName;
// Ring Workload Info
let ringworkload = datatable(Workload:string, AllowedRings:dynamic)[
"M365 Core - IC3", dynamic(["NPE","TDF", "SDF", "MSIT", "GENERAL"]),
"OPG", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
"CAP - Microsoft Teams", dynamic(["TEST", "DOGFOOD", "MSFT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
"Substrate Platform", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
"Others", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"])
];
let ringworkloadlist = ringworkload
| summarize make_set(Workload);
let GetVersionInfo = PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ProjectName, ServiceId = ServiceTreeGuid, Environment, Version,Id
| where  Timestamp between (now()-timeSpan..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary") // 710k rows
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id);
// stages has about 600k
let stages =
    TimelineRecords
    | extend StartTime = todatetime(StartTime)
    | where StartTime >= ago(timeSpan)
    | where not(Status has "unknown")
    | summarize arg_max(StageAttempt, *) by AdoAccount, ProjectId, Id, HostType, Environment, Version, RecordId, tostring(parse_json(Task).Id)
    | summarize
        totalRecords        = count(),
        successfulRecords   = countif(Status in~ ("Succeeded","Skipped","succeededWithIssues")),
        healthCheckTotal    = countif(tostring(parse_json(Task).Name) == "health-check-v1"),
        healthCheckSuccess  = countif(
                                tostring(parse_json(Task).Name) == "health-check-v1"
                                and Status in~ ("Succeeded","Skipped","succeededWithIssues")
                              ),
        Tasks               = make_set(tostring(parse_json(Task).Name)),
        Task_Ids            = make_set(tostring(parse_json(Task).Id)),
        TaskNames           = make_set(tostring(parse_json(Task).Name)),
        HasLockbox          = max(iff(tostring(parse_json(Task).Name) == "lockbox-approval-request-prod_with_onebranch", 1, 0)),
        HasClassic          = max(iff(tostring(parse_json(Task).Name) == "ExpressV2Internal", 1, 0)),
        HasRA               = max(iff(tostring(parse_json(Task).Name) == "Ev2RARollout", 1, 0))
      by AdoAccount, ProjectId, Id, HostType, Environment, Version
    | extend isSuccessful = (totalRecords == successfulRecords)
    | extend StageName = iff(
        Version =~ "V1",
        Environment,
        strcat_array(array_slice(split(Environment, "_"), 1, -1), "_")
      )
    | extend HealthEnabled = iff(healthCheckTotal > 0, 1, 0)
    | extend HealthCheckPassed = iff(
        healthCheckTotal == 0,
        bool(false),
        healthCheckSuccess == healthCheckTotal
      )
    | extend joinKey = strcat (AdoAccount, ProjectId, Version, Id, Environment);
let StageTelemetry_PolicyCompliance =
    Logs
    | where todatetime(Timestamp) > ago(timeSpan)
    | where TaskId == "d98bb041-d191-41c2-b770-3dc3e7b10d7e"
    | where (Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]")
         or Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId"
         or Message has "NamespaceRingJson:"
    | extend AdoAccount = tolower(AdoAccount), ProjectName = tolower(ProjectName)
    | extend MsgType = case(
        Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]", "Policy",
        Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId", "MOBR",
        "Cosmic")
    // --- MOBR enrichment: ring, deploymentType, cloud, trainset ---
    | extend _ring = iff(MsgType == "MOBR", tostring(extract(@"[?&]ring=([^&\s]+)", 1, Message)), "")
    | extend DeploymentRing = iif(isempty(_ring) or tolower(_ring) == "null", "", _ring)
    | extend _depType = iff(MsgType == "MOBR", tostring(extract(@"[?&]deploymentType=([^&\s]+)", 1, Message)), "")
    | extend DeploymentType = iif(isempty(_depType) or tolower(_depType) == "null", "", _depType)
    | extend _cloud = iff(MsgType == "MOBR", tostring(extract(@"[?&]cloud=([^&\s]+)", 1, Message)), "")
    | extend Cloud = iif(isempty(_cloud) or tolower(_cloud) == "null", "", _cloud)
    | extend _trainset = iff(MsgType == "MOBR", tostring(extract(@"[?&]trainset=([^&\s]+)", 1, Message)), "")
    | extend Trainset = iif(isempty(_trainset) or tolower(_trainset) == "null", "", _trainset)
    // --- Cosmic enrichment: ring, namespace ---
    | extend CosmicRing = iff(MsgType == "Cosmic",
        toupper(extract(@"(?i)\\\""ring\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message)),
        "")
    | extend CosmicNamespace = iff(MsgType == "Cosmic",
        extract(@"(?i)\\\""namespace\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message),
        "")
    // --- Policy evidence parsing ---
    | extend PolicyEvidence = iff(MsgType == "Policy", replace_string(Message, "[PolicyEvidenceRecord]", ""), "")
    | extend jObj = parse_json(PolicyEvidence)
    // Keep non-Policy rows alive through mv-expand with a single-element placeholder array
    | mv-expand PolicyResults = iff(MsgType == "Policy", jObj.PolicyResults, pack_array(dynamic(null)))
    | mv-expand RuleResults   = iff(isnotnull(PolicyResults), PolicyResults.RuleResults, pack_array(dynamic(null)))
    | extend
        PolicyName    = tostring(PolicyResults.PolicyName),
        PolicyRan     = tobool(RuleResults.PolicyRan),
        PolicyPassed  = tobool(RuleResults.PolicyPassed),
        PolicyMode    = tostring(RuleResults.PolicyMode)
    | extend PolicyStatus = case(
            PolicyMode == "NotEnabled", 1,
            PolicyRan and PolicyPassed, 1,
            PolicyRan and not(PolicyPassed), 2,
            not(PolicyRan), 3,
            4
        )
    | extend Id = tostring(Id)
    | where PolicyName in ("ring-bake-time", "ring-progression", "stage-bake-time", "min-stage-count") or MsgType != "Policy"
    | summarize
      RingBakeTime_Status =
          min(iff(PolicyName == "ring-bake-time", PolicyStatus, 4)),
      RingProgression_Status =
          min(iff(PolicyName == "ring-progression", PolicyStatus, 4)),
      StageBakeTime_Status =
          min(iff(PolicyName == "stage-bake-time", PolicyStatus, 4)),
      MinStageCount_Status =
          min(iff(PolicyName == "min-stage-count", PolicyStatus, 4)),
      // Enrichment from MOBR logs
      DeploymentRing  = take_anyif(DeploymentRing, DeploymentRing != ""),
      DeploymentType  = take_anyif(DeploymentType, DeploymentType != ""),
      Cloud           = take_anyif(Cloud, Cloud != ""),
      Trainset        = take_anyif(Trainset, Trainset != ""),
      // Enrichment from Cosmic logs
      CosmicRing      = take_anyif(CosmicRing, CosmicRing != ""),
      CosmicNamespace = take_anyif(CosmicNamespace, CosmicNamespace != "")
      by AdoAccount, ProjectName, ProjectId, DefinitionId, Id, Environment
    // --- Ring construction & cosmic stage logic (shifted from YAML) ---
    | extend DeploymentRing = coalesce(DeploymentRing, ""), CosmicRing = coalesce(CosmicRing, "")
    | extend CosmicNamespace = coalesce(CosmicNamespace, ""), DeploymentType = coalesce(DeploymentType, "")
    | extend Cloud = coalesce(Cloud, ""), Trainset = coalesce(Trainset, "")
    | extend Ring = iff(isempty(DeploymentRing), CosmicRing, toupper(DeploymentRing))
    | extend Ring = trim(" ", Ring)
    | extend Ring = iff(Ring contains "$", "", Ring)
    | extend Ring = iff(Ring contains ",", "", Ring)
    | extend isCosmicStage = iff(isnotempty(CosmicRing), 1, 0)
    | extend Namespace = iff(isempty(CosmicNamespace), "Non-Cosmic", CosmicNamespace);
StageTelemetry_PolicyCompliance // 558k
    | extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
    | extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id)
    | lookup kind=leftouter GetVersionInfo on PipelineJoinKey
    | extend joinKey = strcat (AdoAccount, ProjectId, Version, Id, Environment)
    | lookup kind=leftouter stages on joinKey
    | extend StageName = iff(
        Version =~ "V1",
        Environment,
        strcat_array(array_slice(split(Environment, "_"), 1, -1), "_")
      )
    | extend UniqueStageId = strcat(tolower(AdoAccount),"|", tolower(ProjectName),"|",tostring(DefinitionId),"|", Id, "|", StageName)
    // --- ServiceTree enrichment for Onboarded calculation ---
    | join kind=leftouter ServiceTree on ServiceId
    | extend ringworkload = iff(Workload has_any (ringworkloadlist), Workload, "Others")
    | join kind=leftouter ringworkload on $left.ringworkload == $right.Workload
    | extend Onboarded = iff(Ring == "", 0, iff(AllowedRings has Ring, 1, 0))
    | project
        AdoAccount, ProjectName, DefinitionId, Id, Environment, PipelineUrl,
        RingBakeTime_Status, RingProgression_Status,
        StageBakeTime_Status, MinStageCount_Status,
        HealthEnabled, HealthCheckPassed, StageName, UniqueStageId,
        Ring, Namespace, Cloud, DeploymentType, Trainset, isCosmicStage,
        HasLockbox, HasClassic, HasRA, Onboarded,
        ServiceId, Workload, DevOwner, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName