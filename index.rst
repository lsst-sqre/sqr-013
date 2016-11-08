:tocdepth: 1

.. warning::

   This technote is a work in progress.

DocHub's purpose
================

LSST produces a vast number of artifacts on a continual basis.
Software and associated documentation repositories are published through GitHub.
Documents and presentations may also be archived in DocuShare, but also on Zenodo to enable scientific citation.
Conversations about tasks may happen in JIRA, while more general conversations happen on the Community.lsst.org forum.
Papers may be written on GitHub, but are ultimately made available through ADS and a publisher's website.
Ultimately, finding information is more difficult than it should be.

One response is to pare down the number of places where LSST's information artifacts can exist.
While laudable --- and using platforms with identical use cases and feature sets simultaneously is certainly confusing --- retreating to a single platform is not a solution.
Each platform facilitates different types of work.
Arbitrarily forcing work onto platforms that aren't a good fit for that work reduces productivity.

Instead, our approach is to decouple information discovery from the platforms that information exists on.
We will index LSST information artifacts, and make their metadata centrally available through a website and search API.
Then, finding LSST information involves searching through a single website rather than iteratively trawling individual platforms.
This system for indexing LSST information artifacts, storing metadata in a database, and making metadata available through an API and website is called *DocHub*.
This technote describes LSST DocHub's design.

Existing metadata systems
=========================

Information discovery is common issue across large, distributed organizations like LSST.
These are some implementations of DocHub-like systems by other open software and data organizations:

- 18F embeds `.about.yml <https://github.com/18F/about_yml>`__ metadata files in their repositories. These are indexed and published on 18F's `Dashboard <https://18f.gsa.gov/dashboard>`__.
- Code for America has `civic.json <https://github.com/codeforamerica/brigade/blob/master/README-Project-Search.md>`__. These are used by Code for America's `project search page <https://www.codeforamerica.org/brigade/projects>`__. Primarily it is used to denote the status of a project and to provide tags for search. It seems that a lot of information for the search page is also automatically obtained from GitHub metadata, like the project description.
- Code.gov embeds a `code.json <https://code.gov/#/policy-guide/docs/compliance/inventory-code>`__ file in federal repositories. This metadata is tracked and published by the Code.gov website and API. Additional discussion about code.gov's metadata schema is taking place on `a GitHub issue <https://github.com/presidential-innovation-fellows/code-gov-web/issues/41>`__. Earlier discussion also happened on a White House `source-code-policy <https://github.com/WhiteHouse/source-code-policy/issues/117>` issue.
- `CodeMeta <https://github.com/codemeta/codemeta>`__ is minimal metadata schema, written in JSON-LD, that can describe scientific software repositories. CodeMeta's schema is designed to be cross-walked to other 
- `Asset description metadata for software <https://joinup.ec.europa.eu/asset/adms_foss/home>`__ from the EU. See `Issue #41 at codemeta <https://github.com/codemeta/codemeta/issues/41>`__ as well.
- `BibJSON <http://okfnlabs.org/bibjson/>`__ describes resources with JSON objects with fields defined in BibTeX. Being JSON, it's also possible to describe these files with JSON-LD.

These metadata systems share a common pattern: metadata is embedded with the information artifact, centrally indexed, and made available through a search API and website.
This approach scales well since it federates metadata definition and maintenance to the source repositories themselves.
DocHub builds upon this design pattern.

DocHub architecture
===================

DocHub uses a microservice architecture to gain flexibility.
The components are:

- **A metadata schema.** DocHub uses JSON-LD since it is extensible, yet self describing. Like code.gov and similar implementations, this metadata embedded in source repositories whenever possible.
  The same metadata format is used in the database.
