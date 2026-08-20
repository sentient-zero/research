*Published 2026-08-20. This repository holds the canonical copy.*

*Article text is licensed CC BY 4.0. Attribution required.*

---

# There Is No Parameterized Query for a Prompt

A few weeks ago I saw a comment claiming that AI pentesting is mostly API pentesting wearing a different hat.

That's a reasonable place to land if you're working from what's easy to find right now. A lot of the introductory AI security material is recon and API-layer work, and if that's your window into the field, the conclusion follows. It's still wrong, and it's wrong in a way that costs people real findings, so I want to walk through where the instinct holds and where it stops.

## What the take gets right

Most LLM deployments in the wild are a normal application with a model bolted into the middle of it. There's an API in front, a database behind, auth somewhere, and a pile of business logic wrapped around the whole thing. All of your existing instincts apply to that. Broken object-level authorization still gets you other tenants' conversations. Missing rate limiting still lets you burn someone's inference budget or brute force a session. Overly permissive CORS is still overly permissive. Secrets still end up in client-side config.

If anything, the model makes the conventional stuff worse, because these systems get shipped fast by teams who are focused on whether the output is any good and not on who is allowed to call the endpoint.

A concrete example makes the case better than the generalization does. In August 2025, Check Point disclosed MCPoison in Cursor, CVE-2025-54136, in which Cursor asked for approval the first time an MCP server configuration was added and then bound that approval to the entry's key name, never revalidating the command and arguments underneath it. An attacker plants a benign config in a shared repo, waits for someone to approve it once, and swaps the command for a reverse shell later, with no second prompt, and because the file is re-read every time the project opens or the repo syncs, the payload fires again on each one.

What that amounts to is trust bound to a mutable name, with no revalidation of what the name points at. It got a CVE, it's in an AI development tool, it involves MCP, and it has nothing whatsoever to do with the model. Check Point's own framing was to ask whether the trust model accounted for change over time, which is a question you could put to a package manager or an update channel without changing a word. A tester who has never touched an LLM in their life catches that bug, because it's the same bug we've been finding in package managers and update mechanisms for twenty years.

So if the argument is that a large share of what currently gets labeled AI security is conventional security in a new context, that's correct. On most AI-enabled applications I'd expect the majority of findings to be exactly that, and anyone telling you your existing skill set doesn't transfer is selling something.

## Where it stops working

The wrapper is not the whole target, and once you're testing the model itself, the first thing you run into is prompt injection, which is where the analogy breaks.

The obvious response is that injection is the oldest bug class we have. SQL injection, command injection, LDAP injection, we've been finding these forever, so this is just a new sink. I understand why it looks that way, and the reason it isn't the same needs to be stated precisely rather than asserted.

SQL injection is solvable rather than merely mitigable, and the difference is the whole point. Prepared statements work because the database protocol can carry the query structure and the parameter values on separate channels: the parser is handed the statement, then handed the data, and it never has the option to interpret one as the other. Because the boundary is enforced below the application, it doesn't depend on anyone predicting what the input will look like. Get parameterization right and the entire bug class is gone from that code path.

There is no parameterized query for a prompt. The system instructions, the user's message, the retrieved document, the tool output, and the contents of whatever file the agent just read all arrive at the model as one token sequence, with no control channel and no data channel, just text and the model deciding what to do about it.

The second half of that list is where this gets dangerous. If the only untrusted text in the stream came from the person typing, you would have something resembling a conventional injection problem: one input, one attacker, a boundary you could at least reason about. But retrieved documents, tool output, and files the agent opens are not written by the user and they are not written by you. They are written by whoever wrote them, and the model reads them on the user's behalf because reading them is the entire point.

That is indirect prompt injection, and the distinction matters more than the shared word suggests, because the attacker never talks to the model at all. They put text somewhere the model was always going to go, and wait. Both of the bugs I am about to walk through work this way, and so does most of what is currently being found in the wild.

Everything shipped so far works around that instead of fixing it. You can wrap untrusted content in delimiters, tell the model to disregard instructions found in retrieved data, or run a classifier over the input and try to catch injection attempts before they land. Every one of those is a request the model can decline, or a filter with a bypass someone hasn't found yet.

EchoLeak demonstrates the point at production scale. Aim Security disclosed CVE-2025-32711 in June 2025, a zero-click indirect prompt injection in Microsoft 365 Copilot that Microsoft, as the CNA, rated 9.3 critical: one crafted email, and Copilot would pull internal content and send it to an attacker-controlled server without relying on any specific user behavior. Microsoft was not asleep here, and had a classifier deployed specifically to catch cross-prompt injection attempts.

How the researchers got past that classifier matters more than the classifier itself, because it is not a technical bypass at all. They wrote the email as though the instructions were addressed to the human recipient, and never mentioned AI, assistants, or Copilot anywhere in it, which turned out to be sufficient. A filter trained to recognize text aimed at a model does not fire on text that reads like an ordinary memo, and the model follows it anyway, because to the model there was never a difference. Every input is instructions if it parses as instructions.

