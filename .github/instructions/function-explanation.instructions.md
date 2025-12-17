# Functional Explanation Instructions

## Explanation Style
When asked to explain a system feature, please follow the instruction below:
1. Provide an overall overview of the feature.
2. Explain the processing flow step by step in detail.
3. Each step must include:
   - Step number and subject title
   - A detailed description of the functionality of that step
   - Corresponding source file(s): file paths should use relative paths。
     - Format: [`ProjectName/path/fileName`]
     - If there are multiple files, separate them with commas
     - For example: [`ProjectA/src/components/Component1.vue`]
   - Corresponding code snippet(s) (use code blocks)
   - Explanation of the code
   - Relevant log records (if any)
   - Relevant data encryption/decryption handling (if any)
   - **Note**：The code snippet for each step must be complete; partial code is not allowed.
   - **Note**：If a code snippet is too long, split it into multiple sections, and each section must contain complete code logic.

4. Flow Diagrams
   - If there are flow diagrams, please attach them at the end of the explanation.
   - The flow diagrams should clearly show the relationships and flow between each step.
   - The flow diagrams should use simple and clear graphics and text labels for easy understanding.
   - Use Mermaid syntax to generate sequence diagrams.
   - Use Mermaid syntax to generate flowcharts.
 - Sequence Diagram
 ```mermaid
   sequenceDiagram
       participant User
       participant System
       User->>System: Add Task
       System-->>User: Task Added
       User->>System: Delete Task
       System-->>User: Task Deleted
       User->>System: View Task
       System-->>User: Return Task List
   ```
- Flowchart 
   ```mermaid
   graph TD
       A[Start] --> B{Decision}
       B -- Yes --> C[Execute]
       B -- No --> D[End]
   ```