---
layout: post
title:  "ChromiumOS Dev Setup"
date:   2021-08-08 15:11:14 +0800
categories: 
---
## Preparation

1. A high speed proxy against GFW.
   20-30GB source data will be downloaded from google's server.
2. Multi-core processer to speed up first build.
3. OS spec
    * locales = en_US.UTF-8 
        ```
        sudo apt-get install -y locales
        sudo dpkg-reconfigure locales
        ```
    * dep. git, python(>=3.6), etc
        ```bash
        sudo apt-get install -y git-core gitk git-gui curl lvm2 thin-provisioning-tools python-pkg-resources python-virtualenv python-oauth2client xz-utils python3.6
     ```

----
----
## Concept
Like developement flow for other projects, ChromiumOS have similar steps: code, build, test.
Additionally ChromiumOS developement has its own specific tools for each stage.

### Code Mangament 
* depot_tool: repo sync and chroot

### Build
* chroot: containerized dev/build environment, docker's ancestor

### Test
* devserver: 
    Your can update a ChromiumOS instance by cli tools or use a devserver.
* ChromiumOS instance: 
    ChromiumOS installed on vm, usb bar or others.

----
----
## Code Download

Due to some well-known reasons, internet accessibilities in China have been somewhat restricted. 
We need a proxy to complete the task.

```
SOFTWARE -> PROXY CLIENT -> PROXY SERVER -> WWW 
```

Surge/Clash/Vpn is prefered proxy client, it can
proxy nearly all traffic through the network interface. While proxychain can only intercept traffic through socket and shell env settings (HTTP_PROXY&HTTPS_PROXY) is only honored by a few softwares.

For proxy server, choose an nearby foreign datacenter zone, use a valid proxy protocol offered by freedom fighter. 
```bash
docker run -e PASSWORD=<password> -p<server-port>:8388 -p<server-port>:8388/udp -d shadowsocks/shadowsocks-libev
```

For better performance, you need to consider the link beneath networks, like CN2, BGP etc.


Now, we can actually start to download project source.

1. download depot_tools.
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=/{path}/{to}/depot_tools:$PATH
```
2. configure repo to specific branch.
```bash
mkdir chromiumos && cd chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b release-R90-13816.B
```
3. sync repo.
```bash
# download 20G - 30G data
cd chromiumos
repo sync -j16 # -j concurrency
```
**PS**: It will take hours.

Now, we can have a look of the repo folder.
```bash
chromiumos/
    src/
        build/ # build output
        scripts/ # build scripts
        third_party/ # third party package, including kernel
        overlay/ # added the make.conf PORTDIR_OVERLAY
        platform/  
        platform2/
        project_public/
        config/
    chromite/ # test related
    chroot/ # operation system fro chroot
    infra/ # dependency
    infra_virtualenv/ # dependency envrioment
    docs/
```

----
----
## Build


Setup chroot. To ensure everyone uses the same exact environment and tools to build Chromium OS,
   all building is done inside a chroot.
```bash
# download 400M data
cd chromiumos
cros_sdk
```

Now, you have entered chroot environment.

1. set board
```bash
(cr) export BOARD=amd64-generic
(cr) setup_board --board=${BOARD}
```

2. (Optional) set user password for image. `test0000` will used as password in test mode even if you executed `set_shared_user_password.sh`.
```bash
(cr) ./set_shared_user_password.sh
```

3. build packages
```bash
# build 504 packages 90min/4 core
(cr) ./build_packages --board=${BOARD}
```
**PS**: It will take hours.

4. build image
```bash
(cr) ./build_image --board=${BOARD} --noenable_rootfs_verification test
```
The output image can be used to flash on usb driver. 

5. make vm image from the output image from previous step.
```bash
(cr) ./image_to_vm.sh --board=${BOARD} --test_image
```

----

Now let's complete a real task: build a new kernel version(5.10) for our image.
[https://chromium.googlesource.com/chromiumos/docs/+/HEAD/kernel_development.md](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/kernel_development.md)


```bash
(cr) cros-workon-${BOARD} start chromeos-kernel-5_10
FEATURES="noclean" cros_workon_make --board=${BOARD} --install chromeos-kernel-5_10
```

**PS**

I encounter a file collisions issue.
```bash
 * sys-kernel/chromeos-kernel-4_14-4.14.233-r1529:0::chromiumos
 *  /build/amd64-generic/boot/vmlinuz
 *  /build/amd64-generic/usr/lib/debug/boot/vmlinux
 * 
 * Package 'sys-kernel/chromeos-kernel-5_10-9999' NOT merged due to file
 * collisions. If necessary, refer to your elog messages for the whole
 * content of the above message.

