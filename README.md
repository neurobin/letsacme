This script automates the process of getting a signed TLS/SSL certificate from Let's Encrypt using the ACME protocol. It will need to be run on your server and have **access to your private account key**. It gets both the certificate and chain (CABUNDLE) and prints them on stdout unless specified otherwise.

**PLEASE READ THE SOURCE CODE (~350 LINE)! YOU MUST TRUST IT WITH YOUR PRIVATE KEYS!**

#Dependencies:
1. Python
2. openssl

#How to use:

If you just want to renew an existing certificate, you will only have to do Steps 4 and 6.

## 1: Create a Let's Encrypt account private key (if you haven't already):
You must have a public key registered with Let's Encrypt and use the corresponding private key to sign your (certificate) requests. Thus you first need to create a key, which **lcrypt** will use to register an account for you and sign all the following requests.

If you don't understand what the account is for, then this script likely isn't for you. Please, use the official Let's Encrypt client. Or you can read the [howitworks](https://letsencrypt.org/howitworks/technology/) page under the section: **Certificate Issuance and Revocation** to gain a little insight on how certificate issuance works.

The following command creates a 4096bit RSA (private) key:
```sh
openssl genrsa 4096 > account.key
```

**Or use an existing Let's Encrypt key (privkey.pem)**

**Note:** **lcrypt** is using the [PEM](https://tools.ietf.org/html/rfc1421) key format.

##2: Create a certificate signing request (CSR) for your domains.

The ACME protocol used by Let's Encrypt requires a csr to be submitted to it (for renewal too). Once you have the CSR, you can use it as many times as you want. You can create a CSR from the terminal which will require you to create a domain key first, or you can create it from the control  panel provided  by your hosting provider (for example: cpanel). Whatever method you may use to create the CSR, the script doesn't require the domain key, only the CSR.

**Note:** Domain key is not the private key you previously created.

###An example of creating CSR using openssl:
The following command will create a 4096bit RSA (domain) key:
```sh
openssl genrsa 4096 > domain.key
```
Now to create a CSR for a single domain:
```sh
openssl req -new -sha256 -key domain.key -subj "/CN=example.com" > domain.csr
```
For multi-domain:
```sh
openssl req -new -sha256 \
    -key domain.key \
    -subj "/C=US/ST=CA/O=MY Org/CN=example.com" \
    -reqexts SAN \
    -config <(cat /etc/ssl/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=DNS:example.com,DNS:www.example.com,DNS:subdomain.example.com,DNS:www.subdomain.com")) \
    -out domain.csr
```

##3: Create a config file for lcrypt script (optional):
lcrypt uses a JSON file to get the required information it needs to write challenge files on the server. This method is different than the [acme-tiny](https://github.com/diafygi/acme-tiny) script which this script is based on. Acme-tiny requires you to configure your server for completing the challenge, contrary to that I don't intend to do anything at all with the server. Instead of setting up your server, **lcrypt** requires you to provide the document root of each domain in a JSON format. It will create the *.well-known/acme-challenge* directory (if not exists already) and put the temporary challenge files there. An example config file looks like this:

**config.json:**
```json
{
"example.org": {
    "DocumentRoot":"/var/www/public_html"
    },
"subdomain1.example.org": {
    "DocumentRoot":"/var/www/subdomain1"
    },
"subdomain2.example.org": {
    "DocumentRoot":"/var/www/subdomain2"
    },
"subdomain3.example.org": {
    "DocumentRoot":"/var/www/subdomain3"
    },
"subdomain4.example.org": {
    "DocumentRoot":"/var/www/subdomain4"
    },
"subdomain5.example.org": {
    "DocumentRoot":"/var/www/subdomain5"
    }
}
```
##4: Get a signed certificate:
To get a signed certificate, all you need is the private key and the CSR. If you created the config.json file in previous step:
```sh
python lcrypt.py --no-chain --account-key ./account.key --csr ./domain.csr --config-json ./config.json > ./signed.crt
```
If you didn't create the config.json file, then pass the json as an argument:
```sh
python lcrypt.py --no-chain --account-key ./account.key --csr ./domain.csr --config-json '{"example.com":{"DocumentRoot":"/var/www/public_html"},"subdomain.example.com":{"DocumentRoot":"/var/www/subdomain"}}' > ./signed.crt
```
Notice the `--no-chain` option; if you omitted this option then you would get what Let's Encrypt calls a fullchain (cert+chain). Also, you can get the chain, cert and fullchain separately:

```sh
python lcrypt.py --account-key ./account.key --csr ./domain.csr --config-json ./config.json --cert-file ./signed.cert --chain-file ./chain.crt > ./fullchain.crt
```

#5: Install the certificate:
This is an scope beyond the script. You will have to install the certificate manually and setup the server as it requires. An example for nginx:

```nginx
server {
    listen 443;
    server_name example.com, www.example.com;

    ssl on;
    ssl_certificate /path/to/fullchain.crt;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}
```
An example for apache2:
```apache2
<VirtualHost *:443>     
        ...other configuration
        SSLEngine on
        SSLCertificateKeyFile /path/to/domain.key
        SSLCertificateFile /path/to/signed.crt
        SSLCertificateChainFile /path/to/chain.crt
</VirtualHost>
```

#6: Setup an auto-renew cron job:
Let's Encrypt certificate only lasts for 90 days. So you need to renew it in a timely manner. You can setup a cron job to do this for you. A monthly cron job:
```sh
0 0 1 * * python /path/to/lcrypt.py --account-key /path/to/account.key --csr /path/to/domain.csr --config-json /path/to/config.json --cert-file /path/to/signed.crt --chain-file /path/to/chain.crt  > /path/to/fullchain.crt 2>> /var/log/lcrypt.log && service apache2 restart
```

#Permissions:

1. **Challenge directory:** The script needs permissions to write files to the challenge directory which is in the document root of each domain. It simply means that the script requires permission to write to your document root. If that seems to be security issue then you can work it around by creating the challenge directories first. If the challenge directory already exists it will only need permission to write to the challenge directory not the document root.
2. **Account key:** Save the **account.key** file to a secure location. **lcrypt** only needs read permission to it, so you can revoke write permission from it and make it absolutely secure.
3. **Domain key:** Save the **domain.key** file to a secure location. **lcrypt** doesn't use this file. So no permission should be allowed for this file.
4. **Cert files:** Save the **signed.crt**, **chain.crt** and **fullchain.crt** in a secure location. **lcrypt** needs **write** permission for these files as it will update these files in a timely basis.
5. **Config json:** Save it in a secure location (It stores the path to the document root for each domain). **lcrypt** needs only read permission to this file.

As you will want to secure yourself as much as you can and thus give as less permission as possible to the script, I suggest you create an extra user for this script and give that user write permission to the challenge directory and the cert files, and read permission to the private key (account.key) and the config file (config.json) and nothing else.

#For advanced user:

Run it with `-h` flag to get help.
```sh
python lcrypt.py -h
```
It will show you all the available options that are supported by the script. These are the options currently supported:

```sh
optional arguments:
  -h, --help            show this help message and exit
  --account-key ACCOUNT_KEY
                        Path to your Let's Encrypt account private key.
  --csr CSR             Path to your certificate signing request.
  --config-json CONFIG_JSON
                        Configuration JSON file. Must contain
                        "DocumentRoot":"/path/to/document/root" entry for each domain.
  --cert-file CERT_FILE
                        File to write the certificate to. Overwrites if file
                        exists.
  --chain-file CHAIN_FILE
                        File to write the certificate to. Overwrites if file
                        exists.
  --quiet               Suppress output except for errors.
  --ca CA               Certificate authority, default is Let's Encrypt.
  --no-chain            Fetch chain (CABUNDLE) but do not print it on stdout.
  --no-cert             Fetch certificate but do not print it on stdout.
  --force               Apply force. If a directory is found inside the
                        challenge directory with the same name as challenge
                        token, this option will delete the directory and it's
                        content (Use with care).
  --test                Get test certificate (Invalid certificate). This
                        option won't have any effect if --ca is passed.
  --version             Show version info.
```