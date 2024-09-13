# Stage 1: Build the Go application
FROM golang:1.18-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . ./
RUN go build -o main .

# Stage 2: Run the application
FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/main /app/main
EXPOSE 8000
CMD ["./main"]