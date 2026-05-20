---
reviewer: "ask-peer"
review_iterations: 3
check_commands:
  - "jq empty .claude-plugin/marketplace.json plugins/*/.claude-plugin/plugin.json"
test_commands:
  - "Skill(run-tests)"
  - "Skill(verify-bundle-sync)"
hooks:
  on_complete:
    - "Skill(skill-review)"
    - "Skill(verify-diff)"
    - "Skill(publicity-review)"
self_retrospective:
  feedback: "SonicGarden/dev-workflow-issues"
compact_rules: true
---
