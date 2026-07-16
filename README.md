
# DSpace — Nepal Open University Thesis Repository (backend)

This is a customized fork of [DSpace](https://github.com/DSpace/DSpace) 9.2, deployed as the
backend REST API for the **Nepal Open University Thesis Repository**. It is paired with a
matching [frontend fork](https://github.com/Supriya090/nou-dspace-frontend).

**If you are deploying, upgrading, or troubleshooting this repository, start with
[HANDOVER.md](HANDOVER.md), not this file.** It has the actual deployment steps, the config
values you must change before anything will work, and the warnings you need to know about.

## What was customized

All customizations live on the `local-customizations` branch (this is the branch you want, not
`main`), on top of the unmodified `dspace-9.2` tag. On the backend, that's a small set of changes:
Docker network/subnet settings and the `dspace.server.url`/`dspace.ui.url`/`dspace.name` values for
this deployment, and ImageMagick and Ghostscript added to the Docker image (required for thumbnail
generation). The much larger set of customizations — branding, a custom view-only PDF viewer, and
page text — live in the [frontend repo](https://github.com/Supriya090/nou-dspace-frontend); see its
`CUSTOMIZATIONS.md` for the full list.

## Upstream DSpace documentation

For general DSpace questions not specific to this deployment — how DSpace itself works, the REST
API contract, community support — the upstream project's resources still apply:

* [DSpace Documentation Wiki](https://wiki.lyrasis.org/display/DSDOC/)
* [DSpace REST Contract](https://github.com/DSpace/RestContract)
* [DSpace community support](https://wiki.lyrasis.org/display/DSPACE/Support)
* [Upstream DSpace repository](https://github.com/DSpace/DSpace)

Upstream's own installation instructions (Maven build, Tomcat, etc.) do **not** apply to this
deployment, which runs via Docker Compose — see [HANDOVER.md](HANDOVER.md) instead.

## License

DSpace source code is freely available under a standard [BSD 3-Clause license](https://opensource.org/licenses/BSD-3-Clause).
The full license is available in the [LICENSE](LICENSE) file or online at http://www.dspace.org/license/

DSpace uses third-party libraries which may be distributed under different licenses. Those licenses
are listed in the [LICENSES_THIRD_PARTY](LICENSES_THIRD_PARTY) file.
