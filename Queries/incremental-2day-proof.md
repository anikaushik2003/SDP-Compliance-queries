// Incremental 2-Day Proof-of-Concept
// Each day: run all 3 sub-queries as-is with timeSpan=1d, join them
// Then UNION the two days
set notruncation;
set norequesttimeout;
set servertimeout = 1h;
set maxmemoryconsumptionperiterator=32212254720;
// =====================================================================
// DAY 1: ago(2d) to ago(1d)
// =====================================================================
// --- Day 1: all-stage-runs (exact copy with timeSpan=1d scoped to day1) ---
let day1_belowarmpipelines = TimelineRecords
| where Task contains "StratusTriggerTask"
| distinct AdoAccount, ProjectId, Id
| join kind=leftouter PipelineRecords on AdoAccount, ProjectId, Id
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize make_set(PipelineUrl);
let day1_allmobrpipelines = PipelineRecords
| project Timestamp=todatetime(Timestamp), AdoAccount, ProjectId, DefinitionId, ServiceGroupName, ServiceTreeGuid, Version, Id, Environment, ProjectName
| where Timestamp between (ago(2d)..ago(1d))
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend StageName = iff(Version =~ "V1" or Version == "", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| summarize arg_min(Timestamp, *) by Id, Environment
| distinct AdoAccount, ProjectName, ProjectId, DefinitionId, PipelineUrl, Version, Id, Environment, ServiceTreeGuid, StageName, ServiceGroupName;
let day1_runs = day1_allmobrpipelines
| where not(PipelineUrl has_any (day1_belowarmpipelines))
| extend UniqueStageId = strcat(tolower(AdoAccount),"|",tolower(ProjectName),"|",tostring(DefinitionId),"|",Id,"|",StageName)
| distinct PipelineUrl, BuildId=tolong(Id), ServiceTreeGuid, StageName, ServiceGroupName, UniqueStageId, AdoAccount, ProjectId, ProjectName;
// --- Day 1: yaml-to-run-list (exact copy scoped to day1 runs) ---
let day1_RecentRuns = day1_runs | project PipelineUrl, BuildId;
let day1_yaml = BuildYamlMapSnapshot
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| join kind=leftsemi day1_RecentRuns on PipelineUrl, $left.BuildId == $right.BuildId
| project PipelineUrl, BuildId, YamlId;
// --- Day 1: stage-telemetry-policy-compliance (exact copy scoped to day1) ---
let day1_GetVersionInfo = PipelineRecords
| project Timestamp=todatetime(Timestamp), AdoAccount, ProjectId, DefinitionId, ProjectName, ServiceId=ServiceTreeGuid, Environment, Version, Id
| where Timestamp between (ago(2d)..ago(1d))
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id);
let day1_stages = TimelineRecords
| extend StartTime = todatetime(StartTime)
| where StartTime between (ago(2d)..ago(1d))
| where not(Status has "unknown")
| summarize arg_max(StageAttempt, *) by AdoAccount, ProjectId, Id, HostType, Environment, Version, RecordId, tostring(parse_json(Task).Id)
| summarize
    totalRecords        = count(),
    successfulRecords   = countif(Status in~ ("Succeeded","Skipped","succeededWithIssues")),
    healthCheckTotal    = countif(tostring(parse_json(Task).Name) == "health-check-v1"),
    healthCheckSuccess  = countif(tostring(parse_json(Task).Name) == "health-check-v1" and Status in~ ("Succeeded","Skipped","succeededWithIssues")),
    HasLockbox          = max(iff(tostring(parse_json(Task).Name) == "lockbox-approval-request-prod_with_onebranch", 1, 0)),
    HasClassic          = max(iff(tostring(parse_json(Task).Name) == "ExpressV2Internal", 1, 0)),
    HasRA               = max(iff(tostring(parse_json(Task).Name) == "Ev2RARollout", 1, 0))
  by AdoAccount, ProjectId, Id, HostType, Environment, Version
