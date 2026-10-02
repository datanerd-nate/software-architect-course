# Fitness Function
1. Ingestion Latency & Availability Fitness Function
   * Objective: Guarantee that the Student Intake UI remains fast regardless of downstream processing or LLM latency.
   * Metric: $p_{99} \text{ Latency} \le 200\text{ms}$ on POST /v1/answers at $10,000 \text{ req/sec}$.
   * Implementation: Automated CI/CD load test executed against candidate builds simulating end-of-term write surges. If $p_{99} > 200\text{ms}$, the build breaks.
1. Transactional Outbox Lag Fitness Function
   * Objective: Ensure event queues do not stall, maintaining reasonable eventual consistency bounds.
   * Metric: Maximum Outbox Delivery Lag $\le 5,000\text{ms}$ between local database commit and message broker publication.
   * Implementation: Synthetic metric collector querying NOW() - created_at on unprocessed outbox rows. Triggers an architectural alert if lag breaches $5\text{s}$ for over 2 consecutive minutes.
1. Idempotency Assertion Fitness Function
   * Objective: Prevent duplicate events from causing double-writes or corrupting transcripts.
   * Metric: $0\%$ duplicate writes when the exact same event payload is injected $N$ times.
   * Implementation: Integration test suite that publishes duplicate AnswerGraded and GradeValidatedByHuman events with identical event_id and submission_id values, asserting that the target database row count remains unchanged ($1$).
1. AI Calibration & HITL Drift Fitness Function 
   * Objective: Ensure AI grading accuracy does not degrade over time or across new prompt versions.
   * Metric: Human Override Rate on the $X\%$ statistical audit sample must remain $\le 2.0\%$.
   * Implementation: Automated daily pipeline querying Administrative Service database:$$\text{Override Rate} = \frac{\text{Count}(\text{was\_score\_overridden} = \text{TRUE})}{\text{Count}(\text{review\_trigger} = \text{'STATISTICAL\_AUDIT'})}$$If Override Rate $> 2.0\%$, trigger an administrative alert to halt auto-approvals and review the prompt/rubric.