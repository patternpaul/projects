# Projects Directory - Working Memory System

## Purpose

This directory serves as Claude's persistent working memory system to maintain continuity across sessions, after /clear commands, or when conversation context is compacted. Each project folder represents a specific feature, fix, or investigation that can be worked on independently.

## Why This Exists

When working with Claude across multiple sessions, branches, or terminals:

- **Multi-Branch Support**: Different projects can be active on different git branches
- **Parallel Work**: Multiple terminal sessions can work on different projects simultaneously
- **Session Continuity**: Work continues exactly where it left off after /clear or timeout
- **Progress Tracking**: Complex multi-step tasks need persistent tracking per project
- **Knowledge Retention**: Important discoveries and patterns are preserved per project

## Project Index

### Active Projects

| Folder Name | Project Name | Description | Status | Last Updated |
| ----------- | ------------ | ----------- | ------ | ------------ |
| request-routing-detector | Request Routing Detector | Implement intelligent routing between Drive and Legacy systems | Active | 2025-08-27 |

### How to Reference Projects

When starting a new session with Claude, you can say:

- "Let's work on claude-session-improvements"
- "Continue the space-tracking-packing project"
- "I want to work on the Claude Session Improvements project"

Claude will then:

1. Look up the project folder from this index
2. Check the project's TODO.md for current status
3. Review context.md for important information
4. Continue exactly where the work left off

## Directory Structure

```
projects/
├── README.md                  # This file - master index
└── [project-folder-name]/     # Unique project folders
    ├── README.md             # Project documentation
    ├── TODO.md               # Current tasks and progress
    ├── context.md            # Important context to preserve
    ├── references.md         # Quick reference materials
    ├── research/             # Research and analysis
    ├── before-after/         # Performance comparisons
    └── notes.md              # Additional notes
```

## Creating a New Project

When Claude starts work on a new feature or task:

1. Create a uniquely named folder (kebab-case)
2. Add entry to this index with description
3. Create TODO.md with initial task breakdown
4. Create context.md with relevant information
5. Update this README.md with the new project

## Project Lifecycle

- **Active**: Currently being worked on
- **In Progress**: Partially complete, may resume later
- **Ongoing**: Long-term project with periodic updates
- **Completed**: Finished, kept for reference
- **Archived**: No longer relevant but preserved

## For Developers

This directory is managed by Claude to maintain work continuity. The files here contain:

- Task tracking and progress updates per project
- Important technical context and discoveries
- Performance metrics and comparisons
- Decision rationales and trade-offs
- Quick reference materials for common patterns

Feel free to review these files to understand Claude's work process and current state of each project.

## Quick Start for New Sessions

1. Check this index to see available projects
2. Tell Claude which project to work on
3. Claude will load the project context and continue
4. Work resumes exactly where it left off
