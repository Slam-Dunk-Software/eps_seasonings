# EPS Seasonings

A collection of portable, LLM-ready implementation patterns for EPS apps.

Each seasoning is a self-contained markdown file that describes a capability, explains how it works, and tells Claude exactly how to apply it to any EPS web app.

## What is a seasoning?

A seasoning is not code — it's instructions. Hand one to Claude alongside your project and it knows how to implement the pattern correctly. Seasonings are:

- **Composable** — apply as many as fit your app
- **Stack-agnostic** — written for any EPS web app regardless of framework
- **Portable** — version-controlled, shareable, human-readable

## Available seasonings

| Name | Description |
|------|-------------|
| [haptics](seasonings/haptics.md) | iOS Taptic Engine feedback via hidden switch input |
| [pin_gate](seasonings/pin_gate.md) | 4-digit PIN entry screen with on-screen numpad and Fibonacci lockout |

## Usage

Download a seasoning and tell Claude:
> "Apply the haptics seasoning to this project. Here's the seasoning: [paste contents]"

Or reference it directly if your Claude setup has access to your local seasonings folder.
