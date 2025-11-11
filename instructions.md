# Instructions for Creating an MCP Server to Consume REST APIs

## Overview
This document provides guidance for building a Model Context Protocol (MCP) server that consumes REST APIs. The MCP server will act as a bridge between AI assistants (like GitHub Copilot) and external REST APIs, enabling natural language interactions with those APIs.

## Architecture

### MCP Server Components
1. **Server Core**: Handles MCP protocol communication
2. **API Client**: Manages HTTP requests to the REST API
3. **Tool Definitions**: Exposes REST API endpoints as MCP tools
4. **Error Handling**: Manages API errors and rate limiting
5. **Authentication**: Handles API keys and authentication tokens

## Project Structure

```
mcp-rest-server/
├── src/
│   ├── index.ts              # Server entry point
│   ├── server.ts             # MCP server implementation
│   ├── api/
│   │   ├── client.ts         # REST API client
│   │   └── types.ts          # API type definitions
│   ├── tools/
│   │   └── index.ts          # MCP tool definitions
│   └── config/
│       └── settings.ts       # Configuration management
├── package.json
├── tsconfig.json
└── README.md
```

## Implementation Steps

### 1. Initialize the Project

```bash
npm init -y
npm install @modelcontextprotocol/sdk axios dotenv
npm install -D typescript @types/node ts-node
```

### 2. Configure TypeScript

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 3. Create the REST API Client

Create `src/api/client.ts`:
```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';

export class RestApiClient {
  private client: AxiosInstance;

  constructor(baseURL: string, apiKey?: string) {
    this.client = axios.create({
      baseURL,
      headers: {
        'Content-Type': 'application/json',
        ...(apiKey && { 'Authorization': `Bearer ${apiKey}` }),
      },
    });
  }

  async get<T>(endpoint: string, params?: Record<string, any>): Promise<T> {
    const response = await this.client.get<T>(endpoint, { params });
    return response.data;
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await this.client.post<T>(endpoint, data);
    return response.data;
  }

  async put<T>(endpoint: string, data: any): Promise<T> {
    const response = await this.client.put<T>(endpoint, data);
    return response.data;
  }

  async delete<T>(endpoint: string): Promise<T> {
    const response = await this.client.delete<T>(endpoint);
    return response.data;
  }
}
```

### 4. Define MCP Tools

Create `src/tools/index.ts`:
```typescript
import { Tool } from '@modelcontextprotocol/sdk/types.js';

export const tools: Tool[] = [
  {
    name: 'api_get',
    description: 'Perform a GET request to the REST API',
    inputSchema: {
      type: 'object',
      properties: {
        endpoint: {
          type: 'string',
          description: 'API endpoint path (e.g., /users, /posts/1)',
        },
        params: {
          type: 'object',
          description: 'Query parameters as key-value pairs',
        },
      },
      required: ['endpoint'],
    },
  },
  {
    name: 'api_post',
    description: 'Perform a POST request to the REST API',
    inputSchema: {
      type: 'object',
      properties: {
        endpoint: {
          type: 'string',
          description: 'API endpoint path',
        },
        data: {
          type: 'object',
          description: 'Request body data',
        },
      },
      required: ['endpoint', 'data'],
    },
  },
  {
    name: 'api_put',
    description: 'Perform a PUT request to the REST API',
    inputSchema: {
      type: 'object',
      properties: {
        endpoint: {
          type: 'string',
          description: 'API endpoint path',
        },
        data: {
          type: 'object',
          description: 'Request body data',
        },
      },
      required: ['endpoint', 'data'],
    },
  },
  {
    name: 'api_delete',
    description: 'Perform a DELETE request to the REST API',
    inputSchema: {
      type: 'object',
      properties: {
        endpoint: {
          type: 'string',
          description: 'API endpoint path',
        },
      },
      required: ['endpoint'],
    },
  },
];
```

### 5. Implement the MCP Server

Create `src/server.ts`:
```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { RestApiClient } from './api/client.js';
import { tools } from './tools/index.js';

export class RestApiMcpServer {
  private server: Server;
  private apiClient: RestApiClient;

  constructor(baseURL: string, apiKey?: string) {
    this.apiClient = new RestApiClient(baseURL, apiKey);
    this.server = new Server(
      {
        name: 'rest-api-mcp-server',
        version: '1.0.0',
      },
      {
        capabilities: {
          tools: {},
        },
      }
    );

    this.setupHandlers();
  }

  private setupHandlers(): void {
    // List available tools
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools,
    }));

    // Handle tool calls
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      try {
        switch (name) {
          case 'api_get': {
            const result = await this.apiClient.get(
              args.endpoint,
              args.params
            );
            return {
              content: [
                {
                  type: 'text',
                  text: JSON.stringify(result, null, 2),
                },
              ],
            };
          }

          case 'api_post': {
            const result = await this.apiClient.post(
              args.endpoint,
              args.data
            );
            return {
              content: [
                {
                  type: 'text',
                  text: JSON.stringify(result, null, 2),
                },
              ],
            };
          }

          case 'api_put': {
            const result = await this.apiClient.put(
              args.endpoint,
              args.data
            );
            return {
              content: [
                {
                  type: 'text',
                  text: JSON.stringify(result, null, 2),
                },
              ],
            };
          }

          case 'api_delete': {
            const result = await this.apiClient.delete(args.endpoint);
            return {
              content: [
                {
                  type: 'text',
                  text: JSON.stringify(result, null, 2),
                },
              ],
            };
          }

          default:
            throw new Error(`Unknown tool: ${name}`);
        }
      } catch (error: any) {
        return {
          content: [
            {
              type: 'text',
              text: `Error: ${error.message}`,
            },
          ],
          isError: true,
        };
      }
    });
  }

  async start(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error('REST API MCP Server running on stdio');
  }
}
```

