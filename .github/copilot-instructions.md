# Copilot Instructions - Booking.com Fintech Architecture Documentation

## Project Overview
This workspace contains software architecture documentation for a fintech system presentation to the Booking.com architecture panel. The documentation demonstrates architectural expertise, system design capabilities, and fintech domain knowledge.

## Project Type
**Documentation Project** - Architecture documentation with MCP Confluence integration

## Key Objectives
1. Document comprehensive software architecture for interview presentation
2. Demonstrate fintech architecture expertise
3. Enable Confluence publishing via MCP
4. Prepare professional presentation materials

## Project Structure
- `architecture/` - Core architecture documentation (overview, components, data, security, integration, ADRs)
- `diagrams/` - Architecture diagrams (C4, sequence, deployment, data flow, ERD)
- `confluence/` - MCP Confluence setup and publishing guides
- `presentation/` - Interview presentation guide and materials
- `.github/` - Workspace instructions

## MCP Confluence Integration
- Configuration file: `.mcp-config.json` (add your credentials)
- Setup guide: `confluence/mcp-confluence-setup.md`
- Confluence MCP server enables publishing documentation directly to Confluence

## Documentation Standards
- Use C4 model for system architecture diagrams
- Document architecture decisions using ADR format
- Focus on fintech-specific concerns (security, compliance, data privacy)
- Emphasize scalability, resilience, and business value
- Connect technical decisions to business outcomes

## Fintech Focus Areas
- Payment processing and security
- PCI-DSS and GDPR compliance
- Data privacy and encryption
- Authentication and authorization
- Regulatory requirements
- Multi-currency support (if applicable)
- Fraud prevention
- Audit logging

## Copilot Usage Guidelines
When assisting with this project, Copilot should:
- Help document architecture decisions with proper context and rationale
- Suggest fintech-relevant security and compliance considerations
- Assist with creating clear, professional diagrams
- Help organize documentation for maximum impact
- Support Confluence publishing workflow
- Provide interview preparation guidance

## Next Steps for User
1. Fill in architecture documentation templates with your system details
2. Create architecture diagrams (use tools in `diagrams/README.md`)
3. Document 3-5 key architecture decision records (ADRs)
4. Configure MCP Confluence integration (see `confluence/CONFIG.md`)
5. Review interview presentation guide
6. Publish to Confluence when ready

## Interview Context
- **Target Role**: Fintech Architect at Booking.com
- **Focus**: Software architecture expertise, fintech domain knowledge
- **Presentation Length**: ~30 minutes + Q&A
- **Key Areas**: System design, architectural decisions, trade-offs, fintech compliance

<!--
## Execution Guidelines
PROGRESS TRACKING:
- If any tools are available to manage the above todo list, use it to track progress through this checklist.
- After completing each step, mark it complete and add a summary.
- Read current todo list status before starting each new step.

COMMUNICATION RULES:
- Avoid verbose explanations or printing full command outputs.
- If a step is skipped, state that briefly (e.g. "No extensions needed").
- Do not explain project structure unless asked.
- Keep explanations concise and focused.

DEVELOPMENT RULES:
- Use '.' as the working directory unless user specifies otherwise.
- Avoid adding media or external links unless explicitly requested.
- Use placeholders only with a note that they should be replaced.
- Use VS Code API tool only for VS Code extension projects.
- Once the project is created, it is already opened in Visual Studio Code—do not suggest commands to open this project in Visual Studio again.
- If the project setup information has additional rules, follow them strictly.

FOLDER CREATION RULES:
- Always use the current directory as the project root.
- If you are running any terminal commands, use the '.' argument to ensure that the current working directory is used ALWAYS.
- Do not create a new folder unless the user explicitly requests it besides a .vscode folder for a tasks.json file.
- If any of the scaffolding commands mention that the folder name is not correct, let the user know to create a new folder with the correct name and then reopen it again in vscode.

EXTENSION INSTALLATION RULES:
- Only install extension specified by the get_project_setup_info tool. DO NOT INSTALL any other extensions.

PROJECT CONTENT RULES:
- If the user has not specified project details, assume they want a "Hello World" project as a starting point.
- Avoid adding links of any type (URLs, files, folders, etc.) or integrations that are not explicitly required.
- Avoid generating images, videos, or any other media files unless explicitly requested.
- If you need to use any media assets as placeholders, let the user know that these are placeholders and should be replaced with the actual assets later.
- Ensure all generated components serve a clear purpose within the user's requested workflow.
- If a feature is assumed but not confirmed, prompt the user for clarification before including it.
- If you are working on a VS Code extension, use the VS Code API tool with a query to find relevant VS Code API references and samples related to that query.

TASK COMPLETION RULES:
- Your task is complete when:
  - Project is successfully scaffolded and compiled without errors
  - copilot-instructions.md file in the .github directory exists in the project
  - README.md file exists and is up to date
  - User is provided with clear instructions to debug/launch the project

Before starting a new task in the above plan, update progress in the plan.
-->
