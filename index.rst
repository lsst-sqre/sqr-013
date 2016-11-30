:tocdepth: 1

.. sectnum::

Abstract
========

LSST DocHub is a proposed solution to information discovery for LSST Data Management, and the LSST project in general.
LSST documentation and information artifacts are published through a variety of platforms by virtue of the way information is created --- from documents archived on DocuShare, to source code on GitHub, to conversations on `community.lsst.org <https://community.lsst.org>`_.
Currently, staff and users must go to each platform to find information.
This has an overall effect of slowing, and even preventing, knowledge sharing.
**LSST DocHub can solve this problem by decoupling information publication from information discovery.**
DocHub consists of a unified web front-end for documentation browsing, filtering, and search.
The front-end is fed by a web API to centralized metadata and full-text databases.
These databases are populated by *adapters* that monitor each of LSST's information platforms for new and updated artifacts.
DocHub stores metadata as JSON-LD_, which is a community-standard, extensible, and self-describing schema.
This technote establishes the basic design concept for DocHub, including its architecture and JSON-LD metadata patterns.

.. note::

   LSST DocHub, as described in this technote, is a work in progress. Details may change, and many design decisions need to be made.


DocHub's purpose
================

LSST continuously produces a vast number of information artifacts across numerous platforms.
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

- 18F embeds `.about.yml <https://github.com/18F/about_yml>`__ metadata files in their repositories. These metadata are indexed and published on 18F's `Dashboard <https://18f.gsa.gov/dashboard>`__.
- Code for America similarly implements a `civic.json <https://github.com/codeforamerica/brigade/blob/master/README-Project-Search.md>`__ metadata format. These are used by Code for America's `project search page <https://www.codeforamerica.org/brigade/projects>`__. Primarily it is used to denote the status of a project and to provide tags for search. Much of the information for the search page is also automatically obtained from GitHub metadata, like the project description.
- Code.gov embeds a `code.json <https://code.gov/#/policy-guide/docs/compliance/inventory-code>`__ file in federal repositories. This metadata is tracked and published by the Code.gov website and API. Additional discussion about code.gov's metadata schema is taking place on `a GitHub issue <https://github.com/presidential-innovation-fellows/code-gov-web/issues/41>`__. Earlier discussion also happened on a White House `source-code-policy <https://github.com/WhiteHouse/source-code-policy/issues/117>`_ issue.
- CodeMeta_ is a minimal metadata schema, written in JSON-LD_, that can describe scientific software repositories. CodeMeta's schema is designed to be cross-walked `to other metadata schemas <https://github.com/codemeta/codemeta/blob/master/crosswalk.csv>`_ including Dublin Core, schema.org, Python's PyPI, and DataCite.
- `Asset description metadata for software <https://joinup.ec.europa.eu/asset/adms_foss/home>`__ from the EU. See `Issue #41 at codemeta <https://github.com/codemeta/codemeta/issues/41>`__ as well.

These metadata systems share a common pattern: metadata is embedded with the information artifact, centrally indexed, and made available through a search API and website.
This approach scales well since it federates metadata definition and maintenance to the source repositories themselves.
DocHub builds upon this design pattern.

.. _arch:

DocHub architecture
===================

DocHub uses a microservice architecture (:numref:`fig-dochub-arch`) to gain flexibility.
The components are:

- **A metadata schema.** DocHub uses JSON-LD_ since it is extensible, yet self describing. Like code.gov and similar implementations, this metadata embedded in source repositories whenever possible.
  The same metadata format is used in the database.
- **A metadata database.** DocHub uses a MongoDB_ database to store all metadata. MongoDB_ is a document database that works natively with JSON. The JSON-LD_ that's embedded in source repositories is available through MongoDB.
- **A full-text database.** While MongoDB_ is well-suited to querying semi-structured data like JSON-LD_, its full-text search capabilities are more limited. Where possible, the **content** of documents will be stored and made available through Elasticsearch_.
- **Ingest adapters.** Each adapter is a microservice built to transform content and metadata for a particular type of resource into a JSON-LD_ record and full-text entry stored in the MongoDB_ and Elasticsearch databases. This adapter architecture helps DocHub scale: indexing a new arbitrary information source involves deploying a new adapter service. Adapters can either by pushed to (say, by a GitHub webhook), or can poll a platform for new and updated records. Each adapter handles the platform specific challenge of transforming either :ref:`templated JSON-LD <metadata-templating>` stored in a source repository or a platform's native metadata into standardized DocHub JSON-LD_.
- **An API server.** The web API server allows applications to query against DocHub's metadata and full-text databases.
- **A web front end.** This front end is how people typically use DocHub. This website will allow users to browse and filter DocHub information artifacts, and also provide a generic search against the full-text and metadata databases. The website will be editorially designed to some extent. For example, the front page will show featured projects, papers and documents in addition to giving entry points to search and browse against usefully-selected categories. Generally the website (and API) will allow anonymous access. DocHub can be designed to facilitate authorization-based access to non-public documentation (private GitHub repositories) for example, though this will depend on a centralized user database that doesn't exist in the needed form yet.

