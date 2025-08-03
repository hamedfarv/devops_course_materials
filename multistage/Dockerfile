FROM golang:1.14.0-alpine
WORKDIR /mysource
COPY hello.go .
RUN GOOS=linux go build -a -installsuffix cgo -o hello . 
CMD ["./hello"]