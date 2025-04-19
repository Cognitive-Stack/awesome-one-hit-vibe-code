# Instructions for Creating a New MCP Server

## Project Structure 

```
src/
├── config/ # Configuration management
├── services/ # Tool registration
├── tools/ # Tool implementations
├── types/ # TypeScript types
└── index.ts # Server entry
```


## 1. Initial Setup

### Create Project
```bash
mkdir your-mcp-server
cd your-mcp-server
npm init -y
```

### Install Dependencies

```bash
# Core dependencies
npm install fastmcp zod dotenv
# Add your domain-specific dependencies here

# Dev dependencies
npm install -D typescript tsx @types/node jest ts-jest @types/jest
```

## 2. Configuration Files

### package.json
```json
{
  "name": "your-mcp-server",
  "version": "1.0.0",
  "description": "Your MCP Server Description",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "bin": {
    "your-mcp-server": "dist/index.js"
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "dev": "tsx src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "prepare": "npm run build"
  }
}
```

### jest.config.js
```javascript
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
  moduleFileExtensions: ["ts", "js"],
  transform: {
    "^.+\\.ts$": "ts-jest",
  },
  testMatch: ["**/__tests__/**/*.test.ts"],
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/src/$1",
  },
};
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "lib": ["ES2020"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

## 3. Implementation Guide

### Types (src/types/tools.ts)
```typescript
import { z } from "zod";
import { FastMCP } from "fastmcp";

export type ToolConfig = {
  name: string;
  description: string;
  parameters: z.ZodObject<any>;
  execute: (args: any) => Promise<string>;
};

export type Tool = FastMCP["addTool"];
```

### Configuration (src/config/env.ts)
```typescript
export const getConfig = () => {
  const requiredVar = process.env.REQUIRED_VARIABLE;
  if (!requiredVar) {
    throw new Error("REQUIRED_VARIABLE environment variable is not set");
  }
  return { requiredVar };
};
```

### Tool Implementation (src/tools/example-tool.ts)
```typescript
import { getConfig } from "../config/env";

export default async (
  param1: string,
  param2: string,
  options: { option1?: string; option2?: boolean }
) => {
  const config = getConfig();

  // Validate inputs
  if (!options.option1 && !options.option2) {
    throw new Error("Either option1 or option2 must be provided");
  }

  // Implement tool logic
  try {
    // Your implementation here
    return "Operation successful";
  } catch (error) {
    throw new Error(`Operation failed: ${error.message}`);
  }
};
```

### Tool Registration (src/services/tools.ts)
```typescript
import { z } from "zod";
import exampleTool from "../tools/example-tool";
import { ToolConfig } from "../types/tools";

export const tools: ToolConfig[] = [
  {
    name: "example-tool",
    description: "Description of what the tool does",
    parameters: z.object({
      param1: z.string().describe("Description of param1"),
      param2: z.string().describe("Description of param2"),
      option1: z.string().optional().describe("Description of option1"),
      option2: z.boolean().optional().describe("Description of option2"),
    }),
    execute: async (args) => {
      return await exampleTool(args.param1, args.param2, {
        option1: args.option1,
        option2: args.option2,
      });
    },
  },
];
```

### Server Entry Point (src/index.ts)
```typescript
import { FastMCP } from "fastmcp";
import { tools } from "./services/tools";
import { Tool } from "./types/tools";

const server = new FastMCP({
  name: "Your Server Name",
  version: "1.0.0",
});

// Register all tools
tools.forEach((tool) => {
  (server.addTool as Tool)(tool);
});

// Get transport type from environment variable or default to stdio
const transportType = process.env.TRANSPORT_TYPE || "stdio";

if (transportType === "sse") {
  server.start({
    transportType: "sse",
    sse: {
      endpoint: "/sse",
      port: parseInt(process.env.PORT || "8080", 10),
    },
  });
} else {
  server.start({
    transportType: "stdio",
  });
}
```

## 4. Testing

### Example Test (src/tools/__tests__/example-tool.test.ts)
```typescript
import exampleTool from "../example-tool";

jest.mock("../../config/env", () => ({
  getConfig: jest.fn().mockReturnValue({
    requiredVar: "test-value",
  }),
}));

describe("exampleTool", () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it("should handle successful operation", async () => {
    const result = await exampleTool("param1", "param2", { option1: "test" });
    expect(result).toBe("Operation successful");
  });

  it("should throw error when required options are missing", async () => {
    await expect(
      exampleTool("param1", "param2", {})
    ).rejects.toThrow("Either option1 or option2 must be provided");
  });
});
```

## 5. Best Practices

### Error Handling
- Use descriptive error messages
- Implement proper error types
- Handle all potential error cases
- Validate inputs thoroughly

### Type Safety
- Use TypeScript strictly
- Define interfaces for all data structures
- Use Zod for runtime validation
- Avoid any type when possible

### Testing
- Write unit tests for all tools
- Mock external dependencies
- Test error cases
- Test configuration validation

### Configuration
- Use environment variables for configuration
- Validate all required variables
- Provide clear error messages for missing config
- Use TypeScript types for config objects

### Code Organization
- Keep tools modular and focused
- Separate concerns (config, tools, services)
- Use consistent naming conventions
- Document public interfaces

## 6. Publishing

1. Update package.json with your details
2. Build the project: `npm run build`
3. Test the build: `npm test`
4. Publish to npm: `npm publish`

## 7. Environment Variables
Create a `.env` file:
```env
REQUIRED_VARIABLE=your-value
TRANSPORT_TYPE=stdio
PORT=8080
```

Remember to add `.env` to your `.gitignore` file.