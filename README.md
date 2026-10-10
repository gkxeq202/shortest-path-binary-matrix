# shortest path in binary matrix: BFS in 8 directions, the off-by-one that breaks answers, and how to spot the pattern fast

You open LeetCode 1091, read the description, and it looks like every other grid problem you've solved. Then you notice two things: moves are 8-directional, and the answer counts *cells*, not steps. Those two details are where most submissions die.

This is the article I'd want if I were staring at 1091 for the first time, or if I'd already solved it once and needed a refresher before an interview. Algorithm first, then code, then the mistakes that actually show up in practice.

## What the problem asks for, stated plainly

Given an `n x n` binary matrix `grid`, return the length of the shortest **clear path** from the top-left cell `(0, 0)` to the bottom-right cell `(n - 1, n - 1)`.

Clear means two things:

- Every cell you visit has value `0`.
- Every consecutive pair of cells is 8-directionally connected — they share an edge or a corner.

The **length of the clear path is the number of cells visited**, not the number of moves. That single sentence is responsible for a large share of wrong answers. A path that takes 3 moves visits 4 cells.

If no clear path exists, return `-1`.

The constraints are small: `1 <= n <= 100`, and `grid[i][j]` is `0` or `1`. So `n²` tops out around 10,000 cells. Any reasonable `O(n²)` search is fine.

The three canonical examples:

| Input | Output | Why |
| --- | --- | --- |
| `[[0,1],[1,0]]` | `2` | One diagonal move from `(0,0)` to `(1,1)`, so 2 cells |
| `[[0,0,0],[1,1,0],[1,1,0]]` | `4` | `(0,0) → (0,1) → (1,2) → (2,2)` |
| `[[1,0,0],[1,1,0],[1,1,0]]` | `-1` | Start cell is blocked, nothing to search |

Example 2 is worth sitting with for a second, because it's the example that shows why diagonals matter. Without them you'd have to squeeze through the right column and the answer would be longer.

## Why BFS is the answer, not just a good idea

Every move costs exactly 1. There are no weights, no terrain costs, no obstacles you can pay to remove. That makes this an **unweighted shortest path** problem, and unweighted shortest path means breadth-first search.

BFS expands outward in rings. Ring 1 is the start cell. Ring 2 is everything reachable in one move. Ring 3 is everything reachable in two moves, and so on. Because the queue is FIFO, the first time you pull the destination out of the queue, you've reached it by the shortest possible route. That guarantee is the whole reason the algorithm is correct here — it isn't a heuristic, it's structural.

DFS has the opposite problem. It commits to one direction and goes deep, which means it can find *a* path long before it finds the *shortest* one. You'd end up exploring every path and comparing lengths, and on a 100×100 grid that's a non-starter. (The trade-off flips if you need all paths or path counting — see the follow-up section.)

If you're building pattern recognition rather than memorizing solutions, the recognition step here is short: **grid + uniform move cost + "shortest" in the question = BFS.** Anything else and you're likely overthinking it.

## The 8 directions are the real twist

A standard grid BFS uses four direction vectors:

python
directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]


1091 needs eight. The cleanest way to write it is nested loops over `-1, 0, 1`, skipping the `(0, 0)` self-move:

python
for dr in (-1, 0, 1):
    for dc in (-1, 0, 1):
        if dr == 0 and dc == 0:
            continue
        nr, nc = r + dr, c + dc


Or list them explicitly if you find that more readable under interview pressure:

python
directions = [(-1,-1), (-1,0), (-1,1), (0,-1), (0,1), (1,-1), (1,0), (1,1)]


Either works. What matters is that you don't leave four-direction code in place while telling the interviewer you're solving 1091 — that produces correct-looking code and wrong answers.

The geometric consequence of eight directions is that "distance" here behaves like Chebyshev distance, `max(|Δrow|, |Δcol|)`, not Manhattan. On an empty grid the answer would be `max(n-1, n-1) + 1 = n`. Obstacles only ever make it longer, which is a decent sanity check when you're testing your own solution.

## CircuitPython-free walkthrough: how example 2 resolves

Take `grid = [[0,0,0],[1,1,0],[1,1,0]]`, target `(2,2)`.

Start: `(0,0)`, length 1. Its clear neighbors are `(0,1)` — enqueued at length 2 — and `(1,1)` is blocked, `(1,0)` is blocked.

Level 2: `(0,1)`. Neighbors: `(0,0)` already visited, `(0,2)` enqueued at length 3, `(1,2)` enqueued at length 3. `(1,0)` and `(1,1)` are blocked.

