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

Existing repository metadata systems
====================================

Projects for lightweight  repository discovery and management
-------------------------------------------------------------

- 18F has ``.about.yml``
- Code for America has `civic.json <https://github.com/codeforamerica/brigade/blob/master/README-Project-Search.md>`__. These are used by Code for America's `project search page <https://www.codeforamerica.org/brigade/projects>`__. Primarily it is used to denote the status of a project and to provide tags for search. It seems that a lot of information for the search page is also automatically obtained from GitHub metadata, like the project description.

JSON-LD
-------

- `JSON-LD best practices <http://json-ld.org/spec/latest/json-ld-api-best-practices/>`__.
- `Buidling a better book in the browser <http://journal.code4lib.org/articles/10668>`__
- `Linked Data Patterns <http://patterns.dataincubator.org/book/index.html>`__
- `Indexing bibliographic linked data with JSON-LD, ElasticSearch <http://journal.code4lib.org/articles/7949>`__.
- `JSON-LD: Building meaningful data APIs <http://blog.codeship.com/json-ld-building-meaningful-data-apis/>`__.


Metadata schemas (including those built around JSON-LD)
-------------------------------------------------------

- Overview of software metadata schemes in `Issue #117 of White House open source policy <https://github.com/WhiteHouse/source-code-policy/issues/117>`__.
- https://github.com/codemeta/codemeta
- http://schema.org
- http://www.w3.org/TR/vocab-dcat/
- `Asset description metadata for software <https://joinup.ec.europa.eu/asset/adms_foss/home>`__ from the EU. See `Issue #41 at codemeta <https://github.com/codemeta/codemeta/issues/41>`__ as well.
- `BibJSON <http://okfnlabs.org/bibjson/>`__, which is compatible with JSON-LD.
