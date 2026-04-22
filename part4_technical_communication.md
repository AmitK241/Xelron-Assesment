# Part 4: Technical Communication

## Scenario Response

**Reviewer question:** "Why did you choose this specific PR over the others? What made it comprehensible to you, and what challenges do you anticipate in implementing it?"

---

I picked beets #3877 for a few reasons, and I want to be honest about both what drew me to it and where I know I could get tripped up.

The main thing that made this PR readable to me was that the change is narrow. There is one config key, one before-request hook, and a documentation update. I do not need to understand the entire beets plugin system or how MusicBrainz tagging works to reason about whether the implementation is correct. The scope is small enough that I could trace the logic end to end: config is read, a hook is registered, HTTP method is checked, request either proceeds or gets a 405. That kind of traceability matters — PRs that touch many systems at once are harder to reason about even if the individual pieces are simple.

My background with Flask also helped. I have worked with Flask's `before_request` pattern before in small projects, so I already knew that was the right mechanism to use rather than modifying each route handler separately. That recognition saved me from going down a wrong implementation path.

What makes this PR comprehensible compared to the others in the beets list is that the others touch things I would need more context for. PR #3214 involves the BPD plugin which implements an MPD protocol server — understanding whether specific protocol commands are handled correctly requires knowing the MPD 0.16 spec. PR #3279 and others involve the MusicBrainz matching logic, which has a lot of domain-specific behavior around release group disambiguation. Those are not impossible to understand, but they need more ramp-up time.

On implementation challenges: the trickiest part is probably getting the config default right. beets has its own `ConfigView` system layered on top of YAML, and a missing config key does not always behave the same way across plugins depending on how defaults are registered. If I register the default incorrectly, a config file with no `readonly` key would raise an error instead of falling back to `True`. I would test this explicitly with a minimal config before assuming it works.

The second challenge is deciding what to do with HTTP methods beyond GET, DELETE, and PATCH — specifically OPTIONS and HEAD. Getting that decision wrong could break CORS in browser-based clients. I would look at how the existing codebase handles CORS (if at all) before deciding whether to block or pass those methods through.
