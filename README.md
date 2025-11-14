# Rails Orchestrator

Rails Orchestrator is an autonomous system for planning and executing programming tasks based on Ruby on Rails 8+. It serves as a secure server for the Model Context Protocol (MCP), enabling integration of AI tools with software development processes.

## Key Ideas of the System:

- **Autonomous Task Planning** - The system automatically analyzes user ideas and transforms them into detailed implementation plans through a multi-stage processing pipeline, from business analysis to implementation graph generation.

- **Multi-Layer Quality Validation** - Built-in quality control mechanisms include automated tests, security analysis, syntax checking, code linting, and test coverage validation, ensuring high quality of delivered software.

- **Background Task Orchestration** - Uses SolidQueue for automatic background task processing, enabling continuous development cycles without manual intervention, with automatic transitions between task states.

- **User Requirements Analysis** - The system processes user stories, detects gaps in acceptance criteria, and automatically generates system tests and implementation scope analyses.

- **MCP Tools for AI** - Provides a rich set of MCP tools enabling AI models to perform specific actions, from project analysis to running tests and code reviews.

- **Project Documentation Management** - Stores and manages project documentation in the database, allowing planning agents access to specifications, requirements, and architecture.

- **Frontend Contracts** - The system validates frontend requirements for Rails 8+ applications with Stimulus and Tailwind, preventing user interface degradation.

- **Security and Reliability** - Implements multi-layer protection against data loss, automatic error detection, and recovery mechanisms after failures.

## How the System Works (Diagram):

```
┌─────────────┐
│ User Idea   │
└─────────┬───┘
          ↓
┌─────────────────────┐
│ Analysis & Planning │
└─────────┬───────────┘
          ↓
┌───────────────────────────┐
│ Automatic Task Execution  │
└─────────┬─────────────────┘
          ↓
┌───────────────────────────┐
│ Quality Checks & Testing  │
└─────────┬─────────────────┘
          ↓
┌──────────────────┐
│ Finished Product │
└──────────────────┘
```

The system focuses on automating the entire software project lifecycle, from idea to working application, with emphasis on quality, security, and development process efficiency.
