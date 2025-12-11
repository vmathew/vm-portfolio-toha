Here is a clean, executive-ready talking script you can use during the meeting.
It is structured, concise, and aligned to the audience (Governance, Risk, Controls, Automation Leadership).
You can read it as-is, or keep it as a reference while speaking.

⸻

Bedrock Model Access Governance – Talking Script for Today’s Meeting

Opening & Context
“Thanks everyone for joining. I want to walk through a scaling challenge we’re starting to face with our Bedrock model-access governance and highlight a couple of decisions we may need to make going forward.”

⸻

1. How Model Access Is Controlled Today (Current Process)

“As most of you know, when Amazon Bedrock originally launched, AWS gave a UI-based Model Access page that allowed us to enable or disable specific foundation models per account.

Earlier this year, AWS removed that feature and mandated that all model access must be controlled exclusively through Service Control Policies (SCPs) at the AWS Organizations level.

Since then, our Cloud Engineering team has been maintaining a structured allow-list of Bedrock models inside SCPs.
This SCP-based enforcement has been the primary mechanism ensuring:
	•	Only approved models are accessible
	•	Access is consistent across OUs
	•	We remain compliant with internal governance, risk, and model-review guidelines

So far, this process has worked well.”

⸻

2. The Scaling Problem With SCPs

“However, SCPs come with strict size limits — both on character count and policy structure.
When Bedrock had a smaller set of models, this wasn’t an issue.

But now we have:
	•	A rapidly expanding list of foundation models across providers
	•	Frequent new model releases
	•	Multiple model versions per provider
	•	Internal demand shifting toward the latest models

This has caused our SCPs to grow significantly in size.
We are already optimizing and compressing them as much as possible, but we are approaching the SCP maximum size threshold.

In short: We can continue to onboard new models for now, but we will reach a hard cap in the foreseeable future.”

⸻

3. Challenge: Granular Control vs. Manageability

“Our existing approach enforces access at a per-model level, which gives the bank strong control and reduces model-risk exposure.

But the granularity is also what causes the SCP bloat.

This raises an important governance question:

**Should we continue restricting model access at the individual model level?

Or should we move to a ‘provider-level’ enforcement model?”**

⸻

4. Option 1 — Provider-Level Control (Relaxed Granularity)

“Under this model, instead of listing every single model ARN in the SCP, we control access at the provider level — for example:
	•	Allow all Anthropic models
	•	Allow all Amazon Titan models
	•	Allow all Meta models
	•	Etc.

Pros:
	•	SCP size becomes manageable and stable
	•	Lower operational overhead
	•	Faster onboarding of new models
	•	Cleaner Org-level implementation

Cons:
	•	We lose precise control per model
	•	Governance & risk would need to treat entire providers as ‘approved’ blocks
	•	Increased exposure if a provider launches a model that has not yet been reviewed or risk-assessed

Given the governance audience, this may not be acceptable without significant process changes.”

⸻

5. Option 2 — Stay With Per-Model Control + Purge Unused Models

“If provider-level access is not acceptable — which is likely from a risk standpoint — then our best alternative is:

A continuous purge-and-optimize process.

Here’s how it would work:
	•	Use Splunk as the source of truth for Bedrock usage
	•	Identify models that have zero or extremely low invocation activity
	•	Validate with IT Governance/Model Review whether those unused models can be disabled
	•	Remove them from the SCP allow-list
	•	Reclaim character space so newer models can be onboarded

This allows us to maintain granular control while still staying under SCP limits for as long as possible.”

**Short-term: this gives us breathing room.
Long-term: it only delays — but does not eliminate — capacity pressure.”

⸻

6. Long-Term Considerations

“I want to highlight this clearly:

Even with purge cycles, as Bedrock continues to grow, SCP size limits will become an ongoing constraint.

We will eventually need either:
	•	A governance-driven shift to broader provider-level approvals, or
	•	A supplemental control layer (e.g., internal approval workflows, tagging-based runtime checks, IAM-level filters), backed by risk teams

This meeting is to start that dialogue early.”

⸻

7. What We Are Proposing Immediately

“For now, the Cloud Engineering team recommends:
	1.	Continue with per-model SCP control (status quo)
	2.	Introduce a regular, data-driven purge process using Splunk usage analytics
	3.	Align with Governance/Risk on criteria for model removal
	4.	Document a future-state decision path in case SCP limits are hit again

This ensures we stay compliant, keep the environment secure, and maintain operational continuity while we evaluate long-term options.”

⸻

8. Closing

“We wanted to bring this forward now so leadership across governance, risk, controls, and automation has visibility — and so we can jointly define the future direction.

Happy to walk deeper into the specifics and discuss what governance model best supports both scalability and risk alignment.”

⸻

If you’d like, I can also generate:

✔ A one-slide executive summary
✔ A diagram showing the SCP control model
✔ A decision tree for model vs provider-level control
✔ A formal proposal document to circulate after the meeting