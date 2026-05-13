<form version="1.1" theme="dark">
  <label>Bedrock Token Intelligence — Account Summary</label>
  <description>Per-account view of AWS Bedrock invocations, token consumption, and user identity across all AWS accounts in the organization.</description>

  <!-- ============================================================
       GLOBAL FILTERS
       ============================================================ -->
  <fieldset submitButton="false" autoRun="true">

    <!-- TIME PICKER -->
    <input type="time" token="time_tok" searchWhenChanged="true">
      <label>Time Range</label>
      <default>
        <earliest>-7d@d</earliest>
        <latest>now</latest>
      </default>
    </input>

    <!-- MODEL FILTER (dynamic — auto-populates from your data) -->
    <input type="dropdown" token="model_tok" searchWhenChanged="true">
      <label>Model</label>
      <choice value="*">All Models</choice>
      <populatingSearch fieldForLabel="modelFamily" fieldForValue="modelFamily">
        <![CDATA[
          index="app_am_2703_main" eventSource="*bedrock*"
              (eventName="InvokeModel" OR eventName="InvokeModelWithResponseStream"
               OR eventName="Converse" OR eventName="ConverseStream")
          | eval modelId = coalesce('requestParameters.modelId', "unknown")
          | eval modelFamily = case(
              like(modelId, "%claude%"),   "claude",
              like(modelId, "%titan%"),    "titan",
              like(modelId, "%llama%"),    "llama",
              like(modelId, "%amazon%"),   "amazon",
              like(modelId, "%mistral%"),  "mistral",
              like(modelId, "%cohere%"),   "cohere",
              like(modelId, "%ai21%"),     "ai21",
              like(modelId, "%stability%"),"stability",
              true(),                      "other"
            )
          | stats count BY modelFamily
          | sort - count
          | fields modelFamily
        ]]>
      </populatingSearch>
      <default>*</default>
      <initialValue>*</initialValue>
    </input>

    <!-- ACCOUNT FILTER (dynamic) -->
    <input type="dropdown" token="account_tok" searchWhenChanged="true">
      <label>Account</label>
      <choice value="*">All Accounts</choice>
      <populatingSearch fieldForLabel="recipientAccountId" fieldForValue="recipientAccountId">
        <![CDATA[
          index="app_am_2703_main" eventSource="*bedrock*"
              eventName="InvokeModel"
          | stats count BY recipientAccountId
          | sort recipientAccountId
          | fields recipientAccountId
        ]]>
      </populatingSearch>
      <default>*</default>
      <initialValue>*</initialValue>
    </input>

  </fieldset>

  <!-- ============================================================
       BASE SEARCH — runs once, all panels reference it
       ============================================================ -->
  <search id="base_bedrock">
    <query>
      index="app_am_2703_main" eventSource="*bedrock*"
          (eventName="InvokeModel" OR eventName="InvokeModelWithResponseStream"
           OR eventName="Converse" OR eventName="ConverseStream")
          recipientAccountId="$account_tok$"
      | eval inputTokens  = coalesce('additionalEventData.inputTokens', 0)
      | eval outputTokens = coalesce('additionalEventData.outputTokens', 0)
      | eval totalTokens  = inputTokens + outputTokens
      | eval modelId      = coalesce('requestParameters.modelId', "unknown")
      | eval userIdentity = coalesce(
          'userIdentity.arn',
          'userIdentity.sessionContext.sessionIssuer.arn',
          'userIdentity.userName',
          'userIdentity.principalId'
        )
      | where like(modelId, "%$model_tok$%")
    </query>
    <earliest>$time_tok.earliest$</earliest>
    <latest>$time_tok.latest$</latest>
  </search>

  <!-- ============================================================
       ROW 1 — KPI CARDS (4 single-value panels)
       ============================================================ -->
  <row>
    <!-- KPI 1: Total Invocations -->
    <panel>
      <title>Total Invocations</title>
      <single>
        <search base="base_bedrock">
          <query>
            | stats count AS totalInvocations
          </query>
        </search>
        <option name="drilldown">none</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x1182f3","0x1182f3"]</option>
        <option name="rangeValues">[0]</option>
        <option name="useColors">1</option>
        <option name="numberPrecision">0</option>
        <option name="underLabel">across all accounts</option>
      </single>
    </panel>

    <!-- KPI 2: Total Input Tokens -->
    <panel>
      <title>Total Input Tokens</title>
      <single>
        <search base="base_bedrock">
          <query>
            | stats sum(inputTokens) AS totalInputTokens
          </query>
        </search>
        <option name="drilldown">none</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x7c3aed","0x7c3aed"]</option>
        <option name="rangeValues">[0]</option>
        <option name="useColors">1</option>
        <option name="numberPrecision">0</option>
        <option name="underLabel">prompt tokens consumed</option>
      </single>
    </panel>

    <!-- KPI 3: Total Output Tokens -->
    <panel>
      <title>Total Output Tokens</title>
      <single>
        <search base="base_bedrock">
          <query>
            | stats sum(outputTokens) AS totalOutputTokens
          </query>
        </search>
        <option name="drilldown">none</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x10b981","0x10b981"]</option>
        <option name="rangeValues">[0]</option>
        <option name="useColors">1</option>
        <option name="numberPrecision">0</option>
        <option name="underLabel">completion tokens generated</option>
      </single>
    </panel>

    <!-- KPI 4: Distinct Users -->
    <panel>
      <title>Distinct Users</title>
      <single>
        <search base="base_bedrock">
          <query>
            | stats dc(userIdentity) AS distinctUsers
          </query>
        </search>
        <option name="drilldown">none</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0xf59e0b","0xf59e0b"]</option>
        <option name="rangeValues">[0]</option>
        <option name="useColors">1</option>
        <option name="numberPrecision">0</option>
        <option name="underLabel">unique IAM identities</option>
      </single>
    </panel>
  </row>

  <!-- ============================================================
       ROW 2 — CHARTS (bar + pie side-by-side)
       ============================================================ -->
  <row>
    <!-- Token Consumption by Account (Bar Chart) -->
    <panel>
      <title>Token Consumption by Account</title>
      <chart>
        <search base="base_bedrock">
          <query>
            | stats
                sum(inputTokens)  AS "Input Tokens",
                sum(outputTokens) AS "Output Tokens"
              BY recipientAccountId
            | eval _total = 'Input Tokens' + 'Output Tokens'
            | sort - _total
            | fields - _total
          </query>
        </search>
        <option name="charting.chart">bar</option>
        <option name="charting.chart.stackMode">stacked</option>
        <option name="charting.drilldown">none</option>
        <option name="charting.legend.placement">bottom</option>
        <option name="charting.axisTitleX.text">Tokens</option>
        <option name="charting.axisTitleY.text">Account ID</option>
        <option name="charting.fieldColors">{"Input Tokens":0x7c3aed,"Output Tokens":0x10b981}</option>
        <option name="height">350</option>
      </chart>
    </panel>

    <!-- Model Distribution (Pie / Donut Chart) -->
    <panel>
      <title>Model Distribution</title>
      <chart>
        <search base="base_bedrock">
          <query>
            | stats count AS invocations BY modelId
            | sort - invocations
          </query>
        </search>
        <option name="charting.chart">pie</option>
        <option name="charting.drilldown">none</option>
        <option name="charting.chart.showPercent">1</option>
        <option name="height">350</option>
      </chart>
    </panel>
  </row>

  <!-- ============================================================
       ROW 3 — Invocation Trend Over Time (Timechart)
       ============================================================ -->
  <row>
    <panel>
      <title>Invocation Trend Over Time</title>
      <chart>
        <search base="base_bedrock">
          <query>
            | timechart span=1d count AS invocations BY recipientAccountId
          </query>
        </search>
        <option name="charting.chart">line</option>
        <option name="charting.drilldown">none</option>
        <option name="charting.legend.placement">bottom</option>
        <option name="charting.axisTitleX.text">Date</option>
        <option name="charting.axisTitleY.text">Invocations</option>
        <option name="charting.chart.nullValueMode">zero</option>
        <option name="height">250</option>
      </chart>
    </panel>
  </row>

  <!-- ============================================================
       ROW 4 — Per-Account Summary Table
       ============================================================ -->
  <row>
    <panel>
      <title>Per-Account Summary</title>
      <table>
        <search base="base_bedrock">
          <query>
            | stats
                count                AS "Invocations",
                sum(inputTokens)     AS "Input Tokens",
                sum(outputTokens)    AS "Output Tokens",
                sum(totalTokens)     AS "Total Tokens",
                dc(userIdentity)     AS "Distinct Users",
                values(userIdentity) AS "User Identities",
                values(modelId)      AS "Models Used"
              BY recipientAccountId
            | sort - "Total Tokens"
            | rename recipientAccountId AS "Account ID"
          </query>
        </search>
        <option name="drilldown">row</option>
        <option name="count">20</option>
        <option name="dataOverlayMode">none</option>
        <option name="wrap">false</option>
        <!-- Drill-down: clicking a row opens Dashboard 2 filtered to that account -->
        <drilldown>
          <link target="_blank">
            /app/search/bedrock_dashboard_2_user_drilldown?form.time_tok.earliest=$time_tok.earliest$&amp;form.time_tok.latest=$time_tok.latest$&amp;form.account_tok=$row.Account ID$&amp;form.model_tok=$model_tok$
          </link>
        </drilldown>
      </table>
    </panel>
  </row>

</form>
