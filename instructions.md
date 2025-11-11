# Creating an MCP Server in Golang to Consume REST APIs

This guide provides step-by-step instructions for building a Model Context Protocol (MCP) server using Golang that can consume and expose REST API endpoints.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Understanding MCP](#understanding-mcp)
- [Project Setup](#project-setup)
- [MCP Server Implementation](#mcp-server-implementation)
- [REST API Integration](#rest-api-integration)
- [Testing](#testing)
- [Deployment](#deployment)

## Prerequisites

Before you begin, ensure you have the following installed:

- **Go** (version 1.19 or higher): [Download Go](https://golang.org/dl/)
- **Git**: For version control
- Basic understanding of:
  - Go programming language
  - REST API concepts
  - JSON-RPC protocol
  - Standard input/output (stdio) communication

## Understanding MCP

The Model Context Protocol (MCP) is a protocol that enables AI models to interact with external data sources and tools. An MCP server:

- Exposes **resources** (data sources like files, databases, or API endpoints)
- Provides **tools** (executable functions that perform actions)
- Implements **prompts** (templated interactions)
- Communicates via JSON-RPC 2.0 over stdio or SSE (Server-Sent Events)

### MCP Architecture

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│   AI/LLM    │ ◄─────► │ MCP Server  │ ◄─────► │  REST API   │
│   Client    │   MCP    │  (Golang)   │   HTTP   │  Endpoint   │
└─────────────┘          └─────────────┘          └─────────────┘
```

## Project Setup

### 1. Initialize Your Go Project

```bash
# Create project directory
mkdir mcp-rest-server
cd mcp-rest-server

# Initialize Go module
go mod init github.com/yourusername/mcp-rest-server
```

### 2. Install Required Dependencies

```bash
# Install MCP SDK for Go (if available) or JSON-RPC libraries
go get github.com/sourcegraph/jsonrpc2
go get github.com/go-resty/resty/v2  # For REST API calls
```

### 3. Project Structure

Create the following directory structure:

```
mcp-rest-server/
├── main.go
├── internal/
│   ├── server/
│   │   ├── server.go
│   │   └── handlers.go
│   ├── api/
│   │   └── client.go
│   └── types/
│       └── types.go
├── config/
│   └── config.go
├── go.mod
└── go.sum
```

## MCP Server Implementation

### 1. Define MCP Types (`internal/types/types.go`)

```go
package types

// MCP Protocol types
type InitializeRequest struct {
    ProtocolVersion string                 `json:"protocolVersion"`
    Capabilities    ClientCapabilities     `json:"capabilities"`
    ClientInfo      ClientInfo             `json:"clientInfo"`
}

type InitializeResponse struct {
    ProtocolVersion string              `json:"protocolVersion"`
    Capabilities    ServerCapabilities  `json:"capabilities"`
    ServerInfo      ServerInfo          `json:"serverInfo"`
}

type ClientCapabilities struct {
    Roots    *RootCapability    `json:"roots,omitempty"`
    Sampling *SamplingCapability `json:"sampling,omitempty"`
}

type ServerCapabilities struct {
    Resources *ResourceCapability `json:"resources,omitempty"`
    Tools     *ToolCapability     `json:"tools,omitempty"`
    Prompts   *PromptCapability   `json:"prompts,omitempty"`
}

type ResourceCapability struct {
    Subscribe bool `json:"subscribe"`
    ListChanged bool `json:"listChanged"`
}

type ToolCapability struct {
    ListChanged bool `json:"listChanged"`
}

type PromptCapability struct {
    ListChanged bool `json:"listChanged"`
}

type RootCapability struct{}
type SamplingCapability struct{}

type ClientInfo struct {
    Name    string `json:"name"`
    Version string `json:"version"`
}

type ServerInfo struct {
    Name    string `json:"name"`
    Version string `json:"version"`
}

// Tool definitions
type Tool struct {
    Name        string      `json:"name"`
    Description string      `json:"description"`
    InputSchema ToolSchema  `json:"inputSchema"`
}

type ToolSchema struct {
    Type       string                 `json:"type"`
    Properties map[string]interface{} `json:"properties"`
    Required   []string               `json:"required,omitempty"`
}

type CallToolRequest struct {
    Name      string                 `json:"name"`
    Arguments map[string]interface{} `json:"arguments,omitempty"`
}

type CallToolResponse struct {
    Content []ContentItem `json:"content"`
    IsError bool          `json:"isError,omitempty"`
}

type ContentItem struct {
    Type string `json:"type"`
    Text string `json:"text"`
}

// Resource definitions
type Resource struct {
    URI         string `json:"uri"`
    Name        string `json:"name"`
    Description string `json:"description,omitempty"`
    MimeType    string `json:"mimeType,omitempty"`
}

type ListResourcesResponse struct {
    Resources []Resource `json:"resources"`
}

type ReadResourceRequest struct {
    URI string `json:"uri"`
}

type ReadResourceResponse struct {
    Contents []ResourceContent `json:"contents"`
}

type ResourceContent struct {
    URI      string `json:"uri"`
    MimeType string `json:"mimeType,omitempty"`
    Text     string `json:"text,omitempty"`
}
```

### 2. REST API Client (`internal/api/client.go`)

```go
package api

import (
    "encoding/json"
    "fmt"
    "github.com/go-resty/resty/v2"
)

type Client struct {
    baseURL string
    client  *resty.Client
}

func NewClient(baseURL string) *Client {
    return &Client{
        baseURL: baseURL,
        client:  resty.New(),
    }
}

// GetData fetches data from the REST API
func (c *Client) GetData(endpoint string) (map[string]interface{}, error) {
    resp, err := c.client.R().
        SetHeader("Accept", "application/json").
        Get(c.baseURL + endpoint)
    
    if err != nil {
        return nil, fmt.Errorf("request failed: %w", err)
    }

    if resp.StatusCode() != 200 {
        return nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode())
    }

    var result map[string]interface{}
    if err := json.Unmarshal(resp.Body(), &result); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }

    return result, nil
}

// PostData sends data to the REST API
func (c *Client) PostData(endpoint string, data interface{}) (map[string]interface{}, error) {
    resp, err := c.client.R().
        SetHeader("Content-Type", "application/json").
        SetHeader("Accept", "application/json").
        SetBody(data).
        Post(c.baseURL + endpoint)
    
    if err != nil {
        return nil, fmt.Errorf("request failed: %w", err)
    }

    if resp.StatusCode() < 200 || resp.StatusCode() >= 300 {
        return nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode())
    }

    var result map[string]interface{}
    if err := json.Unmarshal(resp.Body(), &result); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }

    return result, nil
}
```

### 3. MCP Server Implementation (`internal/server/server.go`)

```go
package server

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log"

    "github.com/sourcegraph/jsonrpc2"
    "github.com/yourusername/mcp-rest-server/internal/api"
    "github.com/yourusername/mcp-rest-server/internal/types"
)

type MCPServer struct {
    apiClient *api.Client
    info      types.ServerInfo
}

func NewMCPServer(apiBaseURL string) *MCPServer {
    return &MCPServer{
        apiClient: api.NewClient(apiBaseURL),
        info: types.ServerInfo{
            Name:    "REST API MCP Server",
            Version: "1.0.0",
        },
    }
}

// Handle implements the JSON-RPC handler interface
func (s *MCPServer) Handle(ctx context.Context, conn *jsonrpc2.Conn, req *jsonrpc2.Request) (interface{}, error) {
    switch req.Method {
    case "initialize":
        return s.handleInitialize(req)
    case "tools/list":
        return s.handleListTools(req)
    case "tools/call":
        return s.handleCallTool(req)
    case "resources/list":
        return s.handleListResources(req)
    case "resources/read":
        return s.handleReadResource(req)
    default:
        return nil, &jsonrpc2.Error{
            Code:    jsonrpc2.CodeMethodNotFound,
            Message: fmt.Sprintf("method not found: %s", req.Method),
        }
    }
}

func (s *MCPServer) handleInitialize(req *jsonrpc2.Request) (interface{}, error) {
    var initReq types.InitializeRequest
    if err := json.Unmarshal(*req.Params, &initReq); err != nil {
        return nil, err
    }

    log.Printf("Initialized by client: %s v%s", initReq.ClientInfo.Name, initReq.ClientInfo.Version)

    return types.InitializeResponse{
        ProtocolVersion: "2024-11-05",
        Capabilities: types.ServerCapabilities{
            Tools: &types.ToolCapability{
                ListChanged: false,
            },
            Resources: &types.ResourceCapability{
                Subscribe:   false,
                ListChanged: false,
            },
        },
        ServerInfo: s.info,
    }, nil
}

func (s *MCPServer) handleListTools(req *jsonrpc2.Request) (interface{}, error) {
    tools := []types.Tool{
        {
            Name:        "fetch_api_data",
            Description: "Fetch data from the REST API endpoint",
            InputSchema: types.ToolSchema{
                Type: "object",
                Properties: map[string]interface{}{
                    "endpoint": map[string]string{
                        "type":        "string",
                        "description": "The API endpoint path to fetch data from",
                    },
                },
                Required: []string{"endpoint"},
            },
        },
        {
            Name:        "post_api_data",
            Description: "Send data to the REST API endpoint",
            InputSchema: types.ToolSchema{
                Type: "object",
                Properties: map[string]interface{}{
                    "endpoint": map[string]string{
                        "type":        "string",
                        "description": "The API endpoint path to post data to",
                    },
                    "data": map[string]string{
                        "type":        "object",
                        "description": "The data to send to the endpoint",
                    },
                },
                Required: []string{"endpoint", "data"},
            },
        },
    }

    return map[string]interface{}{
        "tools": tools,
    }, nil
}

func (s *MCPServer) handleCallTool(req *jsonrpc2.Request) (interface{}, error) {
    var callReq types.CallToolRequest
    if err := json.Unmarshal(*req.Params, &callReq); err != nil {
        return nil, err
    }

    switch callReq.Name {
    case "fetch_api_data":
        return s.handleFetchAPIData(callReq.Arguments)
    case "post_api_data":
        return s.handlePostAPIData(callReq.Arguments)
    default:
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: fmt.Sprintf("Unknown tool: %s", callReq.Name),
                },
            },
            IsError: true,
        }, nil
    }
}

