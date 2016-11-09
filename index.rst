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
- CodeMeta_ is a minimal metadata schema, written in JSON-LD, that can describe scientific software repositories. CodeMeta's schema is designed to be cross-walked to other 
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

All DocHub metadata records share a common JSON-LD (linked data) schema.
Through a ``@context``, JSON-LD documents map the names of fields to semantic definitions in http://schema.org and other vocabularies.
Specifically, DocHub adopts and extends the codemeta_, which is a minimal schema of concepts needed to describe a scientific software repository.
CodeMeta_ JSON-LD object can be cross-walked to other repository metadata schemas to enable automatic submission pipelines from GitHub to a repository like Zenodo, for example.

This JSON exists in two contexts:
DocHub metadata exists in two contexts: the metadata database, and in artifact repositories (such as GitHub repositories).
Metadata at rest in DocHub's database is intended to be complete and authoritative, while metadata embedded in repositories is *templated*.
Metdata templates are transformed by :ref:`ingest-adapters` into complete JSON-LD stored by DocHub.
This section describes these DocHub metadata in these two contexts.

JSON-LD in the metadata database
--------------------------------

DocHub's metadata database is MongoDB so that JSON-LD documents can be persisted and queried natively.
This design greatly simplifies the API server's design by returning documents in essentially the same form as they are stored.
MongoDB also obviates schema migrations.
By building upon JSON-LD and CodeMeta_, the API server is inherently backwards-compatible with any JSON-LD document, even metadata records with new fields not originally known by the API server.
As new types of fields are added to metadata records, the API server and front-end can evolve independently to provide new functionality based on this data.

Representing versioned resources in JSON-LD and the metadata database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

From a user's perspective, DocHub is a way to browse software and documentation projects, and see what versions are published on LSST the Docs.

CodeMeta_ JSON-LD is best suited for describing single versions of a project in individual JSON-LD metadata objects.
But software or documentation artifact (especially one backed by GitHub) is not a single version:

- There are multiple versions of the software and documentation (and its corresponding metadata) and individual branches and tags
- Multiple editions on LSST the Docs, corresponding to GitHub branches and tags.
- Zenodo depositions corresponding to tags.
- An ADS entry
- JIRA conversations
- Community.lsst.org conversations.

Although it could be possible to combine all of these resources and versions in a single MongoDB document, treating a MongoDB documents as a holistic description of a project, the schema for combining several JSON-LD resources in a MongoDB document would be ad-hoc.
Instead, DocHub maps MongoDB documents one-to-one with JSON-LD documents.

In this case, a JSON-LD and MongoDB document would refer to a single branch HEAD or tagged commit.

.. note::

   In this design, DocHub only tracks the HEAD of Git branches and tags. Individual commits aren't tracked. Tracking commits would enable interesting software provenance tracking, but this would also be a significant scope-creep for DocHub. Since LSST the Docs editions only track branches and editions, it makes sense for DocHub to also work at that level.

CodeMeta's ``relationships`` field enables one metadata document to refer to another.
For one JSON-LD document to refer to its parent Git repository:

.. code-block:: json

   {
     "@context": "...",
     "version": "master"
     "relationships": [
       {
         "relationshipType": "isPartOf",
         "relationshipType": "wasRevisionOf",
         "namespace": "http://www.w3.org/ns/prov#",
         "relatedIdentifier": "https://github.com/lsst-sqre/sqr-013.git",
         "relatedIdentifierType": "URL"
       }
     ]
   }

The ``wasRevisionOf`` relationship type is defined in PROV.
The PROV ontology includes other relationship types, though CodeMeta_ does not restrict ``relationships`` to use *only* PROV types.

Given this relationship, the MongoDB query for all JSON-LD records belonging to a GitHub project are:

.. code-block:: txt

   find({
     relationships: {$elemMatch: {relationshipType: "wasRevisionOf",
                                  relatedIdentifier: "https://github.com/lsst-sqre/sqr-013.git"}}
   })

It makes sense to use the metadata for the ``master`` branch as the 'main' record for a GitHub repository.
The ``master`` metadata is queried with:

