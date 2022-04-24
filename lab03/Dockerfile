FROM alpine

RUN apk add python3

WORKDIR /files
RUN echo "Arquivo de teste" > teste.txt

CMD ["python3", "-m", "http.server"]