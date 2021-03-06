
	mod_http_upload - HTTP File Upload (XEP-0363)

	Author: Holger Weiss <holger@zedat.fu-berlin.de>
	Requirements: ejabberd 15.06, 15.07, or 15.09
	Note: This module comes bundled with ejabberd 15.10 and newer


	DESCRIPTION
	-----------

This module allows for requesting permissions to upload a file via HTTP.  If
the request is accepted, the client receives a URL to use for uploading the
file and another URL from which that file can later be downloaded.

Automatic quota management can be configured by also enabling
mod_http_upload_quota.


	CONFIGURATION
	-------------

In order to use this module, add configuration snippets such as the
following:

  listen:
    # [...]
    -
      module: ejabberd_http
      port: 5443
      tls: true
      certfile: "/etc/ejabberd/example.com.pem"
      request_handlers:
        "": mod_http_upload

  access:
    # [...]
    soft_upload_quota:
      all: 1000 # MiB
    hard_upload_quota:
      all: 1100 # MiB

  modules:
    # [...]
    mod_http_upload:
      docroot: "/home/xmpp/upload"
      put_url: "https://@HOST@:5443"
    mod_http_upload_quota:
      max_days: 100

The configurable mod_http_upload options are:

- host (default: "upload.@HOST@")

  This option defines the JID for the HTTP upload service.  The keyword
  @HOST@ is replaced with the virtual host name.

- name (default: "HTTP File Upload")

  This option defines the Service Discovery name for the HTTP upload
  service.

- access (default: 'local')

  This option defines the access rule to limit who is permitted to use the
  HTTP upload service.  The default value is 'local'.  If no access rule of
  that name exists, no user will be allowed to use the service.

- max_size (default: 104857600)

  This option limits the acceptable file size.  Either a number of bytes
  (larger than zero) or 'infinity' must be specified.

- secret_length (default: 40)

  This option defines the length of the random string included in the GET
  and PUT URLs generated by mod_http_upload.  The minimum length is 8
  characters, but it is recommended to choose a larger value.

- jid_in_url (default: 'sha1')

  When this option is set to 'node', the node identifier of the user's JID
  (i.e., the user name) is included in the GET and PUT URLs generated by
  mod_http_upload.  Otherwise, a SHA-1 hash of the user's bare JID is
  included instead.

- file_mode (default: 'undefined')

  This option defines the permission bits of uploaded files.  The bits are
  specified as an octal number (see the chmod(1) manual page) within double
  quotes.  For example: "0644".

- dir_mode (default: 'undefined')

  This option defines the permission bits of the 'docroot' directory and any
  directories created during file uploads.  The bits are specified as an
  octal number (see the chmod(1) manual page) within double quotes.  For
  example: "0755".

- docroot (default: "@HOME@/upload")

  Uploaded files are stored below the directory specified (as an absolute
  path) with this option.  The keyword @HOME@ is replaced with the home
  directory of the user running ejabberd.

- put_url (default: "http://@HOST@:5444")

  This option specifies the initial part of the PUT URLs used for file
  uploads.  The keyword @HOST@ is replaced with the virtual host name.

  NOTE: Different virtual hosts cannot use the same PUT URL domain.

- get_url (default: $put_url)

  This option specifies the initial part of the GET URLs used for
  downloading the files.  By default, it is set to the same value as the
  'put_url', but you can set it to a different value in order to have the
  files served by a proper HTTP server such as Nginx or Apache.  The keyword
  @HOST@ is replaced with the virtual host name.

- service_url (default: 'undefined')

  If a 'service_url' is specified, HTTP upload slot requests are forwarded
  to this external service instead of being handled by mod_http_upload
  itself.  Such a service is available as a Django app, for example:

  https://github.com/mathiasertl/django-xmpp-http-upload

  An HTTP GET query such as the following is issued whenever an
  HTTP upload slot request is accepted as per the 'access' rule:

  http://localhost:5444/?jid=juliet%40example.com&size=10240&name=example.jpg

  In order to accept the request, the service must return an HTTP status
  code of 200 or 201 and two lines of text/plain output.  The first line is
  forwarded to the XMPP client as the HTTP upload PUT URL, the second line
  as the GET URL.

  In order to reject the request, the service should return one of the
  following HTTP status codes:

  - 402
    In this case, a 'resource-constraint' error stanza is sent to the
    client.  Use this to indicate a temporary error after the client
    exceeded a quota, for example.

  - 403
    In this case, a 'not-allowed' error stanza is sent to the client.  Use
    this to indicate a permanent error to a client that is not permitted to
    upload files, for example.

  - 413
    In this case, a 'not-acceptable' error stanza is sent to the client.
    Use this if the file size was too large, for example.

  In any other case, a 'service-unavailable' error stanza is sent to the
  client.

- custom_headers (default: [])

  This option specifies additional header fields to be included in all HTTP
  responses.  For example:

    custom_headers:
      "Access-Control-Allow-Origin": "*"
      "Access-Control-Allow-Methods": "OPTIONS, HEAD, GET, PUT"

- rm_on_unregister (default: 'true')

  This option specifies whether files uploaded by a user should be removed
  when that user is unregistered.  It must be set to 'false' if this is not
  desired.

The configurable mod_http_upload_quota options are:

- max_days (default: 'infinity')

  If a number larger than zero is specified, any files (and directories)
  older than this number of days are removed from the subdirectories of the
  'docroot' directory, once per day.  By default, this won't be done.

- access_hard_quota (default: 'hard_upload_quota')

  This option defines which access rule is used to specify the "hard quota"
  for the matching JIDs.  That rule must yield a positive number for any JID
  that is supposed to have a quota limit.  This is the number of megabytes a
  corresponding user may upload.  When this threshold is exceeded, ejabberd
  deletes the oldest files uploaded by that user until their disk usage
  equals or falls below the specified soft quota (see below).

- access_soft_quota (default: 'soft_upload_quota')

  This option defines which access rule is used to specify the "soft quota"
  for the matching JIDs.  That rule must yield a positive number of
  megabytes for any JID that is supposed to have a quota limit.  It is
  recommended to make sure the soft quota will be smaller than the hard
  quota.  See the description of the 'access_hard_quota' option for details.

  NOTE: It's not necessary to specify the 'access_hard_quota' and
  'access_soft_quota' options in order to use the quota feature.  You can
  stick to the default names and just specify access rules such as the
  following:

  access:
    # [...]
    soft_upload_quota:
      all: 400
    hard_upload_quota:
      all: 500

  This sets a soft quota of 400 MiB and a hard quota of 500 MiB for all
  users.
