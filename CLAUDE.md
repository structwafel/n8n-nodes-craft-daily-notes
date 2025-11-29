# CLAUDE.md - n8n Custom Node Development

This file provides guidance for Claude Code (claude.ai/code) when working with this n8n custom node development repository.

## Project Purpose

This repository teaches how to build custom n8n community nodes. It contains:
- Working node examples (declarative + programmatic patterns)
- Credential implementations (API key + OAuth2)
- 30+ documentation files covering all aspects of n8n node development

## Key Commands

```bash
npm run dev      # Start n8n with hot reload (http://localhost:5678)
npm run build    # Compile TypeScript to dist/
npm run lint     # Check code quality
npm run lint:fix # Auto-fix lint issues
npm run release  # Create new release
```

## Architecture Decision

When building a new node, choose:

1. **DECLARATIVE** (for REST APIs with CRUD)
   - Reference: `nodes/GithubIssues/GithubIssues.node.ts`
   - Uses `routing.send` for request building
   - No custom `execute()` method needed

2. **PROGRAMMATIC** (for SDKs, complex logic, non-REST)
   - Reference: `nodes/Example/Example.node.ts`
   - Custom `execute()` method with full control

## File Structure for New Node

```
nodes/MyService/
├── MyService.node.ts           # Main node file
├── MyService.node.json         # Metadata
├── myservice.svg               # Icon
├── resources/                  # Resource folders
│   └── user/
│       ├── index.ts           # Resource entry
│       ├── create.ts          # Create operation
│       └── getAll.ts          # List with pagination
├── listSearch/                 # Dynamic dropdowns
│   └── getUsers.ts
└── shared/
    ├── transport.ts           # API request wrapper
    └── descriptions.ts        # Reusable UI components
```

## Critical Code Patterns

### Credentials (API Key)

```typescript
// credentials/MyServiceApi.credentials.ts
export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'MyService API';
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];
  authenticate = {
    type: 'generic',
    properties: {
      headers: { Authorization: '=Bearer {{$credentials.apiKey}}' },
    },
  };
}
```

### Declarative Routing

```typescript
// resources/user/create.ts
export const userCreateFields: INodeProperties[] = [
  {
    displayName: 'Name',
    name: 'name',
    type: 'string',
    displayOptions: { show: { resource: ['user'], operation: ['create'] } },
    routing: { send: { type: 'body', property: 'name' } }
  }
];
```

### Programmatic Execute

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const credentials = await this.getCredentials('myServiceApi');
  const client = new SDK(credentials.apiKey); // Init ONCE
  
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

## Common Mistakes to Avoid

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| Init SDK per item | Init SDK ONCE before loop |
| Forget `pairedItem` | Always include `pairedItem: { item: i }` |
| Hardcode credentials | Use `this.getCredentials()` |
| Skip error handling | Use `this.continueOnFail()` pattern |
| Use `any` types | Define TypeScript interfaces |

## Documentation Quick Reference

| Task | Primary Doc |
|------|------------|
| New node | `docs/10-creating-your-first-node.md` |
| REST routing | `docs/12-declarative-routing.md` |
| SDK node | `docs/13-custom-execute-methods.md` |
| API key auth | `docs/08-api-key-credentials.md` |
| OAuth2 auth | `docs/09-oauth2-credentials.md` |
| Resources | `docs/11-resources-and-operations.md` |
| Pagination | `docs/16-pagination-handling.md` |
| Publishing | `docs/25-preparing-for-publication.md` |

## Package.json Registration

Always register nodes and credentials:

```json
{
  "n8n": {
    "nodes": ["dist/nodes/MyService/MyService.node.js"],
    "credentials": ["dist/credentials/MyServiceApi.credentials.js"]
  }
}
```

## Reference Files

- **Declarative**: `nodes/GithubIssues/GithubIssues.node.ts`
- **Programmatic**: `nodes/Example/Example.node.ts`
- **API Key Auth**: `credentials/GithubIssuesApi.credentials.ts`
- **OAuth2 Auth**: `credentials/GithubIssuesOAuth2Api.credentials.ts`
- **Resource Pattern**: `nodes/GithubIssues/resources/issue/`
- **List Search**: `nodes/GithubIssues/listSearch/`
