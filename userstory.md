

Healthcare Claim Injury Classification System — User Story
Business Context

The client wants to build an AI-powered healthcare claim classification system that can analyze claim-related documents and determine whether an injury is:

Work-related
Non-work-related
Undetermined

The goal is to reduce the manual effort required by claim reviewers while maintaining high classification accuracy, explainability, and compliance.

The system should process multiple types of documents such as:

Claim forms
Doctor notes
Medical conversations/transcripts
Incident reports
Employer statements
Supporting medical records

The AI system must identify the root cause of the injury and determine whether the injury originated during work activities or outside of work.

Example Scenarios
Example 1 — Non-Work-Related Injury

A conversation between the patient and doctor indicates that the patient initially developed neck pain after sleeping in an incorrect position at home. The pain later worsened while operating machinery at work.

Expected Outcome
Classification: NON_WORK_RELATED
Secondary Tag: WORK_AGGRAVATED
Reason:
The injury originated outside the workplace.
Work activities aggravated the existing condition.
Example 2 — Work-Related Injury

An incident report states that the employee fell down the stairs while carrying documents during work hours.

Expected Outcome
Classification: WORK_RELATED
Reason:
The injury occurred while performing work-related duties.
Example 3 — Undetermined Case

The available documents contain conflicting statements or insufficient information to determine the origin of the injury.

Expected Outcome
Classification: UNDETERMINED
Reason:
The evidence is incomplete, inconsistent, or inconclusive.
Business Objective

The objective of the system is to:

Reduce manual claim review effort
Improve claim processing efficiency
Minimize incorrect classifications
Provide explainable AI-driven decisions
Support human reviewers through confidence-based escalation
Key Classification Goals

The system should minimize:

Risk Type	Description
False Positives	Incorrectly classifying non-work injuries as work-related
False Negatives	Missing genuine work-related injuries

The system must support:

confidence scoring
explainability
human review escalation
auditability