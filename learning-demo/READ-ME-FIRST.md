# Responsive lomda kit — after extracting

You extracted this zip into the **root of your lomda project** (the folder
that contains `index.html` or your app folder). The four files landed in the
right places automatically:

```
your-lomda-project/
├── CLAUDE.md                    ← auto-loaded by Claude Code, no action needed
├── RESPONSIVE-GUIDELINES.md     ← reference doc for humans
├── READ-ME-FIRST.md             ← this file (delete after reading)
├── .githooks/
│   └── pre-commit               ← blocks bad commits
└── viewer.html                  ← QA preview (may need to move — see below)
```

## Do these two things now

### 1. Activate the commit hook (once per clone, every teammate)

From the project root:

```
git config core.hooksPath .githooks
```

Verify:
```
git config core.hooksPath
```
Should print `.githooks`. Done — every commit is now checked.

### 2. Place `viewer.html` next to your `index.html`

If your `index.html` lives at the project root, `viewer.html` is already in
the right place. Nothing to do.

If your `index.html` lives inside an app folder (e.g. `app/index.html`),
move `viewer.html` next to it:
```
mv viewer.html app/viewer.html
```

Then open `viewer.html` and update the screen list near the bottom of the
file to match your project's screen names.

## Optional cleanup

- Delete `READ-ME-FIRST.md` — you don't need it in the repo.
- Commit the remaining three files (`CLAUDE.md`, `RESPONSIVE-GUIDELINES.md`,
  `.githooks/pre-commit`) and `viewer.html`.

## If a commit gets blocked later

Read the error message — it names the rule and the offending file+line.
Full fix instructions for each violation type are in
`RESPONSIVE-GUIDELINES.md` and in the team setup guide.

## Design grid different from 1280×710?

If your lomda uses a different design size, search-and-replace `1280` and
`710` in these files: `RESPONSIVE-GUIDELINES.md`, `CLAUDE.md`,
`.githooks/pre-commit`. Do it before your first commit so the hook checks
the right dimensions.
