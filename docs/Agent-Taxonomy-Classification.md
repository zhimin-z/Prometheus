# Prometheus SWE Agent Taxonomy Classification Report

This document provides a comprehensive multi-label classification of the Prometheus agent according to the SWE Agent Taxonomy framework.

---

## Agent Architecture

**Labels:** MULTI-AGENT-HIERARCHICAL, MULTI-AGENT-COLLABORATIVE

**Justification:**

Prometheus implements a **hierarchical multi-agent architecture** with specialized agents coordinated through LangGraph state machines. The system features:

1. **Hierarchical Coordination**: The `IssueGraph` in `prometheus/lang_graph/graphs/issue_graph.py` acts as a manager that delegates tasks to specialized worker agents based on issue classification (bug, feature, question, documentation). The Issue Classification Agent routes work to appropriate specialized subgraph agents (Bug Pipeline, Feature Pipeline, Question Pipeline, Documentation Pipeline).

2. **Collaborative Specialization**: Multiple specialized agents work together with complementary roles:
   - **Issue Classification Agent**: Categorizes incoming issues
   - **Context Retrieval Agent**: Gathers relevant code and documentation from the knowledge graph
   - **Bug Reproduction Agent**: Attempts to reproduce reported bugs
   - **Issue Resolution Agent**: Generates and validates bug fix patches
   - **Question Analysis Agent**: Handles Q&A with tool-augmented LLM

Each agent has a specific expertise domain and contributes to the overall solution through coordinated workflow, as documented in `docs/Multi-Agent-Architecture.md`. The state flows through nodes managed by LangGraph, with parent states inherited by child subgraphs and results passed back through state returns.

---

## Agent Workflow

**Labels:** ITERATIVE, SELF-IMPROVING

**Justification:**

Prometheus employs both **iterative** and **self-improving** workflow patterns:

1. **Iterative Pattern**: The system implements continuous feedback loops throughout its execution:
   - **Bug Reproduction Subgraph** (`prometheus/lang_graph/subgraphs/bug_reproduction_subgraph.py`): Executes reproduction tests, observes results, and uses feedback to refine the next attempt in a Thought → Action → Observation cycle until successful reproduction or max iterations.
   - **Context Retrieval Subgraph** (`prometheus/lang_graph/subgraphs/context_retrieval_subgraph.py`): Iteratively refines queries and retrieves context (2-4 loops), adjusting based on previous retrieval results.
   - **Patch Generation**: The Issue Resolution Agent generates multiple candidate patches, validates them against tests, and iterates with error feedback if tests fail.

2. **Self-Improving Pattern**: The agent learns from past experiences through its Athena semantic memory system:
   - **Memory Retrieval Node** (`prometheus/lang_graph/nodes/memory_retrieval_node.py`): Retrieves previously successful solutions from episodic memory.
   - **Memory Storage Node**: Stores successful context retrievals and solutions for future reuse.
   - The agent evaluates and reflects on what worked or failed, using recorded memories (Athena semantic store) to improve future performance on similar issues.

The workflow critically evaluates past actions and outcomes to refine future strategies, implementing learning from mistakes through memory-augmented iteration.

---

## Agent Planning

**Labels:** HIERARCHICAL, REPLANNING, RETRIEVAL-AUGMENTED-PLANNING

**Justification:**

Prometheus demonstrates three complementary planning strategies:

1. **Hierarchical Planning**: The system decomposes high-level goals into increasingly detailed sub-tasks:
   - Top-level: Issue classification determines the overall pipeline (bug/feature/question/documentation)
   - Mid-level: Within each pipeline, subgraphs break down into specialized phases (context retrieval → reproduction → resolution)
   - Low-level: Each subgraph further decomposes into granular node operations (query refinement → memory retrieval → KG traversal → extraction)
   
   This creates a hierarchical tree of tasks as evidenced by the nested subgraph structure in `prometheus/lang_graph/`.