| extend HealthEnabled = iff(healthCheckTotal > 0, 1, 0)
| extend HealthCheckPassed = iff(healthCheckTotal == 0, bool(false), healthCheckSuccess == healthCheckTotal)
| extend joinKey = strcat(AdoAccount, ProjectId, Version, Id, Environment);
let day1_StageTelemetry = Logs
| where todatetime(Timestamp) between (ago(2d)..ago(1d))
| where TaskId == "d98bb041-d191-41c2-b770-3dc3e7b10d7e"
| where (Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]")
     or Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId"
     or Message has "NamespaceRingJson:"
| extend AdoAccount = tolower(AdoAccount), ProjectName = tolower(ProjectName)
| extend MsgType = case(
    Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]", "Policy",
    Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId", "MOBR",
    "Cosmic")
| extend _ring = iff(MsgType == "MOBR", tostring(extract(@"[?&]ring=([^&\s]+)", 1, Message)), "")
| extend DeploymentRing = iif(isempty(_ring) or tolower(_ring) == "null", "", _ring)
| extend _depType = iff(MsgType == "MOBR", tostring(extract(@"[?&]deploymentType=([^&\s]+)", 1, Message)), "")
| extend DeploymentType = iif(isempty(_depType) or tolower(_depType) == "null", "", _depType)
| extend _cloud = iff(MsgType == "MOBR", tostring(extract(@"[?&]cloud=([^&\s]+)", 1, Message)), "")
| extend Cloud = iif(isempty(_cloud) or tolower(_cloud) == "null", "", _cloud)
| extend _trainset = iff(MsgType == "MOBR", tostring(extract(@"[?&]trainset=([^&\s]+)", 1, Message)), "")
| extend Trainset = iif(isempty(_trainset) or tolower(_trainset) == "null", "", _trainset)
| extend CosmicRing = iff(MsgType == "Cosmic", toupper(extract(@"(?i)\\\""ring\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message)), "")
| extend CosmicNamespace = iff(MsgType == "Cosmic", extract(@"(?i)\\\""namespace\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message), "")
| extend PolicyEvidence = iff(MsgType == "Policy", replace_string(Message, "[PolicyEvidenceRecord]", ""), "")
| extend jObj = parse_json(PolicyEvidence)
| mv-expand PolicyResults = iff(MsgType == "Policy", jObj.PolicyResults, pack_array(dynamic(null)))
| mv-expand RuleResults   = iff(isnotnull(PolicyResults), PolicyResults.RuleResults, pack_array(dynamic(null)))
| extend PolicyName = tostring(PolicyResults.PolicyName),
         PolicyRan  = tobool(RuleResults.PolicyRan),
         PolicyPassed = tobool(RuleResults.PolicyPassed),
         PolicyMode = tostring(RuleResults.PolicyMode)
| extend PolicyStatus = case(PolicyMode == "NotEnabled", 1, PolicyRan and PolicyPassed, 1, PolicyRan and not(PolicyPassed), 2, not(PolicyRan), 3, 4)
| extend Id = tostring(Id)
| where PolicyName in ("ring-bake-time","ring-progression","stage-bake-time","min-stage-count") or MsgType != "Policy"
| summarize
    RingBakeTime_Status    = min(iff(PolicyName == "ring-bake-time", PolicyStatus, 4)),
    RingProgression_Status = min(iff(PolicyName == "ring-progression", PolicyStatus, 4)),
    StageBakeTime_Status   = min(iff(PolicyName == "stage-bake-time", PolicyStatus, 4)),
    MinStageCount_Status   = min(iff(PolicyName == "min-stage-count", PolicyStatus, 4)),
    DeploymentRing  = take_anyif(DeploymentRing, DeploymentRing != ""),
    DeploymentType  = take_anyif(DeploymentType, DeploymentType != ""),
    Cloud           = take_anyif(Cloud, Cloud != ""),
    Trainset        = take_anyif(Trainset, Trainset != ""),
    CosmicRing      = take_anyif(CosmicRing, CosmicRing != ""),
    CosmicNamespace = take_anyif(CosmicNamespace, CosmicNamespace != "")
  by AdoAccount, ProjectName, ProjectId, DefinitionId, Id, Environment