.. code-block:: txt

   find({
     version: "master",
     relationships: {$elemMatch: {relationshipType: "wasRevisionOf",
                                  relatedIdentifier: "https://github.com/lsst-sqre/sqr-013.git"}}
   })

Relationships to projects
^^^^^^^^^^^^^^^^^^^^^^^^^

CodeMeta_\ ‘s ``relationships`` field can be used to make other associations, like associating a single GitHub repository to a larger stack.
For example, we want to associate Science Pipelines packages to Science Pipelines itself.

For this, we'd use a `isPartOf` relationship:

.. code-block:: json

   {
     "@context": "...",
     "version": "master"
     "relationships": [
       {
         "relationshipType": "isPartOf",
         "relatedIdentifier": "https://github.com/lsst/pipelines_docs.git",
         "relatedIdentifierType": "URL"
       }
     ]
   }

The difficulty
Choosing a ``relatedIdentifier`` is an unsolved problem.
In this example, the metadata record is declared as a part of the ``pipelines_docs`` GitHub repo, since ``pipelines_docs`` 'represents' the LSST Science Pipelines.

Alternatively, it might be useful to create JSON-LD metadata records corresponding to a product or product, such as ``lsst_apps``.

.. note::

   `isPartOf <https://schema.org/isPartOf>`_ is a schema.org term.

Embedded metadata templates
---------------------------

Although complete JSON-LD metadata documents can be embedded in GitHub (and similar) repositories, managing metadata this way may not be sustainable.
First, some metadata changes with each commit, and the time of commit (such ``dateModified``).
Second, a lot of metadata is inherent to a repository and its content.
Git commit trees contain information to build contributor metadata, the ``LICENSE`` file authoritatively defines the repository's license, and the document's text authoritatively describes its content.
Repeating information inherent to the GitHub repository in a metadata file introduces fragility.

DocHub's approach is to shift the responsibility of building a complete metadata record to the :ref:`ingest adapter <ingest-adapter>`.
To help the ingest adapter, and to store metadata that *can* be statically managed, we store *metadata templates* in the Git repository.

For example, consider the ``licenseId`` field in a DocHub JSON-LD metadata object:

.. code-block:: json

   {
     "@context": "...",
     "licenseId": "MIT"
   }

Instead of hard-coding the license's `SPDX Id <https://spdx.org/license-list>`__, we can direct the adapter to interpolate a metadata template to include license information from the GitHub API:

.. code-block:: json

   {
     "@context": "...",
     "licenseId": {"@template": "licenseId"}
   }

A field, such as ``licenseId`` whose value is an object with a key named ``@template`` triggers the metadata interpolator.
The value of the ``@template`` is the name of a metadata interpolator (which maps to a specific API in the DocHub metadata interpolation library).

The interpolation object may contain additional fields that act as arguments to the interpolation function.
For example, The ``gitAgents`` interpolator can take additional agents who aren't reflected in the Git history:

.. code-block:: json

   {
     "@context": "...",
     "agents": {"@template": "gitAgents",
                "additionalAgents": [
                  {
                    "@type": "organization",
                    "name": "Science Quality and Reliability Engineering Team",
                    "parentOrganization": "Data Management",
                    "isRightHolder": false,
                    "isMaintainer": true
                  }
                ]
   }

These additional agents can be organizations (shown in this example), or additional authors that aren't Git contributors.

.. _ingest-adapters:

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

Appendix: JSON-LD reading list
==============================

- `JSON-LD best practices <http://json-ld.org/spec/latest/json-ld-api-best-practices/>`__.
- `Building a better book in the browser <http://journal.code4lib.org/articles/10668>`__.
- `Linked Data Patterns <http://patterns.dataincubator.org/book/index.html>`__
- `Indexing bibliographic linked data with JSON-LD, ElasticSearch <http://journal.code4lib.org/articles/7949>`__.
- `JSON-LD: Building meaningful data APIs <http://blog.codeship.com/json-ld-building-meaningful-data-apis/>`__.

.. CodeMeta: https://github.com/codemeta/codemeta
