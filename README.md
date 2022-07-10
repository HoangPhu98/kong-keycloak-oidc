# Kong + Keycloak OIDC 

```/bin/bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm upgrade -i kong-db bitnami/postgresql -f kong-db/override_values.yaml

kubectl apply -f kong/

curl -s -X POST http://localhost:30081/services \
  -d name=mock-service \
  -d url=http://mockbin.org/request \
  | python3 -mjson.tool

# Note id value in response

curl -s -X POST http://localhost:30081/services/7fe48815-afd6-4b46-868d-c76a74fad31a/routes -d "paths[]=/mock" \
    | python3 -mjson.tool

curl -s http://localhost:30080/mock

curl -s http://localhost:30081 | jq .plugins.available_on_server.oidc

kubectl apply -f konga/

helm repo add bitnami https://charts.bitnami.com/bitnami

helm upgrade -i kong-keycloak bitnami/keycloak -f keycloak/override_values.yaml

HOST_IP=`ip address show dev eth0 | grep "inet " \
| grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' \
| head -1`
CLIENT_SECRET="0vTixzgbdtRfLU125DPgyHyc75aT40NL"
REALM="experimental"

curl -s -X POST http://localhost:30081/plugins \
  -d name=oidc \
  -d config.client_id=kong \
  -d config.client_secret=${CLIENT_SECRET} \
  -d config.bearer_only=yes \
  -d config.realm=${REALM} \
  -d config.introspection_endpoint=http://${HOST_IP}:30180/realms/${REALM}/protocol/openid-connect/token/introspect \
  -d config.discovery=http://${HOST_IP}:30180/auth/realms/${REALM}/.well-known/openid-configuration \
  | python3 -mjson.tool


curl "http://${HOST_IP}:30080/mock" \
-H "Accept: application/json" -I
```

```text
Connection: keep-alive
WWW-Authenticate: Bearer realm="kong",error="no Authorization header found"
Server: kong/1.3.0
HTTP/1.1 401 Unauthorized
Date: Sun, 10 Jul 2022 20:19:31 GMT
Connection: keep-alive
WWW-Authenticate: Bearer realm="experimental",error="no Authorization header found"
X-Kong-Response-Latency: 2023
Server: kong/2.8.1
```

```/bin/bash
RAWTKN=$(curl -s -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "username=demouser" \
        -d "password=demouser" \
        -d 'grant_type=password' \
        -d "client_id=myapp" \
        http://${HOST_IP}:30180/realms/experimental/protocol/openid-connect/token \
        |jq . )

echo $RAWTKN

export TKN=$(echo $RAWTKN | jq -r '.access_token')
echo $TKN


