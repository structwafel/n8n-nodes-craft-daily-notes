# n8n Custom Node Development: AI Agent Rules

> **Version**: 1.0.0  
> **Last Updated**: 2025-11-29  
> **Purpose**: Comprehensive guidance for AI coding assistants building n8n custom nodes.

---

## 1. PROJECT OVERVIEW

This is the official n8n custom node starter repository. It teaches AI assistants how to build custom n8n community nodes using real-world examples, comprehensive documentation, and established patterns. The repository contains working node implementations, credential configurations, and 30+ documentation files covering all aspects of n8n node development.

---

## 2. TECHNOLOGY STACK

| Component | Technology | Version |
|-----------|------------|---------|
| **Runtime** | Node.js | v22+ |
| **Language** | TypeScript | 5.x (strict mode) |
| **Framework** | n8n Node SDK | Latest |
| **CLI Tool** | @n8n/node-cli | Latest |
| **Linting** | ESLint | 9.x |
| **Formatting** | Prettier | Latest |

---

## 3. PROJECT STRUCTURE

```
craft-n8n-node/
├── credentials/                    # Authentication configurations
│   ├── GithubIssuesApi.credentials.ts      # API key auth example
│   └── GithubIssuesOAuth2Api.credentials.ts # OAuth2 auth example
├── nodes/                          # Node implementations
│   ├── Example/                    # Basic programmatic node
│   │   ├── Example.node.ts
│   │   └── Example.node.json
│   └── GithubIssues/              # Advanced declarative node
│       ├── GithubIssues.node.ts   # Main entry point
│       ├── GithubIssues.node.json # Node metadata
│       ├── resources/             # Resource-based organization
│       │   ├── issue/             # Issue CRUD operations
│       │   └── issueComment/      # Comment operations
│       ├── listSearch/            # Dynamic dropdown methods
│       └── shared/                # Reusable utilities
│           ├── transport.ts       # API request wrapper
│           └── descriptions.ts    # UI component definitions
├── docs/                          # 30+ documentation files (00-30)
├── icons/                         # Node icons (SVG)
├── package.json                   # Node registration & metadata
└── tsconfig.json                  # TypeScript configuration
```

---

## 4. NODE DEVELOPMENT DECISION TREE

When starting a new n8n node, follow this decision tree:

```
Is it a REST API with standard CRUD operations?
├─ YES → Use DECLARATIVE style
│  └─ Read: docs/12-declarative-routing.md
│  └─ Reference: nodes/GithubIssues/GithubIssues.node.ts
│
├─ Does it have an official SDK/client library?
│  └─ YES → Use PROGRAMMATIC style
│     └─ Read: docs/13-custom-execute-methods.md
│     └─ Reference: nodes/Example/Example.node.ts
│
├─ Is it GraphQL, WebSocket, or non-REST?
│  └─ YES → Use PROGRAMMATIC style
│
└─ Complex authentication (request signing, multi-step)?
   └─ YES → Use PROGRAMMATIC style
```

---

## 5. CODE PATTERNS

### Pattern A: Declarative REST API Node

**When to use**: Standard REST APIs with CRUD operations

**File structure**:
```bash
nodes/MyService/
├── MyService.node.ts              # Main entry point
├── MyService.node.json            # Metadata
├── myservice.svg                  # Icon
├── resources/                     # One folder per resource
│   ├── user/
│   │   ├── index.ts              # Operations + common fields
│   │   ├── create.ts             # Create operation
│   │   ├── get.ts                # Get operation
│   │   ├── getAll.ts             # List with pagination
│   │   ├── update.ts             # Update operation
│   │   └── delete.ts             # Delete operation
│   └── project/
│       └── [same structure]
├── listSearch/                    # Dynamic dropdowns
└── shared/
    ├── descriptions.ts            # Reusable UI components
    ├── transport.ts               # API request wrapper
    └── utils.ts                   # Helper functions
```

**Main node structure**:
```typescript
// nodes/YourService/YourService.node.ts
import type { INodeType, INodeTypeDescription } from 'n8n-workflow';
import { userDescription } from './resources/user';
import { projectDescription } from './resources/project';

export class YourService implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Your Service',
    name: 'yourService',
    icon: 'file:yourservice.svg',
    group: ['transform'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Interact with Your Service API',
    defaults: { name: 'Your Service' },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [
      { name: 'yourServiceApi', required: true }
    ],
    requestDefaults: {
      baseURL: 'https://api.yourservice.com/v1',
      headers: { 'Content-Type': 'application/json' }
    },
    properties: [
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          { name: 'User', value: 'user' },
          { name: 'Project', value: 'project' }
        ],
        default: 'user',
      },
      ...userDescription,
      ...projectDescription,
    ],
  };
}
```

