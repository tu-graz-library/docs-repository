# OAI-PMH 

OAI-PMH (Open Archives Initiative Protocol for Metadata Harvesting) is a low-barrier mechanism for repository interoperability.
**Data Providers** are repositories that expose structured metadata via OAI-PMH. **Service Providers** then make OAI-PMH service requests to harvest that metadata. OAI-PMH is a set of six verbs or services that are invoked within HTTP.


## Endpoints

### Public
The following endpoints are public. The full documentation of the verbs can be found here: [Link](https://www.openarchives.org/OAI/openarchivesprotocol.html#ProtocolMessages)

|Verb               |Description                                                |Example URL|
---                 | ---                                                       |---
|GetRecord          |Request a record by ID in the specified format             |[https://127.0.0.1:5000/oai2d?verb=GetRecord&identifier=oai:arXiv.org:cs/0112017&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=GetRecord&identifier=oai:arXiv.org:cs/0112017&metadataPrefix=oai_dc)|
|Identify           |Retrieve information about the repository                  |[https://127.0.0.1:5000/oai2d?verb=Identify](https://127.0.0.1:5000/oai2d?verb=Identify)|
|ListIdentifiers    |Retrieve header information of records only                |[https://127.0.0.1:5000/oai2d?verb=ListIdentifiers&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=ListIdentifiers&metadataPrefix=oai_dc)|
|ListMetadataFormats|Retrieve metadata formats availabe                         |[https://127.0.0.1:5000/oai2d?verb=ListMetadataFormats](https://127.0.0.1:5000/oai2d?verb=ListMetadataFormats)|
|ListMetadataFormats|Retrieve records in the specified format                   |[https://127.0.0.1:5000/oai2d?verb=ListRecords&metadataPrefix=oai_dc](https://127.0.0.1:5000/oai2d?verb=ListRecords&metadataPrefix=oai_dc)|
|ListSets           |Retrieve set structure                                     |[https://127.0.0.1:5000/oai2d?verb=Identify](https://127.0.0.1:5000/oai2d?verb=Identify)|


### Non Public
The following endpoints are only available as admin and all deal with sets of OAI-PMH.

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


## Issues

- Records
    - Missing the `"_oai"` field necessary for record and identifier retrieval
- elasticsearch index
    - It is possible to define the index used by elasticsearch for all relevant verbs via a config variable `OAISERVER_RECORD_INDEX='records'`
    - Currently, it is set to use the 'records' index which does not exist
    - The correct index for records on my machine looks like this `rdmrecords-records-record-v2.0.0-1621247047`
    - Available indices can be retrieved by visiting [http://localhost:9200/_cat/indices?v&health=yellow&pretty](http://localhost:9200/_cat/indices?v&health=yellow&pretty)