2. **Replanning**: The system dynamically adjusts plans when conditions change:
   - **Bug Reproduction Subgraph**: Contains retry logic with `git_reset_node` that reverts changes and attempts new approaches when reproduction fails, making surgical adjustments to fix broken steps.
   - **Issue Verified Bug Subgraph** (`prometheus/lang_graph/subgraphs/issue_verified_bug_subgraph.py`): Adjusts patch generation strategy based on test failures, creating new plans from feedback rather than rigidly following initial approach.
   - Conditional edges in LangGraph allow adaptation to deviations or errors during execution.

3. **Retrieval-Augmented Planning**: The agent finds similar problems solved in the past and uses them as templates:
   - **Athena Memory System** (`prometheus/utils/memory_utils.py`): Retrieves previously successful plans and contexts for similar queries.
   - **Memory-First Strategy**: The Context Retrieval Subgraph tries semantic memory before falling back to knowledge graph traversal, reusing successful old plans as guides for current situations.
   - The `retrieve_memory` function queries historical solutions to inform current planning decisions.

---

## Agent Reasoning

**Labels:** CHAIN-OF-THOUGHT, REACT, TOOL-COMPOSITION, SELF-EVALUATION, RETRIEVAL-AUGMENTED-PLANNING

**Justification:**

Prometheus implements multiple sophisticated reasoning techniques:

1. **Chain-of-Thought**: The agent breaks down complex problems into explicit logical steps:
   - Bug analysis in `IssueBugAnalyzerNode` generates structured hypotheses about root causes before patch generation.
   - Context refinement in `ContextRefineNode` structures queries into essential_query, extra_requirements, and purpose components.
   - The system clearly lists intermediate reasoning steps throughout its workflow.

2. **ReAct (Reasoning + Acting)**: The agent interleaves thought, action, and observation in continuous cycles:
   - **Bug Reproduction**: Thinks about reproduction strategy → Writes reproduction test (action) → Executes in Docker (observation) → Thinks about results → Adjusts approach.
   - **Context Provider Node** (`prometheus/lang_graph/nodes/context_provider_node.py`): Uses LLM with Neo4j traversal tools, reasoning about what to query next based on previous tool outputs.
   - The ToolNode pattern throughout the codebase shows tight integration of reasoning with tool execution and observation-based iteration.

3. **Tool-Composition**: The agent chains multiple tools in sequence and parallel to accomplish complex tasks:
   - **Graph Traversal Tools** (`prometheus/tools/graph_traversal.py`): Provides 15+ specialized tools for FileNode, ASTNode, and TextNode retrieval that are composed to navigate the knowledge graph.
   - **File Operation Tools** (`prometheus/tools/file_operation.py`): Read, create, edit, and delete operations are strategically coordinated.
   - **Container Command Tools**: Execution tools are combined with file operations for comprehensive patch testing.
   - The Context Retrieval workflow strategically coordinates memory retrieval → KG traversal → extraction tools in sequence.

4. **Self-Evaluation**: The agent validates and scores the quality of its outputs:
   - **Bug Reproduction Structured Node**: Evaluates whether bug was successfully reproduced based on test results.
   - **Final Patch Selection Node** (`prometheus/lang_graph/nodes/final_patch_selection_node.py`): Generates multiple patch candidates and selects the best one based on test pass/fail results.
   - **Multi-level validation**: Patches are tested against reproduction tests, regression tests, and existing test suites with confidence measures applied.

5. **Retrieval-Augmented Planning** (also a reasoning technique): The agent recalls similar problems from memory and adapts known successful strategies, integrating historical knowledge into its reasoning process through the Athena semantic memory system.

---

## Agent Memory

**Labels:** PERSISTENT, EPISODIC, SEMANTIC, VECTOR-DB, GRAPH-STRUCTURED, SHARED-MEMORY

**Justification:**

Prometheus implements a sophisticated multi-tiered memory architecture:

1. **Persistent Memory**: The Athena semantic memory service (`prometheus/utils/memory_utils.py`) stores knowledge that endures across multiple sessions, enabling long-term continuity. The system maintains memory in a separate service accessible via HTTP API with `store_memory` and `retrieve_memory` functions.

