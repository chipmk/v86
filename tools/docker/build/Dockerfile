FROM alpine:3.14 as v86-builder
WORKDIR /work

RUN apk add --update curl clang make openjdk8 npm python3

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

ENV PATH="/root/.cargo/bin:${PATH}"

RUN rustup target add wasm32-unknown-unknown

COPY . .

CMD ["make", "all"]
