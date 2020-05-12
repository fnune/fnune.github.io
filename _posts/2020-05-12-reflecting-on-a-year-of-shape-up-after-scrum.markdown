---
layout: post
title: 'Reflecting on a year of Shape Up after Scrum'
comments: true
date: 2020-05-12 17:37:25 +0200
excerpt: "My developer's perspective on a year of working with Shape Up after switching over from Scrum"
---

In January 2019 [we](https://www.store2be.com) decided to slowly start rebuilding our core product and to release it step-by-step. Since this was a completely new setup, two engineers were assigned the task of initializing the project, and I was one of them. There were no rules constraining our work: we were free to choose, and we weren't even given a hard deadline.

The rest of the team continued working using Scrum, working on our peripheral products, to which most of our efforts had been dedicated until then. This split led to a situation where we had an easily accessible comparison of my small team building the new product to the bigger team working on the rest of the products.

We quickly found that working without a deadline, and leaving project management almost completely in our hands, had a colourful array of effects. Some good, such as, increased quality, morale and learning opportunities, and some bad: communication of progress worsened, different expectations from the other teams caused some distress, and the fact that we were working split from the rest of the team made us want to join forces again.

Once we were done with the initial release, we were overloaded with ideas about how we could make project management better. We gathered feedback, and we started to design a solution. It was in the middle of this that, as if summoned by our need, Ryan Singer from Basecamp released his book [Shape Up](https://basecamp.com/shapeup), which summarizes how they work. David, one of our product managers, introduced us to the book and organized a call with Ryan. The meeting was great—thank you both!—and so was Shape Up: it was so different from Scrum, and also so close to what we were envisioning, that we fell in love with it.

David has also written [his recollection of our experience with Shape Up](https://signsofastruggle.net/2020/01/11/implementing-shape-up-a-case-study/), I'd recommend you read it, too!

## A quick word on Shape Up

It would be a waste of time for me to introduce Shape Up in this article. The book is short and easy to read, and I invite you to check it out.

To help you empathize: I'm a web developer leading a small team of front-end engineers. My involvement with product developers is frequent and constant.

For now, you'll have to be content with my opinion of it after having spent a year trying to make it work for my team. I'll summarize the positives in one short paragraph: the web is already full of those.

## What's going great

Many of the results are positive:

- Both the engineers and product managers love having no backlogs. It's a freeing and stress-relieving way of working, and having to settle on goals more frequently gives our team a stronger feeling of ownership and purpose.
- Cooldown weeks always as well as cycle weeks feel like a fresh start, and for some reason, we respect cooldown more diligently than our previous slack weeks.
- The separation between the shaping track and the building track results in clearer goals it feels like a natural way to separate work.

**Note:** please don't be discouraged by the extensiveness of the next section. I'm purposefully trying to focus on what our team has struggled with. And all-in-all, I think the switch has been greatly beneficial for the team.

## What's not going so well

Teamwork is complicated. Following what another team does because it works for them does not necessarily mean it'll work for yours. It also doesn't mean you'll face the same problems.

### Who shapes?

Fat-marker sketches are a double-edged sword. They allow you to focus on fewer things, but it's still up to you to decide what the right things to focus on are. There's a fine line between "not concrete" and "not thought-through": if it's not thought-through or too abstract, engineers and designers are going to end up shaping during the cycle, and this interference is one of the things Shape Up is meant to solve.

Now, the "ticket taker" or the "code monkey" analogy is also a double-edged sword:

> We give full responsibility to a small integrated team of designers and programmers. They define their own tasks, make adjustments to the scope, and work together to build vertical slices of the product one at a time. This is completely different from other methodologies, where managers chop up the work and programmers act like ticket-takers. [[1]](#citation-1)

Giving development squads the power to influence the product is great. Finding fundamental problems with a solution when the cycle has already started because the fat-marker sketches were made with too fat a marker is not great.

Shaping is a closed-door, creative process. But for a project to be ready for the betting table, you'll probably need some help! It's not enough to be technically literate in order to address risks and to think of all the possible rabbit holes. It's not enough to make the pitch documents accessible to everyone in the team: they won't read them. You need to ask them to do it once you think it's the right time.

### "Appetite as opposed to estimations"

Utilizing the appetite mechanic doesn't remove the need for estimations. At some point, the person representing the engineering team in the betting table is going to have to give their opinion on the feasibility of a project, considering its variable scope, in the appetite it's been given. This is an estimation and it should be treated as such. Use all the tools at your disposition to measure the work well. Your gut feeling is not enough: software projects are notoriously hard to estimate, and if you don't make the effort to measure the project, **variable scope won't save the team** from being unable to finish their work within the cycle.

The [mention of a small batch and a big batch](https://basecamp.com/shapeup/2.2-chapter-08#team-and-project-sizes) in Shape Up caused some problems in our team. The decisions taken in the betting table are often plagued with preconceptions of who's going to be in the big batch, and who in the small batch. This often influences the decision on appetite: "the senior developer should work on this, they can definitely finish this in two weeks". This is a recipe for disaster.

### Betting table meetings during cooldown

It's Wednesday of the second week of cooldown, and you're betting on projects. Decisions are being made based on a growing urgency that nobody wants to mention: we only have two more days before the cycle starts. Nobody has a great feeling about the decisions, but the engineering team needs something to work on next Monday, right?

This is a terrible situation to be in. The solution is to spend a significant amount of time shaping pitches and gaining both supporters and critics of your idea, long before the betting table meetings happen.

## If you're going to switch to Shape Up

Avoid the mistakes that led to the problems I described.

Involve as many people as necessary in the **shaping process**. The prime time of shaping work is the moment misled problem statements are judged, overblown solutions are superseded by simpler ideas, and priorities are clarified: at the right time, before a lot of hours have been spent on details.

- By involving project stakeholders, you can judge the problem better.
- Involve designers to produce ideas both simpler and more elegant.
- Include engineers, and they will help you choose a solution that fits within the appetite you're planning to pitch.

The key to this is to involve others early enough. If you decide to hold the betting table meetings on Tuesday and you're still working on a basic pitch the Friday before that, you've failed. Give yourself weeks for shaping a pitch, it's work best done in collaboration.

Don't choose the **level of abstraction** for your pitch on your own. What's "shaped" for you may be too abstract for the engineer, and too concrete for the designer. Realize that you are all on the same boat: ask them what they think about it before you pitch it.

Save enough time to think of what **variable scope** means for your project. Depending on how important your pitch is, you may be better of with a simple solution that doesn't require so much work. Or if you do need the fully-fledged solution, then ask for help from designers and engineers to help you cut down the scope and identify what's core from what's a nice-to-have.

Make **saying "no"** your default during betting table meetings. You're there to be convinced of whether a team of people should spend weeks of their time working on something. If you're not convinced: say no! There's no need to fill the next six weeks of work immediately if the pitches are not quite there yet. It's better to postpone starting a cycle than to start a cycle that's destined to fail.

Finally, assign responsibilities regarding **communication of progress**. If there are frequent updates, nobody needs to go around asking "how's the project going".

---

The devil is in the detail. Perhaps we weren't doing Scrum the right way, or maybe we're not doing Shape Up the right way, but we're definitely learning a lot about our team, and ultimately building up trust by slowly but steadily discussing every issue that arises with the intention of improving. May I suggest, if things are not going as you'd like in your team: try new ideas! Every team is different, and no existing solution will fit it perfectly.

Switching to Shape Up has made us ask ourselves important questions, but also important discoveries. The team wants to continue doing it: hopefully, I'll make another post when the time comes.

---

### References

1. <div id="citation-1"></div>Singer, R. (2019). Shape Up: Stop Running in Circles and Ship Work that Matters. Retrieved May 12, 2020, from <a href="https://basecamp.com/shapeup/0.3-chapter-01#making-teams-responsible">https://basecamp.com/shapeup/0.3-chapter-01#making-teams-responsible</a>.
