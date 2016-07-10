:tocdepth: 1

.. warning::

   This technote is a work in progress.

   The purpose of this document is to document research and design towards a metadata system that can be used to annotate and index all LSST code and documentation repositories.
   Such metadata would be consumed by the LSST DocHub search service, and be used by Zenodo, ADS, and other web services.

Use cases for metadata embedded in Git repositories
===================================================

- Drive content for static site generators (e.g., title, revision date, and author list).
- Metadata content for deposition in external archives/indices like NASA/SAO ADS, Zenodo and Figshare. It should be possible for a repo to always be immediately depositable when a new release or tag is made.
- Metadata for LSST DocHub to aid project discovery. The metadata will be ingested in ElasticSearch and accessible from a web API. This metadata will drive the summary display of projects in addition to providing search facets.
- Metadata for project administration (e.g., who is the owner of a project?)
- Analytics for LSST projects (how many people contribute to different projects, when was a project last updated?)