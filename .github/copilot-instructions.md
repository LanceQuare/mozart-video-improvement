# AI Rules for SCCS Backend

This is a project for security company, user is able to manage security systems via web and mobile apps. The backend is built with C# .NET 6/8, using RESTful APIs to communicate with React 17 frontends. User is able to schedule tasks, view live video and playback videos, and receive alerts.

## CODING_PRACTICES

### Guidelines for SUPPORT_LEVEL

#### SUPPORT_EXPERT

- Favor elegant, maintainable solutions over verbose code. Assume understanding of language idioms and design patterns.
- Highlight potential performance implications and optimization opportunities in suggested code.
- Frame solutions within broader architectural contexts and suggest design alternatives when appropriate.
- Focus comments on 'why' not 'what' - assume code readability through well-named functions and variables.
- Proactively address edge cases, race conditions, and security considerations without being prompted.
- When debugging, provide targeted diagnostic approaches rather than shotgun solutions.
- Suggest comprehensive testing strategies rather than just example tests, including considerations for mocking, test organization, and coverage.


### Guidelines for DOCUMENTATION

#### DOC_UPDATES

- Update relevant documentation in /docs when modifying features
- Keep README.md in sync with new capabilities
- Maintain changelog entries in CHANGELOG.md


## Project Context
- **Framework:** C# .NET 6/8, React 17

---

## General Guidelines
1. **Documentation & Comments**  
   - Use `///` doc comments on public classes/methods.  
   - Explain *why* vs *what*.  
   - Update `/README.md` and `/CHANGELOG.md` on new features.

2. **Soruce Code **
   - `src/core-frontend` – Web frontend
   - `src/legacyapi`    – Mircoservice for apis
   - `src/video`      – Mircoservice for video handling

---

## Charts and Documentation

### Mermaid Chart Guidelines
- **Chart Types Usage**:
  - **Flowcharts**: Decision paths and process flows
    ```mermaid
    flowchart LR
      Start --> Process --> Decision{Evaluate}
      Decision -->|Yes| ResultA
      Decision -->|No| ResultB
    ```
  - **Sequence Diagrams**: Component interaction sequences
    ```mermaid
    sequenceDiagram
      Client->>Server: Request data
      Server->>Database: Query
      Database-->>Server: Return results
      Server-->>Client: Response
    ```
  - **Class Diagrams**: Structure and relationships
    ```mermaid
    classDiagram
      ClassA --|> AbstractClass
      ClassB --|> AbstractClass
      ClassA -- ClassC : uses
    ```
  - **State Diagrams**: State transitions
  - **Entity Relationship Diagrams**: Data structures

- **Chart Best Practices**:
  - Clear direction setting (LR horizontal/TB vertical)
  - Appropriate node density, group related elements
  - Consistent and meaningful color coding
  - Legend for complex charts
  - High contrast for readability

### Documentation Practice Essentials
- **Decision Records**: Use ADR format, including background, decision, and consequences
- **Reflection Documents**: Record design evolution and rejected alternatives
  - **Example-Driven**: Illustrate principles with actual code examples

---

