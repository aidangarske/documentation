# Loading wolfProvider

## Configuring OpenSSL to Enable Provider Usage

For documentation on how applications use and consume OpenSSL providers, refer to the OpenSSL documentation:

[OpenSSL 3.0](https://www.openssl.org/docs/man3.0/man7/provider.html)

If the application is configured to read/use an OpenSSL config file, additional provider setup steps can be done there. For OpenSSL config documentation, reference the OpenSSL documentation:

[OpenSSL 3.0](https://www.openssl.org/docs/man3.0/man5/config.html)

An application can read and consume the default OpenSSL config file (openssl.cnf) or config as set by OPENSSL\_CONF environment variable and default [openssl\_conf] section.

Alternatively to using an OpenSSL config file, applications can explicitly initialize and register wolfProvider using the desired OSSL\_PROVIDER_\* APIs. As one example, initializing wolfProvider and registering for all algorithms could be done using:
```
    OSSL_PROVIDER *prov = NULL;
    const char *build = NULL;
    OSSL_PARAM request[] = {
        { "buildinfo", OSSL_PARAM_UTF8_PTR, &build, 0, 0 },
        { NULL, 0, NULL, 0, 0 }
    };

    prov = OSSL_PROVIDER_load(NULL, "libwolfprov");
    if (prov != NULL) {
        if (OSSL_PROVIDER_get_params(prov, request))
            printf("Provider 'libwolfprov' buildinfo: %s\n", build);
        else
            ERR_print_errors_fp(stderr);
        OSSL_PROVIDER_unload(prov);
    }
    else {
        ERR_print_errors_fp(stderr);
    }
```

Loading a provider makes it available; it does not by itself force every operation to use it. To require wolfProvider for a given operation, fetch algorithms with a property query using wolfProvider's registered property `"provider=wolfprov"` (for a FIPS build, `"provider=wolfprov,fips=yes"`) via the `EVP_*_fetch()` APIs. Note that `libwolfprov` is the module name used to *load* the provider, while `provider=wolfprov` is the property used to *select* its algorithms.

wolfProvider does not currently implement a provider self-test dispatch (`OSSL_FUNC_PROVIDER_SELF_TEST`). `OSSL_PROVIDER_self_test()` on the loaded provider therefore does not run a wolfProvider self-test and should not be relied on as one.

## Loading wolfProvider from an OpenSSL Configuration File

wolfProvider can be loaded from an OpenSSL config file if an application using OpenSSL is set up to process a config file. An example of how the wolfProvider library may be added to a config file is below.

```
openssl_conf = openssl_init

[openssl_init]
providers = provider_sect

[provider_sect]
libwolfprov = libwolfprov_sect

[libwolfprov_sect]
activate = 1
```

## wolfProvider Static Entrypoint

When wolfProvider is built as a static library, it is loaded as an OpenSSL built-in provider rather than as a dynamically loaded module. The application links against static wolfProvider (and its wolfSSL and OpenSSL dependencies), registers wolfProvider's entry point with `OSSL_PROVIDER_add_builtin()`, and then loads it by name:
```
#include <openssl/provider.h>
#include <wolfprovider/wp_wolfprov.h>

int load_wolfprovider(void)
{
    OSSL_PROVIDER *prov;

    if (OSSL_PROVIDER_add_builtin(NULL, "libwolfprov",
            wolfssl_provider_init) != 1) {
        return -1;
    }
    prov = OSSL_PROVIDER_load(NULL, "libwolfprov");
    if (prov == NULL) {
        return -1;
    }
    /* use prov, then call OSSL_PROVIDER_unload(prov) when finished */
    return 0;
}
```
The entry point `wolfssl_provider_init()` is declared in `wolfprovider/wp_wolfprov.h`; it is passed to `OSSL_PROVIDER_add_builtin()` and is not called directly by the application.

## Replace-Default Mode

wolfProvider can be built to *replace* OpenSSL's default provider rather than loading alongside it. In replace-default mode, OpenSSL requests for the built-in `default` and `fips` providers resolve to wolfProvider (the `libwolfprov` module), so applications use wolfSSL cryptography with no code or configuration changes. (The `legacy` provider is redirected only in static-legacy builds.) OpenSSL's `base` provider, which offers encoders, decoders, and related non-cryptographic services, still loads normally.

Enabling replace-default has two parts. The `--enable-replace-default` configure option (or defining `-DWOLFPROV_REPLACE_DEFAULT` in `CFLAGS`, useful for Yocto-style builds) builds wolfProvider's replacement default provider; it does not by itself patch or rebuild OpenSSL. Making wolfProvider the default also requires building OpenSSL with wolfProvider's `provider_predefined.c` replacement, so that OpenSSL's built-in `default`/`fips` entries load wolfProvider. The `scripts/build-wolfprovider.sh --replace-default` path performs both steps; see the wolfProvider Integration Guide (`docs/INTEGRATION_GUIDE.md`) for the authoritative procedure.

In replace-default mode no `OPENSSL_CONF` or `OPENSSL_MODULES` configuration is required, since wolfProvider is already the default provider. The `scripts/env-setup` helper detects this mode automatically and skips setting those variables.

Replace-default mode is useful for FIPS deployments: it removes the OpenSSL default-provider fallback path, reducing the risk that an application inadvertently uses non-FIPS algorithms. It does not by itself guarantee system-wide FIPS compliance, as explicitly loaded providers and direct low-level calls remain outside its control. See the FIPS 140-3 Support chapter and the wolfProvider FIPS Integration Guide for details.