Level 3: `(0,2)` and `(1,2)`. From `(1,2)`, the diagonal move lands on `(2,2)` at length 4. From `(0,2)`, `(1,2)` is already claimed and `(1,1)` is blocked.

Level 4: `(2,2)` comes off the queue. Answer: `4`.

Notice that `(1,2)` got claimed at level 3 by two different routes and was only enqueued once. That deduplication isn't an optimization detail — it's what keeps BFS at `O(n²)` instead of blowing up into every possible path.

## The Python solution

python
from collections import deque
from typing import List

class Solution:
    def shortestPathBinaryMatrix(self, grid: List[List[int]]) -> int:
        n = len(grid)

        # Start or end blocked: no clear path can exist
        if grid[0][0] == 1 or grid[n - 1][n - 1] == 1:
            return -1

        queue = deque([(0, 0, 1)])   # row, col, path length so far
        grid[0][0] = 1                # mark visited by flipping to 1

        while queue:
            r, c, length = queue.popleft()

            if r == n - 1 and c == n - 1:
                return length

            for dr in (-1, 0, 1):
                for dc in (-1, 0, 1):
                    if dr == 0 and dc == 0:
                        continue
                    nr, nc = r + dr, c + dc
                    if 0 <= nr < n and 0 <= nc < n and grid[nr][nc] == 0:
                        grid[nr][nc] = 1           # mark on enqueue
                        queue.append((nr, nc, length + 1))

        return -1


Three decisions in there are worth defending out loud in an interview:

**The `(0, 0, 1)` seed.** Storing `1`, not `0`, bakes in the "path length counts cells" rule instead of patching it with a `+1` at the return.

**Marking on enqueue, not on dequeue.** If you wait until you pop a cell, the same cell can be pushed multiple times by different neighbors before it's ever processed. The result is usually still correct but the queue can grow far beyond `n²`. Mark when you push.

**Early exit at the destination.** Because BFS processes in increasing length order, returning the moment `(2, 2)` surfaces is safe. There's no shorter route left to find.

The blocked-endpoint check matters more than it looks. It's cheap, it handles the `n == 1` edge case cleanly (`[[0]]` returns `1`, `[[1]]` returns `-1`), and it prevents any weirdness where the start cell *is* the destination.

## Complexity

|  |  |
| --- | --- |
| Time | `O(n²)` — each cell is enqueued at most once, and each cell checks a constant 8 neighbors |
| Space | `O(n²)` — worst case the queue holds an entire BFS level, which can approach the cell count on an open grid |

With the in-place variant above, you skip the extra `visited` matrix, so the practical memory footprint is the queue plus the small amount of state per entry. That's a real difference when `n = 100` and you're writing this on a whiteboard with limited scratch space.

## Five mistakes that actually cost people this problem

**1. Counting moves instead of cells.** Return `steps + 1`, or seed the queue with length 1. Pick one and stay consistent.

**2. Four directions instead of eight.** The most common silent failure. The code runs, the examples partly pass, and the diagonal shortcuts are missing.

**3. Marking visited on dequeue.** Works on small inputs, degrades on large ones, and looks sloppy under review. Mark when the neighbor is pushed.

**4. Skipping the endpoint check.** Forget to test `grid[n-1][n-1] == 1` and you'll return `-1` anyway on most inputs — but the check is what makes the `n == 1` case behave, and it's free.

**5. Using a `visited` set for coordinates when you don't need one.** If mutating the input is acceptable (it usually is on LeetCode), flipping `0` to `1` is faster and simpler than maintaining a `set` of tuples. If it isn't acceptable, use a boolean matrix.

> One caveat on the in-place version: it destroys the caller's grid. In an interview, say so out loud. Interviewers notice when you flag a side effect before they ask about it.

## In-place marking vs. a separate visited structure

Both approaches are correct. The difference is what you're optimizing and what you're promising.

Flipping cells to `1` gives you `O(1)` extra lookup cost, no hashing, and shorter code. It also mutates the input, which is fine on a judge and fine in most interviews, but worth stating explicitly.

A `visited` set or boolean matrix keeps the input intact and reads a little more literally — "this cell has been reached." On a 100×100 grid the overhead is negligible. Choose based on whether the original grid needs to survive the call.

A quick way to decide out loud: *"I'll mark in place by flipping to 1, since the grid isn't needed afterward. If you'd rather I preserve it, I'll add a boolean matrix instead."*

## The follow-ups interviewers actually ask

The base problem takes maybe ten minutes once you see the BFS. The follow-ups are where the conversation goes.

