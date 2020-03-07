# certbot-ocsp-fetcher
`certbot-ocsp-fetcher` helps you setup OCSP stapling in nginx. It fetches and
verifies OCSP responses for TLS certificates issued with [Certbot], to be used
by nginx. This primes the OCSP cache of nginx, which is needed because of
nginx's flawed implementation (see bug [#812]).

In order for all this to be useful, you should know how to correctly set up
OCSP stapling in nginx, for which you can take a look at
[Mozilla's SSL Configuration Generator] for instance. If you use Certbot's
`nginx` plugin, you can also add the `--staple-ocsp` flag to your
`certbot --nginx` command(s) to configure OCSP stapling.

The script works by utilizing the OCSP Responder URL embedded in a certificate,
and saving the OCSP responses in staple files that can be referenced in the
nginx configurations of the websites that use the certificates. The script can
behave in two ways:

  - When this script is called by Certbot as a deploy/renew hook, only the OCSP
    response for the specific certificate that is issued, is fetched.

  - When Certbot's variables are not passed, the script cycles through all sites
    that have a certificate lineage in Certbot's folder, and fetches an OCSP
    response.

The use of this script makes sure OCSP stapling in nginx works reliably, which
makes e.g. the adoption of [OCSP Must-Staple] possible.

## Dependencies
- Bash 4.3+
- BSD column (included in stock Ubuntu within the `util-linux` package)
- Certbot 0.5.0+
- nginx (tested with 1.14.0)
- OpenSSL (1.1.0+)

## Usage
The script should be run with privileges that allow it to access the directory
that Certbot stores its certificates in (by default `/etc/letsencrypt/live`).
You should run it daily, for instance by using the included systemd service +
timer, or by adding it to the user's crontab. It can be run as follows:

`# ./certbot-ocsp-fetcher.sh [-c/--certbot-dir DIRECTORY] [-f/--force-fetch]
[-h/--help] [-n/--cert-name CERTNAME] [-o/--output-dir DIRECTORY]
[-v/--verbose] [-w/--no-reload-webserver]`

The filename of the OCSP staple is the name of the certificate lineage (as used
by Certbot) with the DER extension. Be sure to point nginx to the staple(s) by
using the `ssl_stapling_file` directive in the nginx configuration of the
website, so e.g. `ssl_stapling_file /etc/nginx/ocsp-cache/example.com.der;`.

When you want to use this tool as a deploy hook (available in Certbot >=0.17.0),
append `--deploy-hook "/path/to/certbot-ocsp-fetcher.sh"` to the Certbot command
you would normally use when requesting a certificate.

When you can't use Certbot >=0.17.0, use the `--renew-hook` flag in your
Certbot command instead. The difference between `--deploy-hook` and
`--renew-hook` is that a renew hook is not invoked during the first issuance in
a certificate lineage, but only during its renewals. Be aware that in Certbot
<0.10.0, hooks were [not saved] in the renewal configuration of a certificate.

**Note:** If there is an OCSP staple with the target name already existing in
the output directory which doesn't expire within two days, a new OCSP response
will **not** be fetched. Use the `-f/--force-fetch` flag to override this
behavior (see below).

### CLI parameters
- `-c, --certbot-dir`\
  Specify the configuration directory of the Certbot instance, that is used to
  process the certificates. When not passed, this defaults to
  `/etc/letsencrypt`.\
  This flag cannot be used when the script is invoked as a deploy hook by
  Certbot. In that case, the path to Certbot and the certificate is inferred from
  the call that Certbot makes.

- `-f, --force-fetch`\
  Ignore possibly existing valid OCSP responses on disk, and always fetch new
  responses from the OCSP responder.\
  This flag cannot be used when the script is invoked as a deploy hook by
  Certbot.

- `-h, --help`\
  Print the correct usage of the script.

- `-n, --cert-name`\
  Specify the name of the certificate lineage (as used by Certbot) that you
  want to fetch an OCSP response for. When not specified, all certificate
  lineages in Certbot's configuration directory will be processed.\
  This flag cannot be used when the script is invoked as a deploy hook by
  Certbot.

- `-o, --output-dir`\
  Specify the directory where OCSP staple files are saved. When not passed, this
  defaults to the working directory.

- `-v, --verbose`\
  Makes the tool verbose. This prints informational messages about operations
  performed on certificate lineages. This can be specified multiple times for
  more verbosity.

- `-w/--no-reload-webserver`\
  By default, this script tries to reload a service named `nginx` if at least
  one OCSP response was fetched. This flag disables this behavior.

 [Certbot]: https://github.com/certbot/certbot
 [#812]: https://trac.nginx.org/nginx/ticket/812
 [Mozilla's SSL Configuration Generator]: https://mozilla.github.io/server-side-tls/ssl-config-generator/
 [OCSP Must-Staple]: https://scotthelme.co.uk/ocsp-must-staple/
 [ocsp_host]: https://github.com/tomwassenberg/certbot-ocsp-fetcher/blob/e080b9838c1ee2f1cf05c6e9f366c19f986dc128/certbot-ocsp-fetcher.sh#L183
 [openssl-syntax-issue]: https://github.com/tomwassenberg/certbot-ocsp-fetcher/issues/16
 [not saved]: https://github.com/certbot/certbot/issues/3394
