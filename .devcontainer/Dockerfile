FROM alpine:3.20

# Install dependencies for Hugo Extended
RUN apk add --no-cache \
    curl \
    ca-certificates \
    git \
    libstdc++ \
    libc6-compat

# Install Hugo Extended 0.152.2
ARG HUGO_VERSION=0.152.2
ARG HUGO_TAR="hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz"

RUN curl -L -o /tmp/hugo.tar.gz \
        "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/${HUGO_TAR}" \
    && tar -xzf /tmp/hugo.tar.gz -C /usr/local/bin hugo \
    && chmod +x /usr/local/bin/hugo \
    && rm /tmp/hugo.tar.gz

WORKDIR /app
