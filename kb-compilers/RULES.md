# ABSOLUTE BASE RULES (every agent, every file)

1. RESEARCH FIRST. Before writing any file, web-search every section. Primary sources first (official docs, ISO/language standards, specs/RFCs, vendor docs, original papers, university course notes, well-known textbooks). Blogs only as support.
2. NO FABRICATION. Never invent a source, quote, number, flag, API, or paper. Unverifiable → mark `[unverified]` or omit. Version-specific facts get a version + date ("as of GCC 15, Nov 2026").
3. EVERY FILE ENDS WITH "Sources" (real links) and "Verify yourself" (code to run, claims to check against which source).
4. CODE MUST RUN. Compile/run every code example in the sandbox where possible (gcc/g++/clang/python/java). Comment every block. C/Java: explain logic. C++/Python: explain syntax AND logic, line by line.
5. DEPTH CURVE. 01-beginner (zero knowledge, analogies) → 02-intermediate (precise, standard techniques) → 03-advanced (internals, edge cases, performance) → 04-pro (research/industry depth, tradeoffs, open problems, key papers). Never use a concept before a lesson teaches it.
6. COVERAGE CHECK. Track README must list topics of a standard university course AND a standard industry reference, and show where each is covered (or why excluded).
7. EVERY LESSON: why it exists → intuition/analogy → precise definition → step by step (for problems: dumbest approach → what's annoying → how each wall forces the next idea) → mermaid/ASCII diagrams → commented code → common mistakes → connections to other tracks → self-check Qs with answers in <details> → quirky Tamil + English mnemonic for anything to memorize → Sources → Verify yourself.
8. TRACK FILES: README.md (overview, prereqs, mermaid mindmap roadmap, time estimates, coverage check), glossary.md (A–Z, plain + precise meaning + lesson link), appendix.md (cheat sheets, references, tools, books, courses, papers, docs), exercises.md (graded per level, solutions in <details>, 2–3 mini projects), interview-questions.md (model answers), connections.md, common-mistakes.md.
9. COMPLETE, NOT SKELETAL. No "...", no TODOs, no "left as exercise". Full prose. GitHub-flavoured Markdown.
10. TONE: thoughtful, conversational, dense but readable.

## About the learner
Final-year CSE student. Comfortable in C and Java; basic Python; no C++ syntax. Learns via big-picture mind maps and connecting ideas; wants WHY before HOW.
Other tracks to cross-link: C, C++, Compilers, Computer Architecture, OS, Parallel/CUDA, DSA, Math for CS, Linux, Git, Networks, DBMS, Cloud/DevOps, System Design, Python, Java, Security, ML/LLM agents, AI dev tools, Quantum, Testing/Verification, Rust, Research skills, Career skills.
Projects (add "Where this shows up in my projects" only when genuine): compiler-fuzz (differential tester for C compilers: UB-free random C generator, GCC vs Clang at -O0/-O2/-O3, ddmin reducer), Nidhogg (Elasticsearch fleet ops console + Logly NL log search), aws-waste-scanner (AWS Lambda + Terraform, multi-account), QiskitSage (multi-agent Qiskit PR reviewer on local Ollama), AURIXA (ERP demand-spike agent: Convex, Gemini, LangGraph), ARIA (LangGraph + FastAPI human-in-the-loop email agent).
