.PHONY: run test lint build docker-build

run:
	go run ./cmd/api

test:
	go test ./...

lint:
	gofmt -l . && go vet ./...

build:
	CGO_ENABLED=0 go build -ldflags="-s -w" -o bin/api ./cmd/api

docker-build:
	docker build -t tineng-api:latest .
