index="app_am_main" eventSource="*bedrock*"
    (eventName="InvokeModel" OR eventName="InvokeModelWithResponseStream" OR eventName="Converse" OR eventName="ConverseStream")

| eval inputTokens  = coalesce('responseElements.inputTokenCount',
                               'requestParameters.inputTokenCount', 0)
| eval outputTokens = coalesce('responseElements.outputTokenCount',
                               'requestParameters.outputTokenCount', 0)
| eval totalTokens  = inputTokens + outputTokens

| eval userIdentity = coalesce(
    'userIdentity.arn',
    'userIdentity.userName',
    'userIdentity.sessionContext.sessionIssuer.arn',
    'userIdentity.principalId'
  )

| eval modelId = coalesce(
    'requestParameters.modelId',
    'requestParameters.accept',
    "unknown"
  )

| stats
    count                    AS invocationCount,
    sum(inputTokens)         AS totalInputTokens,
    sum(outputTokens)        AS totalOutputTokens,
    sum(totalTokens)         AS totalTokens,
    dc(userIdentity)         AS distinctUsers,
    values(userIdentity)     AS userIdentities,
    values(modelId)          AS modelsUsed
  BY recipientAccountId

| sort - totalTokens

| eval totalTokens_fmt     = tostring(totalTokens,     "commas")
| eval totalInputTokens_fmt  = tostring(totalInputTokens,  "commas")
| eval totalOutputTokens_fmt = tostring(totalOutputTokens, "commas")

| table recipientAccountId
        invocationCount
        totalInputTokens_fmt
        totalOutputTokens_fmt
        totalTokens_fmt
        distinctUsers
        userIdentities
        modelsUsed
