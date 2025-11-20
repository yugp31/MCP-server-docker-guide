# Setting Up MCP Servers with Docker

A practical guide to running **Model Context Protocol (MCP)** servers using Docker, enabling AI assistants to interact with your local tools, files, and APIs in a safe, containerized environment.

---

## What is MCP?

**Model Context Protocol (MCP)** is an open standard that allows AI models to connect with external tools, APIs, and databases through a standardized interface. Instead of building custom integrations for every tool, MCP provides a universal plug-and-play system for AI applications.

### Key Benefits

- **Reduces hallucinations** by providing LLMs with access to real-time, reliable data sources.
- **Enables AI agents** to perform complex tasks like file operations, web searches, and database queries.
- **Modular architecture** allows switching models or tools without changing integration code.
- **Standardized protocol** using JSON-RPC 2.0 for communication.

---

## Architecture Overview

MCP follows a **client-host-server** architecture with three main components:

- **Host:** Applications like Claude Desktop, Cursor, or Windsurf that contain the LLM
- **Client:** Lightweight protocol client embedded within the host, maintaining 1:1 connections with servers
- **Server:** Independent processes that expose capabilities (data access, tools, prompts) through the MCP standard

### Transport Methods

- **STDIO:** For local integrations where the server runs in the same environment (most common)
- **HTTP+SSE:** For remote connections using Server-Sent Events

---

## Why Use Docker for MCP?

- **Isolation:** Each MCP server runs in its own container, preventing conflicts
- **No dependency hell:** Skip installing Node.js, Python, or other runtimes on your system
- **Easy cleanup:** Remove containers without affecting your base system
- **Portability:** Same setup works across different Linux distributions
- **Security:** Containers limit filesystem and network access to only what you specify

---

## ⚠️ Security Notice

**IMPORTANT:** MCP servers have access to your system resources. Always:

- ✅ Review server source code before building
- ✅ Use Docker containers for isolation (recommended)
- ✅ Use read-only mounts (`:ro`) when possible
- ✅ Store API keys in environment variables, not config files
- ✅ Limit filesystem access to specific directories only
- ✅ Never expose MCP servers directly to the internet
- ✅ Keep Docker images updated regularly

For detailed security guidelines, see [MCP Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices).


## Prerequisites

Before starting, ensure you have:

