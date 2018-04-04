# Antivirus Check Service

The __Antivirus Check Service__ provides the ability to scan files with a locally installed clamav daemon. In addition, the service offers a URL scan using [virustotal](https://www.virustotal.com).
The __Antivirus Check Service__ processes incoming scan requests and sends the scan result to a specified web hook.

## Usage
__Antivirus Check Service__ provides two interfaces.

### WebAPI
The WebAPI is the most common interface to use __Antivirus Check Service__.
All requests besides of the root resource `/` have to be authenticated using basic access authentication.

A GET request to `https://<antivirus-check-service>/` gives a detailed usage api doc:
~~~json
"scan file request": {
    "description": "Download file and scan against virus (using local clamd), report back to given webhook uri",
    "path": "/scan/file",
    "method": "POST",
    "params": {
        "download_uri": {
            "type": "string",
            "description": "Complete uri to the downloadable file"
        },
        "callback_uri": {
            "type": "string",
            "description": "Complete uri to the callback uri"
        },
    }
},
"scan url request": {
    "description": "Scan Url (using virustotal), report back to given webhook Uri",
    "path": "/scan/url",
    "method": "POST",
    "params": {
        "url": {
            "type": "string",
            "description": "Url to scan using virustotal"
        },
        "callback_uri": {
            "type": "string",
            "description": "Complete Uri to the callback uri"
        },
    }
},
"clamav daemon version": {
    "description": "Get clamav daemon version and last database update",
    "path": "/antivirus-version",
    "method": "GET"
},
~~~

To get the clamav daemon version and last database update, you can send a request to the WebAPI `/antivirus-version`.
The response is similar to:
~~~json
{"clamd-version": "0.99.2/24389/Tue", "clamd-database-version": "2018/03/13 - 08:12:22"}
~~~

### AMQP
The __Antivirus Check Service__ provides an AMQP API, which is uses by the WebAPI as well. 
Authenticate and publish a message to the regarding queue using the routing_key:

- url: `amqp://<user>:<password>@<antivirus-check-service>/antivirus`

#### scan file:
 - routing key: `scan_file`
 - message:
    ~~~json
    {
      "download_uri": "https://<uri-to-file>",
      "callback_uri": "https://<uri-to-report-endpoint>"
    }
    ~~~

#### scan file:
 - routing key: `scan_url`
 - message:
    ~~~json
    {
      "url": "https://<url-to-be-scanned>",
      "callback_uri": "https://<uri-to-report-endpoint>"
    }
    ~~~

### Reports
The reports are PUT requests to the given webhook Uri. The payload differs reagrding the scan type.

#### scan file payload
~~~json
{"virus_detected": "<true|false>", "virus_signature": "<null|signature name>"}
~~~

#### scan url payload
~~~json
{"blacklisted": "<true|false>", "full_report": "<virustotal's full report>"}
~~~

### Error
If an error occures the __Antivirus Check Service__ will try to send an error page (500) with the error message as json:
~~~json
{"error": "<error message>"}
~~~

## Install
As prerequisite you have to have installed: `git` and `make`.
In general, follow the instruction in the vagrant provision script.

To get the services running, you need access to rabbitmq-server and clamav-daemon.

### Rabbitmq
- Install rabbitmq-server or use an already installed rabbitmq-server
- Create a vhost and a user with full permissions to the created vhost

### Clamav-Daemon
- Install clamav-daemon clamav-freshclam clamav-unofficial-sigs or use an already installed instance.
- To access the clamav-daemon via TCP socket append the config values to `/etc/clamav/clamd.conf`:
  ```
  TCPSocket 3310
  TCPAddr 0.0.0.0
  ```
- __BE PATIENT!__ At the first run, freshclam has to download all signatures, which can take a
  while and prevent clamav-daemon from working (don't forget to restart clamav-daemon).

### VirusTotal
An API-Key is needed to use virustotal. To get this, an account on virustotal has to be created. The API-Key can be found in the account's settings.

### Service
`git clone` this repository to a modern debian (currently stretch). Change to the new
directory and run as `root`: `make install`. This will install all necessary
packages.

- Copy `/antivirus_service/config.template.yml` to `/antivirus_service/config.yml`
  and adjust the config file.
- Add `auth_keys` to the webserver section, format: `<username>:<password>`
- The python setup routine installs the packages locally (the clone path) and
  creates a link to `/usr/local/bin/antivirus`
- The installation process installs __Antivirus Check Service__ as three systemd
  services (for webserver, scan_file and scan_url). 
- Start the service with:
  `systemctl start antivirus-<webserver|scanfile|scanurl>.service`
- Control the service with: `systemctl status antivirus-<webserver|scanfile|scanurl>.service`
- The logs can be read with: `journalctl -f -u antivirus-<webserver|scanfile|scanurl>.service`

## Verify installation
- Change to the `antivirus_check_service/tests` and start the development webserver with:
  `python3 develop_webserver.py`. The minimal webserver simulates the fileserver and listens for the webhook.
- To initiate the scan request run: `curl -v -d@scan_virus_file_payload.json -u <username>:<password> http://<antivirus-check-service>:8080/scan/file`, 
  whereby `scan_virus_file_payload.json` has the payload:
  ```json
  {
    "download_uri": "http://localhost:7000/scanfile?name=virus.txt",
    "callback_uri": "http://localhost:7000/report"
  }
  ```
- The develop webserver should output:
  ```
  ======== Running on http://0.0.0.0:7000 ========
  (Press CTRL+C to quit)
  --- Scan Request Result ---
  {"virus_detected": true, "virus_signature": "Eicar-Test-Signature"}
  ```

## Update
Change to install directory and run `make update`

## Development & Testing

This project can be developed and tested in a vagrant box. `debian/stretch64` is used as predefined image.
It is strongly recommended to use the vagrant-vbguest plugin: by `vagrant plugin install vagrant-vbguest`.
(The virtualbox guest additions provides synchronizing the sources)

The vagrant command `vagrant up` starts a virtual machine and provision __Antivirus Check Service__
within. At the end of the provision the __Antivirus Check Service__ service will
be started.