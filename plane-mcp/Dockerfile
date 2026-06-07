FROM python:3.11-slim

# Plane's official MCP server (PyPI: plane-mcp-server)
RUN pip install --no-cache-dir plane-mcp-server

EXPOSE 8211

# HTTP transport. Listens on 0.0.0.0:8211.
# Endpoints:
#   /http/api-key/mcp  -> header auth (Authorization: Bearer <PAT> + X-Workspace-slug)
#   /http/mcp          -> OAuth (needs real Plane OAuth app; unused here)
CMD ["plane-mcp-server", "http"]
