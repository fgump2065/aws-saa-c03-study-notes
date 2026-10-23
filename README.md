# AWS Solutions Architect Associate (SAA-C03) — Topic Guides

**37 focused, exam-oriented study guides** for the AWS Solutions Architect Associate (SAA-C03) exam, written with plain-language explanations and practical analogies.

> Adapted from [RonitSachdev's original SAA-C03 topic guides](https://github.com/RonitSachdev/aws-saa-c03-guides). This copy has been edited for accuracy and currency, including factual corrections and an updated scope note.

## What makes these different

Every guide follows the same format:

1. **The idea** — each topic explained from the ground up with a real-world analogy (Multi-AZ = a spare tire, Read Replicas = extra checkout lanes, Route 53 = a phonebook, NAT = a receptionist mailing your letters…)
2. **Core concepts** — key facts, limits, trade-offs, and clearly marked **"THE trap"** callouts
3. **Question patterns** — exam-style scenarios, answers, and the clues that point to them
4. **Pocket card** — a keyword → answer table for rapid review

The guides emphasize decision patterns such as Multi-AZ vs. Read Replicas, Gateway vs. Interface Endpoints, and SQS vs. SNS vs. Kinesis vs. EventBridge.

## Start here

📖 **[Topic Guides Index](topic-guides/00-README.md)** — the index links to all 37 topic guides and suggests a reading order.

For a focused review, start with:

- [VPC](topic-guides/06-VPC.md) — subnets, routing, security groups, and connectivity
- [S3](topic-guides/02-S3.md) — storage classes, encryption, and object protection
- [RDS & Aurora](topic-guides/09-RDS-Aurora.md) — availability, replicas, and database choices
- [SQS, SNS & Kinesis](topic-guides/17-SQS-SNS-Kinesis.md) — messaging and streaming patterns
- [Exam Traps & Key Patterns](topic-guides/36-Exam-Traps-KeyPatterns.md) — useful after reviewing the service guides

## Exam-technique reminders

1. **Read the final requirement carefully** — phrases like “most cost-effective,” “least operational overhead,” and “highly available” can change the right choice.
2. **Count the requirements** — “X as well as Y” means the solution must satisfy both.
3. **Check absolute qualifiers** — words such as *serverless, automatically, at no cost, immediately, directly, cannot,* and *always* can make an otherwise plausible option wrong.
4. **Compare the deciding clause** when two options look similar.
5. **Use stated numbers as constraints** — retrieval time, IOPS, RPO, and RTO can rule out options before cost or operational effort is compared.
6. **Consider less familiar services when they fit the stated requirements**; verify unfamiliar features against AWS documentation.

## Scope and currency

These guides are a focused study aid, not a complete syllabus or an AWS publication. The SAA-C03 exam tests architecture decisions across four domains, and AWS says its in-scope service list is non-exhaustive and subject to change. Use the [official SAA-C03 exam guide](https://docs.aws.amazon.com/pdfs/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.pdf) and [current in-scope services list](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/saa-03-in-scope-services.html) alongside these notes. Verify changing availability, quotas, features, and prices against current AWS documentation. Reviewed and updated October 2026.

## Disclaimer

Community study notes; not affiliated with or endorsed by AWS. These notes reflect a candidate's study and exam experience and do not guarantee exam results.

## License and attribution

[CC BY 4.0](LICENSE). You may share and adapt these materials, including commercially, if you give appropriate credit to **RonitSachdev**, link to the [original source repository](https://github.com/RonitSachdev/aws-saa-c03-guides), identify changes you made, and do not imply the author endorses your work. The repository's LICENSE file contains the full terms.