func (s *MCPServer) handleFetchAPIData(args map[string]interface{}) (types.CallToolResponse, error) {
    endpoint, ok := args["endpoint"].(string)
    if !ok {
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: "Missing or invalid 'endpoint' parameter",
                },
            },
            IsError: true,
        }, nil
    }

    data, err := s.apiClient.GetData(endpoint)
    if err != nil {
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: fmt.Sprintf("Error fetching data: %v", err),
                },
            },
            IsError: true,
        }, nil
    }

    jsonData, _ := json.MarshalIndent(data, "", "  ")
    return types.CallToolResponse{
        Content: []types.ContentItem{
            {
                Type: "text",
                Text: string(jsonData),
            },
        },
    }, nil
}

func (s *MCPServer) handlePostAPIData(args map[string]interface{}) (types.CallToolResponse, error) {
    endpoint, ok := args["endpoint"].(string)
    if !ok {
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: "Missing or invalid 'endpoint' parameter",
                },
            },
            IsError: true,
        }, nil
    }

    data, ok := args["data"]
    if !ok {
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: "Missing 'data' parameter",
                },
            },
            IsError: true,
        }, nil
    }

    result, err := s.apiClient.PostData(endpoint, data)
    if err != nil {
        return types.CallToolResponse{
            Content: []types.ContentItem{
                {
                    Type: "text",
                    Text: fmt.Sprintf("Error posting data: %v", err),
                },
            },
            IsError: true,
        }, nil
    }

    jsonData, _ := json.MarshalIndent(result, "", "  ")
    return types.CallToolResponse{
        Content: []types.ContentItem{
            {
                Type: "text",
                Text: string(jsonData),
            },
        },
    }, nil
}

