# Prompt 01b — Analyze Raw data flow

Continuing analysis of the XMLToHive pipeline.

**Do not write any code yet. Analysis only.**

Look at how data flows and tell me:
1. How Sequence Files are listed and opened
2. How XML/JSON is parsed from each record
3. How the 13 DataFrames are created (list each with target table name)
4. For each DataFrame: write mode, storage format, partitioning, HDFS location
5. Is there any natural key that links a row back to its source message?

[PASTE: The file processing / XML parsing code here]

[PASTE: The DataFrame creation and write code here]