2. **Episodic Memory**: The agent remembers specific past events with context:
   - Memory storage includes not just what was retrieved but also the query context (essential_query, extra_requirements, purpose) and when it occurred.
   - The system maintains a log of interactions through repository_id-indexed storage, enabling recall of specific problem-solving episodes.

3. **Semantic Memory**: The agent stores structured conceptual knowledge:
   - **Athena Memory Client** (`AthenaMemoryClient` class): Uses semantic embeddings to represent and retrieve knowledge based on meaning rather than exact matches.
   - Queries are embedded and matched against stored contexts using semantic similarity.
   - The system stores facts and relationships about code entities, enabling sophisticated reasoning about conceptual connections.

4. **Vector-DB**: Information is converted into dense numerical embeddings:
   - The Athena service performs vector similarity search to find conceptually similar contexts even when exact words differ.
   - The `retrieve_memory` function uses vector-based retrieval to match queries against stored memories by semantic meaning.

5. **Graph-Structured Memory**: The Neo4j knowledge graph (`prometheus/graph/knowledge_graph.py`) stores information as a connected web:
   - **Nodes**: FileNode (files/directories), ASTNode (tree-sitter syntax nodes), TextNode (text chunks)
   - **Edges**: HAS_FILE (directory hierarchy), HAS_AST (file-to-AST), HAS_TEXT (file-to-text), PARENT_OF (AST structure), NEXT_CHUNK (text sequence)
   - This graph structure enables the agent to trace complex connections and follow chains of thought through codebase relationships.
   - The knowledge graph is persisted in Neo4j as documented in the README.md.

6. **Shared-Memory**: Multiple agents (context retrieval, bug reproduction, issue resolution) read from and write to the centralized Neo4j knowledge graph and Athena semantic memory, enabling coordination and knowledge sharing. The LangGraph state is also shared across agents and can be checkpointed in PostgreSQL as noted in the README.md.

---

## Agent Tool

**Labels:** FILE-MANAGEMENT, CODE-EDITING, STRUCTURAL-RETRIEVAL, KNOWLEDGE-GRAPH, VERSION-CONTROL, PYTHON-TOOLS, TESTING-TOOLS, DATABASE-TOOLS, SYSTEM-UTILITIES, TEXT-PROCESSING, SHELL-SCRIPTING, CONTAINER-TOOLS, WEB-TOOLS

**Justification:**

Prometheus provides an extensive toolkit across multiple capability domains:

1. **FILE-MANAGEMENT**: Basic file system operations from `prometheus/tools/file_operation.py`:
   - Navigation and inspection: File path resolution, directory traversal
   - Manipulation: Read, create, delete operations on files and directories
   - All operations support relative path handling from repository root

2. **CODE-EDITING**: Specialized tools for modifying source code:
   - **edit_file**: String-based content replacement with exact matching and occurrence validation
   - **create_file**: New file creation with automatic parent directory creation
   - **read_file_with_line_numbers**: Line-numbered code viewing with configurable ranges
   - Tools enforce relative path usage and prevent duplicate content issues

3. **STRUCTURAL-RETRIEVAL**: Advanced code comprehension via `prometheus/tools/graph_traversal.py`:
   - **Symbol-based searches**: find_file_node_with_basename, find_file_node_with_relative_path
   - **AST-based analysis**: find_ast_node_with_text_in_file, find_ast_node_with_type_in_file
   - **Reference tracking**: Tree-sitter node type searches (function_definition, class_definition)
   - **Pattern matching**: Text-based searches within AST nodes
   - All operations leverage tree-sitter parsing for syntax-aware retrieval

4. **KNOWLEDGE-GRAPH**: Graph-based code representation via Neo4j:
   - The `KnowledgeGraph` class in `prometheus/graph/knowledge_graph.py` constructs and queries structured graphs
   - Represents entities (FileNode, ASTNode, TextNode) and relationships (HAS_FILE, HAS_AST, PARENT_OF, HAS_TEXT, NEXT_CHUNK)
   - Enables semantic understanding through graph traversal for code comprehension
   - The system uses Neo4j with APOC plugins for advanced graph operations as documented in README.md

