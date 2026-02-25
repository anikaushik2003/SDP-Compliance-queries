set notruncation;
set norequesttimeout;
set servertimeout = 1h;
set maxmemoryconsumptionperiterator=32212254720;
let timeSpan = 60d;
let GetVersionInfo = cluster('onebranchm365release.eastus').database('onebranchreleasetelemetry').PipelineRecords
| project Timestamp=todatetime(Timestamp),AdoAccount,ProjectId,DefinitionId, ProjectName, ServiceId = ServiceTreeGuid, Environment, Version,Id
| where  Timestamp between (now()-60d..now())
| extend PipelineUrl = strcat("https://dev.azure.com/",tolower(AdoAccount),"/",ProjectId,"/_build?definitionId=",DefinitionId,"&_a=summary") // 710k rows
| extend PipelineJoinKey = strcat(PipelineUrl, Environment, Id);
// stages has about 600k
let stages =
    cluster('onebranchm365release.eastus')
    .database('onebranchreleasetelemetry')
    .TimelineRecords
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
        TaskNames           = make_set(tostring(parse_json(Task).Name))
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
    cluster('onebranchm365release.eastus')
    .database('onebranchreleasetelemetry')
    .Logs
    | where todatetime(Timestamp) > ago(timeSpan)
    | where TaskId == "d98bb041-d191-41c2-b770-3dc3e7b10d7e"
    | where Message startswith "[PolicyEvidenceRecord]" and Message endswith "[PolicyEvidenceRecord]"
    | extend PolicyEvidence = replace_string(Message, "[PolicyEvidenceRecord]", "")
    | extend AdoAccount = tolower(AdoAccount), ProjectName = tolower(ProjectName)
    | extend jObj = parse_json(PolicyEvidence)
    | mv-expand PolicyResults = jObj.PolicyResults
    | mv-expand RuleResults = PolicyResults.RuleResults
    | extend
        PolicyName    = tostring(PolicyResults.PolicyName),
        PolicyVersion = tostring(PolicyResults.Version),
        PolicyRan     = tobool(RuleResults.PolicyRan),
        PolicyPassed  = tobool(RuleResults.PolicyPassed),
        PolicyMode    = tostring(RuleResults.PolicyMode),
        RuleData      = tostring(RuleResults.Data)
    | extend PolicyStatus = case(
            PolicyMode == "NotEnabled", 1,
            PolicyRan and PolicyPassed, 1,
            PolicyRan and not(PolicyPassed), 2,
            not(PolicyRan), 3,
            4
        )
    | extend Id = tostring(Id)
    | where PolicyName in ("ring-bake-time", "ring-progression", "stage-bake-time", "min-stage-count")
    | summarize
      RingBakeTime_Status =
          min(iff(PolicyName == "ring-bake-time", PolicyStatus, 4)),
      RingProgression_Status =
          min(iff(PolicyName == "ring-progression", PolicyStatus, 4)),
      StageBakeTime_Status =
          min(iff(PolicyName == "stage-bake-time", PolicyStatus, 4)),
      MinStageCount_Status =
          min(iff(PolicyName == "min-stage-count", PolicyStatus, 4))
      by AdoAccount, ProjectName, ProjectId, DefinitionId, Id, Environment;
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
    | project
        AdoAccount, ProjectName, DefinitionId, Id, Environment, PipelineUrl,
        RingBakeTime_Status, RingProgression_Status,
        StageBakeTime_Status, MinStageCount_Status,
        HealthEnabled, HealthCheckPassed, UniqueStageId;
StageTelemetry_PolicyCompliance