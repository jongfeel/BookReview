# BookReview AGENTS.md

Shortcut commands for agents working in this repository.
Detailed layout rules for the README table live in `CLAUDE.md` → **Add Finish Book Task**; this file only adds what lets the task run from an issue number alone.

## "Issue #{N} 책 README에 추가시켜줘"

**Trigger:** any request like `Issue #1741 책 README에 추가시켜줘`, `#1741 README에 추가`, `Add finish book #1741`.

**Only parameter:** `ISSUE_NUMBER` (taken from the request). `IMAGE_LINK` is **not** asked for; resolve it from Aladin as below.
If the user does give an image link, use theirs instead of the resolved one.

### 1. Read the issue

```
gh issue view {ISSUE_NUMBER} --repo jongfeel/BookReview --json title,createdAt,body
```

- `title` → book title used in README alt text, commit message, and PR title
- `createdAt` → start date (`YYYY-MM-DD`)
- Aladin link from `body`: first match of `https?://(www\.)?aladin\.(kr/p/[A-Za-z0-9]+|co\.kr/shop/wproduct\.aspx\?ItemId=[0-9]+)`
  - Keep the link **exactly as written in the issue** for the README (`aladin.kr/p/...` short link).
  - No Aladin link in the body → search Aladin by title:
    `https://www.aladin.co.kr/search/wsearchresult.aspx?SearchTarget=Book&SearchWord={url-encoded title}`,
    take the first `wproduct.aspx?ItemId=...` result, and say so in the final report so the user can check it.

### 2. Resolve the end date

The milestone (same name as the issue title) also contains the parent issue itself, so **exclude it**:

```
gh issue list --repo jongfeel/BookReview --milestone "{title}" --state closed --json number,closedAt --limit 200 \
  --jq '[.[] | select(.number != {ISSUE_NUMBER})] | max_by(.closedAt)'
```

- `closedAt` → end date (`YYYY-MM-DD`)
- If there are still open sub-issues in the milestone, stop and ask the user whether the book is really finished.

### 3. Resolve the cover image (IMAGE_LINK)

Fetch the Aladin page with a browser User-Agent (follows the short-link redirect) and read `og:image`:

```
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0 Safari/537.36"
curl -sL -A "$UA" "{aladin_link}" | grep -o 'og:image" content="[^"]*' | head -1 | sed 's/.*content="//'
```

- Expected form: `https://image.aladin.co.kr/product/{a}/{b}/cover500/{file}.jpg`
- If it is not a `cover500` URL (e.g. `coversum`, `cover200`), replace that size segment with `cover500`.
- If `og:image` is missing, fall back to the first `https://image.aladin.co.kr/product/.../cover500/...` URL in the page HTML.
- Never use GitHub attachment images (`github.com/user-attachments/...`) from the issue body; README uses Aladin images only.
- Verify the URL responds: `curl -sI -A "$UA" "{IMAGE_LINK}"` → `200` and `content-type: image/...`.

### 4. Run the Add Finish Book Task

With `ISSUE_NUMBER`, `IMAGE_LINK`, title, aladin link, start/end dates in hand, follow `CLAUDE.md` → **Add Finish Book Task** steps 2–7 (branch via `gh issue develop`, year table by end date, col1-then-col0 placement, book count +1, commit, push, PR).

### 5. Report back

In the reply (and in the PR body) include:
- the resolved Aladin link and cover image URL, so the user can verify the cover
- start/end dates and which year table / row / column the book went into
- the PR URL

## Notes

- `og:image` is the cover of the exact product the link points to. If the user prefers a different edition's cover, they can pass the image link explicitly (old style: `Issue number: N / Image link: URL`, see issue #1710).
