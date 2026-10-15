# bug-hunt-workflow

A Claude Code skill for finding bugs in a big code base: many fresh "hunter" agents each read one slice of the code, then a separate "skeptic" agent tries to disprove every bug that was reported. Only the survivors are kept.

Based on Anthropic's published "dynamic workflows" test (70 bugs hidden in 116,000 lines; one agent found 14-27, a workflow found 66). **Status: not yet tested on real code.**

## Install
Copy the `bug-hunt-workflow` folder into `~/.claude/skills/` (Windows: `C:\Users\<you>\.claude\skills\`). Then ask Claude Code to "bug hunt this code" or "use a workflow to review this".

Note: it spends a lot of tokens (many agents), so it only runs when you ask for a thorough review.