### 6. Create Entry Point

Create `src/index.ts`:
```typescript
import dotenv from 'dotenv';
import { RestApiMcpServer } from './server.js';

dotenv.config();

const BASE_URL = process.env.API_BASE_URL || 'https://api.example.com';
const API_KEY = process.env.API_KEY;

const server = new RestApiMcpServer(BASE_URL, API_KEY);
server.start().catch(console.error);
```

### 7. Environment Configuration

Create `.env`:
```
API_BASE_URL=https://api.example.com
API_KEY=your_api_key_here
```

### 8. Update package.json

Add the following scripts:
```json
{
  "name": "mcp-rest-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node src/index.ts"
  }
}
```

## Configuration for GitHub Copilot

### Claude Desktop Configuration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "rest-api": {
      "command": "node",
      "args": ["/path/to/your/project/dist/index.js"],
      "env": {
        "API_BASE_URL": "https://api.example.com",
        "API_KEY": "your_api_key_here"
      }
    }
  }
}
```

## Advanced Features

### 1. Rate Limiting

Add rate limiting to prevent API abuse:

```typescript
import Bottleneck from 'bottleneck';

const limiter = new Bottleneck({
  maxConcurrent: 5,
  minTime: 200, // 200ms between requests
});

// Wrap API calls
const result = await limiter.schedule(() => this.apiClient.get(endpoint));
```

### 2. Response Caching

Implement caching for frequently accessed data:

```typescript
import NodeCache from 'node-cache';

const cache = new NodeCache({ stdTTL: 300 }); // 5 minutes

async get<T>(endpoint: string, params?: Record<string, any>): Promise<T> {
  const cacheKey = `${endpoint}-${JSON.stringify(params)}`;
  const cached = cache.get<T>(cacheKey);
  
  if (cached) {
    return cached;
  }
  
  const response = await this.client.get<T>(endpoint, { params });
  cache.set(cacheKey, response.data);
  return response.data;
}
```

### 3. Request/Response Logging

Add logging for debugging:

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Log requests
logger.info('API Request', { method: 'GET', endpoint, params });
```

### 4. Custom Error Handling

Implement robust error handling:

```typescript
class ApiError extends Error {
  constructor(
    message: string,
    public statusCode?: number,
    public response?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// In API client
try {
  const response = await this.client.get<T>(endpoint, { params });
  return response.data;
} catch (error: any) {
  if (error.response) {
    throw new ApiError(
      `API Error: ${error.response.statusText}`,
      error.response.status,
      error.response.data
    );
  }
  throw error;
}
```

## Testing

### Unit Tests

Create `src/__tests__/api.test.ts`:

```typescript
import { RestApiClient } from '../api/client';
import axios from 'axios';

jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

describe('RestApiClient', () => {
  it('should make GET request', async () => {
    const mockData = { id: 1, name: 'Test' };
    mockedAxios.create.mockReturnValue({
      get: jest.fn().mockResolvedValue({ data: mockData }),
    } as any);

    const client = new RestApiClient('https://api.example.com');
    const result = await client.get('/users/1');

    expect(result).toEqual(mockData);
  });
});
```

## Best Practices

1. **Security**
   - Never commit API keys to version control
   - Use environment variables for sensitive data
   - Implement proper authentication mechanisms
   - Validate and sanitize all inputs

2. **Performance**
   - Implement caching for frequently accessed data
   - Use connection pooling for HTTP requests
   - Set appropriate timeouts
   - Implement rate limiting

3. **Error Handling**
   - Provide meaningful error messages
   - Log errors for debugging
   - Implement retry logic for transient failures
   - Handle network timeouts gracefully

4. **Documentation**
   - Document all available tools and their parameters
   - Provide examples of API usage
   - Include API endpoint documentation
   - Document authentication requirements

5. **Monitoring**
   - Log all API requests and responses
   - Track error rates and response times
   - Monitor rate limit usage
   - Set up alerts for critical failures

## Troubleshooting

### Common Issues

1. **Connection Refused**
   - Check if the API base URL is correct
   - Verify network connectivity
   - Ensure firewall rules allow outbound connections

2. **Authentication Errors**
   - Verify API key is correct and not expired
   - Check if the API key has necessary permissions
   - Ensure authentication headers are properly formatted

3. **Rate Limiting**
   - Implement exponential backoff
   - Reduce request frequency
   - Consider caching responses

4. **MCP Server Not Starting**
   - Check if Node.js version is compatible (v18+)
   - Verify all dependencies are installed
   - Check for syntax errors in configuration

## Example Usage

Once the server is running, users can interact with it through natural language:

```
User: "Get all users from the API"
-> Tool: api_get with endpoint="/users"

User: "Create a new user named John Doe"
-> Tool: api_post with endpoint="/users" and data={"name": "John Doe"}

User: "Update user 123's email to john@example.com"
-> Tool: api_put with endpoint="/users/123" and data={"email": "john@example.com"}
```

## Resources

- [MCP SDK Documentation](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [Axios Documentation](https://axios-http.com/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)

## Contributing

When extending this server:
1. Follow the existing code structure
2. Add appropriate error handling
3. Update tool definitions as needed
4. Document new features
5. Write tests for new functionality

## License

Specify your license here (e.g., MIT, Apache 2.0)
