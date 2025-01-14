FROM --platform=$BUILDPLATFORM node:20.2.0-bullseye-slim@sha256:dc1906714d1993d291e1e7b5f236291236b0a0b6dfacdb164e4a9ea44d09c52e AS builder

# Build arguments to change source url, branch or tag
ARG CODIMD_REPOSITORY
ARG HEDGEDOC_REPOSITORY=https://github.com/hedgedoc/hedgedoc.git
ARG VERSION=master
#necessary on ARM because puppeteer doesn't provide a prebuilt binary
ENV PUPPETEER_SKIP_DOWNLOAD=true
ENV YARN_CACHE_FOLDER=/tmp/.yarn

RUN if [ -n "${CODIMD_REPOSITORY}" ]; then echo "CODIMD_REPOSITORY is deprecated. Please use HEDGEDOC_REPOSITORY instead" && exit 1; fi

# Clone the source and remove git repository but keep the HEAD file
RUN --mount=target=/var/lib/apt/lists,type=cache,sharing=locked \
    --mount=target=/var/cache/apt,type=cache,sharing=locked \
    apt-get update && \
    apt-get install --no-install-recommends -y git jq ca-certificates python-is-python3 build-essential
RUN git clone --depth 1 --branch "$VERSION" "$HEDGEDOC_REPOSITORY" /hedgedoc
RUN git -C /hedgedoc log --pretty=format:'%ad %h %d' --abbrev-commit --date=short -1
RUN git -C /hedgedoc rev-parse HEAD > /tmp/gitref
RUN rm -rf /hedgedoc/.git/*
RUN mv /tmp/gitref /hedgedoc/.git/HEAD
RUN jq ".repository.url = \"${HEDGEDOC_REPOSITORY}\"" /hedgedoc/package.json > /hedgedoc/package.new.json
RUN mv /hedgedoc/package.new.json /hedgedoc/package.json

# Install app dependencies and build
WORKDIR /hedgedoc

RUN --mount=type=cache,sharing=locked,target=/tmp/.yarn yarn install --immutable
RUN yarn run build

FROM node:20.2.0-bullseye-slim@sha256:dc1906714d1993d291e1e7b5f236291236b0a0b6dfacdb164e4a9ea44d09c52e AS modules-installer
WORKDIR /hedgedoc

ENV NODE_ENV=production
ENV YARN_CACHE_FOLDER=/tmp/.yarn

COPY --from=builder /hedgedoc /hedgedoc

RUN --mount=target=/var/lib/apt/lists,type=cache,sharing=locked \
    --mount=target=/var/cache/apt,type=cache,sharing=locked \
    apt-get update && \
    apt-get install --no-install-recommends -y git ca-certificates python-is-python3 build-essential

RUN --mount=type=cache,sharing=locked,target=/tmp/.yarn yarn workspaces focus --production

FROM node:20.2.0-bullseye-slim@sha256:dc1906714d1993d291e1e7b5f236291236b0a0b6dfacdb164e4a9ea44d09c52e AS app
WORKDIR /hedgedoc

ARG UID=10000
ENV NODE_ENV=production
ENV UPLOADS_MODE=0700

RUN apt-get update && \
    apt-get install --no-install-recommends -y gosu && \
    rm -r /var/lib/apt/lists/*

# Create hedgedoc user
RUN adduser --uid $UID --home /hedgedoc/ --disabled-password --system hedgedoc

COPY --chown=$UID --from=modules-installer /hedgedoc /hedgedoc

# Add configuraton files
COPY ["resources/config.json", "/files/"]

# Healthcheck
COPY --chown=$UID /resources/healthcheck.mjs /hedgedoc/healthcheck.mjs
HEALTHCHECK --interval=15s CMD node healthcheck.mjs

# For backwards compatibility
RUN ln -s /hedgedoc /codimd

# Symlink configuration files
RUN rm -f /hedgedoc/config.json
RUN ln -s /files/config.json /hedgedoc/config.json

EXPOSE 3000

COPY ["resources/docker-entrypoint.sh", "/usr/local/bin/docker-entrypoint.sh"]
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["node", "app.js"]
