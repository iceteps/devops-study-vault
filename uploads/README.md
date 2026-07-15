# 📤 uploads — the teacher's materials, class by class

Everything Yariv uploads (slide decks, assignment docs) lands here, one folder per
drive upload, **named to mirror his drive-folder numbering + the topic**. Heads-up:
his numbers drift from the real calendar (the "class 3" Git deck's title slide says
**Class 6**) — the **topic suffix is the real key**, the number is just a label:

```
uploads/
├── class-02-docker/     class2-docker.pptx · docker-assignment-1.docx
├── class-03-git/        class3-git.pptx · git-assignment-branching-merging-conflicts.docx
├── class-04-kubernetes/ class4-kubernetes.pptx · kubernetes-basics-cli-assignment.docx
└── class-NN-<topic>/    ← next class goes here
```

> Official course repo (the teacher's): https://github.com/yfreifeld/devops-course

## The weekly ritual (after each class)

1. Download whatever Yariv uploaded to the drive.
2. Create `uploads/class-NN-<topic>/` and drop the files in — **rename them clean**
   (lowercase, dashes, no emoji/spaces) so links and git stay happy.
3. In the matching class note, add/update:
   - **📬 The REAL assignment** — the graded homework, distilled step-by-step.
   - **🖥️ From the class deck** — anything the slides teach that the note didn't cover.
4. Put your solution work under `homework/class-NN-<topic>/`.
5. Commit + push.

Every mined deck/assignment so far is already summarized inside its class note —
the files here are the originals for reference.
