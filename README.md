# slack-mcp-server

A [MCP(Model Context Protocol)](https://www.anthropic.com/news/model-context-protocol) server for accessing Slack API. This server allows AI assistants to interact with the Slack API through a standardized interface.

> [!CAUTION]
> This repository is **NOT** the original `ubie-oss/slack-mcp-server`. It has been forked and modified to use the Slack User OAuth Token instead of the Bot Token. Use this server at your own risk.

## Transport Support

This server supports both traditional and modern MCP transport methods:

- **Stdio Transport** (default): Process-based communication for local integration
- **Streamable HTTP Transport**: HTTP-based communication for web applications and remote clients

## Features

Available tools:

- `slack_list_channels` - List public channels and the private channels you are a member of, with pagination
- `slack_post_message` - Post a new message to a Slack channel
- `slack_reply_to_thread` - Reply to a specific message thread in Slack
- `slack_add_reaction` - Add a reaction emoji to a message
- `slack_get_channel_history` - Get recent messages from a channel
- `slack_get_thread_replies` - Get all replies in a message thread
- `slack_get_users` - Retrieve basic profile information of all users in the workspace
- `slack_get_user_profiles` - Get multiple users' profile information in bulk (efficient for batch operations)
- `slack_search_messages` - Search for messages in the workspace with powerful filters:
  - Basic query search
  - Location filters: `in_channel`
  - User filters: `from_user`, `with`
  - Date filters: `before` (YYYY-MM-DD), `after` (YYYY-MM-DD), `on` (YYYY-MM-DD), `during` (e.g., "July", "2023")
  - Content filters: `has` (emoji reactions), `is` (saved/thread)
  - Sorting options by relevance score or timestamp

### Canvas Tools

- `slack_list_canvases` - List canvases in the workspace with optional filtering:
  - Filter by user ID or channel ID
  - Pagination support with configurable page size
  - Returns canvas metadata including ID, name, creation date, and URLs

- `slack_get_canvas_sections` - Get sections from a canvas for reading content or identifying sections for editing:
  - Filter by section types: `h1`, `h2`, `h3`, or `any_header`
  - Search for sections containing specific text
  - Returns section IDs and metadata for targeted operations
  - **Note**: At least one of `section_types` or `contains_text` must be provided

- `slack_edit_canvas` - Edit canvas content with multiple operation types:
  - **Insert operations**: `insert_after`, `insert_before`, `insert_at_start`, `insert_at_end`
  - **Modify operations**: `replace` (entire canvas or specific section)
  - **Remove operations**: `delete` (specific section)
  - Supports markdown content with rich formatting
  - Batch operations: Apply multiple changes in a single request

### File Tools

- `slack_download_file` - Download a file shared in Slack to a local directory:
  - File IDs are in the `files` field of messages returned by `slack_get_channel_history`, `slack_get_thread_replies` and `slack_search_messages`
  - Saves to `output_dir`, `SLACK_DOWNLOAD_DIR`, or `slack-mcp-server` in the OS temp directory (in that order) and returns the saved path
  - External files (e.g. Google Drive) cannot be downloaded

- `slack_upload_file` - Upload a local file to Slack:
  - Shares the file in `channel_id` (as a thread reply with `thread_ts`) with an optional `initial_comment`
  - Without `channel_id`, the file is uploaded privately and not shared anywhere

## Quick Start

### Installation

```bash
npm install @j-i-k-o/slack-mcp-server
```

NOTE: Its now hosted in GitHub Registry so you need your PAT.

### Configuration

You need to set the following environment variables:

- `SLACK_BOT_TOKEN`: Slack Bot User OAuth Token (legacy, no longer used)
- `SLACK_USER_TOKEN`: Slack User OAuth Token (required for all operations)

You can also create a `.env` file to set these environment variables:

```
SLACK_BOT_TOKEN=xoxb-your-bot-token  # Legacy, no longer used
SLACK_USER_TOKEN=xoxp-your-user-token  # Required for all operations
```

**Important**: All operations now use the User OAuth Token (`SLACK_USER_TOKEN`) instead of the Bot Token. This provides broader access to Slack APIs and ensures consistent functionality across all tools.

#### Required User Token Scopes

Because only the user token is used, add scopes under **User Token Scopes** (not Bot Token Scopes) in your Slack app's OAuth & Permissions page, then reinstall the app.

| Feature | User Token Scopes |
| --- | --- |
| Public channels (list, history, threads, `in_channel` search filter) | `channels:read`, `channels:history` |
| Private channels you are a member of (list, history, threads, `in_channel` search filter) | `groups:read`, `groups:history` |
| Post messages and thread replies | `chat:write` |
| Add reactions | `reactions:write` |
| Users and profiles | `users:read`, `users.profile:read` |
| Search messages | `search:read` |
| Download files | `files:read` |
| Upload files | `files:write` |
| Canvases | `canvases:read`, `canvases:write` (listing also needs `files:read`) |

`slack_list_channels` lists public and private channels together. Without `groups:read`, it falls back to public channels only.

### Usage

#### Start the MCP server

**Stdio Transport (default)**:
```bash
npx @j-i-k-o/slack-mcp-server
```

**Streamable HTTP Transport**:
```bash
npx @j-i-k-o/slack-mcp-server -port 3000
```

You can also run the installed module with node:
```bash
# Stdio transport
node node_modules/.bin/slack-mcp-server

# HTTP transport  
node node_modules/.bin/slack-mcp-server -port 3000
```

**Command Line Options**:
- `-port <number>`: Start with Streamable HTTP transport on specified port
- `-h, --help`: Show help message

#### Client Configuration

**For Stdio Transport (Claude Desktop, etc.)**:

```json
{
  "slack": {
    "command": "npx",
    "args": [
      "-y",
      "@j-i-k-o/slack-mcp-server"
    ],
    "env": {
      "NPM_CONFIG_//npm.pkg.github.com/:_authToken": "<your-github-pat>",
      "SLACK_BOT_TOKEN": "<your-bot-token>",
      "SLACK_USER_TOKEN": "<your-user-token>"
    }
  }
}
```

**For Streamable HTTP Transport (Web applications)**:

Start the server:
```bash
SLACK_BOT_TOKEN=<your-bot-token> SLACK_USER_TOKEN=<your-user-token> npx @j-i-k-o/slack-mcp-server -port 3000
```

Connect to: `http://localhost:3000/mcp`

See [examples/README.md](examples/README.md) for detailed client examples.

## Implementation Pattern

This server adopts the following implementation pattern:

1. Define request/response using Zod schemas
   - Request schema: Define input parameters
   - Response schema: Define responses limited to necessary fields

2. Implementation flow:
   - Validate request with Zod schema
   - Call Slack WebAPI
   - Parse response with Zod schema to limit to necessary fields
   - Return as JSON

For example, the `slack_list_channels` implementation parses the request with `ListChannelsRequestSchema`, calls `userClient.conversations.list`, and returns the response parsed with `ListChannelsResponseSchema`.

## Development

### Available Scripts

- `npm run dev` - Start the server in development mode with hot reloading
- `npm run build` - Build the project for production
- `npm run start` - Start the production server
- `npm run lint` - Run linting checks (ESLint and Prettier)
- `npm run fix` - Automatically fix linting issues

### Contributing

1. Fork the repository
2. Create your feature branch
3. Run tests and linting: `npm run lint`
4. Commit your changes
5. Push to the branch
6. Create a Pull Request