.. figure:: /_static/dochub_arch.svg
   :name: fig-dochub-arch
   :alt: LSST DocHub implementation architecture. See :ref:`arch`.

   **DocHub's architecture.**
   Adapter microservices pull metadata and content from information artifacts, which could be GitHub repositories, JIRA issues, forum topics, ADS entries.
   The adapters build JSON-LD metadata documents and persist them into a MongoDB metadata database.
   An Elasticsearch cluster stores full text content from the adapters, where possible.
   The API server provides GraphQL and RESTful interfaces to the MongoDB and Elasticsearch databases.

DocHub's JSON-LD metadata
=========================

All DocHub metadata records share a common JSON-LD_ (linked data) schema.
Through a ``@context``, JSON-LD_ documents map the names of fields to semantic definitions in http://schema.org and other vocabularies.
Specifically, DocHub adopts and extends codemeta_, which is a minimal schema of concepts needed to describe a scientific software repository.
CodeMeta_ JSON-LD_ object can be cross-walked to other repository metadata schemas to enable automatic submission pipelines from GitHub to a repository like Zenodo, for example.

DocHub metadata exists in two contexts: the metadata database, and in artifact repositories (such as GitHub repositories).
Metadata at rest in DocHub's database is intended to be complete and authoritative, while metadata embedded in repositories is *templated*.
Metadata templates are transformed by :ref:`ingest adapters <ingest-adapters>` into complete JSON-LD_ stored by DocHub.
This section describes these DocHub metadata as it is authoritatively stored in the metadata database.

**See also:** :ref:`json-ld-reading-list`.

JSON-LD in MongoDB
------------------

DocHub's metadata database is MongoDB_ so that JSON-LD_ documents can be persisted and queried natively.
This design greatly simplifies the RESTful API server by allowing it to return documents in essentially the same form as they are stored.

MongoDB_ also obviates schema migrations.
By building upon JSON-LD_ and CodeMeta_, the API server is inherently backwards-compatible with any JSON-LD_ document, even metadata records with new fields not originally known by the API server.
As new types of fields are added to metadata records, the API server and front-end can evolve independently to provide new functionality based on this data.

.. todo::

   How are collections structured?
   One collection per data class?
   Or, one collection for everything?

JSON-LD Applications
--------------------

