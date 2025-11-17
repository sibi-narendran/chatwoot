# Base stage with common dependencies
FROM node:23-alpine as node
FROM ruby:3.4.4-alpine3.21 AS base

RUN apk update && apk add --no-cache \
  build-base \
  libffi-dev \
  git \
  imagemagick \
  libpq \
  openssl \
  postgresql-client \
  postgresql-dev \
  tzdata \
  vips

# Configure bundler
ENV BUNDLE_PATH="/gems" \
    BUNDLE_WITHOUT="development:test" \
    BUNDLER_VERSION="2.5.11"
RUN gem install bundler

# Configure Node.js and pnpm
ARG NODE_VERSION="23.7.0"
ARG PNPM_VERSION="10.2.0"
ENV NODE_VERSION=${NODE_VERSION} \
    PNPM_VERSION=${PNPM_VERSION} \
    NODE_OPTIONS="--max-old-space-size=4096 --openssl-legacy-provider"

COPY --from=node /usr/local/bin/node /usr/local/bin/
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
  && ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx
RUN npm install -g pnpm@${PNPM_VERSION}

WORKDIR /app

# Stage for installing gems
FROM base AS bundle_install
COPY Gemfile Gemfile.lock ./
RUN bundle install -j4

# Stage for precompiling assets
FROM base AS assets
ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
COPY --from=bundle_install /gems /gems
COPY Gemfile Gemfile.lock package.json pnpm-lock.yaml ./
RUN pnpm install
COPY . .
RUN mkdir log
RUN RAILS_ENV=production RACK_ENV=production SECRET_KEY_BASE=precompile_placeholder DATABASE_URL="postgresql://dummy:dummy@dummy/dummy" RAILS_LOG_TO_STDOUT=enabled bundle exec rake assets:precompile

# Final application image
FROM base AS final
ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV} \
    RAILS_SERVE_STATIC_FILES=true \
    RAILS_LOG_TO_STDOUT=true

COPY --from=bundle_install /gems /gems
COPY --from=assets /app/public /app/public
COPY . .

# Generate .git_sha file with current commit hash
RUN git rev-parse HEAD > /app/.git_sha

EXPOSE 3000

ENTRYPOINT ["./docker/entrypoints/rails.sh"]
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