**Count the number of shortest paths.** Run the same BFS but carry two things per cell: distance and path count. On first visit, record the distance and inherit the parent's count. On a re-visit at the *same* distance, add the parent's count. Same `O(n²)`, one extra matrix.

**Return all shortest paths.** Also `O(n²)`: build a parent map during BFS where each cell stores every predecessor that reached it at the optimal distance, then backtrack from `(n-1, n-1)` to `(0, 0)`. Note that the *output* can be exponential even though the search isn't.

**Add obstacles you can remove.** That's LeetCode 1293, where the state becomes `(row, col, remaining_eliminations)` and the visited structure gains a third dimension. Same BFS skeleton, richer state.

**Move to a weighted grid.** BFS stops applying the moment edge weights differ. You'd reach for Dijkstra, and if weights are `0` or `1`, a 0-1 BFS with a deque.

That progression — plain BFS, then state-augmented BFS, then weighted shortest path — is one of the more reusable arcs in interview prep. 1091 sits at the front of it.

## Turning the solution into a pattern you can reuse

Solving 1091 once teaches you 1091. The transferable skill is recognizing *unweighted shortest path on an implicit graph* in a problem that doesn't say "graph" anywhere. Grids, word ladders, knight moves, rotting oranges, puzzle state spaces — same skeleton, different dressing.

That's the gap a pattern-oriented course is designed to close. [AlgoMonster](https://algo.monster?fpr=li69) groups grid problems under a matrix-as-graph framing, where cells become nodes and legal moves become edges, which is exactly the reframe 1091 demands before BFS becomes obvious. It also has a Speedrun mode for timed practice and an AI assistant you can query without a usage cap on paid plans.

Two things to know before you pay for anything, from third-party reviews: the content is text and diagrams, with no video walkthroughs, and it's built for self-directed learners — there's no mock interviewing or mentorship, and system design coverage is introductory. If those are your gaps, budget for them separately.

### AlgoMonster plans and current pricing

AlgoMonster runs periodic promotions, so the annual and lifetime numbers move. Here's what the official pricing page has shown across recent checks:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Monthly | Full content access, unlimited AI assistant, unlimited Speedrun, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials | $18.99 | Billed monthly, 1 month of full access | [See the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything above, plus member discount on live bootcamps and 12 months of access | $8.25/mo, billed annually ($99/year) | Annual; promo periods have shown $69.99/year | [Compare the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates and all future courses | $189 | One-time; promo periods have shown $89 | [Check the lifetime price](https://algo.monster/subscribe?fpr=li69) |

Prices were observed on the official pricing page and change with active promotions — confirm the current number on the page itself before buying. The lifetime plan is the one that makes financial sense if you expect to interview more than once over the next few years; otherwise the annual plan covers a normal prep cycle.

## FAQ

**Is BFS always the right call for this problem?**
Yes, for the shortest-path version. Any time every move costs the same, BFS gives you the optimum. The moment moves have different costs, you need Dijkstra.

**Can I solve it with dynamic programming?**
Not cleanly. DP wants a fixed direction of dependency — top-left to bottom-right, then back — and that works for problems like 01 Matrix where the nearest-zero distance can be relaxed in two passes. Here the eight-directional connectivity combined with "shortest path from a single source" makes BFS both simpler and more defensible.

**Why does the answer count cells and not moves?**
That's the problem's definition, stated in the description. It translates to seeding the queue with `1` and returning the distance when you reach the destination. Guessing wrong here costs you the problem even with a correct search.

**What's the smallest input to test?**
`[[0]]` should return `1`. `[[1]]` should return `-1`. Those two catch the endpoint-check bug immediately, before you get distracted by bigger grids.

**Does the order of the 8 directions matter?**
Not for correctness. Any order explores the same level correctly. It can change which shortest path you happen to return first when several tie, which matters only for the "return all paths" follow-up.

## Bottom line

1091 is a BFS problem wearing a grid costume. Seed the queue with a path length of 1, check both endpoints for blocks up front, walk all 8 directions, mark each cell the moment you enqueue it, and return the length the first time the bottom-right cell comes off the queue. That's `O(n²)` time and `O(n²)` space, and it's the whole solution.

Get the off-by-one and the 8 directions right and you're done. Get them wrong and you'll spend twenty minutes debugging code that was structurally correct the whole time — which is, honestly, the most 1091 thing about 1091.

If you want the pattern itself to stick rather than the one solution, the practice that pays off is running the same BFS skeleton on three or four different grids until the recognition step is instant. That's the part interviewers are scoring.
