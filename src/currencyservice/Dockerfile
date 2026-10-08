# install dependencies
FROM node:24-bookworm-slim@sha256:d6aa754f16b3197301076f047b5def2f02ea1dbbc2ca920407d46d7ec7f87b20 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts

# runtime
FROM gcr.io/distroless/nodejs24-debian12:nonroot@sha256:14d42e2511532589a7c7e01a753667a74fcc96266e137e8125006b87b0c32d0a
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY proto ./proto
COPY data ./data
COPY server.js ./
ENV NODE_ENV=production
ENV PORT=7000
ENV DISABLE_PROFILER=1
EXPOSE 7000
USER 65532
CMD ["server.js"]