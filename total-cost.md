index="app_am_2703_main" eventSource="*bedrock*"
    (eventName="InvokeModel" OR eventName="InvokeModelWithResponseStream"
     OR eventName="Converse" OR eventName="ConverseStream")
| eval inputTokens  = coalesce('additionalEventData.inputTokens', 0)
| eval outputTokens = coalesce('additionalEventData.outputTokens', 0)
| eval modelId      = coalesce('requestParameters.modelId', "unknown")

| eval inputPricePerMTok = case(
    like(modelId, "%opus-4%"),        5.00,
    like(modelId, "%sonnet-4%"),      3.00,
    like(modelId, "%sonnet-3-5%"),    3.00,
    like(modelId, "%claude-3-5-sonnet%"), 3.00,
    like(modelId, "%haiku%"),         1.00,
    like(modelId, "%nova-pro%"),      0.80,
    like(modelId, "%nova-lite%"),     0.06,
    like(modelId, "%nova-micro%"),    0.035,
    like(modelId, "%titan-text-express%"), 0.20,
    like(modelId, "%titan-text-lite%"),    0.15,
    like(modelId, "%titan-embed%"),   0.10,
    like(modelId, "%llama3-70b%"),    0.72,
    like(modelId, "%llama3-8b%"),     0.22,
    like(modelId, "%llama3.3%"),      0.72,
    like(modelId, "%llama4%"),        0.72,
    like(modelId, "%mistral-large%"), 0.50,
    like(modelId, "%mistral-small%"), 0.20,
    like(modelId, "%deepseek%"),      0.62,
    true(),                           1.00
  )

| eval outputPricePerMTok = case(
    like(modelId, "%opus-4%"),        25.00,
    like(modelId, "%sonnet-4%"),      15.00,
    like(modelId, "%sonnet-3-5%"),    15.00,
    like(modelId, "%claude-3-5-sonnet%"), 15.00,
    like(modelId, "%haiku%"),         5.00,
    like(modelId, "%nova-pro%"),      3.20,
    like(modelId, "%nova-lite%"),     0.24,
    like(modelId, "%nova-micro%"),    0.14,
    like(modelId, "%titan-text-express%"), 0.60,
    like(modelId, "%titan-text-lite%"),    0.20,
    like(modelId, "%titan-embed%"),   0.00,
    like(modelId, "%llama3-70b%"),    0.72,
    like(modelId, "%llama3-8b%"),     0.22,
    like(modelId, "%llama3.3%"),      0.72,
    like(modelId, "%llama4%"),        0.72,
    like(modelId, "%mistral-large%"), 1.50,
    like(modelId, "%mistral-small%"), 0.60,
    like(modelId, "%deepseek%"),      1.85,
    true(),                           5.00
  )

| eval inputCost  = (inputTokens / 1000000) * inputPricePerMTok
| eval outputCost = (outputTokens / 1000000) * outputPricePerMTok
| eval totalCost  = inputCost + outputCost

| stats
    count                AS "Invocations",
    sum(inputTokens)     AS "Input Tokens",
    sum(outputTokens)    AS "Output Tokens",
    sum(inputCost)       AS "Input Cost ($)",
    sum(outputCost)      AS "Output Cost ($)",
    sum(totalCost)       AS "Total Cost ($)"
  BY modelId

| eval "Input Cost ($)"  = "$" . tostring(round('Input Cost ($)', 2), "commas")
| eval "Output Cost ($)" = "$" . tostring(round('Output Cost ($)', 2), "commas")
| eval "Total Cost ($)"  = "$" . tostring(round('Total Cost ($)', 2), "commas")
| eval "Input Tokens"    = tostring('Input Tokens', "commas")
| eval "Output Tokens"   = tostring('Output Tokens', "commas")

| sort - "Total Cost ($)"
