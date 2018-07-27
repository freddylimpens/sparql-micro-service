# SPARQL Micro-Services

The SPARQL Micro-Service architecture [1] is meant to allow the combination of Linked Data with data from Web APIs. It enables querying non-RDF Web APIs with SPARQL, and allows on-the-fly assigning dereferenceable URIs to Web API resources that do not have a URI in the first place.

This project is a prototype PHP implementation for JSON-based Web APIs. It comes with several example SPARQL micro-services, designed in the context of a biodiversity-related use case:
- ```flickr/getPhotosByGroupByTag```: searches a Flickr group for photos with a given tag. We use it to search the [*Encyclopedia of Life* Flickr group](https://www.flickr.com/groups/806927@N20) for photos of a given taxon: photos of this group are tagged with the scientific name of the taxon they represent, formatted as ```taxonomy:binomial=<scientific name>```.
- ```macaulaylibrary/getAudioByTaxon``` retrieves audio recordings for a given taxon name from the [Macaulay Library](https://www.macaulaylibrary.org/), a scientific media archive related to birds, amphibians, fishes and mammals.
- ```musicbrainz/getSongByName``` searches the [MusicBrainz music information encyclopedia](https://musicbrainz.org/) for music tunes whose titles match a given name with a minimum confidence of 90%.
- ```bhl/getArticlesByTaxon``` searches the [Biodiversity Heritage Library](https://www.biodiversitylibrary.org/) for scientific articles related to a given taxon name.
- ```eol/getTraitsByTaxon``` searches the searches the [Encyclopedia of Life traits bank](http://eol.org/traitbank) for data related to a given taxon name.

**Each micro-service is further detailed in its dedicated folder**.

## The SPARQL Micro-Service Architecture

A SPARQL micro-service is a lightweight, task-specific SPARQL endpoint that provides access to a small, resource-centric virtual graph, while dynamically assigning dereferenceable URIs to Web API resources that do not have URIs beforehand. The graph is delineated by the Web API service being wrapped, the arguments passed to this service, and the restricted types of RDF triples that this SPARQL micro-service is designed to spawn.


## How to use SPARQL micro-services?

A SPARQL micro-service is typically called from a SPARQL SERVICE clause.

The query below retrieves the URI of the common dolphin species (*Delphinus delphis*) from the SPARQL endpoint of TAXREF-LD, a Linked Data representation of the taxonomy maintained by the french National Museum of Natural History.
Then, it enriches this description with 2 photos retrieved from Flickr, 2 audio recordings from the Macaulay Library, and 1 music tune from MusicBrainz (the tune Web page URL).

If any of the 3 Web APIs invoked is not available (network error, internal failure etc.), the micro-service returns an empty result. In case this happens, the OPTIONAL clauses make it possible to still get (possibly partial) results.

    prefix rdfs:   <http://www.w3.org/2000/01/rdf-schema#>
    prefix owl:    <http://www.w3.org/2002/07/owl#>
    prefix foaf:   <http://xmlns.com/foaf/0.1/>
    prefix schema: <http://schema.org/>

    CONSTRUCT {
        ?species
            schema:subjectOf ?photo; schema:image ?img; schema:thumbnailUrl ?thumbnail;
            schema:contentUrl ?audioUrl;
            schema:subjectOf ?musicPage.
    } WHERE {
        SERVICE <http://taxref.mnhn.fr/sparql>
        { ?species a owl:Class; rdfs:label "Delphinus delphis". }

        OPTIONAL {
          SERVICE <https://example.org/sparqlms/flickr/getPhotosByGroupByTag?group_id=806927@N20&tags=taxonomy:binomial=Delphinus+delphis>
          { SELECT * WHERE { ?photo schema:image ?img; schema:thumbnailUrl ?thumbnail.  } LIMIT 2 }
        }

        OPTIONAL {
          SERVICE <https://example.org/sparqlms/macaulaylibrary/getAudioByTaxon?name=Delphinus+delphis>
               { SELECT ?audioUrl WHERE { [] schema:contentUrl ?audioUrl. } LIMIT 2 }
        }

        OPTIONAL {
          SERVICE <https://example.org/sparqlms/musicbrainz/getSongByName?name=Delphinus+delphis>
          { [] schema:sameAs ?page. }
        }
    }


## Folders structure

    src/sparqlms/
        config.ini                # generic configuration of the SPARQL micro-service engine
        service.php               # core of the SPARQL micro-services
        utils.php                 # utility functions
        Context.php
        Cache.php
        Metrology.php

        <Web API>/                # directory of the services related to one Web API
            <service>/            # one service of this Web API
                config.ini        # micro-service specific configuration
                profile.jsonld    # JSON-LD profile to translate the JSON response into JSON-LD
                insert.sparql     # optional SPARQL INSERT query to create triples that JSON-LD cannot create
                construct.sparql  # optional SPARQL CONSTRUCT query used to process URI dereferencing queries
                service.php       # optional script. Complements the config.ini in case specific actions are required.
                                  # See folder 'manual_config_example' for a detailed example.
            <service>/            # one service of this Web API
            ...
        <Web API>/                # directory of the services related to one Web API
            <service>/            # one service of this Web API
            ...
    docker/
        docker-compose.yml        # run SPARQL micro-services with Docker       
    apache_cfg/
        httpd.cfg                 # Apache rewriting rules for HTTP access
        ssl.cfg                   # Apache rewriting rules for HTTPS access


## Installation

To install this project, you will need an Apache Web server and a write-enabled SPARQL endpoint and RDF triple store. In our case, we used the [Corese-KGRAM](http://wimmics.inria.fr/corese) lightweight in-memory triple-store.

#### Pre-requisites

The following packages must be installed before installing the SPARQL micro-services.
  * PHP 5.3+. Below we assume our current vesrion is 5.6
  * Addition PHP packages: ```php56w-mbstring``` and ```php56w-xml```, ```php56w-devel```, ```php-pear``` (PECL)
  * [Composer](https://getcomposer.org/doc/)
  * Make sure the time zone is defined in the php.ini file, for instance:
```
  [Date]
  ; Defines the default timezone used by the date functions
  ; http://php.net/date.timezone
  date.timezone = 'Europe/Paris'
```
  * To use MongoDB as a cache, install the [MongoDB PHP driver](https://secure.php.net/manual/en/mongodb.installation.manual.php) and add the following line to php.ini:```extension=mongodb.so```

#### Installation procedure

Copy the ```src``` and ```vendor``` directories to a directory exposed by Apache, typically ```/var/www/html/sparqlms``` or ```public_html/sparqlms``` in your home directory.

Do __NOT__ use PHP ```composer``` to update the libraries in the vendor directory. This would override changes we made in Json-LD and EasyRDF.

Set the URL of your write-enabled SPARQL endpoint in ```src/sparqlms/config.ini```. This endpoint does not need to be exposed publicly on the Web, only the SPARQL micro-services should have access to it. For instance:
```
sparql_endpoint = http://localhost:8080/sparql
```

Create directory ```logs``` with 777 rights (chmod 777 logs). You should now have the following directory structure:

    sparqlms/
        src/sparqlms/
        vendor/
        logs/


Customize the dereferenceable URIs generated in the different services: replace the ```http://example.org``` URL with the URL of your server. See the comments in construct.sparql and insert.sparql files.

Set Apache [rewriting rules](http://httpd.apache.org/docs/2.4/rewrite/) to invoke micro-services using SPARQL or URI dereferencing: check the Apache rewriting examples in ```apache_cfg/httpd.conf``` and ```apache_cfg/ssl.conf```. See details in the sections below.

#### Rewriting rules for SPARQL querying

Micro-service URL pattern:
    ```http://example.org/sparqlms/<Web API>/<service>?param=value```

Rule:
    ```RewriteRule "^/sparqlms/([^/?]+)/([^/?]+).*$" http://example.org/~userdir/sparqlms/src/sparqlms/service.php?querymode=sparql&service=$1/$2 [QSA,P,L]```

Usage Example:
```
    SELECT * WHERE {
        SERVICE <https://example.org/sparqlms/macaulaylibrary/getAudioByTaxon?name=Delphinus+delphis>
        { [] <http://schema.org/contentUrl> ?audioUrl. }
    }
```

#### Rewriting rules for URI dereferencing

The apache_cfg directory contains more detailed Apache http/https configuration examples.

URI pattern:
    ```http://example.org/ld/<Web API>/<service>/<identifier>```

Rule:
    ```RewriteRule "^/ld/flickr/photo/(.*)$" http://example.org/~userdir/sparqlms/src/sparqlms/service.php?querymode=ld&service=flickr/getPhotoById&query=&photo_id=$1 [P,L]```

Usage Example:
    ```curl --header "accept:text/turtle" http://example.org/ld/flickr/photo/31173091516```


## Deploy with Docker

**Note: the Docker image is currently outdated compared to the current status of the code. In particular it does not use any cache database.**

You can test SPARQL micro-services using the two [Docker](https://www.docker.com/) images we have built:
- [frmichel/corese](https://hub.docker.com/r/frmichel/corese/): built upon debian:buster, runs the [Corese-KGRAM](http://wimmics.inria.fr/corese) RDF store and SPARQL endpoint. Corese-KGRAM listens on port 8081 but it is not exposed to the Docker server.
- [frmichel/sparql-micro-service](https://hub.docker.com/r/frmichel/sparql-micro-service/): provides the Apache Web server, PHP 5.6, and three of the SPARQL micro-services described above (Flicrk, Macauly Library, MLusicbrainz) configured and ready to go. Apache listens on port 80, it is exposed as port 81 of the Docker server.

To run these images, simply download the file ```docker/docker-compose.yml``` on a Docker server and run ```docker-compose up -d```.

#### Check application logs

To access the SPARQL micro-servcices log file, add a ```volumes``` parameter in the ```docker/docker-compose.yml``` file, like this:

    sms-apache:
        image: frmichel/sparql-micro-service
        networks:
          - sms-net
        ports:
          - "81:80"
        volumes:
          - "./logs:/var/www/html/sparql-ms/logs"

This will mount the SPARQL micro-service logs directory to the Docker host in directory ```./logs```.

You may have to set access mode 777 on this directory for the container to be able to write log files.


## Test the installation

#### Test SPARQL querying

You can test the services by typing the following commands in a bash:

    curl --header "Accept: application/sparql-results+json" "http://example.org/sparqlms/flickr/getPhotoById?query=select%20*%20where%20%7B%3Fs%20%3Fp%20%3Fo%7D&photo_id=31173091246"

    curl --header "Accept: application/sparql-results+json" "http://example.org//sparqlms/macaulaylibrary/getAudioByTaxon?query=select%20*%20where%20%7B%3Fs%20%3Fp%20%3Fo%7D&name=Delphinus+delphis"

    curl --header "Accept: application/sparql-results+json" "http://lexample.org//sparqlms/musicbrainz/getSongByName?query=select%20*%20where%20%7B%3Fs%20%3Fp%20%3Fo%7D&name=Delphinus+delphis"

That should return a JSON SPARQL result.

*If you used the Docker deployment, simply replace example.org with localhost:81 in the commands above.*


#### Test URI dereferencing

Enter this URL in your browser: http://example.org/ld/flickr/photo/31173091246 or the following command in a bash:

    curl --header "Accept: text/turtle" http://example.org/ld/flickr/photo/31173091246

*If you used the Docker deployment, simply replace example.org with localhost:81.*

That should return an RDF description of the photographic resource:

    @prefix schema: <http://schema.org/> .
    @prefix cos: <http://www.inria.fr/acacia/corese#> .
    @prefix dce: <http://purl.org/dc/elements/1.1/> .
    @prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
    @prefix sd: <http://www.w3.org/ns/sparql-service-description#> .
    @prefix ma: <http://www.w3.org/ns/ma-ont#> .
    @prefix api: <http://sms.i3s.unice.fr/schema/> .
    @prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

    <http://example.org/ld/flickr/photo/31173091516> dce:creator "" ;
        rdf:type schema:Photograph ;
        dce:title           "Delphinus delphis 1 (13-7-16 San Diego)" ;
        schema:author       <https://flickr.com/photos/10770266@N04> ;
        schema:subjectOf    <https://www.flickr.com/photos/10770266@N04/31173091516/> ;
        schema:thumbnailUrl <https://farm6.staticflickr.com/5567/31173091516_f1c09fa5d5_q.jpg> ;
        schema:image        <https://farm6.staticflickr.com/5567/31173091516_f1c09fa5d5_z.jpg> .


## Publications

[1] Franck Michel, Catherine Faron-Zucker and Fabien Gandon. *SPARQL Micro-Services: Lightweight Integration of Web APIs and Linked Data*. In Proceedings of the Linked Data on the Web Workshop (LDOW2018). https://hal.archives-ouvertes.fr/hal-01722792

[2] Franck Michel, Olivier Gargominy, Sandrine Tercerie & Catherine Faron-Zucker (2017). *A Model to Represent Nomenclatural and Taxonomic Information as Linked Data. Application to the French Taxonomic Register, TAXREF*. In Proceedings of the 2nd International Workshop on Semantics for Biodiversity (S4BioDiv) co-located with ISWC 2017 vol. 1933. Vienna, Austria. CEUR. https://hal.archives-ouvertes.fr/hal-01617708