- **A metadata database.** DocHub uses a MongoDB database to store all metadata. MongoDB is a document database that works natively with JSON. The JSON-LD that's embedded in source repositories is available through MongoDB.
- **A full-text database.** While MongoDB is well-suited querying semi-structured data like JSON-LD, its full-text search capabilities are more limited. Where possible, the **content** of documents will be stored and made available through Elasticsearch.
- **Ingest adapters.** Each adapter is a microservice built to transform content and metadata for a particular type of resource into a JSON-LD record and full-text entry stored in the MongoDB and Elasticsearch database. This adapter architecture helps DocHub scale; indexing a new arbitrary information source involves deploying a new adapter service. These adapter can either by pushed to (say, by a GitHub webhook), or can poll a platform for new and updated records. Each adapter handles the platform specific challenge of transforming either templated JSON-LD stored in a source repository or a platforms native metadata into standardized DocHub JSON-LD.
- **An API server.** The web API server allows applications to query against DocHub's metadata and full-text databases.
- **A web front end.** This front end is how people typically use DocHub. This website will allow users to browse and filter DocHub information artifacts, and also provide a generic search against the full-text and metadata databases. The website will be editorially designed to some extent. For example, the front page will show featured projects, papers and documents in addition to giving entry points to search and browse against usefully-selected categories. Generally the website (and API) will allow anonymous access. DocHub can be designed to facilitate authorization-based access to non-public documentation (private GitHub repositories) for example, though this will depend on a centralized user database that doesn't exist in the needed form yet.

DocHub's JSON-LD metadata
=========================

JSON-LD Reading List
--------------------

- `JSON-LD best practices <http://json-ld.org/spec/latest/json-ld-api-best-practices/>`__.
- `Buidling a better book in the browser <http://journal.code4lib.org/articles/10668>`__
- `Linked Data Patterns <http://patterns.dataincubator.org/book/index.html>`__
- `Indexing bibliographic linked data with JSON-LD, ElasticSearch <http://journal.code4lib.org/articles/7949>`__.
- `JSON-LD: Building meaningful data APIs <http://blog.codeship.com/json-ld-building-meaningful-data-apis/>`__.

Ingest Adapters
===============

Ingest adapters are microservices that take an artifact in its native form, and index it in the DocHub databases.
That is, it transforms the artifact's native metadata into DocHub JSON-LD metadata.
Each type of artifact has a dedicated ingest adapter microservice.
This way all platform-specific logic is contained within individual ingest adapter code bases.
The DocHub API server does not largely need to know about platforms; it only needs to interpret metadata in DocHub's schema.

Ingest adapters can either be designed for pulling artifact updates, or being pushed update's from the artifact's platform.
For example, GitHub repositories can emit webhook events that trigger ingest adapters.
Alternatively, ingest adapters can poll for updates from platforms that do not support webhooks.

Example: Sphinx Technote Adapter
--------------------------------

This section explores how adapters work through the example of DM's Sphinx technotes.
Technotes are GitHub repositories published through LSST the Docs.

This adapter is a web (HTTP) server.
It needs a public ingress, and should be in the same cluster (namely, Kubernetes cluster) as the MongoDB and Elasticsearch databases.

The adapter has a ``HTTP POST`` endpoint that receives a `GitHub webhook <https://developer.github.com/webhooks/>`_ that is configured directly in the technote's GitHub repository.
GitHub triggers webhooks for different events; the `PushEvent <https://developer.github.com/v3/activity/events/types/#pushevent>`_ is useful since it's triggered whenever the repository is updated with new content, regardless of the branch.
From the webhook ``POST``, the adapter receives a payload of information about the commits in the push, including:

- ``ref``: The Git ref that was pushed to (typically a branch name),
- ``head``: The SHA ref of the HEAD of the commits. For GitHub repositories, DocHub only tracks the head of each branch or a tag, not individual commits.
- ``commits``: an array of commit objects, including ``commits[][url]``, the API URL of each commit in the push.

From this commit information, the adapter begins to build a metadata record for the repository.
First, the adapter looks at the ``lsstmeta.json`` file in the repository.
Most likely, this is a templated JSON-LD file (TODO: link to previous section), which requires the adapter to run metadata interpolators to build a complete ``lsstmeta.json`` JSON-LD file.
To facilitate this, the adapter performs a shallow clone of the entire repository so that the adapter's interpolation pipeline can scrape metadata from the repository content (such as the document's title and abstract).
The adapter can also GitHub's API to query for structured information that GitHub has about the repository, such as committers to build authorship metadata, or parsed license information.
Once built, the adapter inserts the JSON-LD object in the resource's MongoDB document.

In addition, the adapter also extracts text from the technote's reStructedText and inserts that content into Elasticsearch.
