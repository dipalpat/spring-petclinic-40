# Modernize to Spring Boot 4 - Quick Reference

## Usage

"Modernize my app to Spring Boot 4"
- Source: `./legacy-app`
- Target: `./modern-app`

## Workflow

```
Phase 1: Extract → Phase 2: Translate (one story per context)
```

## Phases

**Phase 1: Code to Spec**
- Invoke: `tanzu-extract-story-detail` skill
- Output: `spec/` folder with user stories, technical context, test scenarios
- Duration: One-time upfront scan

**Phase 2: Spec to Boot**
- Invoke: `tanzu-rewrite-story` skill (one story per new context)
- Output: Spring Boot 4 application in target folder
- Incremental: User selects each story via AskQuestion

## Required Inputs

- Source code location (folder/repo to analyze)
- Target folder (where to write Spring Boot 4 app)
- Optional: Spec folder location (default: `./spec`)

## Key Files

- Main skill: /tanzu-modernize-app
- Code to Spec: /tanzu-extract-story-detail
- Spec to Boot: /tanzu-rewrite-story