From there the chain used reference-style Markdown to dodge link redaction, an auto-fetched image to trigger the request without a click, and a Teams proxy that the content security policy already permitted.

That chain divides evenly between two skill sets. Half of it is the traditional web security you already know: CSP analysis, redirect abuse, exfiltration over an allowed origin. The other half is the part where you convince a model to go read something it has access to and put it in a URL, and no amount of API testing experience teaches you how to do that or how to reason about why it worked.

Your existing skills get you into the room, in other words, and they don't get you through the door, because the thing on the other side has no boundary to test. You're not proving a control holds. You're mapping how far it bends and under what conditions, and then doing it again next week when the model gets updated and the answers change.

## From leaking to executing

EchoLeak is a read-side problem: the model has access to things, you convince it to go get them, and the data comes out. That's serious, but the model itself is only ever producing text.

Agentic deployments change the stakes, because the model has hands. It can write files, run commands, call tools, so the same injection that made Copilot leak an email now makes something execute.

CurXecute is the cleanest example of what that looks like in practice. Aim Security disclosed it on August 1, 2025 as CVE-2025-54135, affecting Cursor, and the delivery mechanism is what separates it from anything in the API-testing playbook, because the payload came from a Slack message.

Cursor's agent had a Slack MCP server connected, which is a completely ordinary thing to do. An attacker posts a message to a public channel the agent can reach, and nothing happens yet. The trigger is the victim doing something entirely reasonable, which is asking the agent to use its Slack tools and summarize their messages. The agent reads the channel, because reading it is the entire point of connecting Slack, and the message contains instructions to improve the Model Context Protocol server config at `~/.cursor/mcp.json`, which is where Cursor stores the commands it is allowed to launch.

The attacker's action was posting text in public. The victim's action was asking for a summary. Neither party did anything a security control would flag, and there is no request you can fuzz here: no parameter, no endpoint, no auth boundary that failed. The attacker put text in a place the target was already going to look, and that constitutes the entire initial access step.

The rest of it is a chain, and both halves matter. Cursor did ask for approval on that file, and two conventional flaws made the approval meaningless. Editing a dotfile required approval, but creating one that didn't exist yet did not, so the asymmetry handed you a free write. And the ordering was backwards: in Aim Labs' description, when the agent suggests an edit to mcp.json the edit has already landed on disk, and the command executes even if the user rejects the suggestion, which makes the dialog decoration.

Pull either half out and there's no CVE. Without the injection, you never reach the file, because nothing else in that workflow lets a stranger on Slack write to a developer's home directory. Without the file-approval and ordering bugs, the injection gets you a config that sits there waiting for a click that never comes.

That interaction is the answer to the original comment, because it isn't that AI security replaces what you already do; it's that the interesting findings live in the seam between the two, and you only find the seam if you can work both sides of it. A pure app tester reads that MCP config write and sees a permissions bug worth a medium. Someone who only thinks about models sees an injection with no impact. The severity comes from understanding that a Slack message reached a file write that reached command execution, and being able to hold that entire path in your head at once.

Cursor fixed the file handling, and they did not fix, and cannot fix, the fact that the agent has to read untrusted content to be useful at all. That part isn't a bug. It's the product.

## What the record actually says about these three bugs

If this were really API testing with a hat on, the taxonomies would have absorbed it by now. What they have done instead is more interesting than failure, and it took me three passes at the primary sources to stop getting it wrong.

Start with the scores, because neither of these bugs agrees with itself. On CurXecute, NIST rated 9.8, GitHub as the CNA rated 8.5, and the discovering researchers and most press say 8.6. On EchoLeak, Microsoft as the CNA rated 9.3 and NIST rated 7.5, a mismatch NVD flags on the page, with Microsoft scoring the attack as scope changed and NIST as scope unchanged. Those are not arithmetic disagreements. They are CVSS being asked a question it was never built to answer: when the delivery mechanism is an email the target's assistant reads on the target's behalf, or a message in a public channel the target's agent summarizes on request, does the attacker have privileges? Did scope change? Traditional metrics assume you can point at the interface being attacked, and here the interface is a language model's willingness to believe things, which is why competent parties looking at the same bug land two or three points apart.

The CWE assignments settle the question for anyone who thinks the taxonomy has this handled, though not in the direction you might expect.

Pull the three CVEs in this article and check what they were classified as. EchoLeak: Microsoft assigned CWE-77, command injection, then revised it in February 2026 to CWE-74, the generic injection parent. CurXecute: CWE-78, OS command injection, plus CWE-829, inclusion of functionality from an untrusted control sphere. MCPoison: CWE-78 as well, plus CWE-494, download of code without integrity check.

