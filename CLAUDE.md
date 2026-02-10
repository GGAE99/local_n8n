# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a **monorepo workspace** containing two interconnected n8n projects:

1. **n8n-mcp** - Model Context Protocol server that provides AI assistants with comprehensive access to n8n node information
2. **n8n-skills** - Collection of Claude Code skills that teach AI assistants how to build n8n workflows using n8n-mcp

### Project Relationship

```
n8n-mcp (MCP Server)
├── Provides: Data access layer (800+ nodes, validation, templates, workflow management)
├── Exposes: 20 MCP tools for node discovery, validation, and workflow management
└── Used by: Claude Desktop, Claude.ai, IDEs with MCP support

n8n-skills (Claude Skills)
├── Provides: Expert guidance on HOW to use n8n-mcp tools
├── Contains: 7 complementary skills for workflow building
└── Requires: n8n-mcp MCP server installed and configured

docker-compose.yml
└── Local n8n instance for development/testing
```

## Common Development Commands

### Workspace Level

```bash
# Start local n8n instance
docker-compose up -d

# View n8n logs
docker-compose logs -f n8n

# Stop n8n instance
docker-compose down
```

### n8n-mcp Project

```bash
cd n8n-mcp

# Build and Setup
npm run build          # Build TypeScript
npm run rebuild        # Rebuild node database from n8n packages
npm run validate       # Validate all node data in database

# Testing
npm test               # Run all tests (3,336 tests)
npm run test:unit      # Run unit tests only
npm run test:integration # Run integration tests
npm run test:coverage  # Run tests with coverage report

# Run a single test file
npm test -- tests/unit/services/property-filter.test.ts

# Running the Server
npm start              # Start MCP server in stdio mode
npm run start:http     # Start MCP server in HTTP mode
npm run dev            # Build, rebuild database, and validate

# Linting
npm run lint           # Check TypeScript types
npm run typecheck      # Check TypeScript types
```

### n8n-skills Project

```bash
cd n8n-skills

# No build step required - skills are markdown-based
# Test by loading into Claude Code or Claude.ai

# Distribution
# Individual skills are in skills/ directory
# Each skill contains:
#   - SKILL.md (skill definition)
#   - Reference files (examples, patterns, etc.)
```

## Architecture Overview

### n8n-mcp Architecture

**Core Components:**

1. **MCP Server** (`mcp/server.ts`)
   - Implements Model Context Protocol for AI assistants
   - Provides 20 tools for node discovery, validation, and workflow management
   - Supports both stdio (Claude Desktop) and HTTP modes

2. **Database Layer** (`database/`)
   - SQLite database storing all n8n node information
   - Universal adapter pattern (better-sqlite3 and sql.js)
   - Full-text search with FTS5

3. **Node Processing Pipeline**
   - Loader → Parser → Property Extractor → Docs Mapper
   - Processes 1,084 nodes (537 core + 547 community)

4. **Service Layer** (`services/`)
   - Property filtering (AI-friendly essentials)
   - Multi-profile validation (minimal, runtime, ai-friendly, strict)
   - Type structure validation
   - Expression and workflow validation

5. **Template System** (`templates/`)
   - 2,709 workflow templates from n8n.io
   - Template fetching, storage, and search
   - Pre-extracted configurations from popular templates

**Key Design Patterns:**
- Repository Pattern: All database operations through repository classes
- Service Layer: Business logic separated from data access
- Validation Profiles: Different strictness levels for different contexts
- Diff-Based Updates: Efficient workflow updates using operation diffs

### n8n-skills Architecture

**7 Complementary Skills:**

1. **n8n-expression-syntax** - Teaches correct n8n expression syntax ({{}} patterns)
2. **n8n-mcp-tools-expert** (HIGHEST PRIORITY) - How to use MCP tools effectively
3. **n8n-workflow-patterns** - 5 proven workflow architectural patterns
4. **n8n-validation-expert** - Interprets validation errors and guides fixing
5. **n8n-node-configuration** - Operation-aware node configuration
6. **n8n-code-javascript** - Write JavaScript in n8n Code nodes
7. **n8n-code-python** - Write Python in n8n Code nodes

**Skill Activation:**
- Skills activate automatically based on query patterns
- Work together seamlessly for comprehensive workflow building
- Progressive disclosure: minimal → standard → full detail levels

## Development Workflow

### Working on n8n-mcp

1. **Make Code Changes**
   ```bash
   cd n8n-mcp
   # Edit source files in src/
   npm run build
   ```

2. **Test Changes**
   ```bash
   npm test                    # Run all tests
   npm run test:unit           # Quick unit tests
   npm run test:integration    # Integration tests (requires clean DB)
   ```

3. **Test with MCP Client**
   - After building, reload MCP server in Claude Desktop
   - Test with actual MCP tool calls
   - Check `~/.claude/logs/mcp-server-n8n-mcp.log` for debugging

