# AWX Operator Verify Cert

This repo contains an example configuration of an AWX deployment for connecting to a postgres instance that uses client certificate authentication.

You will need the client certificate (public key + private key) and the Certificate Authority of your postgresql instance.

## Certificate generation

If you want to generate your own CA/Server/Client certs to set up your DB instance, you can use these commands with openssl :

CA cert generation : 
- `openssl req  -nodes -new -x509  -keyout ca.key -out ca.cert`

Server cert generation :
- `openssl genrsa -out server.key 2048`
- `openssl req -new -key server.key -out server.csr`

    Don't forget to put the resolvable hostname or IP in the Common Name field or the client will refuse the connection in `verify=full` mode.

- `openssl req -in server.csr -CA ca.cert -CAkey ca.key -out server.cert`

Client cert generation :
- `openssl genrsa -out client.key 2048`
- `openssl req -new -key client.key -out client.csr`

    Don't forget to put the username in the Common Name field or the server will refuse to authenticate the user in `clientcert=verify-full` mode.

- `openssl req -in client.csr -CA ca.cert -CAkey ca.key -out client.cert`

> :zap: These commands are not safe to use in production. For example the certificate constraints are not set correctly.

## Postgresql configuration

To configure SSL on your instance, set the paths to the root ca cert and server cert and key in the `potgresql.config` file :

    ...
    # - SSL -

    ssl = on
    ssl_ca_file = '/etc/ssl/certs/root.crt'
    ssl_cert_file = '/etc/ssl/certs/server.crt'
    #ssl_crl_file = ''
    ssl_key_file = '/etc/ssl/private/server.key'
    ...

> :exclamation: The `ssl_key_file` needs to be in `0600` mode for postgres to use it.

> :memo: I have chosen the /etc/ssl path arbitrary, I think that you could chose whatever you want.

Finally, set the authentication to `clientcert=verify-full` mode in the `pg_hba.conf` file :

    ...
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    hostssl all             all             0.0.0.0/0               md5    clientcert=verify-full
    ...

Restart your postgresql instance and you should be able to log with your client cert and key to the instance : `psql "port=5432 host=192.168.50.84 user=postgres sslcert=./client/client.cert sslkey=./client/client.key sslrootcert=./ca/ca.cert sslmode=verify-full"`

Once logged in you can check if your connection to the database is encrypted with the following query : `SELECT * FROM pg_stat_ssl;`

## AWX set up

Once you have your client certificat + provate key and the Certificate Authority cert, you can configure AWX to use them to authenticate to your DB instance.