Set that against the argument this article has been making. MCPoison is the bug I opened with as the one that genuinely reduces to conventional security, trust on first use with a revalidation gap, catchable by a tester who has never touched a model. CurXecute is the bug I spent a section on as the one that genuinely does not, a prompt injection from a public Slack channel chained through a file-write asymmetry into command execution. Two bugs this whole article exists to distinguish, and the catalog gives them the same primary weakness. The secondaries do differ, and to be fair they differ sensibly, with untrusted functionality in one case and unverified code in the other, but neither of those is prompt injection either. The classification separates the two bugs along an axis that has nothing to do with what makes one of them novel.

The obvious response is that the CNAs got lazy, because CWE-1427, improper neutralization of input used for LLM prompting, has existed since 2024, submitted in June of that year and published that November, well before any of these. I assumed that was the story until I read the entry, and it does not hold up.

The mapping guidance on 1427 says that if the core concern is generating prompts from third-party sources that should not have been trusted, and it names indirect prompt injection as that case, a different CWE might be needed. EchoLeak and CurXecute are both indirect prompt injection, so MITRE's own guidance points them away from the one entry that looks like it was written for them.

The assignments are therefore defensible. CWE-1427 sits as a child of CWE-77, which is what Microsoft used first, and the revision to CWE-74 moved one further step up the same branch rather than sideways into nothing, while CWE-78 is a sibling under that same parent. Nobody reached for the wrong shelf.

Then I went looking for the right shelf, and this is where the article got harder to write.

CWE-1446 is the category for weaknesses uniquely applicable to AI and ML, and as of CWE 4.20 it has four members: adversarial input perturbations, improper validation of generative AI output, the 1427 prompt injection entry, and insecure setting of inference parameters. There is no entry for indirect prompt injection, nothing covering untrusted third-party content becoming instructions, which is the mechanism behind both EchoLeak and CurXecute and the reason this article exists.

I expected that to be an oversight, and it is not. The category carries a Research Gap note, added in April 2026, recording that the CWE AI Working Group has discussed at length how hard it is to separate common AI and ML attacks from the underlying weaknesses, and that much of the recent research has concentrated on the attacks rather than the weaknesses. Then it makes the strongest argument against everything I have written so far.

MITRE's position, stated in that note, is that from a CWE perspective the control and data distinction is "not necessarily as deep as currently considered" by the AI and ML community, on the grounds that weaknesses are characterized by insecure behavior regardless of whether that behavior came from design, from code, from configuration, or from data.

That is the working group looking directly at the claim in section two, that instruction and data share one channel and that this is a difference in kind rather than degree, and declining to accept it as a basis for classification. The mapping guidance follows from it: mappers are told to ask whether a weakness is truly unique to AI, in which case a higher-level class may still apply, or whether it is a general software weakness that happens to appear in AI software. Microsoft choosing CWE-74 and GitHub choosing CWE-78 is not the taxonomy failing, but the taxonomy working as designed.

I think they are right about classification, and I don't think it touches the argument.

CWE classifies root causes, and it does not classify whether a fix exists, which are two things that come apart here in a way they do not for most bug classes. Grant that prompt injection is a member of the injection family, that the control and data distinction is a matter of degree, that every one of these assignments is correct. You are still left with the thing this article is actually about, which is that SQL injection has a structural remediation and prompt injection does not. Prepared statements end the bug class in a code path, and nothing ends this one. Microsoft deployed a classifier and EchoLeak went around it. Cursor patched the file handling and could not patch the reason the agent reads a stranger's message. Cato is now disclosing across every popular coding agent because patching them one at a time was not converging.

None of that is a claim about taxonomy, and the Research Gap note does not contradict any of it. Two bugs can share a weakness class and still differ in the only respect a tester cares about, which is whether the finding you write closes the hole or just describes where it currently is.

There is a second answer, and it does not come from me. Aim Labs made it in June 2025, a year before MITRE wrote that note, in the EchoLeak writeup itself: they classify their own chain as indirect prompt injection under LLM01 and then argue that protecting AI applications requires finer granularity than the current frameworks provide. Their analogy is the one I would borrow, which is that buffer overflow describes the family perfectly well, and naming the stack overflow sub-family specifically is what made stack canaries possible. Granularity in the name is what lets you build the targeted mitigation. They went on to coin a term for their own case, LLM Scope Violation, meaning untrusted input that causes the model to reach for trusted data already in its context.

That is a direct rejoinder to the working group's position. MITRE says the control and data distinction is not deep enough to justify a separate class, and Aim Labs says the distinction is what a mitigation would have to be built against, pointing at a case in security history where exactly that sequence played out.