curl "http://${HOST_IP}:30080/mock" \
-H "Accept: application/json" \
-H "Authorization: Bearer $TKN"
```

```text
{
  "startedDateTime": "2022-07-10T20:25:30.133Z",
  "clientIPAddress": "192.168.65.3",
  "method": "GET",
  "url": "http://172.24.165.70/request",
  "httpVersion": "HTTP/1.1",
  "cookies": {},
  "headers": {
    "host": "mockbin.org",
    "connection": "close",
    "accept-encoding": "gzip",
    "x-forwarded-for": "192.168.65.3,42.116.167.5, 172.70.142.7",
    "cf-ray": "728c07690c6a46eb-SIN",
    "x-forwarded-proto": "http",
    "cf-visitor": "{\"scheme\":\"http\"}",
    "x-forwarded-host": "172.24.165.70",
    "x-forwarded-port": "80",
    "x-forwarded-path": "/mock",
    "x-forwarded-prefix": "/mock",
    "user-agent": "curl/7.68.0",
    "accept": "application/json",
    "authorization": "Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJJbHFsaDRTU1lqOVZrcnE1NzVoX3h3UEZCcWtrdWQ1T2JVT1VtUGh1cnowIn0.eyJleHAiOjE2NTc0ODQ4ODAsImlhdCI6MTY1NzQ4NDU4MCwianRpIjoiZTU5N2NjNzktZDBlZS00Y2ViLWFkOGEtYWYxODQzY2JjMWIxIiwiaXNzIjoiaHR0cDovLzE3Mi4yNC4xNjUuNzA6MzAxODAvcmVhbG1zL2V4cGVyaW1lbnRhbCIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiI2ZjZlYTdkOC02Nzg3LTRiMmYtOWI5Mi0yNjNjOWVmYzhhZDIiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJteWFwcCIsInNlc3Npb25fc3RhdGUiOiJhMDg2N2FjMC1hOWM2LTQ5MmMtYmQxYi1kMTlkN2U2MDI2ZTEiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtZXhwZXJpbWVudGFsIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoicHJvZmlsZSBlbWFpbCIsInNpZCI6ImEwODY3YWMwLWE5YzYtNDkyYy1iZDFiLWQxOWQ3ZTYwMjZlMSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFtZSI6IkRlbW8gVXNlciIsInByZWZlcnJlZF91c2VybmFtZSI6ImRlbW91c2VyIiwiZ2l2ZW5fbmFtZSI6IkRlbW8iLCJmYW1pbHlfbmFtZSI6IlVzZXIiLCJlbWFpbCI6ImRlbW9AdGVzdC5jb20ifQ.WGVk43oz5wWjTBQhxnxiFjurcnJkrQYp-U-ZVPnDFv2o0ustHOcyDtY2tC7mxR-cIesdfZGzD1Pt8iNKTweLhpFOnoeb3ll-zCHwZR9nc7wMUy4xTTEqGmjL-8Md3W7KKT943YYCmHtdILVJMIzW03v0ypoNhqNrXlkPbdhr2Qf-lzpNSsLoIQHHNMWFh2iqoAFGuI56W51zv4uzen9kccf7jLzXgNl0iWiIrcsmw2S6wd2NQSJbCbQ78QaE0X7YeHH0FIRz3sbIS8mEysYf-HvFzakFm7JTBitMA_9tqTZzAl0YE_HPmY7AkBx3_O4exAW96XtRWzzg4ybaQU1MlQ",
    "x-userinfo": "eyJleHAiOjE2NTc0ODQ4ODAsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwiYWNyIjoiMSIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJkZWZhdWx0LXJvbGVzLWV4cGVyaW1lbnRhbCIsIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInByZWZlcnJlZF91c2VybmFtZSI6ImRlbW91c2VyIiwiY2xpZW50X2lkIjoibXlhcHAiLCJhdWQiOiJhY2NvdW50IiwibmFtZSI6IkRlbW8gVXNlciIsInVzZXJuYW1lIjoiZGVtb3VzZXIiLCJzaWQiOiJhMDg2N2FjMC1hOWM2LTQ5MmMtYmQxYi1kMTlkN2U2MDI2ZTEiLCJzdWIiOiI2ZjZlYTdkOC02Nzg3LTRiMmYtOWI5Mi0yNjNjOWVmYzhhZDIiLCJzY29wZSI6InByb2ZpbGUgZW1haWwiLCJqdGkiOiJlNTk3Y2M3OS1kMGVlLTRjZWItYWQ4YS1hZjE4NDNjYmMxYjEiLCJlbWFpbCI6ImRlbW9AdGVzdC5jb20iLCJpZCI6IjZmNmVhN2Q4LTY3ODctNGIyZi05YjkyLTI2M2M5ZWZjOGFkMiIsImFjdGl2ZSI6dHJ1ZSwiaXNzIjoiaHR0cDovLzE3Mi4yNC4xNjUuNzA6MzAxODAvcmVhbG1zL2V4cGVyaW1lbnRhbCIsInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzZXNzaW9uX3N0YXRlIjoiYTA4NjdhYzAtYTljNi00OTJjLWJkMWItZDE5ZDdlNjAyNmUxIiwiaWF0IjoxNjU3NDg0NTgwLCJ0eXAiOiJCZWFyZXIiLCJnaXZlbl9uYW1lIjoiRGVtbyIsImF6cCI6Im15YXBwIiwiZmFtaWx5X25hbWUiOiJVc2VyIn0=",
    "cf-connecting-ip": "42.116.167.5",
    "cdn-loop": "cloudflare",
    "x-request-id": "8b5626aa-440a-4b38-b0d9-3557f1cf086d",
    "via": "1.1 vegur",
    "connect-time": "1",
    "x-request-start": "1657484730131",
    "total-route-time": "0"
  },
  "queryString": {},
  "postData": {
    "mimeType": "application/octet-stream",
    "text": "",
    "params": []
  },
  "headersSize": 3059,
  "bodySize": 0
}
```