func (s *MCPServer) handleListResources(req *jsonrpc2.Request) (interface{}, error) {
    resources := []types.Resource{
        {
            URI:         "rest://api/status",
            Name:        "API Status",
            Description: "Current status of the REST API",
            MimeType:    "application/json",
        },
    }

    return types.ListResourcesResponse{
        Resources: resources,
    }, nil
}

func (s *MCPServer) handleReadResource(req *jsonrpc2.Request) (interface{}, error) {
    var readReq types.ReadResourceRequest
    if err := json.Unmarshal(*req.Params, &readReq); err != nil {
        return nil, err
    }

    // Example: Read resource based on URI
    data, err := s.apiClient.GetData("/status")
    if err != nil {
        return nil, err
    }

    jsonData, _ := json.MarshalIndent(data, "", "  ")
    return types.ReadResourceResponse{
        Contents: []types.ResourceContent{
            {
                URI:      readReq.URI,
                MimeType: "application/json",
                Text:     string(jsonData),
            },
        },
    }, nil
}

// Run starts the MCP server
func (s *MCPServer) Run() error {
    handler := jsonrpc2.HandlerWithError(s.Handle)
    
    log.Println("Starting MCP server...")
    
    conn := jsonrpc2.NewConn(
        context.Background(),
        jsonrpc2.NewBufferedStream(stdinout{}, jsonrpc2.VSCodeObjectCodec{}),
        handler,
    )

    <-conn.DisconnectNotify()
    log.Println("MCP server stopped")
    return nil
}