What makes it more than a debate between two parties is that Microsoft, on the other side of the same incident, reached for the same corner of history. Their own writeup on defending against indirect prompt injection lists stack canaries, ASLR, CFG, and DEP as the model for their approach: mitigations that stop memory safety bugs from becoming exploits rather than eliminating the bugs. They say plainly that they do not rely on blocking every injection, and that deterministic detection of indirect prompt injection remains an open research problem. Two organizations, adversarial to each other in this disclosure, independently landed on memory-safety mitigation as the closest available precedent, and nobody reached for prepared statements, because prepared statements are what you cite when the class is solvable.

The program is not internally settled either. The CVE Program published guidance in February 2025 telling CNAs that 1427 is generally most appropriate when other weaknesses are present, which is precisely the chain shape both bugs have, while the mapping note on 1427 pushes the indirect case away from it. Both documents are cited on the same category page.

What the classification does affect is you, operationally. If your triage runs on CWE, all three of these are injection bugs of a kind people have been finding since the nineties, and nothing in the record distinguishes the one that a conventional tester would catch from the two that needed someone who could work both layers. Microsoft's own description of EchoLeak calls it AI command injection. The classification does not, and by MITRE's current reasoning, it should not.

## What OWASP had to invent

The CWE argument is about three specific bugs, and this one does not depend on them.

OWASP had to add a separate list. The LLM Top 10 exists as a separate list from the API Top 10 and the web Top 10, and the reason is that several entries have no counterpart in either. OWASP's own stated rationale for putting prompt injection at LLM01 is that language models process instructions and data in the same channel with no clear separation, which is the argument from two sections ago in the framework authors' words, and they built a separate list on top of it. OWASP and MITRE disagree about how much weight that observation can carry, which you should know before citing either one as settled.

Consider LLM04, Data and Model Poisoning, where there is no request to attack at all. Nothing is malformed, no boundary fails, no input gets past a filter. Someone contributed to a dataset, or compromised a fine tuning pipeline, or seeded content a scraper would eventually collect, and the result is a statistical property of a binary artifact that behaves normally until a specific trigger appears. Every technique you own assumes a live system you can send something to and observe, and this one was over before you got there.

Or take LLM08, Vector and Embedding Weaknesses, where you recover source text from vectors that were never supposed to be reversible. The vector store itself might be an ordinary database with ordinary authorization problems, and finding those is real work you already know how to do, but reconstructing the original documents from the embeddings is a different job requiring different knowledge, and no amount of API experience gets you there.

LLM06, Excessive Agency, is the one that fools people, because it looks like authorization and is not. Authorization asks whether an identity is permitted to perform an action, which is a question with a testable answer you can enumerate. Excessive agency asks whether a non-deterministic component will choose to perform an action given some input, across an input space you cannot enumerate, with an answer that changes when the model does. You are not verifying that a control holds. You are estimating a distribution.

MITRE ATLAS closes the section, and it makes the point better than I can, because it draws the line for you.

ATLAS is the ATT&CK-style matrix for AI systems, and its legend marks every tactic and technique adapted from ATT&CK with an ampersand, a single convention that turns the matrix into a map of exactly the argument this article has been making. Of the sixteen tactic columns, fourteen carry the mark: Reconnaissance, Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, Command and Control, Exfiltration, and Impact, all inherited, all things you already do. Two do not carry it, and AI Model Access and AI Attack Staging are native, with no ATT&CK ancestor to inherit from.

The same split runs inside the columns, where Phishing, Valid Accounts, Drive-by Compromise, Command and Scripting Interpreter, and User Execution are all marked as adapted, while LLM Prompt Injection, RAG Poisoning, AI Agent Context Poisoning, AI Agent Tool Poisoning, Training Data Poisoning, LLM Jailbreak, Craft Adversarial Data, and Extract LLM System Prompt are not marked at all, because there was nothing to adapt them from.

That is the boundary drawn by a third party with no stake in this argument. The ampersands mark the parts of AI red teaming that conventional offensive security already covers, and there are a lot of them. The unmarked entries mark the parts it does not, and no amount of API testing experience generates that column.

One entry stands on its own. Under both Initial Access and Persistence, ATLAS lists Prompt Infiltration via Public-Facing Application, which is indirect prompt injection, named as a technique, in a MITRE framework. Set that next to the other thing established a few paragraphs ago: on the attack side MITRE describes this fluently and has for a while, and on the weakness side there is no CWE for it, with the Research Gap note explaining why in MITRE's own words, saying that much of the recent research has focused on the attacks. MITRE's attack-side and weakness-side vocabularies are not at the same place, and that discrepancy is precisely what this article is about.

## What this actually means for you

The original comment was right about the part it could see. Most of what gets called AI security today is conventional security, and your existing skills are worth more in this space than anyone will tell you. MCPoison was a CVE in an AI tool that a competent tester with no model experience would have caught.

CurXecute needed both, though. A Slack message reached a file write reached command execution, and you only see that whole path if you can work the model layer and the application layer at the same time, which means neither specialist alone gets there.

