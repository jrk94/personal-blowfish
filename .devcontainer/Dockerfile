# Use Hugo extended version (necessary for some themes like Blowfish)
FROM klakegg/hugo:ext-alpine

# Set working directory
WORKDIR /app

# Copy site contents (or bind-mount with devcontainer)
COPY . .

# Default command (optional for container runs outside devcontainer)
CMD ["hugo", "server", "--bind=0.0.0.0", "--baseURL=http://localhost", "--appendPort=false", "--disableFastRender"]
