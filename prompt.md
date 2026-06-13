Here's a grounded solution. The capability you want already exists in Bedrock — you just need to turn on the guardrail trace and then decide where to route it for monitoring.
Why you currently can't see the cause
When the model output trips the guardrail, the Converse response comes back with stopReason = "guardrail_intervened" and a generic blocked message. The reason (which PII entity or which regex, and on which side) is only emitted if you explicitly enable tracing. Without trace enabled, that diagnostic detail is discarded.
Layer 1 — Per-call diagnosis (enable the trace)
Add trace: "enabled" to the guardrailConfig on the Converse request. The response then carries trace.guardrail, split into inputAssessment and outputAssessments. Since your team is hitting blocks on the model output, the answer lives in outputAssessments.
Each assessment breaks down by policy. For your setup the relevant parts are:

sensitiveInformationPolicy.piiEntities[] → each has type (e.g. EMAIL, SSN), action (ANONYMIZED = masked, BLOCKED), and match (the offending value).
sensitiveInformationPolicy.regexes[] → each has name (e.g. your TAN rule), action, and match.

The field whose action is BLOCKED inside outputAssessments is exactly what stopped the call.
pythonresp = client.converse(
    modelId="...sonnet-4-6...",
    messages=msgs,
    guardrailConfig={
        "guardrailIdentifier": GID,
        "guardrailVersion": GVER,
        "trace": "enabled",
    },
)

if resp.get("stopReason") == "guardrail_intervened":
    culprits = []
    for gid, assessments in resp["trace"]["guardrail"].get("outputAssessments", {}).items():
        for a in assessments:
            sip = a.get("sensitiveInformationPolicy", {})
            for e in sip.get("piiEntities", []):
                if e["action"] == "BLOCKED":
                    culprits.append(("PII", e["type"]))
            for r in sip.get("regexes", []):
                if r["action"] == "BLOCKED":
                    culprits.append(("REGEX", r["name"]))   # e.g. "TAN"
    log.warning("Output blocked by: %s", culprits)
That gives the team an immediate, actionable signal per request — e.g. "blocked because the model echoed a TAN."
A governance caveat for a bank: the match field contains the raw PII value. Do not log match to general logs. Log only type / regex name for monitoring.
Layer 2 — Centralized monitoring
For ongoing visibility rather than per-call inspection, two complementary options:
CloudWatch metrics — Bedrock Guardrails emit metrics dimensioned by GuardrailPolicyType (including SensitiveInformationPolicy) and by action. You can build a dashboard and alarms on output-side interventions without writing any parsing code — good for "how often / which policy" trend analysis.
Model invocation logging → CloudWatch Logs / S3 — captures the full invocation including the guardrail trace, so you can query (CloudWatch Logs Insights) which entity types and regex names drive output blocks over time, and feed that back into prompt engineering. Important for your case: the invocation log stores the original, unmasked input — so if you enable it with PII data, lock down the log group/bucket (KMS, restricted access, retention) per your bank's data-handling policy, or keep this in a dedicated isolated account.
Recommended pattern for your team
Wrap the Converse call so that on guardrail_intervened it parses outputAssessments, and emits a PII-safe structured event (entity type / rule name / side, no raw match) to CloudWatch Logs as a custom metric. Then:

Input-side avoidance: also run the input through the same guardrail (or ApplyGuardrail) so you catch a TAN/PII present in the source legal doc before invocation and strip or mask it pre-prompt.
Prompt engineering: for cases where the model generates/echoes the PII in its output, add system-prompt instructions to not reproduce identifiers verbatim, and the trace tells you exactly which entity types to target.

Net: the input is being blocked/masked fine, but the model's output sometimes reproduces a value matching a blocked PII type or your TAN regex. The trace pinpoints which one every time.
Want me to package this into a deployable Lambda wrapper plus a CloudWatch dashboard/metric-filter template, or a short runbook doc for the team?