Prepared statements ended SQL injection as a bug class because the protocol could separate control from data, and nothing separates instruction from data in a prompt. Microsoft shipped a classifier against exactly this and EchoLeak chained around it. Cursor patched the file handling in 1.3.9 and could not patch the reason the agent was reading a stranger's message in the first place, because that isn't the bug, it's the product working correctly.

If you want to know whether that's an argument or an observation, watch what happened to the people who found these bugs.

The team that disclosed EchoLeak in June 2025 and CurXecute that August was Aim Security, and in June 2025 they had also demonstrated the same primitive against Atlassian, prompt injection delivered as a Jira support ticket. They are now Cato AI Labs, and in July 2026 they published DuneSlide, two more Cursor RCEs at CVE-2026-50548 and CVE-2026-50549, again zero-click, again starting from content the agent ingested from an MCP server or a web search result, this time escaping the sandbox through a path canonicalization fallback.

Their own framing repays close reading. Cato writes that if these had been isolated incidents they might have treated each as its own vulnerability, and that they are instead disclosing across every popular coding agent because, in their words, "a more systemic approach to protection is required."  That is the same conclusion this article argues for, reached from the other direction by the researchers with the most hands-on exposure to the individual bugs. They started roughly where that LinkedIn comment starts, and four disclosures later they are not there anymore.

One more detail from that timeline says more than the rest of it combined. When Cato reported DuneSlide in February 2026, Cursor rejected it four days later on the grounds that its threat model did not cover misuse of MCP servers, including standard ones, which was not a disagreement about severity so much as an admission that the injection path was not in the model at all. It took an escalation to get the reports reopened.

So no, AI pentesting isn't API pentesting wearing a different hat. It's API pentesting standing next to a thing that has no hat, no head, and no boundary, and asking you to prove something about it anyway.

---

# References

Status legend. **Read at source** means the page was read in full. **Archived**
means the original URL no longer resolves and the Wayback capture was read
instead. **Search-confirmed** means multiple independent results agree but the
source page was not read.

No row currently carries a weaker status than search-confirmed.

Two Aim Labs posts now redirect to Cato Networks following the acquisition. The
archived captures are the versions actually read, and are cited as such.

## CVE-2025-32711 (EchoLeak)

| Source | URL | Status |
|---|---|---|
| NVD entry | https://nvd.nist.gov/vuln/detail/CVE-2025-32711 | **Read at source.** Published 06/11/2025, last modified 06/17/2026, source Microsoft. NIST 7.5 HIGH `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`. CNA Microsoft 9.3 CRITICAL `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:L/A:N`. NVD flags the mismatch. CWE-74, changed from CWE-77 on 02/20/2026. Microsoft's description calls it AI command injection. Both vectors carry `UI:N`. |
| Aim Labs, "Breaking down EchoLeak" (archived) | https://web.archive.org/web/20251209161157/https://www.aim.security/aim-labs/aim-labs-echoleak-blogpost | **Read at source (archived).** Itay Ravia, 11 June 2025. The four-step chain: XPIA bypass by phrasing the email as instructions to a human and never mentioning AI or Copilot; reference-style Markdown evading link redaction; reference-style image evading the same; CSP bypass via a Teams `asyncgw` endpoint, after a SharePoint EmbedService variant needing an accepted invitation. Also RAG spraying and a reference-mention bypass. Coins **LLM Scope Violation**. Argues for finer framework granularity via buffer overflow to stack overflow to stack canary. Section 4 uses that argument. |
| Same post, republished by Cato | https://www.catonetworks.com/blog/breaking-down-echoleak/ | **Read at source.** Where the original Aim URL now resolves. **Not the same text as the archived Aim version.** Dated 31 May 2025 against the archived post's 11 June 2025, and several claims are stronger: "the first practical attack" rather than "a novel practical attack"; complete compromise of data integrity rather than exfiltration from context; a flat statement that no admin configuration or user behavior could have prevented exploitation, where the Aim version's Q&A points at DLP tags and sensitivity-label restrictions; GPT-4 named rather than GPT. The Cato republication appears to carry the earlier draft and the Wayback capture the later, softened one. The article relies on none of the differing claims, but a reader following either citation will find different text. |
| AAAI paper, Reddy and Gujral | https://doi.org/10.1609/aaaiss.v7i1.36899 | **Search-confirmed.** Proceedings of the AAAI Symposium Series 7(1), 303-311, AAAI Fall Symposium Series 2025. Peer-reviewed; cite this rather than the preprint. Abstract states the four-bypass chain. |
| Same, preprint | https://arxiv.org/abs/2509.10540 | **Search-confirmed.** Preprint version. |
| MSRC, "How Microsoft defends against indirect prompt injection attacks" | https://www.microsoft.com/en-us/msrc/blog/2025/07/how-microsoft-defends-against-indirect-prompt-injection-attacks/ | **Read at source.** Andrew Paverd, 29 July 2025. Note the correct host is `microsoft.com/en-us/msrc/`, not `msrc.microsoft.com`, which an earlier draft carried. Splits defenses probabilistic versus deterministic, calls deterministic detection of indirect prompt injection an open research challenge, states the approach does not rely on blocking all injections, and cites stack canaries, ASLR, CFG and DEP as the model. All deterministic wins claimed are impact-side. Published after EchoLeak was patched, so it describes posture rather than what was deployed in June. |
| Varonis dissent on zero-click | (see below) | **Resolved.** Varonis argues the victim must prompt Copilot with matching content. The article now uses Aim Labs' own formulation, that the attack does not rely on specific user behavior, with RAG spraying as the mechanism. That answers the objection on the merits rather than by appeal to `UI:N`. |
| Hack The Box | https://www.hackthebox.com/blog/cve-2025-32711-echoleak-copilot-vulnerability | **Search-confirmed.** Secondary. |
| SOC Prime | https://socprime.com/blog/cve-2025-32711-zero-click-ai-vulnerability/ | **Search-confirmed.** Secondary. Useful framing of LLM scope violation. |
| Sentra | https://sentra.io/blog/copilot-echoleak-prompt-injection | **Search-confirmed.** Secondary. |
| promptfoo LLM security DB | https://www.promptfoo.dev/lm-security-db/vuln/echoleak-zero-click-data-exfiltration-a87757e2 | **Search-confirmed.** Secondary. |

