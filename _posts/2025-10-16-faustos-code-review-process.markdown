---
layout: post
title: "Fausto's code review process"
comments: true
categories: [collaboration]
date: 2025-10-16 11:59:00 +0200
excerpt: 'I try to review code quickly and in a standard manner: my goal is to enable colleagues and improve velocity.'
---

- [Guiding principles](#guiding-principles)
  - [Unblock the author](#unblock-the-author)
  - [Reviews must be actionable](#reviews-must-be-actionable)
- [The process](#the-process)
  - [Default action: approve](#default-action-approve)
  - [Use request changes liberally](#use-request-changes-liberally)
  - [Not feeling confident to review?](#not-feeling-confident-to-review)
- [Comment categories](#comment-categories)
  - [Nit](#nit)
  - [Super-nit](#super-nit)
  - ["Would like to see" / questions](#would-like-to-see--questions)
  - [Off-topic](#off-topic)
    - [Off-topic: product decision](#off-topic-product-decision)
  - [Logic issue / bug](#logic-issue--bug)
  - [Celebratory / thanks](#celebratory--thanks)
- [Cross-linking](#cross-linking)

## Guiding principles

### Unblock the author

My goal in code review is not to be a gatekeeper, but to be a collaborative partner in improving code quality while maintaining team velocity. This process is built on trust. I trust my teammates to make good decisions and address feedback appropriately.

A quick, high-quality code review keeps everyone motivated about their work. It prevents context changes, and it leads to a better team that produces work quicker and in higher quality. I try to review pull requests as soon as possible, without breaking my own context too much, by setting aside time that does not break my day in two: for example, before or after lunch, at the beginning of the day, or at the end. For simple changes, 6 hours is too much time for a pull request to go unreviewed. For complex changes, two days is too much time.

### Reviews must be actionable

Pull request reviews should always give the author a clear next step. It's not acceptable for a review to open up a broad discussion without explicitly making a decision about whether the pull request should be blocked until the discussion is resolved.

Comments-only reviews without a clear conclusion are not good: I can't merge because it's not approved, but I'm also not sure if the comments are blocking. I don't know what to do. The author is left unsure whether to address feedback, wait for more input, or proceed with merging.

Almost every review I leave will either approve the pull request or request changes. If I lack the context to make that decision, I redirect to someone who can. This ensures the author always knows their next action.

## The process

<pre class="mermaid">
flowchart TD
    Start([PR Ready for Review]) --> Review[Review PR & leave comments]

    Review --> ConfidentCheck{Feel confident to approve or request changes?}

    ConfidentCheck -->|No| Redirect["Tag someone with context 'I don't feel confident to review this properly. Would ask @person to take a look.'"]
    Redirect --> End([Done, waiting for a better reviewer])

    ConfidentCheck -->|Yes| ReReviewCheck{Expect to need re-review after changes?}

    ReReviewCheck -->|No - DEFAULT 95% of cases| ApproveComments[Approve with comments]
    ApproveComments --> AuthorChoice[Author decides: merge now or after fixes]

    ReReviewCheck -->|Yes| RequestChanges[Request changes, used liberally]
    RequestChanges --> ClearComments[Clear communication of what needs fixing: this is now top of my queue to unblock author ASAP, other reviewers can dismiss to override]
    ClearComments --> ReReview[Author fixes, dismisses, or gets another approval]

    AuthorChoice --> Merge([PR can be merged])
    ReReview --> Review

    class Start nodeStart
    class Merge nodeSuccess
    class End nodeDanger
    class ApproveComments nodeApprove
    class RequestChanges nodeWarning
</pre>

### Default action: approve

To promote what I believe is a healthy team environment, **I default to approving with comments about 95% of the time**. This signals trust and keeps work flowing. The question isn't "is this perfect?" but rather "is this good enough to merge, and can any issues be addressed afterward or through normal iteration?".

To be very clear: I will sometimes approve a pull request even if I think it worsens the quality of the codebase. If I see a structural problem or a pervasive bad practice, I will raise the issue elsewhere and cross-link. I approve even when there are bugs, as long as they're understood, and the intent of the PR is clear.

Context matters: feature-flagged work that users won't see yet warrants a lighter touch and faster merging, while changes to public contracts or APIs deserve more thorough consideration given their broader impact.

### Use request changes liberally

I only choose request changes when I am sure I will need to come back to the pull request to review it again, because I expect that after the changes it'll look very different. This might be when the changes are architectural or complex enough that I want to see the implementation, or I have concerns about the approach that need discussion.

"Request changes" is a communication tool: other reviewers can [dismiss change requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/dismissing-a-pull-request-review) if they have the necessary context, which means I feel comfortable requesting changes in the rare case I believe I need to see the code again after a rework, without fear of becoming a bottleneck. Both requesting changes and dismissing a change request are completely OK given clear and thoughtful communication.

If I request changes, however, I'm **making a commitment to the author**: I communicate clearly what needs to change and why, I prioritize re-reviewing (this goes to the top of my queue), and I unblock the author ASAP once they've addressed my concerns.

### Not feeling confident to review?

If I'm unfamiliar with the codebase area, don't understand the business requirements, or the PR touches complex systems I haven't worked with, I redirect to someone with appropriate context. I don't approve what I can't properly evaluate. I will nevertheless comment with any thoughts I have, but I will make it clear that I'm not the right person to review.

## Comment categories

To set clear expectations and help authors prioritize feedback, I prefix my comments with one of these labels where applicable:

### Nit

Label: `Nit: `

Minor nit-picks about style, naming, small improvements, or behavior that don't critically affect functionality. These are "take it or leave it" suggestions. The weight of my own convictions should not make me lose sight of the main purpose: to unblock the author. A team can align on common preferences, but that should be a separate conversation.

Note: spelling or grammar mistakes visible to users are not labeled as nits, as they're more like bugs if users will see them.

Examples:

- "Nit: this variable could be named `userId` instead of `uid` for clarity."
- "Nit: this comment explains what the code below is doing, but it does not say anything about the 'why', what do you think about (...)."

### Super-nit

Label: `Super-nit: `

Extremely nit-picky comments that are truly optional. I use this for things that are purely personal preference or very minor consistency issues.

Example: "Super-nit: we could use a function that returns this test's dependencies instead of declaring them at the top as nullable variables and setting them up later"

### "Would like to see" / questions

Label: none (use natural phrasing)

Comments that are worth discussing. I may sense that there is an underlying assumption and want to confirm that. Or maybe I have an idea of how something could have been done differently and want to hear the author's opinion.

Examples:

- "Could we add error handling here for the case where the API returns an error?"
- "Question: why did we choose approach X over Y? Just trying to understand the tradeoffs."

### Off-topic

Label: `Off-topic: `

Discussions that are only tangentially related to the PR. I use this for systemic issues, technical debt, or improvements that should happen separately from this PR.

Example: "Off-topic: looks like we could be using a linting rule for X here. We should probably add that to our linter config in a separate PR. I've opened an issue here (...), let's discuss there."

#### Off-topic: product decision

Label: `Off-topic (product decision): `

A specific subclass of off-topic for when the code correctly implements a product decision, but you disagree with the underlying product decision itself. The PR author is just implementing what was decided elsewhere, so this shouldn't block the PR.

Example: "Off-topic (product decision): this feature defaults to opt-in, but I think opt-out would have better adoption. Not a code issue, as the implementation is correct, but worth discussing with the product team separately. I've opened a thread about this here (...)."

Tip: for off-topic discussions, especially product decisions, consider opening a Slack thread and cross-linking it with the GitHub comment. This keeps the PR focused while ensuring the discussion happens in the right forum.

### Logic issue / bug

Label: None (be direct and clear)

I use this when I believe the code has a logic error or will produce unexpected behavior. This is not a style issue or architectural preference but I'm flagging that the code won't work as intended.

I'm clear and specific about what I think will happen versus what should happen. These comments only warrant "request changes" if the bug is not yet understood by both the author and me. I keep in mind that there may be a case of asymmetrical knowledge or I may have misunderstood the intent of the change, in which case it may be worth improving the pull request description or adding explanatory code comments.

Example: "Logic issue: this function returns early when `count === 0`, but I think we still need to update the timestamp in that case. As written, zero-count events won't be tracked." In this case, I approve anyway.

### Celebratory / thanks

Label: None (be genuine and specific)

I call out clever solutions, good refactoring, thorough testing, or helpful documentation. I thank authors for addressing tricky issues or improving code quality.

Examples:

- "This is a really elegant solution to the caching problem!"
- "Thanks for adding those test cases, they caught an edge case I hadn't thought of"
- "Love how this simplifies the error handling"

## Cross-linking

For off-topic discussions, especially those involving product decisions or broader architectural questions, I open a separate thread in the appropriate channel (Slack, the issue tracker, or wherever the team discusses such topics) to discuss the matter there. I always make it clear in my PR comment that it's non-blocking when moving discussion elsewhere. I post my initial comment on the GitHub PR stating it's non-blocking, then open a thread in the appropriate channel with context, link to that thread in my GitHub comment, and link back to the GitHub comment from the thread for full context. This keeps the PR focused on the code changes while ensuring important discussions happen in the right forum with the right stakeholders.