5. **VERSION-CONTROL**: Git-based operations from `prometheus/git/git_repository.py`:
   - **GitDiffNode**: Creates git diffs for patches
   - **GitResetNode**: Reverts changes for retry loops
   - Patch management and version history inspection capabilities
   - Supports playground/working directory separation for safe experimentation

6. **PYTHON-TOOLS**: Python development environment (implied from codebase):
   - Python interpreters and package managers (pip, pip3)
   - The system is built on Python 3.11+ as documented in README.md
   - Package dependencies managed via pyproject.toml

7. **TESTING-TOOLS**: Test execution and validation:
   - **Container-based test execution**: The `BaseContainer` and `GeneralContainer` classes (`prometheus/docker/`) run tests in isolated Docker environments
   - **BugReproducingExecuteNode**: Executes reproduction tests with configurable test commands
   - **RunExistingTestsSubgraph**: Validates patches against existing test suites
   - Support for build verification and regression testing

8. **DATABASE-TOOLS**: Database clients for data persistence:
   - **Neo4j**: Graph database for knowledge graph storage (port 7474/7687)
   - **PostgreSQL**: Relational database for LangGraph state checkpointing (port 5432)
   - Both documented in README.md with connection details

9. **SYSTEM-UTILITIES**: Operating system and process management:
   - Container execution environment provides shell access
   - Environment variable management for configuration
   - Process execution through Docker containers

10. **TEXT-PROCESSING**: Text transformation capabilities:
    - **TextNode chunking**: Text files are split into chunks with configurable size and overlap
    - **Line number prepending**: The `pre_append_line_numbers` utility formats code with line numbers
    - String manipulation and encoding support

11. **SHELL-SCRIPTING**: Shell command execution:
    - **ContainerCommandTool** (`prometheus/tools/container_command.py`): Executes arbitrary shell commands in Docker containers
    - Bash interpreter available in containerized environment
    - Command execution with output capture

12. **CONTAINER-TOOLS**: Docker-based isolation:
    - **BaseContainer** class (`prometheus/docker/base_container.py`): Manages Docker container lifecycle
    - **GeneralContainer**: General-purpose test environment
    - **UpdateContainerNode**: Synchronizes code changes to container
    - Containerized build and test execution for safety and reproducibility

13. **WEB-TOOLS**: Network communication utilities:
    - HTTP clients for Athena memory service communication (`requests` library)
    - REST API integration for semantic memory operations
    - The system itself provides a web API via FastAPI/Uvicorn on port 9002 as documented in README.md

---

## Summary

Prometheus represents a **sophisticated multi-agent system** that combines:

- **Architecture**: Hierarchical and collaborative multi-agent coordination
- **Workflow**: Iterative feedback loops with self-improving learning from experience
- **Planning**: Hierarchical decomposition with dynamic replanning and retrieval-augmented strategies
- **Reasoning**: Chain-of-thought analysis, ReAct integration, tool composition, and self-evaluation
- **Memory**: Multi-tiered persistent, episodic, and semantic memory using vector databases and graph structures
- **Tools**: Comprehensive toolkit spanning file operations, code analysis, knowledge graphs, version control, testing, databases, and containerization

The system is designed for **production-ready autonomous software engineering** with strong emphasis on context-aware reasoning through unified knowledge graphs (Neo4j) and long-term semantic memory (Athena). Its multi-agent architecture enables specialized expertise while maintaining coordinated workflows through LangGraph state machines.

---

## References

- Repository: https://github.com/EuniAI/Prometheus
- Research Paper: https://arxiv.org/abs/2507.19942
- Multi-Agent Architecture Documentation: `docs/Multi-Agent-Architecture.md`
- Implementation: `prometheus/lang_graph/` (2841+ lines of subgraph code)