// stdinout implements io.ReadWriteCloser for stdio
type stdinout struct{}

func (stdinout) Read(p []byte) (int, error) {
    return io.ReadFull(io.Reader(io.Stdin), p)
}

func (stdinout) Write(p []byte) (int, error) {
    return io.Writer(io.Stdout).Write(p)
}

func (stdinout) Close() error {
    return nil
}
```

### 4. Main Entry Point (`main.go`)

```go
package main

import (
    "flag"
    "log"
    "os"

    "github.com/yourusername/mcp-rest-server/internal/server"
)

func main() {
    apiURL := flag.String("api-url", "", "Base URL of the REST API to consume")
    flag.Parse()

    if *apiURL == "" {
        log.Fatal("Please provide the REST API URL using -api-url flag")
    }

    // Set up logging to stderr (stdout is used for MCP communication)
    log.SetOutput(os.Stderr)

    // Create and run the MCP server
    mcpServer := server.NewMCPServer(*apiURL)
    if err := mcpServer.Run(); err != nil {
        log.Fatalf("Server error: %v", err)
    }
}
```

### 5. Configuration (`config/config.go`)

```go
package config

import (
    "os"
)

type Config struct {
    APIURL      string
    APIKey      string
    ServerName  string
    ServerVersion string
}

func LoadConfig() *Config {
    return &Config{
        APIURL:        getEnv("API_URL", "http://localhost:8080"),
        APIKey:        getEnv("API_KEY", ""),
        ServerName:    "REST API MCP Server",
        ServerVersion: "1.0.0",
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

## REST API Integration

### Adding Authentication

If your REST API requires authentication, modify the `api.Client`:

```go
func NewClient(baseURL, apiKey string) *Client {
    client := resty.New()
    
    // Add authentication headers
    if apiKey != "" {
        client.SetHeader("Authorization", "Bearer "+apiKey)
    }
    
    return &Client{
        baseURL: baseURL,
        client:  client,
    }
}
```

### Handling Different Response Formats

Add methods to handle different content types:

```go
func (c *Client) GetXML(endpoint string) (string, error) {
    resp, err := c.client.R().
        SetHeader("Accept", "application/xml").
        Get(c.baseURL + endpoint)
    
    if err != nil {
        return "", err
    }
    
    return string(resp.Body()), nil
}
```

## Testing

### 1. Unit Tests (`internal/server/server_test.go`)

```go
package server

import (
    "testing"
)

func TestMCPServer_Initialize(t *testing.T) {
    server := NewMCPServer("http://localhost:8080")
    
    if server == nil {
        t.Fatal("Failed to create MCP server")
    }
    
    if server.info.Name != "REST API MCP Server" {
        t.Errorf("Expected server name 'REST API MCP Server', got '%s'", server.info.Name)
    }
}
```

### 2. Integration Testing

Create a test script to verify the server works:

```bash
# test_server.sh
#!/bin/bash

# Start the server
go run main.go -api-url "http://localhost:8080" &
SERVER_PID=$!

# Wait for server to start
sleep 2

# Send initialize request
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | go run main.go -api-url "http://localhost:8080"

# Clean up
kill $SERVER_PID
```

### 3. Run Tests

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run with verbose output
go test -v ./...
```

## Building and Running

### Build the Server

```bash
# Build for current platform
go build -o mcp-server main.go

# Build for Linux
GOOS=linux GOARCH=amd64 go build -o mcp-server-linux main.go

# Build for Windows
GOOS=windows GOARCH=amd64 go build -o mcp-server.exe main.go

# Build for macOS
GOOS=darwin GOARCH=amd64 go build -o mcp-server-mac main.go
```

### Run the Server

```bash
# Run directly
go run main.go -api-url "https://api.example.com"

# Run compiled binary
./mcp-server -api-url "https://api.example.com"

# With environment variables
export API_URL="https://api.example.com"
export API_KEY="your-api-key"
./mcp-server -api-url "$API_URL"
```

## Deployment

### 1. Docker Deployment

Create a `Dockerfile`:

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o mcp-server main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /app/mcp-server .

ENTRYPOINT ["./mcp-server"]
```

Build and run with Docker:

```bash
# Build image
docker build -t mcp-rest-server .

# Run container
docker run -e API_URL="https://api.example.com" mcp-rest-server -api-url "$API_URL"
```

### 2. MCP Client Configuration

To use your server with an MCP client (like Claude Desktop), add to the client's configuration:

```json
{
  "mcpServers": {
    "rest-api": {
      "command": "/path/to/mcp-server",
      "args": ["-api-url", "https://api.example.com"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

### 3. Systemd Service (Linux)

Create `/etc/systemd/system/mcp-server.service`:

```ini
[Unit]
Description=MCP REST API Server
After=network.target

[Service]
Type=simple
User=mcp
Environment="API_URL=https://api.example.com"
Environment="API_KEY=your-api-key"
ExecStart=/usr/local/bin/mcp-server -api-url ${API_URL}
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable mcp-server
sudo systemctl start mcp-server
```

## Best Practices

### 1. Error Handling

Always handle errors gracefully and return meaningful error messages:

```go
if err != nil {
    return types.CallToolResponse{
        Content: []types.ContentItem{
            {
                Type: "text",
                Text: fmt.Sprintf("Error: %v", err),
            },
        },
        IsError: true,
    }, nil
}
```

### 2. Logging

Use structured logging for better debugging:

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
logger.Info("API request", "endpoint", endpoint, "method", "GET")
```

### 3. Rate Limiting

Implement rate limiting for API calls:

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(10), 1) // 10 requests per second

func (c *Client) GetData(endpoint string) (map[string]interface{}, error) {
    if err := limiter.Wait(context.Background()); err != nil {
        return nil, err
    }
    // ... rest of the code
}
```

### 4. Caching

Add caching for frequently accessed data:

```go
import (
    "time"
    "github.com/patrickmn/go-cache"
)

type Client struct {
    baseURL string
    client  *resty.Client
    cache   *cache.Cache
}

func NewClient(baseURL string) *Client {
    return &Client{
        baseURL: baseURL,
        client:  resty.New(),
        cache:   cache.New(5*time.Minute, 10*time.Minute),
    }
}
```

### 5. Configuration Management

Use environment variables and config files:

```go
import "github.com/spf13/viper"

viper.SetConfigName("config")
viper.AddConfigPath(".")
viper.AutomaticEnv()
```

## Troubleshooting

### Common Issues

1. **Server not responding**
   - Check that stdio communication is working correctly
   - Verify logging is sent to stderr, not stdout
   - Ensure the server is running with correct permissions

2. **API connection failures**
   - Verify the API URL is correct and accessible
   - Check network connectivity and firewall rules
   - Ensure API authentication credentials are valid

3. **JSON-RPC errors**
   - Validate request/response format matches MCP specification
   - Check protocol version compatibility
   - Review error messages in stderr logs

## Additional Resources

- [MCP Specification](https://modelcontextprotocol.io/)
- [Go Documentation](https://golang.org/doc/)
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [Resty HTTP Client](https://github.com/go-resty/resty)

## Example: Complete Working Server

See the complete implementation above. To get started quickly:

1. Copy all the code snippets into their respective files
2. Update the module path in `go.mod` and import statements
3. Set your API URL: `export API_URL="https://your-api.com"`
4. Build and run: `go run main.go -api-url "$API_URL"`

Your MCP server is now ready to consume your REST API and expose it to AI clients!

## Next Steps

- Add more sophisticated tools based on your API capabilities
- Implement resource subscriptions for real-time updates
- Add comprehensive error handling and validation
- Create automated tests for all endpoints
- Set up CI/CD pipeline for automated deployment
- Monitor server performance and API usage
- Implement request/response caching strategies
- Add support for WebSocket connections if needed
