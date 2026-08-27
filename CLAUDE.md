# BookReview CLAUDE.md

## Add Book TOC Task

Add the book cover image and a table-of-contents checklist to a reading issue when starting a new book.

**Required parameters:**
- `ISSUE_NUMBER` — GitHub issue number (e.g. 1834)

### Steps

1. **Fetch issue body** and extract the Aladin book link (`aladin.kr/p/...` or `aladin.co.kr/shop/wproduct.aspx?ItemId=...`):
   ```
   gh issue view {ISSUE_NUMBER} --repo jongfeel/BookReview --json title,body
   ```
   If the body has no Aladin link, search Aladin by the issue title to find the product page.

2. **Fetch the Aladin product page** with `curl` (use a browser User-Agent) and extract:
   - **Cover image**: the `og:image` meta tag — a `https://image.aladin.co.kr/product/.../cover500/....jpg` URL
     ```
     grep -o 'og:image" content="[^"]*'
     ```
   - **ISBN-13**: `grep -io "isbn[^;]\{0,40\}"` → 13-digit number (e.g. 9791140719914)
   - Note: Aladin's 목차/책소개 sections are loaded dynamically via script, so the TOC **cannot** be scraped from Aladin — that is why Yes24 is used below.

3. **Find the Yes24 product page** by searching with the ISBN:
   - Fetch `https://www.yes24.com/product/search?query={ISBN}` and extract the product URL (`/product/goods/{id}`)

4. **Extract the TOC** from the Yes24 product page HTML (curl + grep; do not rely on a summarized fetch alone — cross-check against the raw HTML):
   ```
   grep -o "[0-9]\{1,2\}부[.: ][^<]\{0,40\}"   # part titles
   grep -o "[0-9]\{1,2\}장[.: ][^<]\{0,80\}"   # chapter titles
   ```

5. **Append to the issue body** (keep the existing body intact):
   - The Aladin cover image if the body doesn't already have a cover image:
     ```
     <img width="500" alt="{title}" src="{og:image URL}" />
     ```
   - A `---` separator, then a `### 목차` section:
     - Part titles in bold: `**1부. {part title}**`
     - Each chapter as a checkbox: `- [ ] 1장. {chapter title}` (used to check off chapters while reading)
   - Update with:
     ```
     gh issue edit {ISSUE_NUMBER} --repo jongfeel/BookReview --body-file {file}
     ```
   - Example result: issue #1834

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

3. **Determine the target year table** from the **end date** year (e.g. end 2025-09-28 → "Book list of 2025"; end 2026-01-21 → "Book list of 2026")

4. **Insert into the correct year table** in `README.md`
   - The table has **two books per row** (columns: Title | Duration | History | Title | Duration | History)
   - Three layout rules:
     1. **Add col1 first, then col0**: new book goes to column 1 (right) of a new top row; the next book (with a later end date) fills column 0 (left) of that same row
     2. **Never blank bottom**: only the top row may have column 0 empty; every row below must have both columns filled
     3. **Stack order by end date**: books are ordered end date descending, top-to-bottom then left-to-right within each row (col0 > col1)
   - **Step A — add new row at column position 1 (right):**
     ```
     |  |  |  | [![{title}]({IMAGE_LINK})]({aladin_link}) | {start} to {end} | [IssuesLink](https://github.com/jongfeel/BookReview/issues/{ISSUE_NUMBER}) |
     ```
     Insert this new row at the top of the table. Reshuffle existing rows if needed so all rows except the top are fully filled.
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