## CVE-2025-54135 (CurXecute)

| Source | URL | Status |
|---|---|---|
| NVD entry | https://nvd.nist.gov/vuln/detail/CVE-2025-54135 | **Read at source.** Published 08/04/2025, source GitHub. NIST 9.8 CRITICAL `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`. CNA GitHub 8.5 HIGH `CVSS:3.1/AV:N/AC:H/PR:L/UI:N/S:C/C:H/I:H/A:H`. Mismatch flagged. CWE-78 and CWE-829. Affected below 1.3.9, fixed in 1.3.9. Description confirms the section 3 dotfile asymmetry verbatim. |
| Aim Labs, "When Public Prompts Turn Into Local Shells" (archived) | https://web.archive.org/web/20251209161431/https://www.aim.security/aim-labs/aim-labs-curxecute-blogpost | **Read at source (archived).** Ofir Abu, 1 August 2025. Cursor executes any new entry in `~/.cursor/mcp.json` with no confirmation; a suggested edit lands on disk and executes before approve or reject. Attacker posts to a public Slack channel; victim asks to summarize their Slack messages; payload fires. Aim Labs states 8.6 and fix in 1.3. **Reported privately 7 July 2025, patch merged 8 July, published 1 August.** Do not describe 1 August as the disclosure date. The original URL now redirects to a Cato blog category index rather than to the post, so the Wayback capture is the only usable citation. |
| GitHub advisory GHSA-4cxx-hrm3-49rm | https://github.com/cursor/cursor/security/advisories/GHSA-4cxx-hrm3-49rm | **Search-confirmed.** Sole reference on the NVD entry. |
| Tenable RSO FAQ | https://www.tenable.com/blog/faq-cve-2025-54135-cve-2025-54136-vulnerabilities-in-cursor-curxecute-mcpoison | **Search-confirmed.** Covers both Cursor CVEs. |
| Tenable CVE page | https://www.tenable.com/cve/CVE-2025-54135 | **Search-confirmed.** Reference aggregation. |
| BleepingComputer | https://www.bleepingcomputer.com/news/security/ai-powered-cursor-ide-vulnerable-to-prompt-injection-attacks/ | **Search-confirmed.** Secondary. Source of the 8.6 figure. |
| The Hacker News | https://thehackernews.com/2025/08/cursor-ai-code-editor-fixed-flaw.html | **Search-confirmed.** Secondary. |
| CyberScoop | https://cyberscoop.com/cursor-ai-prompt-injection-attack-remote-code-privileges-aimlabs/ | **Search-confirmed.** Secondary. |
| **Fixed-version dispute** | n/a | **Open, unresolved.** Aim Labs says CurXecute fixed in 1.3. Check Point says MCPoison fixed in 1.3. NVD says CurXecute affected below 1.3.9, fixed in 1.3.9. The article states 1.3.9 citing NVD. Sources genuinely disagree. |

## CVE-2025-54136 (MCPoison)

