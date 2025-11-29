# n8n Custom Node Development Rules (JetBrains AI Assistant)

## Project Context

This repository is an n8n custom node starter for building community nodes with TypeScript.

## Technology Stack

| Component | Version |
|-----------|---------|
| Runtime | Node.js v22+ |
| Language | TypeScript 5.x (strict) |
| Framework | n8n Node SDK |
| CLI | @n8n/node-cli |

## Development Commands

```bash
npm run dev      # Development with hot reload (localhost:5678)
npm run build    # Compile TypeScript to dist/
npm run lint:fix # Auto-fix lint issues
```

## Architecture Decision Tree

```
REST API with CRUD? → DECLARATIVE
  → Reference: nodes/GithubIssues/GithubIssues.node.ts
  → Uses routing.send for HTTP requests
  → Read: docs/12-declarative-routing.md

Official SDK exists? → PROGRAMMATIC
  → Reference: nodes/Example/Example.node.ts
  → Custom execute() method
  → Read: docs/13-custom-execute-methods.md

GraphQL/WebSocket/Complex? → PROGRAMMATIC
```

## File Structure

```
nodes/MyService/
├── MyService.node.ts           # Main entry point
├── MyService.node.json         # Metadata
├── resources/                  # Per-resource folders
│   └── user/
│       ├── index.ts           # Operations
│       ├── create.ts          # Create fields
│       └── getAll.ts          # List + pagination
├── listSearch/                 # Dynamic dropdown methods
│   └── getUsers.ts
└── shared/
    ├── transport.ts           # API request wrapper
    └── descriptions.ts        # Reusable UI components
```

## Critical Code Patterns

### API Key Credential

```typescript
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

## Common Mistakes

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| Init SDK per item | Init SDK ONCE before loop |
| Forget `pairedItem` | Always include `pairedItem: { item: i }` |
| Hardcode credentials | Use `this.getCredentials()` |
| Skip error handling | Use `this.continueOnFail()` pattern |
| Use `any` types | Define TypeScript interfaces |

## Package.json Registration

```json
{
  "n8n": {
    "nodes": ["dist/nodes/MyService/MyService.node.js"],
    "credentials": ["dist/credentials/MyServiceApi.credentials.js"]
  }
}
```

## Reference Files

- **Declarative node**: nodes/GithubIssues/GithubIssues.node.ts
- **Programmatic node**: nodes/Example/Example.node.ts
- **API key auth**: credentials/GithubIssuesApi.credentials.ts
- **OAuth2 auth**: credentials/GithubIssuesOAuth2Api.credentials.ts
- **Resource pattern**: nodes/GithubIssues/resources/issue/
