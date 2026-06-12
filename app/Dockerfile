# Build stage: compile a static, CGO-free binary.
FROM golang:1.25-alpine AS build
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /app .

# Runtime stage: distroless nonroot. No shell, no package manager, uid 65532.
# Harden further in production by pinning this base by @sha256 digest.
FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /
COPY --from=build /app /app
EXPOSE 8080
USER 65532:65532
ENTRYPOINT ["/app"]