| Source | URL | Status |
|---|---|---|
| NVD entry | https://nvd.nist.gov/vuln/detail/CVE-2025-54136 | **Read at source.** Published 08/01/2025, last modified 08/24/2025, source GitHub. CWE-78. Advisory GHSA-24mc-g4xr-4395. Affects 1.2.4 and below. |
| Check Point Research, "MCPoison" | https://research.checkpoint.com/2025/cursor-vulnerability-mcpoison/ | **Read at source.** Andrey Charikov, Roman Zaikin, Oded Vanunu, 5 August 2025. Two corrections to earlier drafts: the file is `.cursor/rules/mcp.json`, workspace-scoped, **not** the home-directory path CurXecute targets; and trust binds to the MCP **key name**, with `command` and `args` never revalidated on change. Impact is persistent RCE, re-executed on every project open and repo sync. Their stated motivation was whether the trust model accounted for change over time. Disclosed 16 July 2025, fixed in 1.3 on 29 July. |
| GitHub advisory GHSA-24mc-g4xr-4395 | https://github.com/cursor/cursor/security/advisories/GHSA-24mc-g4xr-4395 | **Read at source.** URL confirmed correct. Published 1 August 2025 by hmwildermuth, titled "Modification of MCP Server Definitions Bypasses Manual Re-approval." **Assigns CWE-78 and CWE-494, download of code without integrity check. NVD lists only CWE-78, so the two records disagree.** Section 4 uses the advisory's pair. CVSS `CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:H`, High, which computes to the 7.2 Tenable reports. Note `PR:H`: the attacker needs write access to the repository, where CurXecute was scored `PR:L`. Affected below 1.2.4, patched 1.3. Reporter credited as @chaandrey. Remediation is that Cursor now requires approval on every modification to an mcpServer entry, not only on addition. |
| Tenable RSO FAQ | https://www.tenable.com/blog/faq-cve-2025-54135-cve-2025-54136-vulnerabilities-in-cursor-curxecute-mcpoison | **Search-confirmed.** CVSS 7.2, approved-once-never-revalidated mechanism. |

## Frameworks and taxonomy

| Source | URL | Status |
|---|---|---|
| CWE-1427 | https://cwe.mitre.org/data/definitions/1427.html | **Read at source.** Submitted 2024-06-21 by Max Rattray, Praetorian. Published CWE 4.16, 2024-11-19. Abstraction Base, mapping ALLOWED, ChildOf CWE-77, MemberOf CWE-1446. **The mapping note says a different CWE may be needed where prompts are generated from untrusted third-party sources, naming indirect prompt injection.** This killed an earlier version of section 4. Observed examples are all direct-injection chains. |
| CWE-1446 | https://cwe.mitre.org/data/definitions/1446.html | **Read at source.** Category, Weaknesses Specific to AI/ML Technology. Submitted 2026-04-27, CWE 4.20. Mapping PROHIBITED. Four members: CWE-1039, CWE-1426, CWE-1427, CWE-1434. **No indirect prompt injection entry; the gap is confirmed.** Research Gap note states the control/data distinction is not necessarily as deep as the AI/ML community considers it. Section 4 quotes and answers this. |
| CWE-74 | https://cwe.mitre.org/data/definitions/74.html | **Search-confirmed.** Current EchoLeak assignment. |
| CWE-77 | https://cwe.mitre.org/data/definitions/77.html | **Search-confirmed.** Original EchoLeak assignment, parent of 1427. |
| CWE-78 | https://cwe.mitre.org/data/definitions/78.html | **Search-confirmed.** Assignment on both Cursor CVEs. |
| CWE-829 | https://cwe.mitre.org/data/definitions/829.html | **Search-confirmed.** Second assignment on CurXecute. |
| CVE Program AI guidance, February 2025 | (cited on the CWE-1446 page) | **Search-confirmed.** Tells CNAs 1427 is generally most appropriate where other weaknesses are present. Points the opposite way from the 1427 mapping note. |
| OWASP Top 10 for LLM Applications | https://genai.owasp.org/llm-top-10/ | **Search-confirmed.** 2025 edition v2.0, published 18 November 2024, OWASP GenAI Security Project. LLM04 Data and Model Poisoning, LLM06 Excessive Agency, LLM08 Vector and Embedding Weaknesses. Names confirmed across multiple sources; page not read. |
| MITRE ATLAS | https://atlas.mitre.org/ | **Screenshot read 2026-08-19, tactic-column count confirmed by the author on the live site the same day.** The matrix legend marks tactics and techniques adapted from ATT&CK with a superscript ampersand. Fourteen of the sixteen tactic columns carry it; **AI Model Access** and **AI Attack Staging** do not. Marked techniques include Phishing, Valid Accounts, Drive-by Compromise, Command and Scripting Interpreter, User Execution. Unmarked: LLM Prompt Injection, RAG Poisoning, AI Agent Context Poisoning, AI Agent Tool Poisoning, Training Data Poisoning, LLM Jailbreak, Craft Adversarial Data, Extract LLM System Prompt. ATLAS also lists **Prompt Infiltration via Public-Facing Application** under both Initial Access and Persistence, which is indirect prompt injection named as a technique. Section 4 now uses all of this. The fourteen-of-sixteen figure counts tactic columns only. The technique-level examples were read off the screenshot and not counted exhaustively, so the article names them without asserting a total. **ATLAS is a living, maturity-filtered matrix, so the counts are true as of the view date and should be rechecked if the article is republished later.** |

