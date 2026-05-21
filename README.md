## Introduction
This is a compilation of SKILL set on frappe apps to be feeded for AI agents

## How is it sourced ??
https://github.com/OpenAEC-Foundation/Frappe_Claude_Skill_Package.git is a package that trains AI agents with Frappe Skills

Where ever your project is opened in the AI Agent like Antigravity
```bash
mkdir -p .agents/skills/
```

```bash
git clone https://github.com/OpenAEC-Foundation/Frappe_Claude_Skill_Package.git
cp -r Frappe_Claude_Skill_Package/skills/source/* .agents/skills/
rm -rf Frappe_Claude_Skill_Package
```

## On successful addition of Claude Skills, Feed our skills

```bash
git clone https://github.com/vinodkumarkolli/ai_agent_skills.git
cp -r ai_agentic_skills .agents/skills/
rm -rf ai_agentic_skills
```