**Routing types**:
- `routing.send.type: 'body'` → Request body
- `routing.send.type: 'query'` → URL parameter
- `routing.send.type: 'header'` → HTTP header
- `routing.output.maxResults: '={{$value}}'` → Limit results

---

### Pattern B: Programmatic SDK-Based Node

**When to use**: Complex logic, SDK integration, non-REST protocols

```typescript
export class YourNode implements INodeType {
  description: INodeTypeDescription = { /* ... */ };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    // 1. Get credentials ONCE (not per item)
    const credentials = await this.getCredentials('yourServiceApi') as {
      apiKey: string;
      projectId: number;
    };

    // 2. Initialize SDK client ONCE
    const client = new YourSDK(credentials.apiKey, {
      projectId: credentials.projectId,
    });

    // 3. Get input items
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    // 4. Process each item
    for (let i = 0; i < items.length; i++) {
      try {
        const operation = this.getNodeParameter('operation', i) as string;
        const param1 = this.getNodeParameter('param1', i) as string;

        const result = await client[operation](param1);

        returnData.push({
          json: result,
          pairedItem: { item: i }
        });
      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: { error: error.message },
            pairedItem: { item: i }
          });
          continue;
        }
        throw new NodeOperationError(this.getNode(), error, { itemIndex: i });
      }
    }

    return [returnData];
  }
}
```

---

### Pattern C: Resource Locator (Multi-Mode Selection)

```typescript
export const userSelect: INodeProperties = {
  displayName: 'User',
  name: 'userId',
  type: 'resourceLocator',
  default: { mode: 'list', value: '' },
  required: true,
  modes: [
    {
      displayName: 'From List',
      name: 'list',
      type: 'list',
      placeholder: 'Select a user...',
      typeOptions: {
        searchListMethod: 'getUsers',
        searchable: true,
        searchFilterRequired: false
      }
    },
    {
      displayName: 'By URL',
      name: 'url',
      type: 'string',
      placeholder: 'https://example.com/users/123',
      extractValue: {
        type: 'regex',
        regex: '/users/([0-9]+)'
      }
    },
    {
      displayName: 'By ID',
      name: 'id',
      type: 'string',
      placeholder: '123'
    }
  ]
};
```

---

## 6. CREDENTIALS PATTERNS

### API Key Authentication

```typescript
// credentials/YourServiceApi.credentials.ts
import type { ICredentialType, INodeProperties } from 'n8n-workflow';

export class YourServiceApi implements ICredentialType {
  name = 'yourServiceApi';
  displayName = 'Your Service API';
  documentationUrl = 'https://docs.yourservice.com';
  
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
      headers: {
        Authorization: '=Bearer {{$credentials.apiKey}}',
      },
    },
  };
}
```

### OAuth2 Authentication

Reference: `docs/09-oauth2-credentials.md` and `credentials/GithubIssuesOAuth2Api.credentials.ts`

---

## 7. DOCUMENTATION NAVIGATION

### Complete Documentation Tree (31 Files)

```
docs/
├── docs/00-documentation-index.md          # Master index and navigation
├── docs/01-project-structure-overview.md   # Repository layout explained
├── docs/02-package-json-configuration.md   # npm package setup for n8n
├── docs/03-typescript-configuration.md     # tsconfig.json settings
├── docs/04-node-anatomy-and-architecture.md # Node structure deep dive
├── docs/05-declarative-vs-programmatic-nodes.md # Architecture choice guide
├── docs/06-node-properties-reference.md    # All property types explained
├── docs/07-credentials-system-overview.md  # Auth architecture
├── docs/08-api-key-credentials.md          # API key implementation
├── docs/09-oauth2-credentials.md           # OAuth2 implementation
├── docs/10-creating-your-first-node.md     # Step-by-step tutorial
├── docs/11-resources-and-operations.md     # Multi-resource organization
├── docs/12-declarative-routing.md          # routing.send patterns
├── docs/13-custom-execute-methods.md       # execute() implementation
├── docs/14-list-search-methods.md          # Dynamic dropdown methods
├── docs/15-resource-locators.md            # Multi-mode input fields
├── docs/16-pagination-handling.md          # Cursor/offset pagination
├── docs/17-error-handling-patterns.md      # Error handling best practices
├── docs/18-helper-functions-and-utilities.md # Reusable utilities
├── docs/19-external-sdk-integration.md     # Third-party SDK patterns
├── docs/20-icons-and-branding.md           # SVG icon requirements
├── docs/21-node-json-metadata.md           # node.json configuration
├── docs/22-development-workflow.md         # Dev server and testing
├── docs/23-testing-strategies.md           # Unit and integration tests
├── docs/24-linting-and-code-quality.md     # ESLint configuration
├── docs/25-preparing-for-publication.md    # Pre-publish checklist
├── docs/26-publishing-to-npm.md            # npm publish workflow
├── docs/27-n8n-cloud-verification.md       # Cloud verification process
├── docs/28-complete-code-examples.md       # Full working examples
├── docs/29-common-patterns-and-recipes.md  # Reusable code patterns
└── docs/30-troubleshooting-guide.md        # Common issues and fixes
```