This section explores how different types of metadata can be encoded in CodeMeta_ JSON-LD (and DocHub's extension of it):

- :ref:`json-ld-versioned-resources`.
- :ref:`json-ld-related-identifiers`.
- :ref:`json-ld-projects`.
- :ref:`json-ld-people`.
- :ref:`json-ld-orgs`.
- :ref:`json-ld-org-hierarchy`.
- :ref:`json-ld-non-software-types`.
- :ref:`json-ld-publications`.

.. _json-ld-versioned-resources:

Representing versioned resources in JSON-LD and the metadata database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

From a user's perspective, DocHub is a way to browse software and documentation projects, and see what versions are published on LSST the Docs.

CodeMeta_ JSON-LD_ is best suited for describing single versions of a project in individual JSON-LD_ metadata objects.
But a software or documentation artifact (especially one backed by GitHub) is not a single version:

- There are multiple versions of the software and documentation (and its corresponding metadata) and individual branches and tags
- Multiple editions on LSST the Docs, corresponding to GitHub branches and tags.
- Zenodo depositions corresponding to tags.
- An ADS entry
- JIRA conversations
- Community.lsst.org conversations.

Although it could be possible to combine all of these resources and versions in a single MongoDB_ document, treating a MongoDB_ documents as a holistic description of a project, the schema for combining several JSON-LD_ resources in a MongoDB_ document would be ad-hoc.
Instead, DocHub maps MongoDB_ documents one-to-one with JSON-LD_ documents.

In this case, a JSON-LD_ and MongoDB_ document would refer to a single branch HEAD or tagged commit.

.. note::

   In this design, DocHub only tracks the HEAD of Git branches and tags. Individual commits aren't tracked. Tracking commits would enable interesting software provenance tracking, but this would also be a significant scope-creep for DocHub. Since LSST the Docs editions only track branches and editions, it makes sense for DocHub to also work at that level.

CodeMeta's ``relationships`` field enables one metadata document to refer to another.
For one JSON-LD_ document to refer to its parent Git repository:

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

Given this relationship, the MongoDB_ query for all JSON-LD_ records belonging to a GitHub project are:

.. code-block:: text

   find({
     relationships: {$elemMatch: {relationshipType: "wasRevisionOf",
                                  relatedIdentifier: "https://github.com/lsst-sqre/sqr-013.git"}}
   })

It makes sense to use the metadata for the ``master`` branch as the 'main' record for a GitHub repository.
The ``master`` metadata is queried with:

.. code-block:: text

   find({
     version: "master",
     relationships: {$elemMatch: {relationshipType: "wasRevisionOf",
                                  relatedIdentifier: "https://github.com/lsst-sqre/sqr-013.git"}}
   })

.. _json-ld-related-identifiers:

Related identifiers in ADS and (DOIs)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

CodeMeta_\ ‘s ``relationships`` field can be used to make other associations, like associating a single GitHub repository to a larger project.
For example, a GitHub repository might also be archived on Zenodo, and have a DOI.

.. code-block:: json

   {
     "@context": "...",
     "version": "v1",
     "relationships": [
       {
         "relationshipType": "compiles",
         "relatedIdentifier": "doi:10.5281/zenodo.153867",
         "relatedIdentifierType": "DOI"
       }
     ]
   }

This example shows that the ``v1`` tag of this software repository was compiled into the Zenodo archived entity.

The `Zenodo deposition resource documentation <https://zenodo.org/dev#restapi-rep>`_ describes possible ``relationshipType``\ s.

- isCitedBy
- cites
- isSupplementTo
- isSupplementedBy
- isNewVersionOf
- isPreviousVersionOf
- isPartOf
- hasPart
- compiles
- isCompiledBy
- isIdenticalTo
- isAlternateIdentifier

``relatedIdentifiers`` supported by Zenodo are:

- DOI
- Handle
- ARK
- PURL
- ISSN
- ISBN
- PubMed ID
- PubMed Central ID
- ADS Bibliographic Code
- arXiv
- Life Science Identifiers (LSID)
- EAN-13
- ISTC
- URNs and URLs

.. _json-ld-projects:

Relationships to projects
^^^^^^^^^^^^^^^^^^^^^^^^^

``relationships`` can support linking an artifact to larger multi-repository projects.
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

In this example, the metadata record is declared as a part of the ``pipelines_docs`` GitHub repo, since ``pipelines_docs`` 'represents' the LSST Science Pipelines.
(See below for additional relationship types).

Alternatively, it might be useful to create JSON-LD_ metadata records corresponding to a product or product, such as ``lsst_apps``.

.. note::

   `isPartOf <https://schema.org/isPartOf>`_ is a schema.org term. It is also in the Zenodo relationship vocabulary.

.. _json-ld-people:

Representing people in JSON-LD
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In CodeMeta_ JSON-LD_, authors are specified in an ``agents`` field.
For example:

.. code-block:: json

   {
      "@context": "...",
      "agents": [
        {
          "@id": "https://orcid.org/0000-0003-3001-676X",
          "@type": "person",
          "email": "jsick@lsst.org",
          "name": "Jonathan Sick",
          "affiliation": "AURA/LSST",
          "mustbeCited": true,
          "isMaintainer": true,
          "isRightsHolder": false,
        }
      ]
   }

Note that the ``@id`` field is an ORCiD.
From a linked-data perspective, adopting ORCiDs as identifiers for people allows us to leverage other data sources, including journals and ADS, more effectively.

ORCiD is not currently required by LSST.
An alternative to ORCiD is to treat metadata records served through DocHub's RESTful API as authoritative records.
The DocHub URL for a person's record becomes their ``@id``.

.. _json-ld-orgs:

Representing organizations and copyright holders in JSON-LD
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In addition to authors, ``agents`` can indicate the involvement of organizations, and even indicate what organizations hold copyright:

.. code-block:: json

   {
      "@context": "...",
      "agents": [
        {
          "@type": "organization",
          "name": "Association of Universities for Research in Astronomy",
          "isRightsHolder": true,
          "isMaintainer": false,
          "role": {
            "namespace": "http://www.ngdc.noaa.gov/metadata/published/xsd/schema/resources/Codelist/gmxCodelists.xml#CI_RoleCode",
            "roleCode": "rightsHolder"
          }
         },
      ]
   }

The ``role`` field provides detailed information about the role an agent plays.

.. note::

   In CodeMeta_, examples show the role as ``copyrightHolder``, however the namespace has a ``rightHolder`` instead.

Other roles are:

- ``resourceProvider``: party that supplies the resource.
- ``custodian``: party that accepts accountability and responsibility for the data and ensures appropriate care and maintenance of the resource.
- ``owner``: party that owns the resource.
- ``sponsor``: party that sponsors the resource.
- ``user``: party who uses the resource.
- ``distributor``: party who distributes the resource.
- ``originator``: party who created the resource.
- ``pointOfContact``: party who can be contacted for acquiring knowledge about or acquisition of the resource.
- ``principleInvestorigator``: key party responsible for gathering information and conducting research.
- ``processor``: party who has processed the data in a manner such that the resource has been modified.
- ``publisher``: party who published the resource.
- ``author``: party who authored the resource.
- ``coAuthor``: party who jointly authors the resource.
- ``collaborator``: party who assists with the generation of the resource other than the principal investigator.
- ``editor``: party who reviewed or modified the resource to improve the content.
- ``mediator``: a class of entity that mediates access to the resource and for whom the resource is intended or useful.
- ``rightsHolder``: party owning or managing rights over the resource.
- ``contributor``: party contributing to the resource.
- ``funder``: party providing monetary support for the resource.
- ``stakeholder``: party who has an interest in the resource or the use of the resource.

.. seealso::

   `The codelist schema documentation <http://www.ngdc.noaa.gov/metadata/published/xsd/schema/resources/Codelist/gmxCodelists.xml#CI_RoleCode>`_ authoritatively describes these roles.

.. _json-ld-org-hierarchy:

Describing organizational hierarchy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

One search pattern for DocHub, especially by LSST staff, is to browse artifacts by the organization that made them (LSST subsystems, and teams).
The ``subOrganization`` type and ``parentOrganization`` build an organizational hierarchy:

.. code-block:: json

   {
      "@context": "...",
      "agents": [
        {
          "@type": "organization",
          "name": "Association of Universities for Research in Astronomy",
          "isRightsHolder": true,
          "isMaintainer": false,
          "role": {
            "namespace": "http://www.ngdc.noaa.gov/metadata/published/xsd/schema/resources/Codelist/gmxCodelists.xml#CI_RoleCode",
            "roleCode": "rightsHolder"
          }
         },
         {
           "@type": "organization",
           "name": "Large Synoptic Survey Telescope",
           "parentOrganization": "Association of Universities for Research in Astronomy",
           "isRightHolder": false,
           "isMaintainer": false
         },
         {
           "@type": "organization",
           "name": "Data Management",
           "parentOrganization": "Large Synoptic Survey Telescope",
           "isRightHolder": false,
           "isMaintainer": false
         },
         {
           "@type": "organization",
           "name": "Science Quality and Reliability Engineering Team",
           "parentOrganization": "Data Management",
           "isRightHolder": false,
           "isMaintainer": true
         }

      ]
   }

.. _json-ld-non-software-types:

Types for non-software artifacts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

CodeMeta_ JSON-LD was designed to designed to represent software projects, see the ``@type``:

.. code-block:: json

   {
     "@context":"https://raw.githubusercontent.com/codemeta/codemeta/master/codemeta.jsonld",
     "@type": "SoftwareSourceCode",
   }

schema.org types
""""""""""""""""

``SoftwareSourceCode`` is a schema.org_ ``@type``: http://schema.org/SoftwareSourceCode.
`SoftwareSourceCode`_ is derives from a schema.org_ CreativeWork_.

Some other derived types from schema.org_ that may be useful are:

- `ScholarlyArticle <http://schema.org/ScholarlyArticle>`_ for peer-reviewed articles.
- `Conversation <http://schema.org/Conversation>`_, for forum topics or GitHub issue threads.
- `SocialMediaPosting <http://schema.org/SocialMediaPosting>`_, for tweets.

**See also:** :ref:`json-ld-publications`.

Zenodo types
""""""""""""

These are artifact types defined by the `Zenodo deposition schema <https://zenodo.org/dev#restapi-rep-meta>`_:

- ``publication``: Publication, with ``publication_type``:

  - ``book``: Book
  - ``section``: Book section
  - ``conferencepaper``: Conference paper
  - ``article``: Journal article
  - ``patent``: Patent
  - ``preprint``: Preprint
  - ``report``: Report
  - ``softwaredocumentation``: Software documentation
  - ``thesis``: Thesis
  - ``technicalnote``: Technical note
  - ``workingpaper``: Working paper
  - ``other``: Other

- ``poster``: Poster
- ``presentation``: Presentation
- ``dataset``: Dataset
- ``image``: Image, with ``image_type``:

  - ``figure``: Figure
  - ``plot``: Plot
  - ``drawing``: Drawing
  - ``diagram``: Diagram
  - ``photo``: Photo
  - ``other``: Other

- ``video``: Video/Audio
- ``software``: Software

In a JSON-LD_ sense, DocHub will use schema.org_ types, but should be capable of cross-walking metadata to and from these Zenodo types.

.. _json-ld-publications:

Representation of publications
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

schema.org_ has full support for describing scholarly articles using JSON-LD_:

This is Example 2 from schema.org's ScholarlyArticle_ documentation:

.. code-block:: json

   {
     "@context": "http://schema.org", 
     "@graph": [
       {
           "@id": "#issue", 
           "@type": "PublicationIssue", 
           "issueNumber": "5", 
           "datePublished": "2012", 
           "isPartOf": {
               "@id": "#periodical", 
               "@type": [
                   "PublicationVolume", 
                   "Periodical"
               ], 
               "name": "Cataloging & Classification Quarterly", 
               "issn": [
                   "1544-4554", 
                   "0163-9374"
               ], 
               "volumeNumber": "50", 
               "publisher": "Taylor & Francis Group"
           }
       }, 
       {
           "@type": "ScholarlyArticle", 
           "isPartOf": "#issue", 
           "description": "The library catalog as a catalog of works was an infectious idea, which together with research led to reconceptualization in the form of the FRBR conceptual model. Two categories of lacunae emerge--the expression entity, and gaps in the model such as aggregates and dynamic documents. Evidence needed to extend the FRBR model is available in contemporary research on instantiation. The challenge for the bibliographic community is to begin to think of FRBR as a form of knowledge organization system, adding a final dimension to classification. The articles in the present special issue offer a compendium of the promise of the FRBR model.", 
           "sameAs": "http://dx.doi.org/10.1080/01639374.2012.682254", 
           "about": [
               "Works", 
               "Catalog"
           ], 
           "pageEnd": "368", 
           "pageStart": "360", 
           "name": "Be Careful What You Wish For: FRBR, Some Lacunae, A Review", 
           "author": "Smiraglia, Richard P."
       }
     ]
   }

And Example 3 from ScholarlyArticle_:

.. code-block:: json

   {
     "@context": "http://schema.org", 
     "@graph": [
       {
         "@id": "#issue4",
         "@type": "PublicationIssue",
         "datePublished": "2006-10",
         "issueNumber": "4"
       },
       {
         "@id": "#volume50",
         "@type": "PublicationVolume",
         "volumeNumber": "50"
       },
       {
         "@id": "#periodical",
         "@type": "Periodical",
         "name": "Library Resources and Technical Services"
       },
       {
         "@id": "#article",
         "@type": "ScholarlyArticle",
         "author": "Carlyle, Allyson.",
         "isPartOf": [
           {
             "@id": "#periodical"
           },
           {
             "@id": "#volume50"
           },
           {
             "@id": "#issue4"
           }
         ],
         "name": "Understanding FRBR as a Conceptual Model: FRBR and the Bibliographic Universe",
         "pageEnd": "273",
         "pageStart": "264"
       }
     ]
   }

**Example 3** establishes bibliographic information with a ``@graph`` containing PublicationIssue_, PublicationVolume_, and Periodical_ objects.
These three objects are connected to the publication with ``isPartOf``, however there's no explicit relationship between the issue, volume and periodical.

Alternatively, **Example 2** has two objects in its ``@graph``: a PublicationIssue_ (that includes PublicationVolume_ and Periodical_ metadata in its type), and a ScholarlyArticle_.
The ScholarlyArticle_ links to PublicationIssue_ through an ``isPartOf`` relationship.
Thus **Example 2** establishes a complete semantic relationship between the article, issue, volume and periodical.
**Example 2** is preferred.

The schema.org approach is slightly different from CodeMeta_ since it encapsulates several simultaneous relations in a ``relationships`` array.
This is ideal since it allows us to connect a paper not only to its journal context, but also to associated source code and datasets.

Another difference is that DocHub JSON-LD_ does not tend to use ``@graph``\ s; instead one resource is mapped to a MongoDB_ document.
This is one possible approach to using ``relationships`` and folding Journal information into the relationship type:

.. code-block:: json

   {
     "@context": "...",
     "@type": "ScholarlyArticle",
     "relationships": [
       {
         "relationshipType": "isSupplementTo",
         "relatedIdentifier": "https://github.com/lsst/example_analysis_software.git",
         "relatedIdentifierType": "URL"
       },
       {
         "relationshipType" "isPartOf",
         "@id": "#issue", 
         "@type": [
             "PublicationVolume", 
             "Periodical",
             "PublicationIssue"
         ], 
         "name": "Cataloging & Classification Quarterly", 
         "volumeNumber": "50", 
         "issueNumber": "5",
         "publisher": "Taylor & Francis Group"
         "pageEnd": "368",
         "partStart": "360",
       },
       {
         "relationshipType": "isIdenticalTo",
         "relatedIdentifier": "doi:...",
         "relatedIdentifierType": "DOI"
       },
     ],
     "name": "Article's Name",
     "description": "Article's abstract ..."
   }

.. _metadata-templating:

JSON-LD metadata templates
==========================

Although complete JSON-LD_ metadata documents can be embedded in GitHub (and similar) repositories, managing metadata this way may not be sustainable.
First, some metadata changes with each commit, and the time of commit (such ``dateModified``).
Second, a lot of metadata is inherent to a repository and its content.
Git commit trees contain information to build contributor metadata, the ``LICENSE`` file authoritatively defines the repository's license, and the document's text authoritatively describes its content.
Repeating information inherent to the GitHub repository in a metadata file introduces fragility.

DocHub's approach is to shift the responsibility of building a complete metadata record to the :ref:`ingest adapter <ingest-adapters>`.
To help the ingest adapter, and to store metadata that *can* be statically managed, we store *metadata templates* in the Git repository.

Interpolation objects
---------------------

For example, consider the ``licenseId`` field in a DocHub JSON-LD_ metadata object:

.. code-block:: json

   {
     "@context": "...",
     "licenseId": "MIT"
   }

Instead of hard-coding the license's `SPDX Id <https://spdx.org/license-list>`__, we can direct the adapter to interpolate a metadata template to include license information from the GitHub API:

.. code-block:: json

   {
     "@context": "...",
     "licenseId": {"@template": "GitHubLicenseId"}
   }

An object with ``@template`` field is an *interpolation object*.
The value of ``@template`` is the name of a metadata interpolator known to the :ref:`ingest adapter <ingest-adapters>`.

The interpolation object may contain additional fields that act as arguments to the interpolation function.
For example, The ``GitContributors`` interpolator can take additional agents who aren't reflected in a Git repos's history:

.. code-block:: json

   {
     "@context": "...",
     "agents": {"@template": "GitContributors",
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
That is, it transforms the artifact's native metadata into DocHub JSON-LD_ metadata.
Each type of artifact has a dedicated ingest adapter microservice.
This way all platform-specific logic is contained within individual ingest adapter code bases.
The DocHub API server does not largely need to know about platforms; it only needs to interpret metadata in DocHub's schema.

Ingest adapters can either be designed for pulling artifact updates, or being pushed update's from the artifact's platform.
For example, GitHub repositories can emit webhook events that trigger ingest adapters.
Alternatively, ingest adapters can poll for updates from platforms that do not support webhooks.

Kubernetes deployment pattern
-----------------------------

Since DocHub is deployed with Kubernetes, adapters are expected to be deployed as Kubernetes pods in the same cluster as the API server and databases.

Adapters that recieve HTTP POST requests from webhooks are configured with Kubernetes ingress resources, which gives them an external IP.

Being in the same cluster, the adapters can directly connect with the MongoDB_ and Elasticsearch instances, which removes any need for an intermediate API layer.
This arrangement does require that adapters are trusted.
Every adapter will need to be managed by DocHub's DevOps team.

Example: Sphinx Technote Adapter
--------------------------------

This section explores how adapters work through the example of DM's Sphinx technotes.
Technotes are GitHub repositories published through LSST the Docs.

This adapter is a web (HTTP) server.
It needs a public ingress, and should be in the same cluster (namely, Kubernetes cluster) as the MongoDB_ and Elasticsearch databases.

The adapter has a ``HTTP POST`` endpoint that receives a `GitHub webhook <https://developer.github.com/webhooks/>`_ that is configured directly in the technote's GitHub repository.
GitHub triggers webhooks for different events; the `PushEvent <https://developer.github.com/v3/activity/events/types/#pushevent>`_ is useful since it's triggered whenever the repository is updated with new content, regardless of the branch.
From the webhook ``POST``, the adapter receives a payload of information about the commits in the push, including:

- ``ref``: The Git ref that was pushed to (typically a branch name),
- ``head``: The SHA ref of the HEAD of the commits. For GitHub repositories, DocHub only tracks the head of each branch or a tag, not individual commits.
- ``commits``: an array of commit objects, including ``commits[][url]``, the API URL of each commit in the push.

From this commit information, the adapter begins to build a metadata record for the repository.
First, the adapter looks at the ``lsstmeta.json`` file in the repository.
Most likely, this is a :ref:`templated JSON-LD file <metadata-templating>`, which requires the adapter to run metadata interpolators to build a complete ``lsstmeta.json`` JSON-LD_ file.
To facilitate this, the adapter performs a shallow clone of the entire repository so that the adapter's interpolation pipeline can scrape metadata from the repository content (such as the document's title and abstract).
The adapter can also GitHub's API to query for structured information that GitHub has about the repository, such as committers to build authorship metadata, or parsed license information.
Once built, the adapter inserts the JSON-LD_ object in the resource's MongoDB_ document.

In addition, the adapter also extracts text from the technote's reStructedText and inserts that content into Elasticsearch.

DocHub API server
=================

Authentication and authorization
--------------------------------

DocHub's API will require auth infrastructure:

- Some resources will be embargoed (particularly, draft papers in private GitHub repositories) and classified (for example, access-controlled documents in DocuShare).
- Some fields *within* resources may be access controlled. For example, there may be a desire to make email addresses in records of people available only to authenticated project and science collaboration users.

LSST does not currently have a general purpose authentication system and user database capable of supporting authorization tasks.
There are some work-arounds for this:

- Permit DocHub to *only* index public information. The *metadata* of a classified DocuShare document may be considered public and indexed, but the *content* would not be indexed by Elasticsearch. In this case, the metadata adapters are required to enforce data classification.
- Use GitHub. GitHub OAuth would authenticate users and GitHub's permissions model would be used for authorization. That is, only those who can see a GitHub repository would be able to view it on DocHub. One problem here is that not everyone is LSST is on GitHub. Second, access controls on DocuShare do not map to GitHub organizations.
- Use Slack. This is a tenable authentication solution since everyone in the project and science collaborations have (or can have) an https://lsstc.slack.com Slack account, making `Slack-based OAuth authentication <https://api.slack.com/docs/sign-in-with-slack>`_  possible. The https://slack.com/api/users.identity endpoint can include information about a user's Slack team memberships. This could be a convenient way of establishing authorization.

In the long term, an ideal solution would be to have a central LSST and community user database.
That database provide university user authentication.
It would also be the best place to establish groups that define permissions.
Indeed, DocuShare, GitHub, Slack permissions and groups ought to be derived from this central database.

In the near term, we can launch DocHub as a completely open system, though a system for checking authorizations should be anticipated in the original design.

RESTful API
-----------

DocHub API server will provide a basic RESTful API to access JSON-LD_ documents:

.. code-block:: text

   GET https://dochub.lsst.codes/metadata/identifier.json

This provides two important features for linked-data datasets:

1. The URL for a JSON-LD_ document serves as the universal identifier for a resource, in a linked-data sense. For example, a ``relationships`` field in one JSON-LD_ document can use a DocHub REST API URL of another artifact as the ``relatedIdentifier``.
2. Third-party metadata services can ingest this JSON-LD_.

Implementation
^^^^^^^^^^^^^^

For consistency with LSST Data Management's technology stack, the RESTful API will be deployed as a Flask_ application.

The ID of a DocHub JSON-LD_ document can be derived from its MongoDB_ ``ObjectId``, which is a universally unique identifier for every MongoDB_ document.

Additional questions
^^^^^^^^^^^^^^^^^^^^

1. Should DocHub fully-resolve the metadata of all related resources (as much as is possible) by walking the link tree? This could argument to the HTTP GET request.
2. Should the RESTful API provide JSON-LD_ transformation functionality, like `framing <http://json-ld.org/spec/latest/json-ld-framing/>`_ (customizing the representation of a JSON-LD_ document), `expansion <http://json-ld.org/spec/latest/json-ld-api/#expansion-algorithms>`_ (inlining the context with field names) and `flattening <http://json-ld.org/spec/latest/json-ld/#flattened-document-form>`_ (collecting individual field's data and context in separate JSON objects).

GraphQL API
-----------

In addition to the RESTful API, DocHub should provide a GraphQL_ API through a ``/graphql`` endpoint.
Whereas RESTful APIs are oriented towards CRUD operations on resources, GraphQL_ is designed to efficiently populate data in user interfaces, which usually iterate over a subset of data in many resources.
In REST, it's often necessary to build custom endpoints that efficiently provide data to populate a UI.
With GraphQL_, the query specifies exactly what the shape of the output dataset is.

Implementation
^^^^^^^^^^^^^^

DataHub's GraphQL API will be implemented with the Graphene_ package *within* the Flask application.
All GraphQL_ queries are served from a single ``/graphql`` endpoint.

Type system
^^^^^^^^^^^

GraphQL uses a `type system <http://graphql.org/learn/schema/>`_ so that the server can validate and resolve GraphQL's arbitrary requests.
DocHub's GraphQL implementation will need to distill the various types of information expressed in JSON-LD as basic GraphQL types like Person and Organization, and `interfaces <http://graphql.org/learn/schema/#interfaces>`_ like Artifact for hierarchies that include types like SoftwareRepository, GitRef, LsstTheDocsEdition, DocuShareDeposition, ZenodoDeposition, and so forth.

Overall, the GraphQL API should be designed to efficiently populate DocHub's front-end user interface (whereas the REST and JSON-LD API is designed to be cross-walked to other metadata systems).

.. _json-ld-reading-list:

Appendix: JSON-LD reading list
==============================

- `JSON-LD best practices <http://json-ld.org/spec/latest/json-ld-api-best-practices/>`__.
- `Building a better book in the browser <http://journal.code4lib.org/articles/10668>`__.
- `Linked Data Patterns <http://patterns.dataincubator.org/book/index.html>`__
- `Indexing bibliographic linked data with JSON-LD, ElasticSearch <http://journal.code4lib.org/articles/7949>`__.
- `JSON-LD: Building meaningful data APIs <http://blog.codeship.com/json-ld-building-meaningful-data-apis/>`__.
- `BibJSON <http://okfnlabs.org/bibjson/>`__ describes resources with JSON objects with fields defined in BibTeX. Being JSON, it's also possible to describe these files with JSON-LD.

.. _CodeMeta: https://github.com/codemeta/codemeta
.. _GraphQL: http://graphql.org
.. _Flask: http://flask.pocoo.org
.. _Graphene: http://graphene-python.org
.. _JSON-LD: http://json-ld.org
.. _MongoDB: https://docs.mongodb.com/manual/
.. _zenodo_metadata: https://zenodo.org/dev#restapi-rep-meta
.. _Elasticsearch: https://www.elastic.co/products/elasticsearch

.. _schema.org: http://schema.org
.. _SoftwareSourceCode: http://schema.org/SoftwareSourceCode
.. _CreativeWork: http://schema.org/CreativeWork
.. _ScholarlyArticle: http://schema.org/ScholarlyArticle
.. _PublicationIssue: http://schema.org/PublicationIssue
.. _PublicationVolume: http://schema.org/PublicationVolume
.. _Periodical: http://schema.org/Periodical
