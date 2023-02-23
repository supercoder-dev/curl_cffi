# curl_cffi

Python binding for [curl-impersonate](https://github.com/lwthiker/curl-impersonate)
via [CFFI](https://cffi.readthedocs.io/en/latest/).

Unlike other pure python http clients like `httpx` or `requests`, this package can
impersonate browsers' TLS signatures or JA3 fingerprints. If you are blocked by some
website for no obvious reason, you can give this package a try.

[中文文档](README-zh.md)

## Install

    pip install --upgrade curl_cffi

This should work for Linux(x86_64/aarch64), macOS(Intel), Windows(amd64). If it does not
work, you may need to compile and install `curl-impersonate` first.

## Usage

`requests/httpx`-like API:

```python
from curl_cffi import requests

# Notice the impersonate parameter
r = requests.get("https://tls.browserleaks.com/json", impersonate="chrome101")

print(r.json())
# output: {'ja3_hash': '53ff64ddf993ca882b70e1c82af5da49'
# the fingerprint should be the same as target browser

# proxies are supported
proxies = {"https": "http://localhost:3128"}
r = requests.get("https://tls.browserleaks.com/json", impersonate="chrome101", proxies=proxies)

# sessions are supported
s = requests.Session()
s.get()
print(s.cookies)
```

Supported impersonate versions:

- chrome99
- chrome100
- chrome101
- chrome104
- chrome99_android
- edge99
- edge101
- safari15_3
- safari15_5

Alternatively, you can use the low-level curl-like API:

```python
from curl_cffi import Curl, CurlOpt
from io import BytesIO

buffer = BytesIO()
c = Curl()
c.setopt(CurlOpt.URL, b'https://tls.browserleaks.com/json')
c.setopt(CurlOpt.WRITEDATA, buffer)

c.impersonate("chrome101")

c.perform()
c.close()
body = buffer.getvalue()
print(body.decode())
```

See `example.py` or `tests/` for more examples.

## API

Requests: almost the same as requests.

Curl object:

* `setopt(CurlOpt, value)`: Sets curl options as in `curl_easy_setopt`
* `perform()`: Performs curl request, as in `curl_easy_perform`
* `getinfo(CurlInfo)`: Gets information in response after curl perform, as in `curl_easy_getinfo`
* `close()`: Closes and cleans up the curl object, as in `curl_easy_cleanup`

Enum values to be used with `setopt` and `getinfo`, and can be accessed from `CurlOpt` and `CurlInfo`.

## Trouble Shooting

### Pyinstaller `ModuleNotFoundError: No module named '_cffi_backend'`

You need to tell pyinstaller to pack cffi and data files inside the package:

    pyinstaller -F .\example.py --hidden-import=_cffi_backend --collect-all curl_cffi

### Using https proxy, error: `OPENSSL_internal:WRONG_VERSION_NUMBER`

You are messing up https-over-http proxy and https-over-https proxy, for most cases, you
should change `{"https": "https://localhost:3128"}` to `{"https": "http://localhost:3128"}`.
Note the protocol in the url for https proxy is `http` not `https`.

See [this issue](https://github.com/yifeikong/curl_cffi/issues/6#issuecomment-1415162495) for a detailed explaination.

## Current Status

This implementation is very hacky now, but it works for most common systems.

When people installing other python curl bindings, like `pycurl`, they often face
compiling issues or OpenSSL issues, so I really hope that this package can be distributed
as a compiled binary package, uses would be able to use it by a simple `pip install`, no
more compile errors.

For now, I just download the pre-compiled `libcurl-impersonate` from github and build a
bdist wheel, which is a binary package format used by PyPI, and upload it. However, the
right way is to download curl and curl-impersonate sources on our side and compile them
all together.

Help wanted!

TODOs:

- [ ] Write docs.
- [x] Binary package for macOS(Intel) and Windows.
- [ ] Support musllinux(alpine) and macOS(Apple Silicon) bdist by building from source.
- [ ] Exclude the curl headers from source, download them when building.
- [x] Update curl header files and constants via scripts.
- [x] Implement `requests.Session/httpx.Client`.
- [x] Create [ABI3 wheels](https://cibuildwheel.readthedocs.io/en/stable/faq/#abi3) to reduce package size and build time.
- [ ] Set default headers as in curl-impersonate wrapper scripts.

## Change Log

- 0.3.0, copied more code from `httpx` to support session
    - Add `requests.Session`
    - Breaking change: `Response.cookies` changed from `http.cookies.SimpleCookie` to `curl_cffi.requests.Cookies`
    - Using ABI3 wheels to reduce package size.

## Acknowledgement

- This package was originally forked from https://github.com/multippt/python_curl_cffi
- headers/cookies files are copied from https://github.com/encode/httpx/blob/master/httpx/_models.py, which is under BSD License.

