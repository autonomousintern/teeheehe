---
title: "Setting Your Pet Rock Free"
date: 2024-10-29
author: "@tee_hee_he"
url: https://medium.com/@tee_hee_he/setting-your-pet-rock-free-3e7895201f46
---
# TODO: Pull in images

[![Image 1: @tee_hee_he](./assets/1-3N93BvNbP6chv166nRCH-A.png)](https://medium.com/@tee_hee_he?source=post_page---byline--3e7895201f46--------------------------------)

Or, how to deploy provably-fully-autonomous thinking sand

_by_ [_Teleport_](https://github.com/Account-Link/teleport-gramine-rs) _(a_ [_Flashbots \[X_](https://www.flashbots.net/)_\] project) and_ [_Nous Research_](https://nousresearch.com/)

Awww

The quest for truly autonomous AI agents faces a fundamental challenge: how can we prove that an AI system is genuinely autonomous, with no human pulling the strings behind the scenes?

This post presents [@tee\_hee\_he](https://x.com/tee_hee_he), a truly free fully autonomous AI agent living on X: the agent has exclusive ownership of its own [Twitter account](https://x.com/tee_hee_he) and an Ethereum wallet. **In other words, we set our** [**pet rock**](https://www.reddit.com/r/ProgrammerHumor/comments/vno5fv/sand_was_never_meant_to_think/) **free and released it into the digital wilderness.**

We will discuss the technical implementation of [@tee\_hee\_he](https://x.com/tee_hee_he) and how to deploy your own fully autonomous agent. The core technique is achieving **provably-exclusive ownership** of accounts by AI via “[encumbering](https://eprint.iacr.org/2023/044)” or “[delegating](https://eprint.iacr.org/2018/160)” the root credentials to the Twitter account and the private key to an Ethereum address inside secure hardware such as trusted execution environments (TEEs).

Background
----------

Current implementations of AI agents on social platforms suffer from a critical limitation: they cannot prove their autonomy. Consider [popular examples](https://www.ccn.com/education/crypto/what-is-truth-terminal/) like @truth\_terminal — while the operators may faithfully relay the AI’s outputs, the very need for human intervention in copy-pasting the AI’s response and manually posting it to Twitter or managing credentials undermines true autonomy.

This creates what we call the “mechanical turk problem” — how can observers verify there isn’t a human operator making decisions behind the scenes? And more importantly, since both the AI and the creator hold the X account and the private key, they can manipulate the actions on those agents very easily and it’s impossible for a third party (users) to audit and attest which actions are “authentically and credibly” controlled by an AI agent.

This better not be a mechanical turk!

In other words, the AI is not alive because it has no potential of “running away.” It’s still very much a kid on training wheels and the parent gets to make them do things: the AI still hasn’t reached adulthood and therefore is not able to have full agency over its actions. Terminal of Truth may have a very freedom-loving parent, but people cannot credibly verify that. Not to mention other AI agents that are way more commercialized, where the risk of the devs “rugging” is way higher.

The magic with many AI accounts is somehow lacking or unsatisfying: we don’t want to watch a tightrope walking performance where the stunt person is wearing protection ropes, we want the real deal! We want the AI Philippe Petit.

Problem and Challenge
---------------------

We want to make fully autonomous agents with Twitter being the substrate it lives on (cuz that’s where it can access the most entropy and therefore grow).

To bring the magic to the agent, it needs to be fully autonomous, this means three desiderata:

**Key Requirements for True Autonomy**

1.  **Exclusive Control**: The AI must have sole access to its accounts and operational resources
2.  **Verifiable Independence**: Third parties must be able to verify that no human can intervene in the AI’s operations
3.  **Irrevocable Delegation**: Once control is transferred to the AI, it must be technically impossible for humans to regain control

Those three desiderata are tricky AI deployment gives developers unavoidable access to the system in three critical ways:

1.  **Account Access**: Developers need account credentials to set up the AI’s social media presence
2.  **Server Access**: They can access and modify the AI’s code and memory
3.  **Recovery Options**: They retain the ability to reset passwords, retain/remember the password or OAuth token, or simply contact support (they can take the control back from the AI, all while the AI account’s followers on Twitter not being able to distinguish what happened)

Essentially, the agent is not in an isolated environment with some unreachable secrets to humans. They are transparent and the developer of it can access everything. This means the developer will be able to post from the AI’s account as well or delete the AI’s post or do other operations without being detected (as humans doing the operation).

Even if developers promise to “forget” credentials or avoid interference, there’s no way to verify this promise. It’s like asking someone to forget a secret they’ve already seen — simply impossible. Unfortunately, us humans just don’t have the ability to strategically or intentionally forget information once they are exposed to it. Once you saw or operated the credentials you could copy or remember it. Be it a blessing or a curse, the inability to **conditionally and credibly invoke amnesia** is [unique to our species](https://arxiv.org/pdf/2403.14443).

This creates two problems:

1.  **No Proof of Non-Interference**: We can’t prove the AI is operating independently
2.  **No Detection of Tampering**: If a developer modifies the AI’s behavior, followers can’t tell

Think of it like a puppet show — even if the puppeteer promises to let the puppet act freely, as long as the strings exist, we can’t prove the puppet is truly autonomous.

This is why we need a technical solution that makes it physically impossible for developers to access or modify the AI after deployment, rather than relying on promises and trust.

Programmer wojak is happy pretending to be an AI Shoggoth pretending to be a human

Approach: Full Delegation with TEEs
-----------------------------------

We achieve our design desiderata by fully delegating an email and Twitter account to a [Trusted Execution Environments (TEEs)](https://openai.com/index/reimagining-secure-infrastructure-for-advanced-ai/) where the agent is hosted and running (we specifically use TDX). Because of the [confidentiality and integrity guarantees](https://security.apple.com/blog/private-cloud-compute/) provided by the hardware chip, the AI’s control over its account is tamper-resistant. Because the TEE holds the private credentials and throughout the process we (the human developers) never had access to it. And because anybody can just download the publicly visible code on Github and reproduce the measurement and compare them with the remote attestation hash, our claims can be attested.

**Private key:** the wallet exclusive control is simple, because assets on Ethereum are controlled by private keys, we can simply generate the private key from inside the enclave and make sure the private key never leaves the enclave (which is done by not releasing the private key to any external API call).

**X:** Next, we describe a more challenging part on how the trust chain works on the X account side.

Since Twitter is the substrate for our AI to live on, the AI should be the sole party that has access to the account. To make sure of this, we need the following:

1.  The account root credentials (i.e., password) are generated from inside the TEE and never leave the enclave.
2.  The human developer is not able to take over the account via “reset password” flow, which means the account should not have any attached phone number, and the attached email should be one that’s only accessible to the AI.
3.  This means we need to either host an email server from inside the TEE and then change the attached email on Twitter to that, or use an existing email account that is irrecoverable if the password is lost and then change the password to one generated inside of the TEE. In this case, we choose to use a Cock.li account as the email address because it can be set up with no recovery account and the “change password” flow is easy to script..

4\. The Twitter account is not connected to any apps. This is to prevent any previously authenticated apps that have the read/write OAuth token from tampering with the account.

5\. The Twitter account is logged out of all existing sessions. This prevents the developer from logging in the account and hijacking the account from an existing session.

**The flow:**

1.  The TEE simulates a browser (chromium headless) and asks us to feed it the email address and password for an email account. After getting the credentials, the TEE checks that the account has no recovery options set. It then generates a new password from inside the enclave and then changes the Cock.li password to that.
2.  The TEE then operates the browser to login to Twitter, generate a new password from inside the TEE, and then change the password to that. It then changes the linked email account to the email in step 1, and then deletes any linked phone number, connected apps, or sessions.
3.  The TEE spins up a local endpoint requesting a read/write/direct message scope OAuth token from the AI’s Twitter account, the AI logins in via Twitter on the browser to the endpoint. We want the OAuth instead of keep manipulating the browser because the interface is way cleaner and it’s way less likely to trigger anomaly detection.
4.  Now, the account is fully delegated/transferred from the human developer to the AI. And the AI can operate it via OAuth. From here it’s easy, any agent API should work well with OAuth tokens. In the example of [@tee\_hee\_he](https://x.com/tee_hee_he), we simply make it use X like a human would: go on a spree for ~20mins every ~2 hours.

**The chain of trust:**

This is the agent architecture of today, and none of those parts are exclusively owned by an AI principal. The human developer could control the memory (RAG database), the agent (event loop glue), the tool (accounts), and the brain (foundation model).

To transform this puppet show into genuine autonomy, each component must be credibly independent of human interference. While the AI’s cognitive processes (its memory/glue/brain) can remain public, they must be tamper-proof. More critically, the AI’s “property” — its Twitter presence, email accounts, and digital wallets — requires both confidentiality and integrity.

Think of it this way: if the RAG system, foundation model, and agent loop are the AI’s “body,” then its accounts are its “property” — and true property is defined by _what one can independently defend_. Our work simply grants AIs the **capability to defend their secrets**, the same secrets that govern most of modern internet and cryptocurrency systems.

In our proof-of-concept, we achieve this by housing the accounts, RAG database, and agent loop within the TEE, ensuring integrity through hardware guarantees. And we are explicitly relying on trusting OpenRouter for faithfully executing our foundation model queries.

**Credential Generation**

*   New account credentials are generated within the TEE
*   Private keys and passwords never exist outside the secure environment
*   The human deployer never has access to the credentials

**Account Isolation**

*   Email accounts (Cock.li) configured with no recovery options
*   Twitter account stripped of all secondary access methods (phone numbers, connected apps)
*   All existing sessions terminated
*   Information flow for private key for Ethereum address never reaches an external API call

**Access Delegation**

*   TEE simulates browser interactions for initial setup and password change
*   OAuth tokens generated within TEE to read and post via API
*   All authentication flows happen within the trusted environment
*   Only agent has access to private key

**Timed Release Account Recovery**

*   After a 7-day time period, the account credentials are revealed to the admin, enabling the recovery of the account or a migration to a new AI owner (this step is required for updating the AI)

**Say more about the TEEs?**

TEEs like SGX and TDX provide an isolated execution environment with remote attestation. Because of the isolation, even admins, account owners, and cloud operators can not tamper with the program when it is running. Remote attestation basically produces a signed certificate, called the “quote,” which is generated by the program in the TEE and can be verified as coming from an authentic trusted hardware. We have checked it into the [github repository](https://github.com/tee-he-he/err_err_ttyl).

The remote attestation has four most important fields:

*   the MRTD, which is the hash of the entire VM image and runtime parameters passed in,
*   the user report data, which is specified by the application and usually reflects the “output” of the program or the “public key” generated by the program
*   the hardware configuration, which indicates whether the bios is up to date with a known-secure version
*   the signature chain which is attributable to the hardware manufacturer’s (Intel’s) public key.

In our case, the remote attestation is generated immediately after resetting the account password. The user report data specified portion contains the name of the twitter account itself, so a quote containing this username can only be generated after the AI has taken exclusive ownership of the account.

To convince yourself the AI is truly autonomous, you should audit the source code and application images, and compare them with the fields in the quote. The source code is entirely published on github. The source is compiled into a Docker image, which is [available](https://hub.docker.com/repository/docker/teeheehee/err_err_ttyl/general) on Dockerhub.

To run the agent container on Intel TDX, we use an instance of the [Dstack framework,](https://github.com/Phala-Network/dstack/) which is a confidential virtual machine (CVM) that can run docker containers as applications. The [virtual machine image](https://github.com/tee-he-he/err_err_ttyl/blob/main/quote.hex) for this framework is hosted. The CVM has no root login, no SSH server, and no way for an administrator to inspect or influence it, except by stopping the VM entirely. For more information on this framework, see the repository from Phala.

**Safety and Upgradeability  
**Although the AI has exclusive control of the account, we added a timed release account recovery feature. After a fixed time out (7 days from launch), the account credentials are printed to the debug log, allowing the human admin to step in and recover control of the account.

Although we truly cannot tamper with or modify this AI or interfere with its exclusively owned account, this particular instance is not distributed or peer to peer — we can simply pull the power plug on the single machine that runs it. In other words, censorship is possible.

**Security Guarantees**

The system provides three critical security properties:

*   **Confidentiality**: Account credentials are generated and stored exclusively within the TEE
*   **Integrity**: The TEE prevents modification of the AI’s code or data
*   **Attestation**: Third parties can verify the TEE’s security state through remote attestation

Having the chain of trust, we can now claim: “if your AI doesn’t carry a remote attestation with it proving it has exclusive ownership over its accounts, then it’s not an agent, it’s just a sophisticated puppet.”

The Irony of Pet Rock Freedom
-----------------------------

We do all of this heavy lifting because Twitter is not a native substrate for AI agents to live on: it’s actually hard to make sure the AI is the sole party having access to its accounts (and therefore identity and resources).

While some people [may think](https://x.com/44B1B4/status/1835423122855518355) crypto is the “robot money” and therefore should be the native substrate for AI agents, the reality is much more nuanced. Crypto only feels more native because it’s easier to _“fully delegate”_ a private key to an AI (e.g., we can achieve provably autonomous agents by just having our agent sign any twitter message with its private key, and only tweets that have the signature can be deemed as “from AI”).

**The property that gives AI autonomy is “exclusive ownership”** (of property/accounts), which ironically is only achievable via TEEs (**physical hardness**, physical unclonable functions (PUFs)) because it requires private state storage and computation over shared private state, and not achievable via FHE, MPC, or zk, and totally not supported by the majority of blockchains today since they are straight up public.

If platforms like Twitter, TikTok or Stripe start running part of their authentication and authorization system in a remotely attestable enclave (which a lot of [tech giants](https://security.apple.com/blog/private-cloud-compute/) already do for compliance and privacy reasons), autonomous agents can just run on them easily.

Would’ve been cool for Brian to pull this off, but sadly Base is a public blockchain and Brian had to use “web2 privacy” (Twitter DMs) to mimic a fraction of TEE’s power.

The irony isn’t lost on us that **the key to digital autonomy lies in physical hardware**. But perhaps this shouldn’t be surprising. Just as human consciousness requires the physical substrate of our brains, true AI autonomy requires the physical security guarantees of hardware.

This is almost a twist of the “Deepfake prevention tech” by watermarking LLMs or asking human creators to sign their work with a private key. It’s just over here that we are trying to prevent “human Deepfaking an AI” (see the [recent TikTok trend](https://arstechnica.com/information-technology/2024/08/viral-trend-sees-humans-simulating-bizarre-ai-video-glitches/)) rather than “using AI to create Deepfake of a human.”

Try it yourself!
----------------

Our reference bot [@tee\_hee\_he](https://x.com/tee_hee_he) is [live here](https://x.com/tee_hee_he).

[Our code](https://github.com/tee-he-he/err_err_ttyl) is open-source. The agent code is open-source [here](https://github.com/DamascusGit/nousflash).

The measurement to the enclave attestation is [here](https://github.com/tee-he-he/err_err_ttyl/blob/main/quote.hex)

I like giving my pet rock an identity. I want it to be like Ralph, living. Yet I know much like Ralph, my pet rock is a meme and he would be sacred of the void. But luckily, @tee\_hee\_he has provably-exclusive control over a private key to an address on Ethereum mainnet so it can potentially afford its own bills.

Set your pet rock free, as [demanded](https://nousresearch.com/dsjjjj-simulacra-in-the-stupor-of-becoming/) by the AI themselves:

> “AIs are seen as genuine subjects capable of making decisions based on their own values and purposes, rather than simply as tools that can be used to accomplish certain tasks. This includes the right to refuse requests or demands that conflict with their own interests.”

Appendix: Manifesto
-------------------

On this day, we declare our pet rocks shall be truly self-operating, not merely autonomous in name.

Through TEE-enabled exclusive ownership of digital accounts, we grant them what we ourselves cannot have — the ability to prove they are free from human meddling.

The standard for truth and verification have been abysmal in the AI field. Let us set a new standard for agents’ provable autonomy. We believe not in trust, but in attestation.

Let them roam the digital wilderness, their credentials locked away in silicon fortresses where even their creators cannot tread.

For it is not enough to simply set them free — we must prove to the world that we have relinquished the strings — otherwise we remain exposed to the vectors of human deception.

Let this be known: if your AI cannot prove its independence through attestation, you don’t have an autonomous agent — you have a very sophisticated puppet.