# AI Is probably ready. You Weren't.

"AI made everything worse."

More production incidents. Code nobody can confidently review anymore. A demo that dazzled investors last quarter — and a system quietly on fire this one.

I hear some version of this every week. From developers, platform engineers, leads, architects. They've concluded the model isn't good enough.

It's the wrong conclusion.

AI is ready. You weren't.

---

The engineers telling me this aren't anti-AI. They're exhausted. They can see the gap between what's being promised upstairs and what their codebase can actually support — and nobody above them wants to hear that the problem isn't the model. It's the decade of shortcuts that came before it.

And the most dangerous sentence in the whole conversation is this one: *"We have to use AI. It's a business goal. Investors expect it."*

When adoption is driven by investor optics instead of engineering readiness, you don't get innovation. You get pressure. And pressure without foundation produces exactly what these teams are living through — more incidents, more confusion, more debt — wrapped in a demo that looked great on a slide.

Here's what that pressure actually looks like at 4pm on a Tuesday — not on the slide.

Cursor generates a feature. It works. Nobody on the team would have written it quite that way, and nobody fully traces how it works — but the tests are green, so it ships. The next change gets generated on top of that one. Now you're editing code you didn't write, sitting on code nobody understood, to add behavior no human ever reasoned about end to end.

Then the PR lands. Claude wrote it: 400 lines across nine files, clean description, tidy diff. And here's the quiet part nobody says out loud — *nobody can meaningfully review 400 generated lines touching nine files.* Not in the ten minutes the sprint allows for it. So the review becomes theater. An "LGTM" on a body of work no human has actually held in their head. The approval isn't a judgment anymore. It's a formality with a green checkmark.

You can hear it in how people talk now. A year ago, "does this handle the edge case?" got you "yes, because X." Today it gets you a shrug and "I don't know — Claude wrote it." Said like that settles the matter.

It doesn't settle anything. It's a confession. The person shipping the code has quietly stopped being the person who can vouch for it — and nobody decided that on purpose. It drifted there, one green checkmark at a time.

Multiply that by every engineer, every day. You haven't accelerated your team — you've automated the production of code nobody owns, at a velocity your review process was never built to absorb. "PRs merged" goes up and to the right. "Humans who understand this system" goes quietly down. The moment those two lines cross is the moment the incidents start. It's also the moment you lose the ability to debug them — because the people on call were never in the loop on what got written.

That's not an AI failure. That's a process sprinting past its own ability to verify itself, and calling the speedometer a strategy.

So let's talk about the foundation.

---

## The journey nobody wants to take

Here's how I think about the path every software organization has to walk. Not a framework I read somewhere — a pattern I keep watching play out:

**POC → Domain-Driven Design → Cloud Native → Microservices → Agentic AI**

In my experience, each stage earns the next. You can't teleport past the middle and land on the destination. And most companies sprinting toward AI are skipping straight from "we have a codebase" to "we have agents," with nothing in between.

**POC** is where you prove the idea works. Fast, dirty, disposable. The failure mode is simple: POC code ships to production, nobody throws it away, and now your foundation is built on sand.

**Domain-Driven Design** is where you force yourself to understand your own business before writing another line of code. Not "naming things better." Answering hard questions: How are your services defined? Where are the boundaries, and why? How do they communicate, and what guarantees does that communication carry? What contracts exist between them — and what happens when one is violated? Most orgs skip this because it's slow and ships no features. So they accumulate a codebase nobody fully understands, including the people who wrote it.

**Cloud Native** isn't AWS. It isn't Kubernetes. It isn't Datadog. Those are tools. Cloud Native is a doctrine — loosely coupled systems that are resilient, observable, and recoverable. And the principle I care about most is idempotency: operations you can safely repeat. If something goes wrong, you can replay, retry, and recover without making it worse. The real promise is being able to get out of trouble *faster than you got into it*. Kubernetes gives you a standard way to describe and manage workloads — genuinely useful — but that's a benefit of the tool, not the doctrine.

**Microservices** — done *after* the previous steps — is where the architecture finally breathes. Services scale independently instead of dragging everything with them. You release to a small slice of users first, and expand only if nothing breaks. Each service is simple enough that a new engineer understands it without a two-hour onboarding. No distributed monolith cosplaying as microservices. No "why did changing this break three unrelated things." Real ownership. Real boundaries. Real simplicity.

**Agentic AI** is where autonomous systems make decisions, trigger workflows, call APIs, and act on your behalf. This is where the journey pays off — but only if you actually walked it.

---

## The interface you're not thinking about

