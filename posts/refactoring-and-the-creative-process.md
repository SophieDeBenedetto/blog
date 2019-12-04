# Knock Down Your House: Refactoring and The Creative Process

Recently, my team and I were tasked with the holy grail of assignments--a greenfield project. We were writing brand new code in a brand new app to meet brand new requirements. Like many developers, we jumped at this chance to design our own codebase and solve a new set of problems, outside of the chaffing confines of a big, crufty, legacy app.

We eagerly got to work, but as the blank pages of so many empty text editors stared back at us, we started to succumb to the siren call of perfection.

Over the course of 12 weeks, we at times gave into that "perfection pressure", fought our way out of the corners it backed us into, and finally made peace with the creative process of coding. In other words, we learned to build our house, and we learned to knock it down.

## The Siren Call of Perfection

To shed some light on the tensions we struggle with as writers of code, we can look to Annie Dillard's "The Writing Life":

> The reason to perfect a piece of prose as it progresses...is that original writing fashions a form. It unrolls out into nothingness. It grows cell to cell, bole to bough to twig to leaf; any careful word may suggest a route...out of which much, or all, will develop. Perfecting the work inch by inch, writing from the first word toward the last, displays the courage and fear this method induces.

At first glance, this directive to "perfect a piece of [code] as it progresses", seems necessary and vital. Given an opportunity to build a feature or an app from scratch, we should lay out the perfect approach, design for every potential consideration and ensure that every method we define, a class we build, represents another perfect step towards our perfect goal.

Let's take a look at what this perspective really costs us.

### Trying to Predict the Future

We were building an app responsible for "deploying" content to GitHub. At first, we had two use-cases: deploying content for teachers and deploying content for students. With two use-cases in-hand, I decided to ignore the rule of threes and *start* with a code design. Before writing a single test for a real requirement, I began sketching out a set of classes that would handle both of our use cases.

I got excited about the early stages of my design, and without stopping to think, I shared these initial design thoughts with the pair working on the first use-case. The pair stopped writing code for the specific use-case at hand and started trying to execute a code design that would pre-solve for _both_ scenarios.

### Drowning in Scope Creep
Having tempted a pair to sail too close to the rocks, they start to drown in scope creep. They were writing code to solve a problem that didn't exist yet. They exhibited all the familiar symptoms of scope creep––their progress slowed, their ticket stayed on the right side of the Kanban board for days and days, their standup reports became meandering and confusing.  

Luckily for me, another teammate swooped in with the solution. He encouraged us to *stop* designing and *start* writing code to meet a specific set of requirements. He correctly reminded us that messy code is less expensive than the wrong abstraction.

So, we abandoned our code design and started writing code *just* for the set of requirements in front of us.

## Let the Process Be Your Guide

We let go of the impetus to over-design our code, to plan for every eventuality and thus derail our progress.

> When you write, you lay out a line of words. The line of words is a miner's pick, a wood-carver's gouge, a surgeon's probe. You wield it and it digs a path and you follow. Soon you find yourself deep in new territory...The new place interests you because it is not clear.

Programmers are naturally curious, we're attracted to new problems, to problems we can't readily solve. "The new place interests [us] because it is not clear". We resolved to give into our curiosity about the problem in front of us and stop trying to predict the future.

In the scenario described above, we built two classes that contained *a lot* of duplication. And we walked away, for a time. Then, when a set of requirements around deploying a third kind of resource came up, we were ready to refactor the code we had written in order to meet this new requirement.

This was the time when we sat down to develop a sane, clean design and apply it. We were finally ready to refactor.  

## Process is Nothing, Erase Your Tracks

Instead of perfecting the work *as* is progressed, we let the work proceed. We wrote code that was messy, that wasn't DRY, and we moved on to the next problem. When we revisited our earlier code, with the third set of requirements in hand, we could clearly see what needed to change and why.

> The reason not to perfect a work as it progress is that...original work fashions a form the true shape of which it discovers only as it proceeds, so the early strokes are useless, however their sheen.

> Now the earlier writing looks soft and careless. Process is nothing; erase your tracks.

Despite finally understanding why our code needed to change and how, the directive to "erase our tracks" was still a difficult one to follow. The "sheen" of our earlier work still appealed to us. We had to adjust our perspective yet again in order to refactor.

## Duck!

It's one thing to *know* that code you wrote needs to change. Its another thing to actually change it. The longer our bad code stays in place the more attached to it we become. It begins to have "the ring of the inevitable".

> Several delusions weaken the writer's resolve to throw away work. If he has read his pages too often, those pages will have a necessary quality, the ring of the inevitable, like poetry known by heart; they will perfectly answer their own familiar rhythm. He will retain them.

In revisiting our duplicative classes, we started to convince ourselves why we *couldn't* change certain elements.  

Having given ourselves over the process of writing code, having allowed ourselves to write *bad* code, we needed to allow ourselves to let that bad code go. We took a step back and trusted the instincts that we've cultivated as developers––the instincts that help us identify code smells or tease out good patterns from bad code. In this way, we were able to let go of our bad code, however precious it had begun to seem to us. We had built our house, and now we were ready to tear it down.

> The line of words is a hammer. You hammer against the walls of your house. You tap the walls, lightly everywhere. After giving many years attention to these things, you know what to listen for. Some of the walls are bearing walls; they have to stay, or everything will fall down. Other walls can go with impunity; you can hear the difference. Unfortunately, it is often a bearing wall that has to go. It cannot be helped. There is only one solution, which appalls you, but there it is. Knock it out. Duck