- **Docker** installed and running ([Installation Guide](https://docs.docker.com/get-docker/))
- **Git** (for cloning repositories)
- **An MCP-compatible client** such as:
  - [Claude Desktop](https://claude.ai/download)
  - [Cursor](https://cursor.sh/)
  - [Windsurf](https://codeium.com/windsurf)
- **API keys** (optional, for specific services):
  - Brave Search API
  - Google Maps API

### Quick Docker Setup (Linux)

Debian/Ubuntu
```
sudo apt update
sudo apt install docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```
Arch Linux
```
sudo pacman -S docker docker-compose
sudo systemctl start docker.service
sudo systemctl enable docker.service
```
Fedora
```
sudo dnf install docker docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```
Add your user to docker group (all distributions)
```
sudo usermod -aG docker $USER
```

**Important:** Log out and log back in for group changes to take effect.

### Verify Docker Installation
```
docker --version
docker run hello-world
```
---

## Step 1: Clone the Official MCP Servers Repository
```
git clone https://github.com/modelcontextprotocol/servers.git
cd servers
```
---

## Step 2: Build MCP Server Images

Create a build script to compile all available MCP servers:
```
nano build_servers.sh
```
Paste the following content:
```
#!/bin/bash
set -euo pipefail


# Security: Verify we're in the official MCP repository
if [ ! -f "CONTRIBUTING.md" ] || ! grep -q "modelcontextprotocol" "CONTRIBUTING.md" 2>/dev/null; then
    echo "⚠️  Warning: Not in official MCP servers repository"
    echo "For security, only build from trusted sources"
    read -p "Continue anyway? (yes/no): " confirm
    if [ "$confirm" != "yes" ]; then
        exit 1
    fi
fi

SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE}")" && pwd)
DOCKER_SRC_DIR="$SCRIPT_DIR/src"

for folder in "$DOCKER_SRC_DIR"/*; do
if [ -d "$folder" ]; then
folder_name=$(basename "$folder")
# Skip folders that don't need building
  if [ "$folder_name" == "redis" ]; then
        echo "⏭️  Skipping '$folder_name': Redis does not require build"
        continue
    fi
    
    dockerfile="$folder/Dockerfile"
    
    if [ -f "$dockerfile" ]; then
        # Check if Dockerfile references uv.lock
        if grep -q "uv\.lock" "$dockerfile"; then
            if [ ! -f "$folder/uv.lock" ]; then
                echo "⏭️  Skipping '$folder_name': uv.lock not found"
                continue
            fi
        fi
        
        # Determine build context
        if grep -E -q "COPY[[:space:]]+src/${folder_name}\b" "$dockerfile"; then
            context="$SCRIPT_DIR"
            echo "🔨 Building '$folder_name' with repository root as context"
        else
            context="$folder"
            echo "🔨 Building '$folder_name' with project folder as context"
        fi
        
        docker build -t "$folder_name" -f "$dockerfile" "$context"
        echo "✅ Successfully built '$folder_name'"
    else
        echo "⏭️  Skipping '$folder_name': Dockerfile not found"
    fi
fi
done

echo ""
echo "🎉 Build complete! Run 'docker images' to see all built MCP servers."

```

Make the script executable and run it:
```
chmod +x build_servers.sh
./build_servers.sh
```


**Note:** Building may take 10-20 minutes depending on your internet speed and system performance.

### Verify Built Images
```
docker images
```

You should see images like: `filesystem`, `brave-search`, `git`, `sqlite`, `fetch`, `puppeteer`, `memory`, `google-maps`, and more

---

## Step 3: Configure Your MCP Client

MCP servers communicate via **STDIO** (standard input/output), not HTTP. Configuration differs by client.

### For Claude Desktop (Linux)

Create or edit the configuration file:
```
mkdir -p ~/.config/Claude
nano ~/.config/Claude/claude_desktop_config.json
```

Add the following configuration:
```
{
  "mcpServers": {
    "filesystem": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--mount", "type=bind,src=/home/YOUR_USERNAME/mcp-workspace,dst=/workspace,readonly",
        "--memory", "512m",
        "--cpus", "0.5",
        "filesystem",
        "/workspace"
      ]
    },
    "git": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--mount", "type=bind,src=/path/to/your/repo,dst=/repo",
        "git",
        "--repository", "/repo"
      ]
    },
    "brave-search": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "BRAVE_API_KEY",
        "brave-search"
      ],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    },
    "sqlite": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--mount", "type=bind,src=/home/YOUR_USERNAME/databases,dst=/data",
        "sqlite"
      ]
    },
    "fetch": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "fetch"]
    },
    "memory": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "memory"]
    },
    "puppeteer": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--init",
        "--cap-drop=ALL",
        "--security-opt=no-new-privileges",
        "-e", "DOCKER_CONTAINER=true",
        "puppeteer"
      ]
    },
    "google-maps": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "GOOGLE_MAPS_API_KEY",
        "google-maps"
      ],
      "env": {
        "GOOGLE_MAPS_API_KEY": "${GOOGLE_MAPS_API_KEY}"
      }
    }
  }
}

```
### Configuration Notes

**IMPORTANT:** Update these values:

1. **Filesystem path:** Replace `/home/YOUR_USERNAME/mcp-workspace` with your actual workspace directory
`mkdir -p ~/mcp-workspace`
2. **API keys:** Replace `YOUR_BRAVE_API_KEY_HERE` and `YOUR_GOOGLE_MAPS_API_KEY_HERE` with your actual keys
3. **For other MCP clients:** Check their documentation for configuration file locations

---

## Step 4: Setting Up Environment Variables Securely

Instead of hardcoding API keys in the config file, store them as environment variables:

### Linux (Bash/Zsh)

Add to `~/.bashrc` or `~/.zshrc`:



## Step 5: Test Your Setup

### Restart Your MCP Client

**Completely quit and restart** your MCP client (don't just reload) to load the new configuration.

### Test Each Server

Try these commands in your AI assistant:
**Filesystem:**
```
"Create a new file called test.txt in my workspace"
```

**Brave Search:**
```
"Search the web for latest Docker updates using brave-search"
```

**Git:**
"Show me the git status of the current repository"

**SQLite:**
```
"Create a new SQLite database for tracking my projects"
```

**Memory:**
```
"Remember that my favorite programming language is Python"
```

The AI assistant should request permission before executing operations and show results.

---

## Available MCP Servers

| Server | Description | Requires API Key |
|--------|-------------|------------------|
| **filesystem** | Secure file operations (read, write, delete) | No |
| **brave-search** | Web search via Brave Search API | Yes |
| **git** | Git repository operations and version control | No |
| **sqlite** | SQLite database creation and queries | No |
| **fetch** | Fetch and parse web content | No |
| **puppeteer** | Browser automation and web scraping | No |
| **memory** | Persistent knowledge graph storage | No |
| **google-maps** | Location services and directions | Yes |
| **postgres** | PostgreSQL database access | Connection string |
| **sequentialthinking** | Problem-solving workflows | No |

---

## Handy Docker Commands
List all built images:
```
docker images
```
List running containers:
```
docker ps
```

List all containers (including stopped):
```
docker ps -a
```

Remove unused containers:
```
docker container prune
```

Remove unused images:
```
docker image prune
```

View logs for a specific run (if it fails):
```
docker logs <container_id>
```

Test a server manually:
```
docker run -i --rm filesystem
```

Remove a specific image:
```
docker rmi <image_name>
```

---

## Troubleshooting

### Servers Not Appearing in Client

1. Completely restart your MCP client (quit and reopen, don't just reload)
2. Verify Docker is running: `systemctl status docker` or `docker ps`
3. Check configuration file for JSON syntax errors
4. Ensure all file paths are absolute, not relative

### Permission Errors

Verify you're in the docker group
```
groups $USER
```
If not, add yourself and log out/in
```
sudo usermod -aG docker $USER
```

### Filesystem Mount Errors

- Ensure the directory exists:
  ```
  mkdir -p ~/mcp-workspace
  ```
- Use absolute paths only (no `~` or relative paths)
- Check read/write permissions:
  ```
  ls -la ~/mcp-workspace
  ```

### API Key Failures

- Verify no extra spaces or quotes in API keys
- Test API keys directly with curl if possible
- Check environment variables are properly set in config

### Build Failures
```
Clean Docker cache and rebuild
docker system prune -a
./build_servers.sh
```

---

## Security Best Practices

1. **Limit filesystem access:** Only mount directories the AI needs
2. **Use read-only mounts when possible:**
"--mount", "type=bind,src=/path,dst=/workspace,readonly"

3. **Never hardcode API keys:** Always use environment variables
4. **Review permissions:** Create limited database users for query-only operations
5. **Monitor resources:** Use `docker stats` to track container resource usage
6. **Keep images updated:** Regularly rebuild images for security patches

---

## Advanced: Custom MCP Servers

Want to build your own MCP server? Check these resources:

- [Official MCP Build Guide](https://modelcontextprotocol.io/docs/develop/build-server)
- [Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)

---

## Resources

- **MCP Documentation:** [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
- **MCP Servers Repository:** [https://github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)
- **Docker Documentation:** [https://docs.docker.com](https://docs.docker.com)
- **Claude Desktop MCP Guide:** [https://support.claude.com](https://support.claude.com)

---

**💡 Tip:** Docker makes experimenting with MCP servers safe and simple — if something breaks, just remove the container and start fresh. No system-wide installations to worry about.

---

## Contributing

Contributions are welcome! Feel free to:
- Open issues for bugs or suggestions
- Submit pull requests with improvements
- Fork and create your own version

This project is meant to be community-maintained and updated over time.

## License

This guide is released under the [CC0 v1.0 License](LICENSE). Use it freely, modify it, share it - no restrictions!
