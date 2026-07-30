FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go mod tidy && CGO_ENABLED=0 GOOS=linux go build -o http-server-projeto-korp .

FROM alpine:3.18
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/http-server-projeto-korp /usr/local/bin/http-server-projeto-korp
EXPOSE 8080
ENTRYPOINT ["http-server-projeto-korp"]