### Quick Reference by Task

| Task | Primary Doc | Secondary Doc |
|------|-------------|---------------|
| **Getting Started** | `docs/10-creating-your-first-node.md` | `docs/00-documentation-index.md` |
| **Choose Architecture** | `docs/05-declarative-vs-programmatic-nodes.md` | `docs/04-node-anatomy-and-architecture.md` |
| **REST API Node** | `docs/12-declarative-routing.md` | `docs/06-node-properties-reference.md` |
| **SDK Integration** | `docs/13-custom-execute-methods.md` | `docs/19-external-sdk-integration.md` |
| **API Key Auth** | `docs/08-api-key-credentials.md` | `docs/07-credentials-system-overview.md` |
| **OAuth2 Auth** | `docs/09-oauth2-credentials.md` | `docs/07-credentials-system-overview.md` |
| **Multi-Resource** | `docs/11-resources-and-operations.md` | `docs/04-node-anatomy-and-architecture.md` |
| **Dynamic Dropdowns** | `docs/14-list-search-methods.md` | `docs/15-resource-locators.md` |
| **Pagination** | `docs/16-pagination-handling.md` | `docs/12-declarative-routing.md` |
| **Error Handling** | `docs/17-error-handling-patterns.md` | `docs/30-troubleshooting-guide.md` |
| **Icons & Branding** | `docs/20-icons-and-branding.md` | `docs/21-node-json-metadata.md` |
| **Testing** | `docs/23-testing-strategies.md` | `docs/22-development-workflow.md` |
| **Publishing** | `docs/25-preparing-for-publication.md` | `docs/26-publishing-to-npm.md` |
| **Cloud Verification** | `docs/27-n8n-cloud-verification.md` | `docs/25-preparing-for-publication.md` |
| **Troubleshooting** | `docs/30-troubleshooting-guide.md` | `docs/17-error-handling-patterns.md` |

---

## 8. DEVELOPMENT COMMANDS

```bash
# Install dependencies
npm install

# Development with hot reload
npm run dev

# Build for production
npm run build

# Lint code
npm run lint

# Auto-fix lint issues
npm run lint:fix

# Create release
npm run release
```

---

## 9. PACKAGE.JSON REGISTRATION

**CRITICAL**: All nodes and credentials MUST be registered:

```json
{
  "name": "n8n-nodes-yourservice",
  "n8n": {
    "nodes": [
      "dist/nodes/YourService/YourService.node.js"
    ],
    "credentials": [
      "dist/credentials/YourServiceApi.credentials.js"
    ]
  }
}
```

---

## 10. COMMON MISTAKES TO AVOID

| ❌ DON'T | ✅ DO |
|----------|-------|
| Initialize SDK per item | Initialize SDK ONCE before loop |
| Forget `pairedItem` | Always include `pairedItem: { item: i }` |
| Hardcode credentials | Use `this.getCredentials()` |
| Skip error handling | Use `this.continueOnFail()` pattern |
| Create single large file | Split into resources/operations |
| Use `any` types | Define proper TypeScript interfaces |

---

## 11. FILE REFERENCES

When building nodes, reference these key files:

- **Declarative node**: `nodes/GithubIssues/GithubIssues.node.ts`
- **Programmatic node**: `nodes/Example/Example.node.ts`
- **API key credential**: `credentials/GithubIssuesApi.credentials.ts`
- **OAuth2 credential**: `credentials/GithubIssuesOAuth2Api.credentials.ts`
- **Resource organization**: `nodes/GithubIssues/resources/issue/`
- **List search methods**: `nodes/GithubIssues/listSearch/`
- **Transport layer**: `nodes/GithubIssues/shared/transport.ts`
- **UI descriptions**: `nodes/GithubIssues/shared/descriptions.ts`

---

## 12. QUICK START STEPS

1. **Research API** → Identify auth method, endpoints, parameters
2. **Choose architecture** → Declarative (REST) or Programmatic (SDK)
3. **Create credentials** → `credentials/YourServiceApi.credentials.ts`
4. **Create node structure** → Follow file structure patterns above
5. **Define resources** → One folder per API resource
6. **Implement operations** → Separate file per operation
7. **Register in package.json** → Add to `n8n.nodes` and `n8n.credentials`
8. **Test with `npm run dev`** → Verify in n8n workflow editor
9. **Lint and build** → `npm run lint:fix && npm run build`
10. **Publish** → `npm publish`
