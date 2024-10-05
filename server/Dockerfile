# Use a specific version of Bun for consistency
FROM oven/bun:1 AS base

# Stage for dependencies
FROM base AS deps
WORKDIR /app

# Copy only package files to leverage Docker cache
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

# Stage for building the application
FROM base AS builder
WORKDIR /app

# Copy node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy the rest of the application code
COPY . .

# Build the application
RUN bun run build

# Final stage for running the application
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

# Install only production dependencies
RUN bun install --frozen-lockfile --production

# Create a user and group for better security
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 shakil

# Prepare the dist directory
RUN mkdir dist && chown shakil:nodejs dist

# Copy built files with the correct ownership
COPY --from=builder --chown=shakil:nodejs /app/dist ./dist

USER shakil

# Expose the port for the application
EXPOSE 5500

ENV PORT=5500

# Health check to ensure the application is running
HEALTHCHECK CMD curl --fail http://localhost:5500/health || exit 1

# Command to run your application
CMD ["bun", "dist/server.js"]
