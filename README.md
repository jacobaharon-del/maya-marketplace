# Maya — Talent Sourcing (team plugin marketplace)

This repo is a Claude plugin marketplace. It ships **Maya**, the recruiting
team's LinkedIn Recruiter sourcing agent, so every teammate installs the same
version and gets updates from one place.

## What's in here

```
maya-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # lists the plugins in this marketplace
└── plugins/
    └── maya-sourcing/
        ├── .claude-plugin/
        │   └── plugin.json        # plugin manifest (name + version)
        └── skills/
            └── maya-sourcing/     # the Maya skill itself
                ├── SKILL.md
                └── references/
                    └── linkedin-recruiter.md
```

## For teammates — install once

In Cowork (or Claude Code), run:

```
/plugin marketplace add jacobaharon-del/maya-marketplace
/plugin install maya-sourcing@maya-marketplace
```

After that, Maya triggers on things like "open a new role", "source candidates
for…", or "build a shortlist". Each teammate still needs:
- their **Notion** connector connected, with the shared "Maya — Talent Sourcing"
  pages shared to their account, and
- their own **LinkedIn Recruiter** seat logged in (Maya never enters credentials).

## For the maintainer (Jacob) — ship an update

1. Edit the skill under `plugins/maya-sourcing/skills/maya-sourcing/`.
2. **Bump the `version`** in `plugins/maya-sourcing/.claude-plugin/plugin.json`
   (e.g. `1.0.0` → `1.0.1`). This is the cache key — without a bump, teammates
   won't pull the change.
3. Commit and push.
4. Tell the team to run:

```
/plugin update maya-sourcing
```

## Push this repo (first time)

```
git remote add origin https://github.com/jacobaharon-del/maya-marketplace.git
git branch -M main
git push -u origin main
```

There's no fully-silent auto-update today, so teammates run that one command
when you notify them of a new version.
