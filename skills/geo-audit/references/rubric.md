# GEO scoring rubric

Scoring definitions for `geo-audit`, plus the fix that maps to each failure mode.
The point of a fixed rubric is that a score from March is comparable to a score
from August. Change the definitions and the trend line becomes fiction.

## appear

| Value | Meaning |
|---|---|
| `true` | The brand is named anywhere in the grounded answer. |
| `false` | Not named. |

Named only inside a URL the answer cites, without appearing in the prose, is
`false`. A buyer reads the prose.

## prominence

| Value | Meaning |
|---|---|
| `first` | Named first, or as the direct recommendation. |
| `listed` | In the main list of options, not first. |
| `mention` | An aside, a footnote, a "you could also look at". |
| `absent` | Not named. |

## sentiment

| Value | Meaning |
|---|---|
| `accurate` | Described correctly, in terms the project would use. |
| `hedged` | Named but qualified into uselessness: "less mature", "smaller community", "unclear if maintained". |
| `wrong` | Described incorrectly, or confused with a different entity of the same name. |
| `n/a` | Not named. |

`wrong` is worse than `absent`. An absent project is invisible; a wrongly
described one is actively mis-sold to a buyer who was ready to choose.

## Clean win

`appear == true` **and** `sentiment == accurate`. Prominence does not gate a
clean win, but `first` is the goal on the niche and problem queries.

Headline score = clean wins / total queries. Report it as `3/4`, never as a
percentage: percentages imply a precision that four stochastic samples do not
have.

---

## Failure modes → the fix

| Symptom | Diagnosis | Fix |
|---|---|---|
| `absent` on the **category** query | No presence on the pages models cite | Placement: get into the roundups and awesome-lists in the target list |
| `absent` on the **niche** query | The project's own positioning is not machine-legible | On-site: `llms.txt`, FAQ, JSON-LD, a repo description carrying the category words |
| `absent` on the **problem** query | No content framing the pain in the buyer's words | A comparison or decision-guide page at a URL containing the query terms |
| `wrong` on the **branded** query | Name collision | Disambiguation: explicit "not affiliated with X" in FAQ + schema, consistent two-word naming everywhere off-site |
| `hedged` anywhere | Thin or stale third-party evidence | Citable specifics in public: real numbers, dated changelog, third-party writeups |
| `mention` where `first` is wanted | Present but out-ranked on that page | Improve the entry itself on the cited page, not the site |

## Anti-patterns

- **Scoring more queries to make the number look better.** Four honest queries
  beat twelve padded with terms nobody types.
- **Rewording a query between runs.** That resets the trend line. Queries are
  fixed once chosen; add new ones as additions, never as edits.
- **Counting a citation as a win.** Being *cited* is not being *recommended*.
  Only prose mentions score.
- **Chasing the category query first.** It is the hardest and the least
  winnable. The niche and problem queries move first and matter more.
