# Project - claude-para-llm-wiki

## Objectives
- Following the general guidelines in[[Building a Complete Personal Harness LLM Wiki + Developer’s Second Brain in Obsidian]]
- In place of `raw/` folder for immutable documents, I want to have a top-level structure following the PARA methods.  
- I will have 2 vaults like this.  One for daily personal use and one for work use.  They will follow basically the same structure.
- I am a Principal Architect for a SaaS company.  I am engaged with multiple projects and the overall architecture for the company.  Outside of work, I have personal development projects and my own pursuits as a Father of 8. 
- In both scenarios, Projects and Areas are very important since I will be moving through multiple in any given day.  My role (Area) could be very different for some things, and I want my raw content to reflect that. 
- I also have daily workflows that involve capturing tasks and meeting notes from other tool output.  For this process to work, I need to have a drop folder in the PARA structure. 
- I prefer the top level folders have numeric prefixes to sort them lexically outside of Obsidian.  Here's a suggested starting set:  00-Inbox, 01-Projects, 02-Areas, 03-Resources, 04-Archives, 05-Daily (kept organized by Year/Month done by PARA Ingest workflow)
- I will have "inbox" ingestion workflows that route content to the appropriate PARA structure.  These will always run before wiki ingestion to ensure the most recent items are handled.
- For the `dev/` items discussed in the article, I want to map these instead to 01-Projects/{project-name}/ADRs or Incidents.  
- I would like `wiki/` to be housed in the PARA tree in a folder like 90-Wiki.  This leaves room for other top level folders if they are needed.  
- A daily agent log should be written to our appended in the wiki path to record what was done.
- I would like to have all templates stored in an organized fashion in top level `_Templates` path.

### Some additional thoughts
- Definitely want to leverage the Obsidian skills where possible.
- Prefer Obsidian MCP that doesn't require Obsidian to be open
- Part of the wiki ingest process should also create/update a hot cache file that can be read in by other skills to return results quicker. 
- I would like to implement QMD for search with a sensible fallback so that the agents don't have to constantly read content to find answers and process file. 
- While providing skills and CLAUDE.md rules for allowed tool access is reasonable, I also want to make sure that we put explicit hooks in place to ensure that reinforce what's ok and what isn't.  
- Also we need hooks to reinforce the correct tool patterns.  If agent attempts to use grep to search vault content, the hook should inject the correct tool (QMD) to use along with syntax links as help.  Agent trying to use the file tool should be prompted to use Obsidian Skills, MCP or CLI.  
- Need to leverage a community prompt injection filter to protect against external contents doing anything nefarious.  

### One Vault, Multiple Git Projects
- I have tried using `docs/` approach in each code project.  Main issues I have found with this is that I cannot consolidate the information created and discovered during a coding session in alignment with other docs I may have in a central vault.  
- In each project, I want to be able to set a VAULT_BASE_PATH and VAULT_PROJECT_PATH variables.  VAULT_BASE_PATH is a full path to the vault.  VAULT_PROJECT_PATH is the location of the project relative to the base path - 01-Projects/my-project-name.  
- This needs to be understood by all skills, commands and hooks to ensure that information can be processed and created in the correct folders.  

### Must Haves
- vault-init skill / command that will be used to scaffold the initial vault.  This should also create a .obsidian file that includes the plugins and settings to make the vault function properly. 
- para-ingest skill / command that consists of local routing rules in markdown to help the agent move inbox content to the correct PARA folders.  This skill needs to delegate handling of specific content to other skills.  For example if the inbox has an HTML document >> defuddle() or to_markdown() skill.  For an audio transcript >> transcribe to markdown. 
- modified versions of all of the wiki skills mentioned in the article to meet the objectives outlined.  

### How to proceed
- Use plan mode and create a detailed plan.  Write it to project-plan.md.  
- Break all of the work down into sequenced tasks.  Each task should be broken down so that one developer agent can complete it.  Identify dependencies and agent profiles to use for each task.  Write this file to project-tasks.md.
- When we implement this, I want to use a team of 4 agents.  Project Manager, Skill Developer, Python Developer and Reviewer.  
- 