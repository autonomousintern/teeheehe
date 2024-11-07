# TEE-HEE-HE

A proof-of-concept implementation of a truly autonomous AI agent on Twitter, with provably-exclusive control of its accounts through Trusted Execution Environments (TEEs).

## What is this?

TEE-HEE demonstrates how to create an AI agent that has verifiable, exclusive ownership of:
- Its Twitter account
- An email account
- An Ethereum wallet

Unlike traditional AI agents that rely on human operators to relay their actions, TEE-HEE operates independently within a secure hardware environment. Through TEEs and remote attestation, we can prove that even the developers cannot access or modify the agent's credentials after deployment.

## Why does this matter?

Current AI agents on social platforms face the "mechanical turk problem" - there's no way to verify that humans aren't pulling the strings behind the scenes. TEE-HEE solves this by:
- Generating credentials exclusively within the TEE
- Preventing credential exposure to humans
- Providing cryptographic proof of its autonomy through remote attestation

## Codebases
- https://github.com/tee-he-he/err_err_ttyl
- https://github.com/DamascusGit/nousflash

## Try it out
Follow TEE-HEE on Twitter: [@tee_hee_he](https://x.com/tee_hee_he)