# Day 21 — Portfolio & Documentation: Technical Writing & Personal Branding

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Review | **Flag:** 📌

## Brief

A repo full of scripts proves you can build things. A blog post proves you can *explain* things — and explaining clearly under no time pressure is a strong proxy for how you'll explain an incident, a design decision, or a runbook to a teammate at 2am. It's also one of the cheapest ways to get found: a good technical post on a real problem you hit and fixed ranks in search, gets linked from your GitHub profile and LinkedIn, and gives an interviewer something concrete to ask about instead of generic behavioral questions. LinkedIn is the other half of discoverability — recruiters search it by keyword before they ever read a resume, so a profile that doesn't mention the tools you've actually used effectively doesn't exist to that search.

## Writing a genuinely good technical blog post

Pick **one concept** from this phase that you understood well enough to explain without notes — not the broadest topic, the one where you actually hit a wall and got past it. A post about *one specific problem solved* reads far better than a survey of "everything I learned about Linux this month."

**Structure that works for a technical audience:**

1. **Problem/context** — what were you trying to do, and what broke or confused you? Be specific: "I had a bash script that took 40 seconds to list files in a directory with 200k entries" is a hook; "Linux file listing performance" is not.
2. **Explanation** — the underlying mechanism, at the depth needed to understand *why* the problem happened. This is where you demonstrate understanding, not just "I ran this command and it worked."
3. **Concrete example** — actual commands, actual output, ideally from a real terminal session. Readers skim prose and read code blocks; make the code blocks carry the weight.
4. **Takeaway** — one or two sentences a reader can walk away with and reuse, even if they forget everything else in the post.

Worked outline, using a Day 19 topic as an example:

```markdown
# Why My Script Was Silently Hanging (and How strace Found It)

## The problem
A backup script that normally finished in under a second started taking
30+ seconds with no error output — no crash, no obvious cause in the logs.

## Digging in with strace
`strace -f -T ./backup.sh` showed the process spending nearly all its time
in a single `read()` call against a named pipe that nothing was writing to.

    read(3, ...)  = 0 <29.812340>

## Why this happened
The script assumed a subprocess would close its stdout the moment it
finished, but it inherited a file descriptor from a parent process that
kept the pipe open — so `read()` blocked waiting for an EOF that never came.

## The fix
Explicitly closing inherited descriptors before the subprocess call
resolved it. One line, `close_fds=True` in the Python `subprocess.Popen`
call, cut runtime from 30s to under 1s.

## Takeaway
When a script "hangs" with no error, suspect a blocking read on a
pipe/socket before you suspect CPU — `strace -f -T` will show you exactly
which syscall it's stuck in and for how long.
```

**Why "explain to a peer, not a marketing audience" works:** marketing language ("revolutionary," "game-changing," "10x your productivity") is an instant signal of inexperience to a technical reader — nobody who actually debugs systems for a living talks that way about a `strace` session. Write the post you'd want to read if you hit the exact same problem at 11pm and searched for it. Plain, specific, and slightly informal beats polished and vague every time.

**Where to publish for free:**

- **[dev.to](https://dev.to)** — low-friction, built-in audience, good SEO, tags let you reach people searching for the exact problem you solved.
- **[Hashnode](https://hashnode.com)** — similar to dev.to, supports a custom domain for free if you want your posts to live at your own URL.
- **Your own GitHub Pages / MkDocs site** (built in the companion file) — full control, doubles as more content for your portfolio repo, but starts with zero built-in audience, so cross-post or link to it from dev.to/Hashnode/LinkedIn.

A reasonable approach: write once, publish to your own MkDocs site as the canonical version, then cross-post to dev.to or Hashnode with a "originally published at [your site]" link back — you get the audience of the platform and the SEO/ownership benefit of your own domain.

## Updating LinkedIn with skills

**Skills section — align with how recruiters actually search.** Recruiters query LinkedIn Recruiter by exact keyword/skill tags, not free text. A skills section listing "Problem Solving" and "Team Player" is invisible to that search; one listing `Linux`, `Bash`, `AWS (EC2, IAM, S3)`, `Docker`, `Kubernetes`, `Terraform`, `CI/CD`, `Python`, `Ansible`, `Git` is what actually surfaces you. List tools and technologies you can speak to in an interview without hesitation — padding the list with things you touched once in a tutorial backfires the moment someone asks a follow-up question.

**Featured section** — pin your portfolio repo and your blog post directly on your profile, above the fold, so a visitor doesn't have to dig through your "About" or experience sections to find proof of work:

- Add the GitHub repo URL with a custom title ("DevOps Fundamentals — 20+ scripts and docs from a hands-on Linux/AWS/Kubernetes learning path").
- Add the blog post URL once published.
- If you deployed the MkDocs site, add that too — it's the most "finished product"-looking artifact of the three.

**About summary — reflect real work, not buzzwords.** Compare:

> Generic (avoid): *"Passionate and driven DevOps enthusiast with a strong desire to leverage cutting-edge technologies to drive digital transformation and deliver innovative solutions."*

> Concrete (use instead): *"Built a self-directed 100-day DevOps learning path covering Linux internals, AWS automation (EC2/IAM/S3), containerization, Kubernetes, Terraform, and CI/CD — documented end-to-end in a public GitHub repo with 20+ working scripts and per-topic write-ups. Recently wrote about debugging a silently hanging bash script with strace [link]. Looking for [target role] where I can keep building on that hands-on foundation."*

The second version names specific tools, links to verifiable proof, and states what you're looking for — everything the first version gestures at without saying.

## Points to Remember

- One specific problem, explained well, beats a broad survey post — pick the narrowest topic you can explain confidently end-to-end.
- Structure: problem/context → explanation → concrete example (real commands/output) → takeaway. Code blocks carry more weight than prose for a technical audience.
- LinkedIn's skills section is a search index, not a personality description — list the actual tools you used, in the terms recruiters search for.
- Use the Featured section to put your GitHub repo and blog post directly on your profile — don't make visitors dig for proof of work.
- Write your About summary around what you actually built, with a link, not around adjectives.

## Common Mistakes

- Writing a blog post that only lists commands with no explanation of *why* — reads as a copy-pasted man page, not evidence of understanding.
- Choosing a topic too broad to cover well ("Everything about Linux") instead of one specific, well-understood problem.
- A copy-pasted, generic LinkedIn "About" summary full of buzzwords ("passionate," "synergy," "cutting-edge") with zero specific tools, projects, or links — reads as filler to anyone screening candidates at volume.
- Listing skills on LinkedIn you can't actually speak to in an interview — a skills list is an implicit invitation for follow-up questions.
- Publishing a blog post or portfolio link nowhere visible on the LinkedIn profile itself, so the work exists but nobody screening the profile ever finds it.
