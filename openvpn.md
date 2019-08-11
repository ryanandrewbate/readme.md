# OpenVPN server setup

For use with site-to-site and site-to-remote-user setups over WAN.

## Mikrotik (server)

#### Create certificate for server
* Generate an RSA private key for the server:
`openssl genrsa -out myserver.com.key`
* Create a certificate signing request (CSR) for the server using this key, specifying values when prompted - "GB" country code for the UK.
`openssl req -new -key myserver.com.key -out myserver.com.csr`
* Create an EXT file `myserver.com.ext` file and add the following values:
        
        authorityKeyIdentifier=keyid,issuer
        basicConstraints=CA:FALSE
        keyUsage = digitalSignature, keyEncipherment
        extendedKeyUsage=serverAuth
        subjectAltName = @alt_names
    
        [alt_names]
        DNS.1 = myserver.com
        DNS.2 = www.myserver.com
        DNS.3 = **internaldnsname**
        IP.1 = **internalipaddress**
    *Note: The keyUsage and extendedKeyUsage values are necessary for OpenVPN clients to successfully verify the certificate as a "server" - changing these may cause server validation to fail.*

* Generate the certificate using the CSR and EXT file, signing it with the CA certificate and private key:
`openssl x509 -req -in myserver.com.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out myserver.com.crt -days 3650 -sha256 -extfile myserver.com.ext `

* Upload the certificate, private key and CA certificate to the Mikrotik in `Files`.

    * myserver.com.crt
    * myserver.com.key
    * myCA.pem / myCA.crt
    
* Under `System > Certificates`, click `Import` and add each of these files.

There should now be two rows displayed on the Certificates page, one for the CA with just `T` alongside it, and another for the server with `KT`.

#### Add certificate to WebFig interface (optional)

*   Under `IP > Services` choose `www-ssl`.
*   Select your certificate, `myserver.com.crt` from the Certificate list.
*   Disable and then re-enable `www-ssl` and ensure TCP port 443 is open for incoming traffic.

#### Create OpenVPN profile
This serves as a template for connections made through the OpenVPN server.

*   Under `PPP > Profiles` choose `Add new`.
*   Enter `openvpn` or similar as the profile name.
*   Enter a local address for the server (e.g. 10.50.0.1).
*   For remote address, choose an IP pool to use for remote client address allocation. (e.g. pool 10.50.0.10-10.50.0.254)
*   Select a Bridge to join client connections to (optional).
*   Add any local DNS servers as necessary (particularly of note for Active Directory domains).
* S et `Change TCP MSS` to `Yes`.
*   Leave `Use uPnP` as `Default`.
*   Leave `Use MPLS` as `Default`.
*   Set `Use Compression` to `No` (this depends on your hardware capabilities on either end - advised against for low powered routers on site-to-site links).
*   Set `Use Encryption` to `Required`.
*   Set `Only One` as `Default`.
    *The remaining options can be left blank.*
*   Save the profile by pressing `OK`.

##### Setup OpenVPN Server
*   Under `PPP > Interface` press the `OVPN Server` button.
*   Leave `Port` as `1194`.
*   Set `Mode` to `ip`.
*   Leave `Netmask` as `24`
*   Select the profile made in the previous section as `Profile`.
*   Select the server certificate from earlier as `Certificate`.
*   Ensure `Require Client Certificate` is enabled.
*   Set `Auth.` to only `SHA1`.
*   Set `Cipher` to only `AES 256`.
   *The remaining options can be left blank or at default.*
*   Check the `Enable` box and press `OK`.

##### Create a User
*   Under `PPP > Secrets` choose `Add new`.
*   Set `Name` to the name of the device or the hostname of the remote client you wish to connect.
*   Set `Password` to a 32-bit minimum random string password.
*   Set `Service` to `ovpn`.
*   Set `Profile` to the profile created previously.
*   Set any other values as desired for the client in question - these will override those of the profile.
   *The remaining options can be left blank or at default.*
*   Check the `Enable` box and press `OK`.

# Client

##### Create client certificate
* Generate an RSA private key for the client:
`openssl genrsa -out myclient.key`
* Create a certificate signing request (CSR) for the server using this key, specifying values when prompted.
`openssl req -new -key myclient.key -out myclient.csr`
* Create an EXT file `myserver.com.ext` file and add the following values:
        
        authorityKeyIdentifier=keyid,issuer
        basicConstraints=CA:FALSE
        keyUsage = digitalSignature, keyAgreement
        extendedKeyUsage=clientAuth
        subjectAltName = @alt_names
    
        [alt_names]
        DNS.1 = myclient

* Generate the certificate using the CSR and EXT file, signing it with the **same CA certificate and private key as the server**:
`openssl x509 -req -in myclient.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out myclient.crt -days 3650 -sha256 -extfile myclient.ext `

## Mikrotik Client
#### Install client certificates

*   Upload the client certificate, private key and CA certificate to the Mikrotik in `Files`.
    *   myclient.crt
    *   mysclient.key
    *   myCA.pem / myCA.crt
    
*   Under `System > Certificates`, click `Import` and add each of these files.

There should now be two rows displayed on the Certificates page, one for the CA with just `T` alongside it, and another for the client with `KT`.

#### Create OpenVPN client
*   Under `PPP > Interface` choose `Add new` and then select `OVPN Client` from the list.
*   Set `Connect To` to the hostname of the remote server.
*   Leave `Port` as `1194` as before.
*   Leave `Mode` as `ip` as before.
*   Set `Username` and `Password` to the values set before.
*   Select `default-encryption` for `Profile`.
*   For `Certificate`, select your client certificate from the drop-down.
*   Enable `Verify Server Certificate`.
*   Select `SHA1` for `Auth.`.
*   Select `AES 256` for `Cipher`.
*   Leave `Set Default Route` disabled unless you intend to route all internet traffic over the VPN.
*   Press OK.

The client should now connect and will appear under `PPP > Interface` and `Interfaces > Interface`.

## Other client (e.g. desktop)
#### Create OpenVPN configuration file
*   Create a file `myserver.com.ovpn` on your computer and insert the following values:

        client
        dev tun
        proto tcp-client
        tls-client
        remote [myserver.com] 1194
        remote-cert-tls server
        nobind
        persist-key
        persist-tun
        cipher AES-256-CBC
        auth SHA1
        pull
        verb 2
        mute 3
        auth-user-pass secret
        
        #### Either CA/CERT/KEY files:
        ca myCA.pem
        cert myclient.crt
        key myclient.key
        #### Or a PKCS12 file:
        # pkcs12 myclient.pfx
        #### Or inline PEM for CA/CERT/KEY in tags:
        # <ca>pem</ca>
        # <cert>pem</cert>
        # <key>pem</key>
        
        #### Any routes on remote end
        # route [subnet] [mask]
        route 10.0.0.0 255.255.248.0

#### Create secret file
*   Create another file `secret` in the same directory and put only the username and password on the first and second lines.
*   Set the permissions for this file accordingly to prevent unauthorised access.

You should then be able to open this configuration in Tunnelblick (Mac) or another OpenVPN client and connect.