| extend DeploymentRing = coalesce(DeploymentRing, ""), CosmicRing = coalesce(CosmicRing, "")
| extend CosmicNamespace = coalesce(CosmicNamespace, ""), DeploymentType = coalesce(DeploymentType, "")
| extend Cloud = coalesce(Cloud, ""), Trainset = coalesce(Trainset, "")
| extend Ring = iff(isempty(DeploymentRing), CosmicRing, toupper(DeploymentRing))
| extend Ring = trim(" ", Ring)
| extend Ring = iff(Ring contains "$", "", Ring)
| extend Ring = iff(Ring contains ",", "", Ring)
| extend isCosmicStage = iff(isnotempty(CosmicRing), 1, 0)
| extend Namespace = iff(isempty(CosmicNamespace), "Non-Cosmic", CosmicNamespace)
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id)
| lookup kind=leftouter day1_GetVersionInfo on PipelineJoinKey
| extend joinKey = strcat(AdoAccount, ProjectId, Version, Id, Environment)
| lookup kind=leftouter day1_stages on joinKey
| extend StageName = iff(Version =~ "V1", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| extend UniqueStageId = strcat(tolower(AdoAccount),"|",tolower(ProjectName),"|",tostring(DefinitionId),"|",Id,"|",StageName)
| project PipelineUrl, StageName, UniqueStageId,
    RingBakeTime_Status, RingProgression_Status, StageBakeTime_Status, MinStageCount_Status,
    HealthEnabled, HealthCheckPassed, HasLockbox, HasClassic, HasRA,
    Ring, Namespace, Cloud, DeploymentType, Trainset, isCosmicStage;
// --- Day 1: JOIN all 3 tables ---
let day1_result = day1_runs
| join kind=leftouter day1_yaml on PipelineUrl, BuildId
| join kind=leftouter day1_StageTelemetry on UniqueStageId
| project-away PipelineUrl1, BuildId1, PipelineUrl2, UniqueStageId1
| extend ProcessedDay = "Day1";
// =====================================================================
// DAY 2: ago(1d) to now()
// =====================================================================
// --- Day 2: all-stage-runs (exact copy scoped to day2) ---
let day2_belowarmpipelines = TimelineRecords
| where Task contains "StratusTriggerTask"
| distinct AdoAccount, ProjectId, Id
| join kind=leftouter PipelineRecords on AdoAccount, ProjectId, Id
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| summarize make_set(PipelineUrl);
let day2_allmobrpipelines = PipelineRecords
| project Timestamp=todatetime(Timestamp), AdoAccount, ProjectId, DefinitionId, ServiceGroupName, ServiceTreeGuid, Version, Id, Environment, ProjectName
| where Timestamp between (ago(1d)..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| where not(ServiceTreeGuid has "28A2529F-7085-4BBF-AA02-787135BB26A8") and not(ServiceTreeGuid has "29B815EE-C4FF-4760-B0CC-3255EE740883")
| where not(AdoAccount has_any ("cloudestest","1ESPipelineTemplates-OB","onebranchm365test1"))
| where ServiceGroupName != "AAD First Party Git"
| where not(ServiceGroupName contains "AAD")
| extend StageName = iff(Version =~ "V1" or Version == "", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| summarize arg_min(Timestamp, *) by Id, Environment
| distinct AdoAccount, ProjectName, ProjectId, DefinitionId, PipelineUrl, Version, Id, Environment, ServiceTreeGuid, StageName, ServiceGroupName;
let day2_runs = day2_allmobrpipelines
| where not(PipelineUrl has_any (day2_belowarmpipelines))
| extend UniqueStageId = strcat(tolower(AdoAccount),"|",tolower(ProjectName),"|",tostring(DefinitionId),"|",Id,"|",StageName)
| distinct PipelineUrl, BuildId=tolong(Id), ServiceTreeGuid, StageName, ServiceGroupName, UniqueStageId, AdoAccount, ProjectId, ProjectName;
// --- Day 2: yaml-to-run-list (exact copy scoped to day2 runs) ---
let day2_RecentRuns = day2_runs | project PipelineUrl, BuildId;
let day2_yaml = BuildYamlMapSnapshot
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(OrganizationName),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| join kind=leftsemi day2_RecentRuns on PipelineUrl, $left.BuildId == $right.BuildId
| project PipelineUrl, BuildId, YamlId;
// --- Day 2: stage-telemetry-policy-compliance (exact copy scoped to day2) ---
let day2_GetVersionInfo = PipelineRecords
| project Timestamp=todatetime(Timestamp), AdoAccount, ProjectId, DefinitionId, ProjectName, ServiceId=ServiceTreeGuid, Environment, Version, Id
| where Timestamp between (ago(1d)..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id);
let day2_stages = TimelineRecords
| extend StartTime = todatetime(StartTime)
| where StartTime between (ago(1d)..now())
| where not(Status has "unknown")
| summarize arg_max(StageAttempt, *) by AdoAccount, ProjectId, Id, HostType, Environment, Version, RecordId, tostring(parse_json(Task).Id)
| summarize
    totalRecords        = count(),
    successfulRecords   = countif(Status in~ ("Succeeded","Skipped","succeededWithIssues")),
    healthCheckTotal    = countif(tostring(parse_json(Task).Name) == "health-check-v1"),
    healthCheckSuccess  = countif(tostring(parse_json(Task).Name) == "health-check-v1" and Status in~ ("Succeeded","Skipped","succeededWithIssues")),
    HasLockbox          = max(iff(tostring(parse_json(Task).Name) == "lockbox-approval-request-prod_with_onebranch", 1, 0)),
    HasClassic          = max(iff(tostring(parse_json(Task).Name) == "ExpressV2Internal", 1, 0)),
    HasRA               = max(iff(tostring(parse_json(Task).Name) == "Ev2RARollout", 1, 0))
  by AdoAccount, ProjectId, Id, HostType, Environment, Version
| extend HealthEnabled = iff(healthCheckTotal > 0, 1, 0)
| extend HealthCheckPassed = iff(healthCheckTotal == 0, bool(false), healthCheckSuccess == healthCheckTotal)
| extend joinKey = strcat(AdoAccount, ProjectId, Version, Id, Environment);
let day2_StageTelemetry = Logs
| where todatetime(Timestamp) between (ago(1d)..now())
| where TaskId == "d98bb041-d191-41c2-b770-3dc3e7b10d7e"
| where (Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]")
     or Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId"
     or Message has "NamespaceRingJson:"
| extend AdoAccount = tolower(AdoAccount), ProjectName = tolower(ProjectName)
| extend MsgType = case(
    Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]", "Policy",
    Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId", "MOBR",
    "Cosmic")
| extend _ring = iff(MsgType == "MOBR", tostring(extract(@"[?&]ring=([^&\s]+)", 1, Message)), "")
| extend DeploymentRing = iif(isempty(_ring) or tolower(_ring) == "null", "", _ring)
| extend _depType = iff(MsgType == "MOBR", tostring(extract(@"[?&]deploymentType=([^&\s]+)", 1, Message)), "")
| extend DeploymentType = iif(isempty(_depType) or tolower(_depType) == "null", "", _depType)
| extend _cloud = iff(MsgType == "MOBR", tostring(extract(@"[?&]cloud=([^&\s]+)", 1, Message)), "")
| extend Cloud = iif(isempty(_cloud) or tolower(_cloud) == "null", "", _cloud)
| extend _trainset = iff(MsgType == "MOBR", tostring(extract(@"[?&]trainset=([^&\s]+)", 1, Message)), "")
| extend Trainset = iif(isempty(_trainset) or tolower(_trainset) == "null", "", _trainset)
| extend CosmicRing = iff(MsgType == "Cosmic", toupper(extract(@"(?i)\\\""ring\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message)), "")
| extend CosmicNamespace = iff(MsgType == "Cosmic", extract(@"(?i)\\\""namespace\\\""\s*:\s*\\\""([^\\\""]*)\\\""", 1, Message), "")
| extend PolicyEvidence = iff(MsgType == "Policy", replace_string(Message, "[PolicyEvidenceRecord]", ""), "")
| extend jObj = parse_json(PolicyEvidence)
| mv-expand PolicyResults = iff(MsgType == "Policy", jObj.PolicyResults, pack_array(dynamic(null)))
| mv-expand RuleResults   = iff(isnotnull(PolicyResults), PolicyResults.RuleResults, pack_array(dynamic(null)))
| extend PolicyName = tostring(PolicyResults.PolicyName),
         PolicyRan  = tobool(RuleResults.PolicyRan),
         PolicyPassed = tobool(RuleResults.PolicyPassed),
         PolicyMode = tostring(RuleResults.PolicyMode)
| extend PolicyStatus = case(PolicyMode == "NotEnabled", 1, PolicyRan and PolicyPassed, 1, PolicyRan and not(PolicyPassed), 2, not(PolicyRan), 3, 4)
| extend Id = tostring(Id)
| where PolicyName in ("ring-bake-time","ring-progression","stage-bake-time","min-stage-count") or MsgType != "Policy"
| summarize
    RingBakeTime_Status    = min(iff(PolicyName == "ring-bake-time", PolicyStatus, 4)),
    RingProgression_Status = min(iff(PolicyName == "ring-progression", PolicyStatus, 4)),
    StageBakeTime_Status   = min(iff(PolicyName == "stage-bake-time", PolicyStatus, 4)),
    MinStageCount_Status   = min(iff(PolicyName == "min-stage-count", PolicyStatus, 4)),
    DeploymentRing  = take_anyif(DeploymentRing, DeploymentRing != ""),
    DeploymentType  = take_anyif(DeploymentType, DeploymentType != ""),
    Cloud           = take_anyif(Cloud, Cloud != ""),
    Trainset        = take_anyif(Trainset, Trainset != ""),
    CosmicRing      = take_anyif(CosmicRing, CosmicRing != ""),
    CosmicNamespace = take_anyif(CosmicNamespace, CosmicNamespace != "")
  by AdoAccount, ProjectName, ProjectId, DefinitionId, Id, Environment
| extend DeploymentRing = coalesce(DeploymentRing, ""), CosmicRing = coalesce(CosmicRing, "")
| extend CosmicNamespace = coalesce(CosmicNamespace, ""), DeploymentType = coalesce(DeploymentType, "")
| extend Cloud = coalesce(Cloud, ""), Trainset = coalesce(Trainset, "")
| extend Ring = iff(isempty(DeploymentRing), CosmicRing, toupper(DeploymentRing))
| extend Ring = trim(" ", Ring)
| extend Ring = iff(Ring contains "$", "", Ring)
| extend Ring = iff(Ring contains ",", "", Ring)
| extend isCosmicStage = iff(isnotempty(CosmicRing), 1, 0)
| extend Namespace = iff(isempty(CosmicNamespace), "Non-Cosmic", CosmicNamespace)
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary")
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id)
| lookup kind=leftouter day2_GetVersionInfo on PipelineJoinKey
| extend joinKey = strcat(AdoAccount, ProjectId, Version, Id, Environment)
| lookup kind=leftouter day2_stages on joinKey
| extend StageName = iff(Version =~ "V1", Environment, strcat_array(array_slice(split(Environment, "_"), 1, -1), "_"))
| extend UniqueStageId = strcat(tolower(AdoAccount),"|",tolower(ProjectName),"|",tostring(DefinitionId),"|",Id,"|",StageName)
| project PipelineUrl, StageName, UniqueStageId,
    RingBakeTime_Status, RingProgression_Status, StageBakeTime_Status, MinStageCount_Status,
    HealthEnabled, HealthCheckPassed, HasLockbox, HasClassic, HasRA,
    Ring, Namespace, Cloud, DeploymentType, Trainset, isCosmicStage;
// --- Day 2: JOIN all 3 tables ---
let day2_result = day2_runs
| join kind=leftouter day2_yaml on PipelineUrl, BuildId
| join kind=leftouter day2_StageTelemetry on UniqueStageId
| project-away PipelineUrl1, BuildId1, PipelineUrl2, UniqueStageId1
| extend ProcessedDay = "Day2";
// =====================================================================
// FINAL: UNION both days
// =====================================================================
day1_result
| union day2_result
| summarize Day1=countif(ProcessedDay=="Day1"), Day2=countif(ProcessedDay=="Day2"), Total=count()
