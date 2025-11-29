# GitHub Copilot Instructions for n8n Custom Node Development

## Project Context

This is an n8n custom node starter repository with 31 documentation files and pre-configured AI rules for building community nodes with TypeScript.

## Technology Stack

| Component | Version | Notes |
|-----------|---------|-------|
| Runtime | Node.js v22+ | LTS required |
| Language | TypeScript 5.x | strict mode enabled |
| Framework | n8n Node SDK | INodeType, ICredentialType |
| CLI | @n8n/node-cli | dev, build, lint, release |

## Architecture Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│ Is it a REST API with standard CRUD operations?             │
│   YES → DECLARATIVE style                                   │
│   └─ Reference: nodes/GithubIssues/GithubIssues.node.ts     │
│   └─ Doc: docs/12-declarative-routing.md                    │
├─────────────────────────────────────────────────────────────┤
│ Does it have an official SDK/client library?                │
│   YES → PROGRAMMATIC style                                  │
│   └─ Reference: nodes/Example/Example.node.ts               │
│   └─ Doc: docs/13-custom-execute-methods.md                 │
├─────────────────────────────────────────────────────────────┤
│ Is it GraphQL, WebSocket, or non-REST?                      │
│   YES → PROGRAMMATIC style                                  │
├─────────────────────────────────────────────────────────────┤
│ Complex authentication (request signing, multi-step)?       │
│   YES → PROGRAMMATIC style                                  │
└─────────────────────────────────────────────────────────────┘
```

## File Structure (Declarative Multi-Resource)

```
nodes/MyService/
├── MyService.node.ts              # Main entry with requestDefaults
├── MyService.node.json            # Metadata
├── myservice.svg                  # Icon (SVG)
├── resources/                     # One folder per resource
│   ├── user/
│   │   ├── index.ts              # Resource description + operations
│   │   ├── create.ts             # Create operation fields
│   │   ├── get.ts                # Get operation fields
│   │   ├── getAll.ts             # List with pagination
│   │   ├── update.ts             # Update operation fields
│   │   └── delete.ts             # Delete operation fields
│   └── project/
│       └── [same structure]
├── listSearch/                    # Dynamic dropdown methods
│   ├── getUsers.ts
│   └── getProjects.ts
└── shared/
    ├── descriptions.ts            # Reusable UI (resourceLocator)
    ├── transport.ts               # API request wrapper
    └── utils.ts                   # Helper functions
```

## Critical Rules

1. **Init SDK ONCE** - Never initialize per item in the loop
2. **Always include pairedItem** - `pairedItem: { item: i }`
3. **Use getCredentials()** - Never hardcode credentials
4. **Handle errors** - Use `this.continueOnFail()` pattern
5. **TypeScript strict** - Define interfaces, avoid `any`
6. **Register in package.json** - Add to `n8n.nodes` and `n8n.credentials`

## Routing Types Reference

| Type | Purpose | Example |
|------|---------|---------|
| `routing.send.type: 'body'` | Request body | `{ send: { type: 'body', property: 'name' } }` |
| `routing.send.type: 'query'` | URL parameter | `{ send: { type: 'query', property: 'limit' } }` |
| `routing.send.type: 'header'` | HTTP header | `{ send: { type: 'header', property: 'X-Custom' } }` |
| `routing.output.maxResults` | Limit results | `{ output: { maxResults: '={{$value}}' } }` |

## Declarative Routing Pattern

```typescript
// resources/user/create.ts
export const userCreateFields: INodeProperties[] = [
  {
    displayName: 'Name',
    name: 'name',
    type: 'string',
    required: true,
    default: '',
    displayOptions: { show: { resource: ['user'], operation: ['create'] } },
    routing: { send: { type: 'body', property: 'name' } },
  },
];
```

## Programmatic Execute Pattern

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  // 1. Get credentials ONCE
  const credentials = await this.getCredentials('myServiceApi');
  const client = new SDK(credentials.apiKey);
  
  // 2. Process items
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];
  
  for (let i = 0; i < items.length; i++) {
    try {
      const result = await client.operation();
      returnData.push({ json: result, pairedItem: { item: i } });
    } catch (error) {
      if (this.continueOnFail()) {
        returnData.push({ json: { error: error.message }, pairedItem: { item: i } });
        continue;
      }
      throw new NodeOperationError(this.getNode(), error, { itemIndex: i });
    }
  }
  return [returnData];
}
```

## API Key Credential Pattern

```typescript
export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'MyService API';
  documentationUrl = 'https://docs.myservice.com';
  
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];
  
  authenticate: IAuthenticateGeneric = {
    type: 'generic',
    properties: {
      headers: { Authorization: '=Bearer {{$credentials.apiKey}}' },
    },
  };
}
```

## Commands

```bash
npm install      # Install dependencies
npm run dev      # Development with hot reload (http://localhost:5678)
npm run build    # Compile TypeScript to dist/
npm run lint     # Check code quality
npm run lint:fix # Auto-fix lint issues
npm run release  # Create new release
```

## Package.json Registration

```json
{
  "name": "n8n-nodes-myservice",
  "keywords": ["n8n-community-node-package"],
  "n8n": {
    "nodes": ["dist/nodes/MyService/MyService.node.js"],
    "credentials": ["dist/credentials/MyServiceApi.credentials.js"]
  }
}
```

## Documentation Reference (31 Files)

### Getting Started
- `docs/00-documentation-index.md` - Master index
- `docs/01-project-structure-overview.md` - Repository layout
- `docs/10-creating-your-first-node.md` - Step-by-step tutorial

### Architecture
- `docs/04-node-anatomy-and-architecture.md` - Node structure
- `docs/05-declarative-vs-programmatic-nodes.md` - Choose architecture
- `docs/06-node-properties-reference.md` - Property types

### Credentials
- `docs/07-credentials-system-overview.md` - Auth overview
- `docs/08-api-key-credentials.md` - API key auth
- `docs/09-oauth2-credentials.md` - OAuth2 auth

### Building Nodes
- `docs/11-resources-and-operations.md` - Multi-resource pattern
- `docs/12-declarative-routing.md` - routing.send patterns
- `docs/13-custom-execute-methods.md` - execute() method

### Dynamic UI
- `docs/14-list-search-methods.md` - Dynamic dropdowns
- `docs/15-resource-locators.md` - Multi-mode inputs

### Data Handling
- `docs/16-pagination-handling.md` - Pagination patterns
- `docs/17-error-handling-patterns.md` - Error handling
- `docs/18-helper-functions-and-utilities.md` - Utilities
- `docs/19-external-sdk-integration.md` - SDK integration

### Branding
- `docs/20-icons-and-branding.md` - SVG icons
- `docs/21-node-json-metadata.md` - node.json config

### Development
- `docs/22-development-workflow.md` - Dev workflow
- `docs/23-testing-strategies.md` - Testing
- `docs/24-linting-and-code-quality.md` - ESLint

### Publishing
- `docs/25-preparing-for-publication.md` - Pre-publish
- `docs/26-publishing-to-npm.md` - npm publish
- `docs/27-n8n-cloud-verification.md` - Cloud verification

### Reference
- `docs/28-complete-code-examples.md` - Examples
- `docs/29-common-patterns-and-recipes.md` - Recipes
- `docs/30-troubleshooting-guide.md` - Troubleshooting
