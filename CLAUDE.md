# BookReview CLAUDE.md

## Add Finish Book Task

Add a completed book entry to the README.md book list table and open a pull request.

**Required parameters:**
- `ISSUE_NUMBER` — GitHub issue number (e.g. 1495)
- `IMAGE_LINK` — Aladin cover image URL (e.g. `https://image.aladin.co.kr/product/.../cover500/....jpg`)

### Steps

1. **Fetch issue details** using `gh` CLI:
   ```
   # Issue title, open date (= project start date), and body
   gh issue view {ISSUE_NUMBER} --repo jongfeel/BookReview --json title,createdAt,body

   # Latest sub-issue close date (= project end date)
   gh issue list --repo jongfeel/BookReview --milestone "{issue title}" --state closed --json number,closedAt --limit 1 --order desc
   ```
   - Extract: issue title, `createdAt` (start date), latest `closedAt` among sub-issues (end date)
   - Extract the Aladin book link (`aladin.kr/p/...`) from the `body` field

2. **Create and check out a branch**
   ```
   gh issue develop {ISSUE_NUMBER} --repo jongfeel/BookReview --checkout
   ```

3. **Determine the target year table** from the start date year (e.g. start 2025-08-25 → "Book list of 2025")

4. **Insert into the correct year table** in `README.md`
   - The table has **two books per row** (columns: Title | Duration | History | Title | Duration | History)
   - Books are ordered by end date **descending**, left-to-right then top-to-bottom
   - **Step A — add new row at column position 1 (right):**
     ```
     |  |  |  | [![{title}]({IMAGE_LINK})]({aladin_link}) | {start} to {end} | [IssuesLink](https://github.com/jongfeel/BookReview/issues/{ISSUE_NUMBER}) |
     ```
     Insert this new row at the top of the table.
   - **Step B — next task: fill column position 0 (left) of that same row** (no new row):
     Replace the leading `|  |  |  |` with `| [![{title}]({IMAGE_LINK})]({aladin_link}) | {start} to {end} | [IssuesLink](.../{ISSUE_NUMBER}) |`
   - Increment the book count in the table heading by 1

5. **Commit**
   ```
   git commit -am "Add finish book: {issue title}"
   ```

6. **Push**
   ```
   git push
   ```

7. **Create pull request**:
   ```
   gh pr create --title "Add finish book: {issue title}" --body "..."
   ```
   PR body should include:
   - Book title, duration, table placement rationale
   - `Closes #{ISSUE_NUMBER}`

### Notes

- The 2026 table heading uses `(N book)` (singular form); 2025 and older use `(N books)`.
- `gh` commands work without extra setup after running `gh auth login` and selecting "Authenticate Git with your GitHub credentials".
