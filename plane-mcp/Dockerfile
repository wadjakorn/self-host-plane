FROM python:3.11-slim

# Plane's official MCP server (PyPI: plane-mcp-server).
# Pinned for reproducible builds; bump deliberately.
ARG PLANE_MCP_VERSION=0.2.8
RUN pip install --no-cache-dir "plane-mcp-server==${PLANE_MCP_VERSION}"

# Run as non-root.
RUN useradd --create-home --uid 10001 mcp
USER mcp

EXPOSE 8211

# Liveness: TCP-connect to the listener (any HTTP status, incl. 404, means it's up).
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import socket; socket.create_connection(('127.0.0.1',8211),3).close()" || exit 1

# HTTP transport. Listens on 0.0.0.0:8211 (published to loopback by compose).
# Endpoints:
#   /http/api-key/mcp  -> header auth (Authorization: Bearer <PAT> + X-Workspace-slug)
#   /http/mcp          -> OAuth (needs real Plane OAuth app; unused here)
CMD ["plane-mcp-server", "http"]
