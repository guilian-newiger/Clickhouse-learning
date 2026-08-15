# ClickHouse Learning
I want to become a Core Developer of a relational Database Management System (DBMS) for Analytical Workloads (OLAP) with specialization in Storage.

For this goal, I not only have to understand theoretical concepts but I have to be able to apply them in practice. ClickHouse is a relational OLAP DBMS and it's open source. So I can ask myself questions, look at the source code of ClickHouse and use the GNU Debugger (gdb) to step through an example, inspect the memory. Since I want to specialize in Storage, viewing the raw bytes of files is helpful. For that, I use the hex-mode of vim.

In this repository I curate my experiences in a structured format in order to revisit past learnings, follow my own progress and demonstrate potential future DBMS employers my domain knowledge, analytical skills and documentation prowess.

Disclaimer: In my documentation I refer to code snippets from the GitHub repository of ClickHouse (https://github.com/clickhouse/clickhouse). These references are for educational purposes only.

## Structure
This repository contains my curated experiences from inspecting how ClickHouse works under the hood to understand how theoretical DBMS concepts are implemented in practice.

The directory structure is based on domain knowledge, specifically the [architecture overview of ClickHouse from the viewpoint of a contributor](https://clickhouse.com/docs/resources/develop-contribute/introduction/architecture) (as of 2026-08-07). The file order is chronological and represents my progression as I learn.

Each file is led by a question, followed by my own hypothesis. Then, I step through an example to answer the question. Finally, I summarize my learning and note new questions that came up.

Therefore, a file is divided into four sections:
1. Question
2. Hypothesis
3. Example
4. Conclusion