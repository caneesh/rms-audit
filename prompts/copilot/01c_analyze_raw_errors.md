# Prompt 01c — Analyze Raw error handling

Continuing analysis of the XMLToHive pipeline.

**Do not write any code yet. Analysis only.**

Tell me:
1. Where are exceptions caught today?
2. What happens when a file fails to parse?
3. What happens when a DataFrame write fails?
4. How are errors logged (log4j, println, custom logger)?
5. Does processing continue after a file error, or does the whole job fail?
6. Can two runs execute concurrently, or is there locking?

Based on all analysis (01a, 01b, 01c), recommend:
- Best insertion points for run-level audit (start/end of job)
- Best insertion points for file-level audit (start/end of each file)
- Best insertion points for error capture

[PASTE: Error handling / try-catch code here]

[PASTE: Logging code or imports here]
