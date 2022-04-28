### **`Author`**

        Abdias Natanael Vrech
        Tel. +34 681 93 54 92
        Ontinyent, Valencia, SP
        email: abdias.dev@gmail.com
        Telynet S.A.
___
## README to Add `Certificate Pinning`:
<br>

This is a tutorial to block communications with the server in case the SSL is not the same than the indicated in the App's Communication

Index:
- [Get Sha256 of Cert](#get-sha256-of-cert)
- [Network Security Configuration (NSC)](#network-security-configuration-nsc)
- [OkHttp and CertificatePinner](#okhttp-and-certificatepinner)
- [HttpsURLConnection and TrustManager](#httpsurlconnection-and-trustmanager)

___
<br>

## [Get Sha256 of Cert](#https://developer.android.com/training/articles/security-config)

In order to get the `sha256` needed in the two first methods, you can do it using `openssl`

There is at least two options for running `openssl`
1. Using [Openssl](#https://www.openssl.org/)
2. Using [Git Bash](#https://git-scm.com/downloads) _(this is my prefered one, since it is useful for a lot more options)_

<br>

Once you have it all set, lets do it:
1. position the console in the cert folder
    ```
    cd C:/path/to/your/cert/folder
    ```
2. Run this command

    - Change `cert_name` to the file name
    - Change `ext` to its extension [ `cer, crt, pem, ..`]) <br><br>
    ```
    openssl x509 -in cert_name.cer -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
    ```
3. The prevoius execution will print the code in the console. Copy it, and that's it

___
<br>

## [Network Security Configuration (NSC)](#https://developer.android.com/training/articles/security-config)
_(this will only work for Android SDK >= `24` || Android `7.0` )_

1. Create a network security config file under <br><br>
    ```
    res/xml/network_security_config.xml
    ```
2. Add the `android:networkSecurityConfig` attribute to the application tag <br><br>
    ```
    <?xml version="1.0" encoding="utf-8"?>
    <manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    package="co.netguru.demoapp">
    <application
        android:networkSecurityConfig="@xml/network_security_config">
        ...
    </application>
    </manifest
    ```
3. Set up the configuration file and add fingerprints.<br>
    - _replace `example.com` url with the right domain (without `https://`)_
    - _add as many `pin` → `sha-256` keys as you need_
    - _repeat the block of `domain-config` for each domain_ <br><br>
    ```
    <?xml version="1.0" encoding="utf-8"?>
    <network-security-config>
    <domain-config>
        <domain includeSubdomains="true">example.com</domain>
        <pin-set>
            <pin digest="SHA-256">ZC3lTYTDBJQVf1P2V7+fibTqbIsWNR/X7CWNVW+CEEA=</pin>
            <pin digest="SHA-256">GUAL5bejH7czkXcAeJ0vCiRxwMnVBsDlBMBsFtfLF8A=</pin>
        </pin-set>
    </domain-config>
    </network-security-config>
    ```

___
<br>

## OkHttp and CertificatePinner
This method will require to use `OkHttp` in order to append the `sha-256` keys _(Adapt it at your needs)_

1. Create Method to parse keys as CertificatePinner 

    - `pinningMap`: map of `site` wit a list of `sha-256` <br><br>

    ```
    private CertificatePinner.Builder certPinner( Map<String, List<String>> pinningMap )
        {
            CertificatePinner.Builder certificatePinner = null;
            if( pinningMap != null )
            {
                certificatePinner = new CertificatePinner.Builder();
                Set<Map.Entry<String, List<String>>> entryMap = pinningMap.entrySet();
                for( Map.Entry<String, List<String>> entry : entryMap )
                {
                    if( entry != null )
                    {
                        String key = entry.getKey();
                        List<String> value = entry.getValue();
                        certificatePinner.add( key, value.toArray(new String [value.size()]));
                    }
                }
            }
            
            return certificatePinner;
        }
    ```
2. Create the list of sha-256 to pin to the connection <br><br>
    ```
    String domain = "api.example.com";
    Map<String, List<String>> pinningMap = new HashMap<>();
        pinningMap.put(
                domain,
                Arrays.asList(
                        "sha256/vt8A0euzW8Pv51wHHP0/K8UYGqNL8B3C1jWRdO3obdQ=",
                        "sha256/jQJTbIh0grw0/1TkHSumWb+Fs0Ggogr621gT3PvPKG0=",
                        "sha256/C5+lpZ7tcVwmwQIMcRtPbsQtWLABXhQzejna0wHFr8M="
                )
        );
    ```

3. Stablish the connection pinning the Certificate SHAs
    - Declare `site` variable with the full url _(including the `https` attr)_
    - Call the previous method `certPinner()` passing the `pinningMap` )
    - If there is no match with the `pinningMap` list, will trigger `onFailure()` with the expected `sha-256`
    - If everything is ok, will go to `onResponse()` <br><br>
    ```
    String site = "https://api.example.com/route/params";
    CertificatePinner.Builder certificatePinner = certPinner( pinningMap );
    OkHttpClient.Builder clientHttpBuilder = new OkHttpClient.Builder();

    if( certificatePinner != null )
        clientHttpBuilder.certificatePinner( certificatePinner.build() );

    OkHttpClient client = clientHttpBuilder.build();
    client.newCall( new Request.Builder()
        .url( site )
        .build()
    ).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            Log.d( "Failure", e.getMessage() );
        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
            Log.d( "Response", response.body().string() );
        }
    });
    ```
___
<br>

## HttpsURLConnection and TrustManager
_(The old-school way, this is the one we use on `Telynet`)_

<br>

For this to work, you need a `.cert` file.

If you just have a `.pem`, you can convert it using `openssl`:
```
openssl x509 -inform PEM -in cert1.pem -outform DER -out pem.cer 
``` 
<br>

Lets do it:
1. Add your certificate `cert.cer` file to the app resources, under: <br><br>
    ```
    /res/raw
    ```
2. Write this function to create the `SSLContext` based on a `raw` file: <br><br>
    ```
    public SSLContext ssl( Context context, int raw )
    {
        try
        {
            SSLContext ssl = SSLContext.getInstance( "TLS" );
            KeyStore keyStore = KeyStore.getInstance( "PKCS12" );
            InputStream is = context.getResources().openRawResource( raw );
            CertificateFactory cf = CertificateFactory.getInstance( "X.509" );
            X509Certificate caCert = ( X509Certificate )cf.generateCertificate( is );

            keyStore.load( null );
            keyStore.setCertificateEntry( "caCert", caCert );

            TrustManagerFactory tmf = TrustManagerFactory.getInstance( TrustManagerFactory.getDefaultAlgorithm() );
            tmf.init( keyStore );
            ssl.init( null, tmf.getTrustManagers(), null );

            return ssl;
        }
        catch ( NoSuchAlgorithmException e ) {}
        catch ( KeyStoreException e ) {}
        catch ( FileNotFoundException e ) {}
        catch ( IOException e ) {}
        catch ( CertificateException e ) {}
        catch ( KeyManagementException e ) {}

        // Add this ssl like this:
        // urlConnection.setSSLSocketFactory( ssl.getSocketFactory() );
        return null;
    }
    ```
3. Now you can invoke it on the `HttpURLconnection`

    - Replace `site` with the corresponding url
    - Replace `R.raw.cert` with the filename
    - Get `sslCntxt` from previous function created `ssl()` 
    - Add `setSSLSocketFactory()` with the SSLContext <br><br>
    ```
    ...
    URL url = new URL( site );
    HttpsURLConnection conn = ( HttpsURLConnection )url.openConnection();
    SSLContext sslCntxt = ssl( Activity.this, R.raw.cert );
    if( sslCntxt != null ) conn.setSSLSocketFactory( sslCntxt.getSocketFactory());
    ...
    ```
When reading the response:
- If is empty → the connection was blocked due to ssl unmatched
- If got result → the connection was stablished and SSL Verified

---
---
Extra!!

If you want to get Certificate Info, docall this function:
```
private void httpsCert( HttpsURLConnection conn )
{
    Map<String, String> map = new HashMap<>();
    if( conn != null )
    {
        try
        {
            map.put( "responseCode", conn.getResponseCode() );
            map.put( "cipherSuite", conn.getCipherSuite() );

            Certificates[] certs = conn.getServerCertificates();

            List<Map<String, String>> lists = new ArrayList<>();
            for( Certificate cert: certs )
            {
                Map<String, String> certHash = new HashMap<>();
                certHash.put( "type", cert.getType() );
                certHash.put( "hashCode", cert.hashCode() );
                certHash.put( "publicKey-algorithm", cert.getPublicKey()getAlgorithm() );
                certHash.put( "publicKey-format", cert.getPublicKey()getFormat() );

                lists.add( lists );
            }

            map.put( "certs", lists );
        } catch( SSLPeerUnverifiedException e ) {
            e.printStackTrace();
        } catch( IOException e ) {
            e.printStackTrace();
        }
    }
}
```