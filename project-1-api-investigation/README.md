# Project 1: API Investigation — Ashby Job Posting API

> **Simulated support ticket.** The customer and scenario are fictional. All test results are real and were captured on November 12, 2026.

## Ticket: Customer reports their careers page shows no jobs

**Reported issue:** "We connected our Ashby job board to our careers site, but it isn't showing any jobs."

**Environment**
- Endpoint: `GET https://api.ashbyhq.com/posting-api/job-board/{board}`
- Parameter: `includeCompensation=true`
- Tool: Postman web app (Cloud Agent)
- Date tested: November 12, 2026

**Steps to reproduce**
1. Send `GET https://api.ashbyhq.com/posting-api/job-board/ashby?includeCompensation=true` to establish a working baseline.
2. Send the same request with the board name changed to `ashby-test-not-real-123`.

**Expected result:** `200 OK` with a list of published jobs.

**Actual result:** With an incorrect board name, the API returns `404 Not Found` and no jobs.

## Evidence

**Baseline:** `200 OK`, about 110 ms, 1.73 MB. All 3 automated tests passed: status is 200, `jobs` is a list, and every job has a `title` and `jobUrl`.

**Failure tests** (one change at a time, reverted before the next):

| # | What I changed | Status code | Response | Interpretation |
| --- | --- | --- | --- | --- |
| 1 | Board name that doesn't exist (`ashby-test-not-real-123`) | 404 | Not Found | The API rejects an unknown board name with a clear error. |
| 2 | Typo in the path (`posting-ap`) | 401 | Unauthorized | The API returned an authentication error instead of 404, which could mislead a customer into troubleshooting credentials. |
| 3 | Misspelled parameter (`includeCompensaton`) | 200 | Request successful; pay fields missing | The API silently ignored the unknown parameter, with no warning to the caller. |
| 4 | Method changed from GET to POST | 401 | Unauthorized | The API returned an authentication error instead of 405 (Method Not Allowed), with the same risk as Test 2. |

Screenshots of each test are in the `/screenshots` folder, and the Postman collection is exported as `Ashby Job Board.postman_collection.json`.

## Investigation

- **Ruled out an outage:** the baseline request returned `200 OK` and passed all tests, so the API was healthy.
- **Reproduced the symptom:** the only change that produced "no jobs" was an incorrect board name (Test 1, `404`).
- **Ruled out the pay parameter:** even when misspelled (Test 3), the request still returned jobs; only the pay fields were missing.
- **Watched the tests fail as expected:** on the `404` response there was no `jobs` list, so my test script failed. That confirms the tests catch this problem automatically.
- **Unexpected behavior in Tests 2 and 4:** a wrong path and a wrong method both returned `401` instead of `404` or `405`. **Hypothesis (unconfirmed):** requests that don't match the public posting endpoint are handled by Ashby's authenticated API, which requires an API key.

## Root cause

Most likely a **configuration issue**: the board name in the customer's careers-page setup doesn't match their Ashby job board name.

A second possibility is that the board name is correct but no jobs are published. The two cases can be told apart by the customer's actual request:
- `404 Not Found` → wrong board name (confirmed in Test 1).
- `200 OK` with an empty list → likely no published jobs (expected behavior, not tested here).

## Classification

**Configuration**, not a product defect. The API behaved correctly for the customer's issue.

## Product feedback (separate from this ticket)

1. **Silent parameter typos (Test 3):** an unknown parameter returns `200` with no warning. Suggest a documentation note or a validation warning.
2. **Misleading 401 errors (Tests 2 and 4):** a path typo or wrong method returns "Unauthorized," which points customers toward credentials instead of the real mistake. Suggest a clearer error or a documentation note.

## Customer reply

> Thanks for flagging this! The job board connection is working on our end. The request your careers page is sending uses a board name we don't recognize, so no jobs come back. Could you check that the board name in your careers page setup matches the name at the end of your jobs.ashbyhq.com URL? Once it matches, your published jobs should appear right away. I'll keep this ticket open until you confirm they're showing.

## Next steps

- **Customer:** confirm the board name in their careers page setup.
- **Support:** if jobs still don't appear with the correct name, ask for the exact URL their site calls and the time of the failure, then retest.
- **Engineering:** escalate only if a correct board name returns errors or leaves out jobs that are published.
- **Product/Docs:** share the two product-feedback items above.

## What I learned

- Status codes point to who needs to act: 4xx usually means the request (customer side), 5xx means the server (escalate).
- Error messages can mislead. A `401` doesn't always mean a credentials problem, so I check what was actually sent before trusting the error text.
- While following the Postman course, I hit a `500` error from its practice API. It turned out the API's host (Glitch) shut down project hosting in July 2025, so I switched to Ashby's live API. Checking for an outage before assuming my request was wrong saved time.