Here's what most people miss: agents don't plug into your business logic directly. They need a clean, well-defined interface to operate against. The cleanest one we have right now is an **MCP** — a Model Context Protocol server that exposes your domain's capabilities in a way an agent can consume safely and predictably. Think of it as the contract between your business and the AI acting on its behalf.

Your MCP isn't an afterthought. It's the surface through which an agent reads context, triggers actions, and gets feedback. And it inherits everything underneath it — the clean boundaries from DDD, the recoverability from Cloud Native, the independence of your microservices. If those are missing, the MCP doesn't hide the mess. It *exposes* it: an interface that surfaces ambiguity, permits unsafe operations, and hands an autonomous system the keys to a house that was never properly built.

But even a flawless MCP isn't enough on its own. An agent also needs to know *how your organization actually works*. What does a good PR look like on your team? How do you name things, structure things, escalate things? What patterns should it follow, and which should it avoid?

That's your institutional knowledge — and it has to be explicit, documented, and taught to the agent before it touches anything. It doesn't live in your repos. Scanning a codebase is not onboarding. You wouldn't hire an engineer and say "just read the code." You'd give them context, examples, conventions. Your agents need the same.

An agent that can call your APIs but doesn't understand your conventions will produce work that technically passes and culturally doesn't. Code that clears CI and confuses everyone who reads it. Decisions that are locally correct and globally off-brand. That's the line between an agent that *assists* your team and one that *creates work* for it.

So ask yourself honestly: if your agents can't reason about your domain (no DDD), can't recover from failure safely (no Cloud Native), can't operate against clean isolated services (no microservices) — what exactly is your MCP going to offer them? Automated chaos? Technical debt at scale?

---

## The shortcut illusion

I've watched this play out over and over. A team grafts an LLM onto a monolith that was never properly modeled. It works beautifully in the demo. Then it hallucinates in production, makes confident decisions on ambiguous data, and triggers side effects nobody can trace. Incidents multiply. Trust evaporates.

Then they say AI isn't ready.

Every shortcut you took compounds the moment you introduce an autonomous system. A human developer eventually senses when something feels wrong. An agent doesn't. It just keeps going — at machine speed, in the wrong direction.

The math is uncomfortable:

If your domain model is a mess, your AI inherits the mess and operates confidently inside it. If your infrastructure isn't observable, you won't know what your agents are doing until something breaks. If your services aren't independently deployable, you can't iterate on agent behavior without touching everything.

AI is a tool. A powerful one — but a tool. It won't replace your CI/CD pipeline, your QA, or your reviews. It doesn't make deployments safer, tests optional, or discipline unnecessary. Teams that treat it as a shortcut past engineering discipline are about to learn that discipline was never the bottleneck.

And if you skip the foundation, every agent query gets expensive *twice*. Expensive in tokens, because your context is bloated with the ambiguity a well-modeled system would never produce. And expensive in quality, because the output reflects the confusion it was fed. Garbage in, confident garbage out — at scale, on a subscription.

Skip the journey, and you don't get the destination. You get a very costly approximation of it.

---

## This isn't a reason to wait

I'm not telling you to avoid AI. Explore aggressively. Build prototypes. Let your engineers get their hands dirty and find out what's possible.

But be honest about which stage you're actually at — and what has to be true before you unlock the next one. The organizations that win with AI won't be the ones who rushed in first. They'll be the ones who built the foundation that makes AI *safe to trust*.

The journey isn't a detour. It's the point.

---

## And before any of it — know what you're actually selling

Before the architecture debates, the MCP design, the agent strategy — answer one question. What do your customers actually pay you for? Not what your product does. The thing that, if you got it exactly right, makes them choose you and stay.

That's your position. It's the only thing worth defending with AI.

Code is mutable. Whatever you ship today — your best engineer's work or a model's — is legacy in a few years. AI didn't change that. It just accelerated it. Your codebase was never your moat.

Your moat is your understanding of a problem, your relationships, your domain expertise, the trust you've earned with a specific kind of customer. So when you think about where AI fits, don't start with "what new things can we build?" Start with "what would make us dramatically better at the thing we already own?"

Depth compounds. Width dilutes. Customers don't buy breadth — they buy confidence that you solve *their* problem better than anyone else. Use AI to sharpen that. Win there first. Then expand from strength.

---

Most "AI isn't working for us" stories aren't AI stories. They're foundation stories.

So I'll ask the uncomfortable question: where is your org *actually* on this path — and be honest about it. Are you building an AI-powered business, or an AI-powered incident?

I'd genuinely like to know what you're seeing. 👇
