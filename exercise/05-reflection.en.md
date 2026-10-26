**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:**
**My role on the review board:**

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

Based on the review board meeting, I would trust AI with **summarizing documentation, identifying architectural anti-patterns, and cross-referencing with known issues**. For example, the AI quickly flagged that `emailservice` lacks deduplication and that the shared PostgreSQL/RabbitMQ connection pool could exhaust under load—issues that human reviewers might miss until implementation. 

However, I would keep **evaluating trade-offs, validating business constraints, and setting priorities** for humans. For instance, the AI couldn't know that the marketing campaign was already live, making the "1-minute" SLA a hard requirement, nor could it gauge team skill or hardware lead times. These "soft" constraints often override purely technical recommendations, and only humans in the room can make those calls.

The sweet spot seems to be AI as a high-speed "chief analyst" that surfaces risks and options, while humans remain the "chief strategists" who own the final decision and balance technical trade-offs against business reality.
