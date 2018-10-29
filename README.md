# Identification client par certificat avec Nginx

## Création de l'arborescence de fichiers nécessaire à la config

```$bash
mkdir -p ca/certs/users && \
mkdir ca/crl && \
mkdir ca/private

touch ca/index.txt && \
touch ca/index.txt.attr && \
echo 01 > ca/crlnumber
```

## Génération du certificat pour le chiffrement HTTPS du serveur

```$bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout ca/nginx-selfsigned.key \
    -out ca/nginx-selfsigned.crt
```

## Génération du certificat serveur pour signer les certificats des clients

```$bash
openssl genrsa -des3 -out ca/private/ca.key 4096
```
```$bash
openssl req -new -x509 -days 1095 \
    -key ca/private/ca.key \
    -out ca/certs/ca.crt
```
```$bash
openssl ca -name CA_default -gencrl \
    -keyfile ca/private/ca.key \
    -cert ca/certs/ca.crt \
    -out ca/private/ca.crl \
    -config openssl.cnf \
    -crldays 1095
```

## Génération de certificats pour les utilisateurs

```$bash
openssl genrsa -des3 -out ca/certs/users/<username>.key 1024
```
```$bash
openssl req -new -key ca/certs/users/<username>.key \
    -out ca/certs/users/<username>.csr
```   
```$bash
openssl x509 -req -days 1095 \
    -in ca/certs/users/<username>.csr \
    -CA ca/certs/ca.crt \
    -CAkey ca/private/ca.key \
    -CAserial ca/serial \
    -CAcreateserial \
    -out ca/certs/users/<username>.crt
```
```$bash
openssl pkcs12 -export -clcerts \
    -in ca/certs/users/<username>.crt \
    -inkey ca/certs/users/<username>.key \
    -out ca/certs/users/<username>.p12
```

## Accéder au serveur

### Sans certificat client

```$bash
curl --insecure https://localhost
```

Le serveur refuse la connexion :
```$html
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.15.5</center>
</body>
</html>
```

### Avec certificat client

```$bash
curl --insecure --cert ca/certs/users/<username>.p12 \
    --cert-type p12 https://localhost
```

Le serveur accepte la connexion :
```$html
<html>
    <head>
        <title>My super title</title>
    </head>
    <body>
        <h1>Example page</h1>
    </body>
</html>
```

## Révocation de certification clients

```$bash
openssl ca -config openssl.cnf \
    -revoke ca/certs/users/<username>.crt \
    -keyfile ca/private/ca.key \
    -cert ca/certs/ca.crt
``` 
```$bash
openssl ca -config openssl.cnf -gencrl \
    -keyfile ca/private/ca.key \
    -cert ca/certs/ca.crt \
    -out ca/private/ca.crl \
    -crldays 1095
```

## Notes

⚠ La prise en compte de la révocation des certificats
nécessite un **reload** du serveur Nginx.

⚠ Les connexions existantes ne sont pas interrompues, les clients
connectés et dont le certificat a été révoqué ont donc encore
accès au service jusqu'à ce qu'ils ferment leur connexion ou que
le serveur redémarre.
