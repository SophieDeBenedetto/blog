# We're All Project Managers Now

## Intro
The transition to remote work means async takes the foreground. Whether, like me, you've transitioned into a remote, distributed company, or your formerly in-person team has gone "remote" during the pandemic, async communication and processes are rapidly taking over. It's especially important to keep in mind that your newly remote team isn't just "working at home"--they're working _at_ home while homeschooling their kids, or navigating an increasingly restricted world (someone please tell me where I can buy face masks, thanks), or dealing with stress, family, health issues--you name it. In these situations, we can no longer assume that everyone is able to commit to the same synchronized working hours.

In this new world, the in-person strategies and tactics we used to manage the development life cycle and reliably deliver valuable software are no longer available to us. This creates a set of problems that prevent us from getting stuff done, but luckily there's a solution. Keep reading to learn about the pitfalls of async work and find out how you can combat them by being your own project manager.

## The Dangers of Remote, Async Work

Without things like synchronous, in-person daily standups, or the ability "shoulder tap" your PM, tech lead or teammate, its easy to fall prey to a number of common problems.

* Scope creep on your in-flight work
* Missed deliverables--shipping features with missing or unidentified requirements
* Isolation--lack of collaboration at both the implementation planning stage and in your day-to-day coding

How can we avoid these terrible dangers? How can we ensure that our engineering work stays in-scope and on-time, and that it provides deliverables that meet the goals of the product?

We need YOU to be a project manager! And we need you to do your project management using async, written communication.

## What It Means To Be A Project Manager
Congratulations--you're promoted! Kind of. You're a project manager now!

What does it mean to be a project manager?

First off, it's important to remember that you are not a *product manager*--your PM is still the keeper of the roadmap, the decider of prioritization, the communicator to stakeholders and business leaders. So, what are your responsibilities?

You need to document your work like a project manager. This means more time spent writing clear requirements, enumerating engineering tasks and facilitating collaboration amongst your teammates.

In particular, there are a few specific strategies that you can adopt that will help you excel in your new role (congrats again on your promotion).

On your issues or tickets, make sure you:
  * Declare the deliverable--when is this unit of work done?
  * Outline discrete tasks that team members can contribute to or pair with you on
  * Write up background info so team members can help you if you're stuck or contribute to the larger deliverable

In addition to these written communication strategies, don't forget your PM! They're not seated next to you anymore, and maybe they're not even getting a daily update via a synchronous standup. That doesn't mean you have to give into scope creep or get overwhelmed with a set of tasks that you are powerless to say "no" to. When outlining new work, suggest a priority and ask your PM to thumbs up/down it, build this step into your workflow.

In the next section, we'll explain how each of these strategies help us avoid the pitfalls described above and we'll take a look at how we can leverage them on an async, remote team.

## The Problems: Scope Creep + Missing Requirements
In the past you may have leveraged in-person conversation to raise concerns about scope creep and keep your work on track. On an in-person team, it's easier for your product manager or tech lead to have a high-degree of awareness of how your work is going, how long it's taking to deliver, what you're stuck on, etc. But when you're working from home, in relative isolation, it becomes your responsibility to closely track the progress of your work against your goal and communicate that progress. In order for your product manager, tech lead or teammates to help you stay on track, you need to be your own project manager.

### The Solution: Write Issues with a Clear Deliverable
Adopting a simple strategy when you write up or claim an issue or ticket can help you do exactly that.

Make sure any ticket you work on has a clearly defined **deliverable**. How does this help with the problems of scope creep and missed requirements?

By clearly defining what "done" looks like, we give ourselves the tool to combat scope creep. If we begin taking our work in a new direction, or find ourselves adding new tasks to the ticket or issue, we and our teammates can point back to the deliverable and ask: "Is this work necessary in order to achieve the deliverable?"

Similarly, our clear picture of "done" helps us combat missed deliverables. When your product manager and/or tech lead signs off on what "done" looks like, you avoid pre-launch surprises in which a startled product manager may say: "Oh no! We're not ready to launch this feature, I thought it would _also_ do X!"

At GitHub, we use GitHub (surprise) to manage our project development life cycle. This means we write GitHub issues to document our work on a given project. My team uses templates to structure our issues and we've recently adopted a template with a build-int "Deliverable" section. In this way, engineers are adopting a project management mindset and using async, written communication to help keep our projects on track.

## The Problem: Isolation

