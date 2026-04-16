# data/

This directory stores filer data files generated during the tax refund flow.

```
data/
├── .gitkeep              # Keeps this folder tracked in git
├── README.md             # This file
└── <id>/                 # One directory per filer, named after the Israeli ID number
    ├── info.md           # Collected tax data (income, bank account, tax credits, etc.)
    ├── session.md        # Login session state (written by the login skill)
    ├── submission.md     # Form 135 submission state (written by the form-fill skill)
    └── *.pdf             # Supporting documents (Form 106, Form 867, IDF certificates, etc.)
```

Each `<id>/info.md` file is created by the `collect-info` skill and contains the collected tax information used to fill Form 135.

**This directory is gitignored** — its contents are private and should never be committed.
