# OAI-PMH 

OAI-PMH (Open Archives Initiative Protocol for Metadata Harvesting) is a low-barrier mechanism for repository interoperability. [Documentation](http://www.openarchives.org/OAI/openarchivesprotocol.html)
**Data Providers** are repositories that expose structured metadata via OAI-PMH. **Service Providers** then make OAI-PMH service requests to harvest that metadata. OAI-PMH is a set of six verbs or services that are invoked within HTTP.

As of today (03.08.2021), sets are not working properly and the admin routes are not to be used. A revamp for the sets is about to happen in the official module.


## Endpoints

### Public
The following endpoints are public. The full documentation of the verbs can be found here: [Link](https://www.openarchives.org/OAI/openarchivesprotocol.html#ProtocolMessages)
These endpoints/verbs are used by harvesters (other repositories) to request metadata of available records. Additionally, informations about the repository and available data formats is provided.

|Verb               |Description                                                |Example URL|
---                 | ---                                                       |---
|GetRecord          |Request a record by ID in the specified format             |[https://127.0.0.1:5000/oai2d?verb=GetRecord&identifier=oai:arXiv.org:cs/0112017&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=GetRecord&identifier=oai:arXiv.org:cs/0112017&metadataPrefix=oai_dc)|
|Identify           |Retrieve information about the repository                  |[https://127.0.0.1:5000/oai2d?verb=Identify](https://127.0.0.1:5000/oai2d?verb=Identify)|
|ListIdentifiers    |Retrieve header information of records only                |[https://127.0.0.1:5000/oai2d?verb=ListIdentifiers&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=ListIdentifiers&metadataPrefix=oai_dc)|
|ListMetadataFormats|Retrieve metadata formats availabe                         |[https://127.0.0.1:5000/oai2d?verb=ListMetadataFormats](https://127.0.0.1:5000/oai2d?verb=ListMetadataFormats)|
|ListMetadataFormats|Retrieve records in the specified format                   |[https://127.0.0.1:5000/oai2d?verb=ListRecords&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=ListRecords&metadataPrefix=oai_dc)|
|ListSets           |Retrieve set structure                                     |[https://127.0.0.1:5000/oai2d?verb=Identify](https://127.0.0.1:5000/oai2d?verb=Identify)|


### Non Public
The following endpoints are only available as admin and all deal with sets of OAI-PMH. These endpoints will be revamped in a later update. Functionality is not guaranteed.

|Request Methods            |Description                                                |Example URL|
---                         | ---                                                       |---
|OPTIONS, POST              |Action to be performed on set                              |[https://127.0.0.1:5000/admin/oaiset/action/](https://127.0.0.1:5000//admin/oaiset/action/)|
|GET, HEAD, OPTIONS         |                                                           |[https://127.0.0.1:5000/admin/oaiset/ajax/lookup/](https://127.0.0.1:5000/admin/oaiset/ajax/lookup/)|
|OPTIONS, POST              |                                                           |[https://127.0.0.1:5000/admin/oaiset/ajax/update/](https://127.0.0.1:5000/admin/oaiset/ajax/update/)|
|GET, HEAD, OPTIONS, POST   |Create new set                                             |[https://127.0.0.1:5000/admin/oaiset/new/](https://127.0.0.1:5000/admin/oaiset/new/)|
|OPTIONS, POST              |Delete set                                                 |[https://127.0.0.1:5000/admin/oaiset/delete/](https://127.0.0.1:5000/admin/oaiset/delete/)|
|GET, HEAD, OPTIONS         |Retrieve details of set                                    |[https://127.0.0.1:5000/admin/oaiset/details/](https://127.0.0.1:5000/admin/oaiset/details/)|
|GET, HEAD, OPTIONS, POST   |Edit specified set                                         |[https://127.0.0.1:5000/admin/oaiset/edit/](https://127.0.0.1:5000/admin/oaiset/edit/)|
|GET, HEAD, OPTIONS         |Export set as specified type                               |[https://127.0.0.1:5000/admin/oaiset/export/<export_type\>/](https://127.0.0.1:5000/admin/oaiset/export/<export_type>/)|
|GET, HEAD, OPTIONS         |Main page for sets                                         |[https://127.0.0.1:5000/admin/oaiset/](https://127.0.0.1:5000/admin/oaiset/)|


## Configuration

#### ElasticSearch index
Elastisearch index to be used. This will be set by `invenio-rdm-records`:
```conf
OAISERVER_RECORD_INDEX='records'
```

#### OAI ID Prefix
The prefix that will be applied to the generated OAI-PMH ids. Should be the address of the repository (f.e. repository.tugraz.at):
```conf
OAISERVER_ID_PREFIX = 'repository.tugraz.at': 
```

#### Admin Emails
The e-mail addresses of administrators of the repository
```conf
OAISERVER_ADMIN_EMAILS = [
    'info@inveniosoftware.org',
]:
```

#### Available Metadata Formats
Define the metadata formats available from a repository. These can be completely redefined or modified (extended, replaced, removed) as need be.
```conf
OAISERVER_METADATA_FORMATS = {`
    'oai_dc': {
        'serializer': (
            'invenio_oaiserver.utils:dumps_etree', {
                'xslt_filename': pkg_resources.resource_filename(
                    'invenio_oaiserver', 'static/xsl/MARC21slim2OAIDC.xsl'
                ),
            }
        ),
        'schema': 'http://www.openarchives.org/OAI/2.0/oai_dc.xsd',
        'namespace': 'http://www.openarchives.org/OAI/2.0/oai_dc/',
    },
    'marc21': {
        'serializer': (
            'invenio_oaiserver.utils:dumps_etree', {
                'prefix': 'marc',
            }
        ),
        'schema': 'http://www.loc.gov/standards/marcxml/schema/MARC21slim.xsd',
        'namespace': 'http://www.loc.gov/MARC21/slim',
    }
}
```

The serializer is defined by a function and additional arguments. The function will be called with two fixed arguments (pid, record) and additional specified arguments.
It must return an LXML element instance, to be used in the response of the OAI server.

Adding another format can be achieved by putting following code in a module's `init_app` function, which resides in the module's `ext.py` file:
```python

def my_metadata_serializer(pid, record, **kwargs):
    # record['_source'] will hold the record data
    return = MyMetadataFormat().dump_xml(record['_source'])

def init_app(self, app):
    app.extensions['invenio-oaiserver']['OAISERVER_METADATA_FORMATS'].set('my_metadata_format', {
        'serializer': ('my_module.ext:my_metadata_serializer', {}),
        'schema'    : 'link_to_schema_definition_file',
        'namespace' : 'link_to_schema_definition',
        }
    )
```

After this, verbs supporting the `metadataFormat` attribute will be able to pick up the metadata format.


#### Record Search Class
Record search class for the `ListRecords` verb. This is an ElasticSearch class by default and it should return harvestable records.
```conf
OAISERVER_SEARCH_CLS = 'invenio_oaiserver.query:OAIServerSearch'
```

#### OAI ID Fetcher Function
Will return the OAI ID of a record. This will take two arguments: `function(record_uuid, record_as_dict)`.
```conf
OAISERVER_ID_FETCHER = 'invenio_oaiserver.fetchers:oaiid_fetcher'
```

#### Record Updated Key
Record dictionary key for information on when the record was last updated. Set by `invenio-rdm-records`.
```conf
OAISERVER_LAST_UPDATE_KEY = "_updated"
```

#### Record Created Key
Record dictionary key for information on when the record was created.
```conf
OAISERVER_CREATED_KEY = "_created"
```

#### Record's Sets Fetcher Function
Function to fetch the sets a record belongs to as a list. Takes one argument: `function(record_as_dict)`
```conf
OAISERVER_RECORD_SETS_FETCHER = 'invenio_oaiserver.utils:record_sets_fetcher'
```

#### Record Retrieval Class
Used when an `identifier` parameter is used in a verb and `OAISERVER_GETRECORD_FETCHER` is not overridden.
```conf
OAISERVER_RECORD_CLS = 'invenio_records.api:Record'
```

#### Single Record Fetcher Function
Function to fetch a record and return as dict. Takes one argument: `function(record_uuid)`
```conf
OAISERVER_GETRECORD_FETCHER = 'invenio_oaiserver.utils:getrecord_fetcher'
```
