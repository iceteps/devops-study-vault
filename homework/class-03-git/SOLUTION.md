# Solution — Git Fundamentals Assignment

- **Repo:** https://github.com/iceteps/git-python-practice
- **Status:** done 2026-07-15 — core Parts 1–9 (branch, clean merge, a real hand-resolved merge
  conflict) plus the full 11-task bonus advanced-practice section, tag `v1.0.0`.
- `feature/mistake` is deliberately left unmerged locally — it's the artifact proving the
  reset-recovery scenario (Bonus 11A) worked; git refuses to `-d` delete an unmerged branch.

```
* f350df1 Oops - accidental commit on main
* 567daeb Add .gitignore for .env and Python cache files
* 4f4a264 Rebase practice commit 2
* 98596b9 Rebase practice commit 1
* 84b4502 Add header comment on main
* 95577bc Temporary change
* edaa6a5 Revert "Add a comment to app.py"
* 1951add Add a comment to app.py
*   2364600 Merge feature/add-time into main, resolve greeting conflict
|\
| * 32eb863 Change greeting wording on feature/add-time
* | 908e309 Change greeting wording on main
|/
* 163c6a3 Add current time to greeting message
* a27451b created app.py to be able to push to main
```