```
Delete those files and rebuild will fix this issue.

If you already has a vm runing (this part will be discussd in the next section), you can test you kernel by update to the vm and ssh to your vm.

Update vm.
```bash
(cr)./update_kernel.sh  --remote=127.0.0.1 --ssh_port=9222
```

Ssh to vm and lookup kernel version.
```bash
(local) ssh -p 9222 root@127.0.0.1 # password: test0000
(vm) uname -r
```

----
----
## Test

There is a tool set to interact with ChromiumOS instance for test purpose.

* cros deploy

    [https://chromium.googlesource.com/chromiumos/docs/+/HEAD/cros_deploy.md](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/cros_deploy.md)

* update kernel

    [https://chromium.googlesource.com/chromiumos/docs/+/HEAD/kernel_development.md](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/kernel_development.md)

```bash
(cr)./update_kernel.sh  --remote=127.0.0.1 --ssh_port=9222
```

### VM
Start vm with ChromiumOS vm image.
```bash
(cr) cros_vm --start --board=${BOARD}
```
Now, you can ssh or vnc to this vm.
```bash
ssh -p 9222 root@127.0.0.1 # password: test0000
remmina # vnc 127.0.0.1:5900
```

Stop vm.
```bash
(cr) cros_vm --stop
```
### devserver
Another way to update os.
[https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/master/docs/devserver.md](https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/master/docs/devserver.md)


Devserver need to start in chroot enviroment.
```bash
(cr) sudo start_devserver
```

ChromiumOS instance pull update payload from devserver and update itself.
You need to configure `/etc/lsb-release` first.
```bash
CHROMEOS_AUSERVER=http://ubuntu:8080/update
CHROMEOS_DEVSERVER=http://ubuntu:8080
```

```bash
(vm) update_engine_client --update
```

---


Make docker images for devserver.

1. copy file
    ```bash
    mkir devserver && cd devserver
    cp -r /{path}/{to}/{devserver}/*.py /{path}/{to}/{devserver}/dut-scripts .
    # devserver depend on code in chromite
    cp -r /{path}/{to}/{chromiumos}/chromite .
    ```
2. add `requirements.txt`
    ```txt
    cherrypy
    psutil
    ```
3. `Dockerfile`
    ```
    FROM python:3.9

    WORKDIR /app
    COPY requirements.txt requirements.txt
    RUN pip3 install -r requirements.txt
    RUN useradd chronos
    ENV PORTAGE_USERNAME=chronos
    COPY . .
    CMD python3 devserver.py
    ```
4. build docker image
    ```bash
    docker build --rm -t cr_devserver .
    ```
5. run container instance
    ```bash
    docker run -p 8080:8080 -v /<path_to_chroot_build>:/build -v /<path_to_static>/:/app/static/ cr_devserver
    ```

----
----
## Unresolved Issues
1. Change chroot to docker for devserver may be a better solution. 
2. `update_engine_client` error
    * Vm `update_engine_client` error log:
    ```bash
    (vm) update_engine_client --update=true
    2021-08-08T06:06:03.464109Z INFO update_engine_client: [update_engine_client.cc(511)] Forcing an update by setting app_version to ForcedUpdate.
    2021-08-08T06:06:03.475705Z INFO update_engine_client: [update_engine_client.cc(513)] Initiating update check.
    2021-08-08T06:06:03.546710Z INFO update_engine_client: [update_engine_client.cc(542)] Waiting for update to complete.
    2021-08-08T06:06:53.969584Z ERROR update_engine_client: [update_engine_client.cc(211)] Update failed, current operation is UPDATE_STATUS_IDLE, last error code is ErrorCode::kOmahaErrorInHTTPResponse(37)
    ```

    * Devserver error log:
    ```bash
    [07/Aug/2021:23:06:28] UPDATE Handling update ping as http://ubuntu:8080
    INFO:cherrypy.error.140704208815832:[07/Aug/2021:23:06:28] UPDATE Handling update ping as http://ubuntu:8080
    [07/Aug/2021:23:06:28] UPDATE Using static url base http://ubuntu:8080/static
    INFO:cherrypy.error.140704208815832:[07/Aug/2021:23:06:28] UPDATE Using static url base http://ubuntu:8080/static
    [07/Aug/2021:23:06:28] UPDATE Responding to client to use url http://ubuntu:8080/static/amd64-generic/R90-13816.93.2021_08_07_2006-a1 to get image
    INFO:cherrypy.error.140704208815832:[07/Aug/2021:23:06:28] UPDATE Responding to client to use url http://ubuntu:8080/static/amd64-generic/R90-13816.93.2021_08_07_2006-a1 to get image
    ERROR:root:Failed to read app data from chromiumos_base_image.bin-package-sizes.json ('appid')
    [07/Aug/2021:23:06:28] HTTP 
    Traceback (most recent call last):
      File "/usr/lib64/python3.6/site-packages/cherrypy/_cprequest.py", line 630, in respond
        self._do_respond(path_info)
      File "/usr/lib64/python3.6/site-packages/cherrypy/_cprequest.py", line 689, in _do_respond
        response.body = self.handler()
      File "/usr/lib64/python3.6/site-packages/cherrypy/lib/encoding.py", line 221, in __call__
        self.body = self.oldhandler(*args, **kwargs)
      File "/usr/lib64/python3.6/site-packages/cherrypy/_cpdispatch.py", line 54, in __call__
        return self.callable(*self.args, **self.kwargs)
      File "/usr/lib/devserver/devserver.py", line 1132, in update
        return updater.HandleUpdatePing(data, label, **kwargs)
      File "/usr/lib64/devserver/autoupdate.py", line 152, in HandleUpdatePing
        update_metadata_dir=local_payload_dir)
      File "/usr/lib64/devserver/nebraska.py", line 786, in __init__
        self.update_app_index = AppIndex(update_metadata_dir)
      File "/usr/lib64/devserver/nebraska.py", line 582, in __init__
        self._Scan()
      File "/usr/lib64/devserver/nebraska.py", line 598, in _Scan
        app = AppIndex.AppData(metadata)
      File "/usr/lib64/devserver/nebraska.py", line 719, in __init__
        self.appid = app_data[self.APPID_KEY]
    KeyError: 'appid'
    ```

    Due to `/usr/lib64/devserver/nebraska.py`, it seems like that `chromiumos_base_image.bin-package-sizes.json` does not contain key **appid**.
    ```python
    class AppIndex(object):
      def _Scan(self):
        """Scans the directory and loads all available properties files."""
        if self._directory is None:
          return

        for f in os.listdir(self._directory):
          if f.endswith('.json'):
            try:
              with open(os.path.join(self._directory, f), 'r') as metafile:
                metadata_str = metafile.read()
                metadata = json.loads(metadata_str)
                # Get the name from file name itself, assuming the metadata file
                # ends with '.json'.
                metadata[AppIndex.AppData.NAME_KEY] = f[:-len('.json')]
                app = AppIndex.AppData(metadata)
                self._index.append(app)
            except (IOError, KeyError, ValueError) as err:
              logging.error('Failed to read app data from %s (%s)', f, str(err))
              raise
            logging.debug('Found app data: %s', str(app))

    class AppData(object):
        APPID_KEY = 'appid'
        NAME_KEY = 'name'
        IS_DELTA_KEY = 'is_delta'
        SIZE_KEY = 'size'
        METADATA_SIG_KEY = 'metadata_signature'
        METADATA_SIZE_KEY = 'metadata_size'
        TARGET_VERSION_KEY = 'target_version'
        SOURCE_VERSION_KEY = 'source_version'
        SHA256_HEX_KEY = 'sha256_hex'
        PUBLIC_KEY_RSA_KEY = 'public_key'

        def __init__(self, app_data):
          self.appid = app_data[self.APPID_KEY]
          # Replace the begining of the App ID with the canary version.
          self.canary_appid = ''
          if len(self.appid) >= len(_CANARY_APP_ID):
            self.canary_appid = (_CANARY_APP_ID +
                                 self.appid[len(_CANARY_APP_ID):])
          self.name = app_data[self.NAME_KEY]
          self.target_version = app_data[self.TARGET_VERSION_KEY]
          self.is_delta = app_data[self.IS_DELTA_KEY]
          self.source_version = (
              app_data[self.SOURCE_VERSION_KEY] if self.is_delta else None)
          self.size = app_data[self.SIZE_KEY]
          # Sometimes the payload is not signed, hence the matadata signature is
          # null, but we should pass empty string instead of letting the value be
          # null (the XML element tree will break).
          self.metadata_signature = app_data[self.METADATA_SIG_KEY] or ''
          self.metadata_size = app_data[self.METADATA_SIZE_KEY]
          self.public_key = app_data.get(self.PUBLIC_KEY_RSA_KEY)
          # Unfortunately the sha256_hex that paygen generates is actually a base64
          # sha256 hash of the payload for some unknown historical reason. But the
          # Omaha response contains the hex value of that hash. So here convert the
          # value from base64 to hex so nebraska can send the correct version to the
          # client. See b/131762584.
          self.sha256 = app_data[self.SHA256_HEX_KEY]
          self.sha256_hex = base64.b16encode(
              base64.b64decode(self.sha256)).decode('utf-8')
          self.url = None # Determined per-request.
    ```



{% include disque.html %}