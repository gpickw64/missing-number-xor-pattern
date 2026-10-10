# missing number: solve LeetCode 268 in one pass with O(1) space, dodge the edge cases, and recognise the pattern next time

Type "missing number" into a search box and you're usually in one of two situations. Either you have an array like `[3, 0, 1]` in front of you and a clock running, or you half-remember that this problem has a one-line trick answer that interviewers expect you to find without being told. The brute force is genuinely easy. The interesting part is knowing why the fast version works, which edge cases break naive implementations, and how the same trick shows up again in three other questions you'll meet later in the same interview loop.

Here's the version of the problem most people hit first — LeetCode 268:

> Given an array `nums` containing `n` distinct integers in the range `[0, n]`, return the only number in the range that is missing from the array.

The follow-up is the part that matters: can you do it in O(n) time with O(1) extra space?

Worked examples, because the range boundary trips people up:

| Input | n | Range | Answer | Why |
| --- | --- | --- | --- | --- |
| `[3, 0, 1]` | 3 | `[0, 3]` | 2 | 2 never appears |
| `[0, 1]` | 2 | `[0, 2]` | 2 | The missing value is `n` itself |
| `[9,6,4,2,3,5,7,0,1]` | 9 | `[0, 9]` | 8 | Same idea, longer array |
| `[1]` | 1 | `[0, 1]` | 0 | The missing value is `0` |

Constraints are modest — `n` runs up to 10⁴, every element is unique, and each element sits between 0 and `n`. That's small enough that almost anything passes, which is exactly why interviewers use it: the constraints don't force the clever solution, so they get to see whether you reach for it anyway.

## The answer you can give in 30 seconds: sort, then look for the gap

Sort the array and each index `i` should hold the value `i`. The first index where that breaks is the missing number. If nothing breaks, the answer is `n`.

python
def missing_number(nums):
    nums.sort()
    for i, v in enumerate(nums):
        if v != i:
            return i
    return len(nums)


Time is O(n log n), space depends on the sort implementation. This is the right thing to say first in an interview, because it's correct and you can reason about it out loud. It is not the answer they're waiting for.

## Gauss gets you to O(n) time and O(1) space

The numbers `0` through `n` sum to `n × (n + 1) / 2`. Subtract the sum of the array and whatever is left over is the missing value.

python
def missing_number(nums):
    n = len(nums)
    return n * (n + 1) // 2 - sum(nums)


One pass, one integer of extra state. In Python this is close to bulletproof because integers grow as needed. In Java, C++, or Go, the same line has a real failure mode worth mentioning in an interview: `n * (n + 1) / 2` grows quadratically, so in a 32-bit `int` it overflows once `n` gets past roughly 65,000. With this problem's `n ≤ 10⁴` ceiling you're fine, but if an interviewer scales `n` up to 10⁹ as a follow-up, the sum version silently starts returning garbage while the XOR version keeps working. Saying that out loud is worth more than the code itself.

## The XOR version, and why the bit trick is not magic

XOR has two properties that do all the work here:

- `a ^ a = 0` — a number cancels itself out
- `a ^ 0 = a` — zero is the identity

If you XOR every number from `0` to `n` together with every number in the array, each value except one appears exactly twice and cancels to zero. The missing number appears once, so it survives.

python
def missing_number(nums):
    result = len(nums)          # seeds the value n
    for i, v in enumerate(nums):
        result ^= i ^ v
    return result


Seeding with `len(nums)` handles an edge case that catches a lot of people: if every element is in its "correct" position and the missing number is `n`, the loop never produces it, so you need `n` already in the accumulator.

There's a neat variant that skips the manual seed by pairing each value with an index that starts at 1:

python
from functools import reduce
from operator import xor

def missing_number(nums):
    return reduce(xor, (i ^ v for i, v in enumerate(nums, 1)), 0)


Walk through `[3, 0, 1]` and you get `(1 ^ 3) ^ (2 ^ 0) ^ (3 ^ 1)`, which is `2 ^ 2 ^ 2`, which is `2`. Rearranged with the commutative property, you're really XOR-ing the full range `0..3` against the array `3, 0, 1` — the `0, 1, 3` pairs cancel and `2` is left alone. The `enumerate(nums, 1)` offset is doing the job that seeding with `n` does in the other version. Both are the same algorithm wearing different clothes. This is the approach AlgoMonster's write-up for problem 268 uses, and it walks through the cancellation step by step rather than just handing you the final line.

## Time and space, side by side

| Approach | Time | Extra space | Weak point |
| --- | --- | --- | --- |
| Brute force (scan the range, check membership) | O(n²) | O(1) | Only useful as a starting point to say out loud |
| Sort, then compare index to value | O(n log n) | O(1)–O(n) | Mutates the input; slower than needed |
| Hash set of the array, scan the range | O(n) | O(n) | The obvious "faster" answer; still fails the follow-up |
| Sum formula | O(n) | O(1) | Quadratic growth can overflow fixed-width ints |
| XOR over indices and values | O(n) | O(1) | Requires understanding of XOR to explain |

The pattern in interviews is predictable: candidate offers sorting, interviewer asks for better; candidate offers a hash set, interviewer asks about space; candidate offers sum or XOR, interview ends well.

## The edge cases that actually cost points

Four inputs are worth testing before you declare victory:

1. **The answer is `n`.** `[0, 1]` returns 2. Any solution that only ever compares against indices inside the array will miss this.
2. **The answer is `0`.** `[1]` returns 0. If you seed an accumulator with `0` and never account for the range's lower bound, you can end up returning a wrong value that happens to look plausible.
3. **`n = 1`.** Both `[0]` and `[1]` are valid one-element inputs with different answers (1 and 0). This is the cheapest test set in the whole problem.
4. **Sorted input.** `[0, 1, 2, 3, 5]` returns 4. Sorting-based solutions pass here without exercising anything interesting, which is why it's a bad test case to rely on.

A fifth, less obvious one: if the interviewer forbids modifying the input, the sorting solution is out unless you copy first — and copying costs O(n) space, which puts you right back where the hash set was.

## What interviewers are actually testing

Nobody is screening for whether you memorised the Gauss formula. Two things are being measured:

**Can you use the constraint you were given?** The phrase "distinct numbers in the range `[0, n]`" is doing heavy lifting. It tells you exactly what the complete data set should look like, which means you can compare the actual array against the expected one without building the expected one. Candidates who miss this end up with hash sets and O(n) space.

**Can you explain why a bit trick works?** "I used XOR" is not an answer. "Every number that appears in both the range and the array cancels to zero, so only the missing value survives" is an answer. Interviewers will usually push one step further and ask what happens if two numbers are missing — at which point XOR alone stops being sufficient usage of the pattern and you need to think about grouping by bit, the way problem 260 does.

## Where this problem shows up again

The missing-number idea is one node in a cluster of array and bit-manipulation questions that share the same mental model. Practising it in isolation is fine; recognising it inside an unfamiliar problem is what pays off.

- **Find All Numbers Disappeared in an Array** — the array has duplicates instead of a missing value, and the index-as-hash trick replaces the sum formula.
- **First Missing Positive** — same "the answer lives in `[1, n]`" reasoning, but you have to place values at their correct indices first.
- **Find the Duplicate Number** — the mirror image of this problem, and the place where "compare value to index" and cycle detection both apply.
- **Single Number / Single Number III** — the same XOR cancellation property, one with a single survivor and one where you have to split the array by a differing bit.
- **Missing number in a sorted sequence** — binary search instead of XOR, which is the version that shows up when the input arrives pre-sorted and `n` is huge.

If you find yourself searching for individual solutions to each of these one at a time, that's the actual signal here. They're not five separate tricks. They're one pattern — a bitwise operation that cancels matched pairs, or an index that stands in for a hash key — applied to five different input shapes.

## Where structured pattern practice fits in

This is the gap that AlgoMonster is built to fill. It's a coding-interview prep platform from ex-Google engineers, and instead of a problem bank you work through explicit patterns with written explanations, diagrams, and editorials; the current plan pages list 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, with company-tagged questions refreshed monthly. Problem 268 is covered under the bit-manipulation pattern, which is what lets you connect it to 260 and 136 rather than treating it as a standalone puzzle.

The format is deliberately not video-based — it's written explanations plus illustrations, which works well if you learn by reading and poorly if you need someone talking at you.

### Plans and current pricing

AlgoMonster has one free tier and two paid ones. Pricing on this platform moves with promotions, so the checkout page is the only number worth trusting — third-party trackers that still list lifetime access at $459 are working from older screenshots.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge whether the written format suits you | $0 | — | [Browse the free AlgoMonster content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; monthly-updated company questions; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps; Foundation and Build an LLM bonus courses | $99/year (listed at $198) | Billed annually, 12 months of access, cancel or upgrade anytime | [Get the AlgoMonster Yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, one payment | $189 (listed at $378) | Billed once | [Get AlgoMonster Lifetime access](https://algo.monster?fpr=li69) |

Two details worth knowing before you buy. Specialised courses built with outside partners, such as Quant Interview Prep, are excluded from both plans and sold separately. And there's no monthly option — it's annual or lifetime, so if your interview is in three weeks, you're paying for a year either way.

### Is the discount real?

The 50% reduction appears to be standing pricing rather than a flash sale: the plan page currently displays $99/year and $189 lifetime against $198 and $378. On top of that, a partner promo code, `SCHELLEY`, is reported to stack an additional 10% at checkout, bringing yearly to roughly $89 and lifetime to roughly $170. I haven't verified that code independently, and the only cost of trying it is the thirty seconds it takes to type into the promotion-code box on the payment page — if it doesn't apply, you're still at the displayed price.

👉 [See the current AlgoMonster pricing and apply a code at checkout](https://algo.monster?fpr=li69)

### Who should skip it

Be honest about your situation. If you already have a working pattern framework and just need volume, AlgoMonster's couple-hundred mapped problems are a small bank, and a larger question set will serve you better. If you're interviewing at senior level, the system design material is introductory and won't carry you through a full loop. And if you learn best from video walkthroughs, this isn't the platform for that.

Where it earns its money is a specific case: you have four to eight weeks, you can study one to two hours a day, and your actual weakness is that you see `[3, 0, 1]`, start writing a hash set, and never notice the range constraint pointing at something cheaper.

## Bringing it back to the one line of code

The missing-number problem is a good litmus test for how you study. There's a two-line solution, and there's a reason that solution works. Learning only the two lines gets you through that one question. Learning the reason — every value that appears twice cancels, and the constraint `[0, n]` guarantees exactly one value appears once — gets you through the family of questions that reuse it.

Start by writing the sorting version so you have something correct. Then the sum version, and note where it would overflow. Then the XOR version, with `len(nums)` as the seed, and test it on `[0]`, `[1]`, and `[0, 1]` before you call it done. That's maybe twenty minutes of work for one problem, and it's the difference between recognising the pattern in the next question and starting from scratch.

👉 [Work through the bit-manipulation pattern on AlgoMonster](https://algo.monster?fpr=li69)