4. **Update Database**
   ```bash
   npm run rebuild             # Full rebuild (2-3 minutes)
   npm run validate            # Validate node data
   ```

### Working on n8n-skills

1. **Edit Skills**
   ```bash
   cd n8n-skills/skills/<skill-name>
   # Edit SKILL.md or reference files
   ```

2. **Test Skills**
   - Copy to `~/.claude/skills/` for testing
   - Load in Claude Code and test with queries
   - Check evaluations in `evaluations/` directory

3. **Add New Skill**
   - Create directory under `skills/`
   - Write SKILL.md with frontmatter
   - Add reference files as needed
   - Create 3+ evaluations

### Cross-Project Workflows

**Testing Skills with Local MCP Server:**
```bash
# Terminal 1: Start n8n-mcp in HTTP mode
cd n8n-mcp
npm run build
npm run start:http

# Terminal 2: Configure Claude to use local MCP
# Edit ~/.claude/.mcp.json
# Test skills that call MCP tools
```

**Testing with Local n8n Instance:**
```bash
# Start n8n
docker-compose up -d

# Configure n8n-mcp to use local instance
export N8N_API_URL=http://localhost:5678
export N8N_API_KEY=your-api-key

# Test workflow management tools
npm run start:http
```

## Important Development Notes

### n8n-mcp Specific

- **MCP Server Reload Required**: After changes, ask user to reload MCP server in Claude Desktop
- **Database Rebuilds**: Take 2-3 minutes due to n8n package size
- **HTTP Mode**: Requires proper CORS and auth token configuration
- **Performance**: Use `get_node_essentials()` instead of `get_node_info()` for faster responses
- **Session Persistence**: Export/restore API enables zero-downtime deployments
- **Testing**: Always run `npm run build` before testing changes

### n8n-skills Specific

- **Markdown-Based**: No build step required
- **Skill Activation**: Automatic based on query patterns
- **Cross-Skill Integration**: Skills are designed to work together
- **MCP Dependency**: Requires n8n-mcp server installed and configured
- **Testing**: Manual testing with Claude Code or automated with evaluations

### Workspace Coordination

- **Local n8n**: Use docker-compose.yml for local testing
- **API Configuration**: Set N8N_API_URL and N8N_API_KEY for workflow management
- **Version Sync**: Keep n8n-mcp updated with latest n8n versions
- **Documentation**: Update both project READMEs when making significant changes

## Testing Strategy

### n8n-mcp Tests

**3,336 total tests:**
- Unit tests (2,766): Isolated component testing with mocks
- Integration tests (570): Full system behavior validation
  - n8n API Integration (172 tests): All 18 MCP handler tools
  - MCP Protocol (119 tests): Protocol compliance
  - Database (226 tests): Repository operations, FTS5 search
  - Templates (35 tests): Template operations
  - Docker (18 tests): Configuration, security

**Key Test Commands:**
```bash
npm test                      # Run all tests
npm run test:coverage         # Generate coverage report
npm run test:integration:n8n  # Test n8n API integration
```

### n8n-skills Testing

**Evaluation-Based Testing:**
- Each skill has 3+ evaluation scenarios
- Test against real MCP tool responses
- Iterative testing until 100% pass rate

**Manual Testing:**
- Load skills into Claude Code
- Test with representative queries
- Verify cross-skill integration

## Common Pitfalls

### n8n-mcp

- ❌ Forgetting to rebuild database after package updates
- ❌ Not reloading MCP server after code changes
- ❌ Testing without running `npm run build` first
- ❌ Using wrong nodeType format (nodes-base.* vs n8n-nodes-base.*)

### n8n-skills

- ❌ Skills not activating (check description triggers)
- ❌ MCP server not configured (skills require n8n-mcp)
- ❌ Outdated skill content (keep in sync with MCP tool changes)

### Workspace

- ❌ Wrong working directory (use cd to navigate between projects)
- ❌ Docker n8n not running when testing workflow management
- ❌ Conflicting ports (n8n on 5678, MCP HTTP on various ports)

## Project-Specific CLAUDE.md Files

For detailed project-specific guidance:

- **n8n-mcp**: `n8n-mcp/CLAUDE.md` - Comprehensive MCP server development guide
- **n8n-skills**: `n8n-skills/CLAUDE.md` - Skills development and usage guide

Always check project-specific CLAUDE.md when working in that subdirectory.

## Credits

**Conceived by Romuald Członkowski** - [www.aiadvisors.pl/en](https://www.aiadvisors.pl/en)

Both projects are part of the n8n-mcp ecosystem:
- GitHub: https://github.com/czlonkowski/n8n-mcp
- GitHub: https://github.com/czlonkowski/n8n-skills
- npm: https://www.npmjs.com/package/n8n-mcp

## License

MIT License - See LICENSE files in individual project directories
