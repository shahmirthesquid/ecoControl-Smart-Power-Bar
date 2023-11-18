# HTTPS Requests
Fetching or posting data to the internet is one of the core tasks of an IoT device. Doing so over HTTP is implemented quite well in the default ESP8266 Arduino libraries, but for HTTPS requests things are less straightforward. This class implements arbitrary HTTPS requests in a secure way, without requiring any specific certificates or fingerprints to be hard coded in the application. A full certificate store is automatically downloaded on build, and stored in PROGMEM.

Certificates are only valid for a specific time period; therefore, in order to determine whether a certificate is valid, the ESP8266 must be set to the current date and time. This is usually accomplished using the SNTP protocol to get the current time from a NTP server. Because of this it is mandatory to initialize the time by calling `timeSync.begin()` before attempting any HTTPS requests. More information on the `timeSync` class can be found in it's [documentation](https://github.com/maakbaas/esp8266-iot-framework/blob/master/docs/time-sync.md).

## Class Methods

#### begin

```c++
void begin(String url);
void begin(String url, bool useMFLN);
```
This method initializes a request object to `url`. The URL must include a http:// or https:// prefix. The method is only required if you want to set the `useMFLN` parameter or use the `addHeader` or `setAuthorization` functions. Otherwise you can use one of the shorthand request methods below.

MFLN has the potential to reduce memory needed for HTTPS requests by up to 20kB if it is supported by the server you are requesting from. The reason that it is not enabled by default is that it takes the ESP8266 roughly 5 seconds to detect if MFLN is supported. With this flag you can enable MFLN if this trade-off is worth it in your use case. See the section on [memory usage](https://github.com/maakbaas/esp8266-iot-framework/blob/master/docs/fetch.md#memory-usage) for details.

#### GET

```c++
int GET(String url);
int GET();
```
This method starts a GET request to the URL specified in `url`. The URL must include a http:// or https:// prefix. The method returns the HTTP response code. If you first called the `begin` method, the variant without the `url` argument should be used.

#### POST

```c++
int POST(String url, String body);
int POST(String body);
```
This method starts a POST request to the URL specified in `url` with the payload specified in `body`. The URL must include a http:// or https:// prefix. The method returns the HTTP response code. If you first called the `begin` method, the variant without the `url` argument should be used.

#### PUT

```c++
int PUT(String url, String body);
int PUT(String body);
```
This method starts a PUT request to the URL specified in `url` with the payload specified in `body`. The URL must include a http:// or https:// prefix. The method returns the HTTP response code. If you first called the `begin` method, the variant without the `url` argument should be used.

#### PATCH

```c++
int PATCH(String url, String body);
int PATCH(String body);
```
This method starts a PATCH request to the URL specified in `url` with the payload specified in `body`. The URL must include a http:// or https:// prefix. The method returns the HTTP response code. If you first called the `begin` method, the variant without the `url` argument should be used.

#### DELETE

```c++
int DELETE(String url);
int DELETE();
```
This method starts a DELETE request to the URL specified in `url`. The URL must include a http:// or https:// prefix. The method returns the HTTP response code. If you first called the `begin` method, the variant without the `url` argument should be used.

#### busy

```c++
bool busy();
```
This method returns if the server is still connected or if new bytes are still available to be read.

#### available

```c++
bool available();
```
This method returns if there are still bytes available to read. In general the `busy()` function is more robust to use in a while loop if you want to stream incoming data.

#### read

```c++
uint8_t read();
```
Returns the next byte from the incoming response to the HTTP(S) request.

#### readString

```c++
String readString();
```
Reads the complete response as a string. Use this method only if you are requesting an API or other URL for which you know that the response only contains a limited number of bytes.

#### clean

```c++
void clean();
```
Call this after you have finished handling a request to free up the memory that was used by the `http` and `client` objects.

#### setAuthorization

```c++
void setAuthorization(const char * user, const char * password);
void setAuthorization(const char * auth);
```

Sets the HTTP Auhorization credentials for the request. Username and passord or a base64 encoded string of the credentials can be used. Only Basic authorization is supported.

#### addHeader

```c++
void addHeader(String name, String value);
```

Allows to add headers to your HTTP(S) request.

## Usage Examples

For parsing and processing larger responses you can stream the incoming data such as in this example:

```c++
fetch.GET("https://www.google.com");

while (fetch.busy())
{
    if (fetch.available())
    {
        Serial.write(fetch.read());
    }
}

fetch.clean();
```

For reading in a shorter response you can simply use `readString()` instead:

```c++
fetch.GET("https://www.google.com");

Serial.write(fetch.readString());

fetch.clean();
```

## Memory Usage

As described [here](https://arduino-esp8266.readthedocs.io/en/latest/esp8266wifi/bearssl-client-secure-class.html#mfln-or-maximum-fragment-length-negotiation-saving-ram) each HTTPS request needs roughly 28kB of free memory. Which is the majority of what is available, so can become an issue if your application needs more memory.

To help with this, MFLN is implemented to reduce the memory requirements. But this technology can only be used if it is supported by the server. Before a HTTPS request is executed the `probeMaxFragmentLength` and `setBufferSizes` functions are used to check if the receive buffer size can be reduced. When supported by the server the memory needed for each request is reduced to roughly 6kB, which is quite significant.

## Code Generation

As mentioned earlier a full certificate store is saved in PROGMEM as part of the application. By default this store is located in `certificates.h`, and will be included in the build. These certificates will be used by the ESP8266 Arduino BearSSL class to establish a secure connection.

If you ever want to update or rebuild the certificate store, you can do this by enabling or running the pre-build script `preBuildCertificates.py`. This script will read in all root certificates from the Mozilla certificate store and process these into a format that is compatible with the ESP8266 Arduino layer.

For this step OpenSSL is needed. On Linux this is probably available by default, on Windows this comes as part of something like MinGW, or Cygwin, but is also installed with the Windows Git client. If needed you can edit the path to OpenSSL by adding the build flag below to `platformio.ini`:

**-DOPENSSL="C:/Program Files/Git/usr/bin/openssl.exe"** Path to openssl executable. The location shown here is the default location. If your openssl is in a different location, change this flag accordingly

Another prerequisite is the Python module asn1crypto. If this module is not available, the script will attempt to install it using `pip`.

## Certificate Store Size

The default certificate store contains ~150 certificates, and is roughly 170kB in size. There is a method to reduce this size by only including the root certificates that are needed for a predefined list of domains. These domains can be defined with the `DOMAIN_LIST` build flag in `platformio.ini`. See the [installation guide](https://github.com/maakbaas/esp8266-iot-framework/blob/master/docs/installation-guide.md) for more information on this build flag.

Note that doing this reduces the flexibility of your application. This should not be used for cases where there is a user configurable URL that can change after the build. Also, if the root certificate for a certain domain changes, your application will no longer work. But this scenario is probably not very likely.