When you're working synchronously and/or in-person, one-on-one conversations with teammates can often successfully set the stage for pairing sessions or enable you to get someone up and running on a new unit of work. When we're no longer working in-person, or when your coworker isn't available for live conversations at the same time you are, we can struggle to collaborate. This means that we're more often solving problems alone and delivering work slower than when we had another brain to bounce ideas off of or another set of hands to complete a task in parallel with our own.

Once again, adopting a project management mindset can help us avoid these issues.

### The Solution: Write Issues with Context + Task Items
When you're spec-ing out or claiming a unit of work, put on your project manager hat and make sure that the issue or ticket you're writing has a section on background/context and a section with clearly defined engineering tasks.

How does this help solve the problem of isolation?

By providing background info and breaking out discrete units of work, you enable and invite collaboration. Using any context or background info you provide, your teammates are able to quickly get up to speed in order to pair. Furthermore, teammates are able to work towards the larger deliverable in parallel by completing some tasks while you do others.

On my team at GitHub, we use an issue template that contains a "Task List" section. The engineer owning the issue is responsible for spec-ing out this task list and linking these items to additional issues with their own clearly stated deliverables. This approach to issue writing is employed asynchronously and enables team members to work in parallel to deliver larger units of work.

## The Problem: Shifting Priorities
On an in-person team, the product manager has a high-degree of awareness of your day-to-day work. They can use this awareness to keep a tight reign on project priorities. On an in-person, or a synchronous team, it's also easy for you as an engineer to raise a red flag to your product manager when new issues or considerations arise, always letting them make the final prioritization call.

When we're working remote, and when engineers take on more responsibility for issue or ticket writing, it's easy to loose track of the product roadmap or get confused regarding the priority of new work.

One more project management strategy can help us combat this problem.

### The Solution: Don't Forget Your Product Manager
When you're claiming an issue or ticket, writing a new issue or even triaging a bug report, don't forget your product manager! Just because they're not sitting next to you anymore, doesn't mean you're alone in organizing your priorities.

Write issues with a clearly defined priority, and communicate asynchronously to your product manager on that issue or ticket to ask them to approve or amend that priority.

At GitHub, we label our issues with various priority levels, but we also include a section on issues in which priority is calculated and discussed by looking at the issue's visibility, impact and effort-level. Async communication happens between engineers and product managers on this section of the issue, allowing the team to reliably and accurately determine priority with the help of the product manager.

## Project Management Seems Like A Lot of Work...

As engineers, we tend to chafe at these kinds of project management strategies. We prefer to focus on what we know--writing code! It's understandable for us to feel that the approach outlined here will slow us down. Its also true that many of us are _not_ be the biggest fans of writing, preferring to contribute by getting our hands dirty with code.

I would expect many engineers reading this post to be wondering...

* Won't this slow down my velocity?
* What if I'm not a good writer? Won't it be too burdensome for me to write _so_ much?

Just the opposite. Keep reading to learn how adopting the project management mindset will help you move faster and _avoid_ burdensome writing.

## Project Management Helps You Work Less, Move Faster
The strategies outlined here will help you and your team move faster and save you from doing unnecessary work down the line. While it might feel slower to do more writing up front, rather than just diving into code, it helps create a shared understanding of your work with your product manager and engineering colleagues. This shared understanding means you can easily find engineering collaborators with whom to share tasks or engage in pair programming--helping you deliver faster. And the clear deliverable ensures that you're doing the _right_ work--no more rabbit holes (ok, _fewer_ rabbit holes), pre-launch surprises, or last-minute returns to the drawing board.

Its also important to note that your new project management responsibilities are just that--_management_ responsibilities. In other words, ensuring that a given issue or ticket has a "Deliverable" section doesn't mean that you are _always_ on the hook for completing that section. Ensuring that an issue has a "Background" section doesn't mean you have to be the one to do _all_ the research. In this way you're avoiding the mental overhead of articulating your thought process and the status of your work from scratch every time. And, by outlining these sections, you create units of work that others can plug into--you're sharing the responsibility of answering key questions and empowering your teammates to work productively as your collaborators. This allows everyone to work towards a shared understanding of the work to be done.

This is what it means to be project manager. You're facilitating the forward movement of the project, enabling collaboration and helping guarantee that goals are met.

## Conclusion
Engineers thinking like project managers makes for higher performing teams, whether you're remote and async or not. While we may initially chafe at the additional time spent writing and organizing our work, the strategies outlined here ultimately empower teams to deliver high-value work, quickly.

As engineers, we must take on the responsibilities of fighting scope creep, facilitating collaboration, and ensuring requirements are met. In this way, we participate as full members of a high-functioning team that is always on the same page, capable of moving quickly and effective at solving complex problems--all in time for your next big launch.
