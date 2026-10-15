# build
FROM golang:1.26.8-alpine3.24@sha256:8ac98ca534ac3f51e1f420a1dd2c15e74c75cfa0f23f3ad27eb5d7236c349a0c AS build
WORKDIR /shippingservice
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/shippingservice .

# runtime
FROM gcr.io/distroless/static-debian12:nonroot@sha256:afa5c872c891853ca7fcf1f12c3edb23f7eeef36189728842dd51042ff57f7ab
WORKDIR /app
COPY --from=build /out/shippingservice .
ENV PORT=50051
ENV DISABLE_PROFILER=1
ENV DISABLE_TRACING=1
ENV DISABLE_STATS=1
EXPOSE 50051
USER 65532
ENTRYPOINT ["/app/shippingservice"]
