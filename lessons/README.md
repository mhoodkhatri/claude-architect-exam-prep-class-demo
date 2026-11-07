# Lessons

One lesson per night, written by Claude Code via the **Daily Lesson** workflow (`.github/workflows/daily-lesson.yml`).

- Files are named `YYYY-MM-DD-topic-slug.md` (date in Pakistan time).
- Topics come from `../syllabus.md`, in order.
- Each lesson arrives as an auto-merging pull request that has to pass the Markdown-only check.

To get a lesson right now, open the **Actions** tab → **Daily Lesson** → **Run workflow**.

## One-time setup: the `LESSON_BOT_TOKEN` secret

The workflow opens its pull requests with a personal access token, so GitHub treats them as yours and runs the required check without asking for approval.

1. Go to GitHub → profile picture → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
2. Name it `lesson-bot`. Set an expiration you're comfortable with (for example, 90 days).
3. **Repository access:** *Only select repositories* → this repo.
4. **Permissions → Repository permissions:**
   - **Contents:** Read and write
   - **Pull requests:** Read and write
5. Click **Generate token** and copy it.
6. In this repo: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**. Name: `LESSON_BOT_TOKEN`, Secret: paste the token.

When the token expires, generate a new one and update the secret.
