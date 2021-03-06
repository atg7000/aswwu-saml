ASWWU SAML
----------
The ASWWU SAML container used to authenticate students with the university login.

Endpoints
=========
The SAML container has a few endpoints that handle logging in and out along with viewing the SAML data. There are also query parameters to modify the SAML function.

URLS
++++
- ``/`` - Base SAML page which supports SAML operations
- ``/attrs/`` - Page to view the current user's SAML attributes, useful for debugging or clearing cookies on the SAML site
- ``/metadata/`` - Page that serves our SP metadata, this is used by ADFS

On the base `/` endpoint, multiple query parameters can be used to adjust what type of SAML operation is performed. They are as follows:

Query Parameters
++++++++++++++++
- ``?sso`` - Used to initiate the sign on procedure, can be linked to on the ASWWU sites
- ``?slo`` - Used to initiate the log out procedure, can be linked to on the ASWWU sites
- ``?redirect=URI`` - Used to redirect the user back to a page after login or logout, if not included the user will be redirected to the homepage
- ``?sls`` - Used by ADFS to handle logging out
- ``?acs`` - Used by ADFS to communicate with the container

The first three query parameters can be used by ASWWU sites to handle the login and logout functionality.

Examples
++++++++
Some examples of possible queries to the container would be:

- ``https://saml.aswwu.com/?sso&redirect=/mask`` - This query will start the login process for the user and redirect them to the mask after it completes
- ``https://saml.aswwu.com/?slo`` - This query will start the logout process for the user and redirect them to the default homepage location after it completes

Setup
=====
Setup steps for deploying in production.

NGINX
+++++
NGINX must be configured to serve the SAML site over SSL on ``saml.aswwu.com``. Configure a new site and serve it over https to `saml.aswwu.com`. Also create a new location as follows:

::

  location / {
    proxy_pass              http://127.0.0.1:8000/;
    proxy_set_header        Host               $host;
    proxy_set_header        X-Real-IP          $remote_addr;
    proxy_set_header        X-Forwarded-For    $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Host   $host:443;
    proxy_set_header        X-Forwarded-Server $host;
    proxy_set_header        X-Forwarded-Port   443;
    proxy_set_header        X-Forwarded-Proto  https;
  }

Look at the NGINX repository for more configuration details.

Certificates
++++++++++++
Onelogin Python Toolkit expects that certificates for the SP be stored in the ``saml/certs/`` directory:

- ``sp.key`` - Private Key
- ``sp.crt`` - Public cert
- ``sp_new.crt`` - Future Public cert (optional)

Also you can use other cert to sign the metadata of the SP using the:

- ``metadata.key``
- ``metadata.crt``

You will also need to add the signing and encryption X509 certificates in ``saml/settings.json``. There is a ``settings.json.sample`` file that you can copy and fill in the settings. Also another file will need to be added, ``saml/advanced_settings.json`` there is also ain ``advanced_settings.json.sample`` in the same place that can likely be copied without modification unless settings need to change for ADFS.

Build and Start
+++++++++++++++
Before you can build, you must copy the ``.env.sample file`` to ``.env`` and add in the appropriate details. Each environment variable is described below:

- ``DJANGO_ENV`` - Should be either ``prod`` or ``dev``
- ``DJANGO_SECRET_KEY`` - Should be a randomly generated Django secret key
- ``DJANGO_TAG`` - The Docker tag for the image that is built by Docker Compose
- ``DJANGO_PORT`` - The port that django should start on internally, not through the reverse proxy
- ``SAML_CERTS_DIR`` - The directory where the SP certificates should be stored
- ``SAML_KEY`` - The key that the API server expects to authenticate SAML users and retrieve their cookie
- ``SAML_URL`` - The domain where the SAML container is running, this will likely break the site if not set to ``saml.aswwu.com``
- ``SITE_URL`` - The domain where the SAML container should redirect to, should be ``aswwu.com`` or ``www.aswwu.com``

Once you have setup you ``.env`` file, you can build and run the Docker container:

::

  $ docker-compose up -d --build

Docker Compose can be install with apt if necessary.

Debugging
+++++++++
In the case of errors, there is a settings in ``saml/settings.json`` called ``debug`` that can be set to true. Once this is set, you can restart the server with:

::

  $ docker-compose restart

Restarting the server will apply all new settings and certificates in the ``saml`` directory. You should now see debug messages if there are login issues.

Certificate Maintenance
+++++++++++++++++++++++
ADFS will eventually roll over their certificates, they will need to be updated in the ``saml/settings.json`` file. The certificates can be found in the `Federation Metadata <https://adfs.wallawalla.edu/FederationMetadata/2007-06/FederationMetadata.xml>`_ provided by the university. There is a section in the file call ``IDPSSODescriptor``, within this section there will be three ``KeyDescriptor`` sections containing the encryption certificate first and the two signing certificates next. These long lines are the certificates that can be added in the ``saml/settings.json`` file.

Use the command in the Debugging section to restart the server and use the new certificates.