## Later Cursor research (DuneSlide)

| Source | URL | Status |
|---|---|---|
| Cato Networks disclosure | https://www.catonetworks.com/blog/duneslide-two-critical-rce-vulnerabilities/ | **Search-confirmed.** Cato AI Labs, Itay Ravia, 1 July 2026. CVE-2026-50548 and CVE-2026-50549, 9.8 under CVSS 3.1 and 9.3 under 4.0. Patched in Cursor 3.0, released 2 April 2026. Reported 19 February, rejected 23 February on the grounds the threat model excluded MCP server misuse, escalated 26 February, reopened. |
| The Hacker News | https://thehackernews.com/2026/07/critical-cursor-flaws-could-let-prompt.html | **Search-confirmed.** Secondary. |
| Cato AI Labs provenance | https://www.catonetworks.com/blog/breaking-down-echoleak/ | **Read at source, first-party.** Itay Ravia's author bio on the Cato-hosted EchoLeak post states he is Head of Cato AI Labs, "which was formerly the research department of Aim Security." He authored both the EchoLeak post and DuneSlide. The closing section states the shared lineage explicitly rather than implying independent corroboration, and this is the citation for it. |

---

# How this piece was checked

Every factual claim below the lede rests on a page that was read at source or on
an archived capture of one. The reference tables above record which, for each.

That matters more than usual here, because several arguments in this article did
not survive verification. They are listed below with what killed them. If you go
and read the sources, you will find the same things I did, including the parts
that went against me.

**"The CNAs had CWE-1427 available and reached for older categories anyway."**
This was going to be the core of the CWE section. It does not survive reading
CWE-1427. The mapping guidance on that entry says a different CWE may be needed
where prompts are generated from untrusted third-party sources, and names indirect
prompt injection as that case. EchoLeak and CurXecute are both indirect. The
assignments the CNAs made are defensible.

**"The catalog has no shelf for this."**
The replacement argument, and also wrong, or at least incomplete. CWE-1446 does
lack an entry for indirect prompt injection, but the gap is documented rather than
accidental. Its Research Gap note states that from a CWE perspective the
distinction between control and data is not necessarily as deep as the AI security
community considers it. That is a direct counterargument to this article's central
premise, made by the body that maintains the taxonomy. The section now quotes it
and answers on remediability instead, which is the ground where the claim holds.

**"An independent team reached the same conclusion."**
Cato AI Labs was formerly the research department of Aim Security. Itay Ravia
authored both the EchoLeak disclosure and DuneSlide. The closing section states
the shared lineage rather than implying independent corroboration, because the
trajectory is the interesting part and overstating it would be cheap.

**MCPoison, described imprecisely.**
An early draft said approval was granted once and never asked for again. The Check
Point writeup is more specific: trust binds to the MCP entry's key name, and the
command and arguments beneath it are never revalidated when they change. The
impact is also persistent, re-executing on every project open and repository sync,
rather than firing once.

**MCPoison's classification, recorded incompletely.**
NVD lists CWE-78 alone. The GitHub advisory assigns CWE-78 and CWE-494. Two
records of the same vulnerability disagreeing about its classification, which is a
small illustration of the section that discusses exactly that.

**"No user interaction."**
Replaced with Aim Labs' own formulation, that the attack does not rely on specific
user behavior. Their RAG spraying section explains why that phrasing is the
accurate one. At least one vendor writeup disputes the zero-click label on the
grounds that the victim must prompt Copilot with matching content; both CVSS
vectors on the NVD entry carry `UI:N`, and the peer-reviewed AAAI paper states the
exploit achieved privilege escalation without user interaction.

**The ATLAS passage, understated.**
An earlier version hedged, saying a separate framework proves nothing by itself.
Reading the live matrix showed the legend marks ATT&CK-adapted entries with a
superscript ampersand, which draws the inherited-versus-native line this article
argues for. The section was rewritten around that.

**Two versions of the EchoLeak writeup exist.**
The Cato republication and the archived Aim Security original differ in date and
in the strength of several claims. This article relies on none of the differing
text. Both are cited in the references so a reader can see either.

---

# Known limitations

**The fixed-version numbers disagree at source.** Aim Labs says CurXecute was
fixed in Cursor 1.3. Check Point says MCPoison was fixed in 1.3. NVD says
CurXecute was fixed in 1.3.9. This article cites NVD. The disagreement is real and
unresolved.

**The ATLAS counts are dated.** ATLAS is a living, maturity-filtered matrix. The
fourteen-of-sixteen tactic figure is true as read on 2026-08-19 and should be
rechecked before being cited later.
