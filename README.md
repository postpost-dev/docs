# PostPost API Documentation

REST API for scheduling and publishing social media posts across 10 platforms.

**Website:** [postpost.dev](https://postpost.dev) | **Docs:** [docs.postpost.dev](https://docs.postpost.dev)

## Supported Platforms

X/Twitter, LinkedIn, Instagram, Threads, TikTok, YouTube, Facebook, Bluesky, Mastodon, Telegram

## Quick Start

```bash
curl -X POST https://api.postpost.dev/api/v1/create-post \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Hello from PostPost!",
    "platforms": ["twitter-123", "linkedin-456"],
    "scheduledTime": "2026-04-01T14:00:00.000Z"
  }'
```

## Documentation Structure

```
docs/
├── getting-started.md      # Quick start guide
├── authentication.md       # API keys
├── endpoints/              # API endpoint reference
├── platforms/              # Platform-specific guides
├── guides/                 # Usage guides
├── mcp/                    # MCP server docs
└── examples/               # Code examples (JS, Python, Go, etc.)
```

## License

[MIT](LICENSE)
