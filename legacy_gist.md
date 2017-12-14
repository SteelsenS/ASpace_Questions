# ASpace_Questions
An issue repository for questions being asked about ASpace

Below is the original GIST:

# Data from ASpace for ICE

I'm going to apologize in advance here if I'm recapitulating information that you're already familiar with or if I'm missing key points. I've been in and out of the loop and may be misunderstanding what you're looking for. If that's the case then please let me know. In general I'm going to try to provide explanations based on what I know in the most complete way that I can, even if it involves a lot of rehashing.

## Questions [ Answered ]

### 9/25 email

1:

> Does it look like I'm going about this the right way?

Yes, you've nailed how to navigate within the API (finding the related paths provided) and the sticking point seems to be identifying the exact data points to use for each situation.

2:

> The above looks like it would get me to the resource information, but not the kind of item information that I’m using throughout the table above. What would be the right way to get at that? 
Robert clarified that the only kind of container barcodes that ICE will be handling will be containers that don’t contain any items that have barcodes (he says he can provide examples of these). This makes me wonder even further where the item-type information would be coming from in this case.

Some of the information you're looking for is going to be linked through the resource, some through the archival object, and some through the top container itself. This is pretty much the crux of the issue so I'll go more in depth in the next section.

3:
 
> ~~I presume that a container can contain multiple items. Does that mean we need to retrieve data for all items in the container and load that into the ICE form? (I think this is more of a question for Robert).~~ Robert has clarified that the only A-Space container barcodes that ICE will be handling are those that can be treated as a single item, so that resolves this.

Yup, this one is set.

### 10/2 email

> It seems like an approach like this might work, but please let me know if there’s a better way:
![Proposed Flow](https://i.imgur.com/2aWE6MV.png)

This seems like it should work. My only immediate concern is whether a barcode may exist in a different indicator field (since in theory `indicator_2` itself doesn't tell you what type of data is in that field, you have to refer to the indicator type (`type_2`) but I think that in practice it's fine. 

With that said - I think it may also be more complex than what is absolutely necessary. My explanation follows

## The Barcode Scenarios

There are three barcode scenarios presented:

>1. (ex. 39002125667973) Barcodes for containers that contain items with their own barcode – ICE can’t do anything with these and should reject them
>2. (ex. 39002102378974) Barcodes for individual items that have been entered into the system as containers – ICE should treat these as single items within the ICE system
>3. (ex. 39002125006222) Barcodes for individual items that have been entered as single items – ICE should also treat these as single items

A lot of the problem here is that we're moving back and forth between inventory management and archival management. Colocating a barcoded VHS with several others inside of a barcoded box doesn't matter for their description, but it does very much matter for inventory. 

The "top_container" entity represents that arbitrary box of items, while the "sub_container" entity, an instance linked directly to the "archival_object" entity is, theoretically, a single item. However, as a matter of practice sometimes single items are categorized as top containers (case 2).

I think that *in theory* you should be able to do only the archival object search. Per the table that you generated for the query `{{BaseUrl}}/repositories/{{ActiveRepository}}/search?type[]=archival_object&page=1&q={{barcode}}`:

1.	Multiple items returned
2.	Single item lacking indicator_2
3.	Single item with indicator_2

For the second case the barcode appears in the top container object - the `barcode` property of the `top_container` property. In the third the barcode appears in the `indicator_2` property of the `sub_container` property. Top container has a different value. It looks like the newest version of ArchivesSpace also includes a handy `child_container_u_sstr` value right in the results json (without needing to parse further) which should instantly let you identify whether the barcode is a sub_container (there's a value there) or top_container (no value).

This allows you to simplify the logic a bit:
![Modified Flow](https://i.imgur.com/KqpHkwG.png)

The problem here is that neither query accounts for the edge case of a repository which typically barcodes their AV at the sub_container level presents for scanning a top container which has not yet been filled with objects. In practice this wouldn't happen - you'd instantly realize that the box wasn't an AV object and didn't have anything in it - but if you were to scan it then it would populate the form since it looks like a case 2 object. 

Overall it seems like this could easily be a business rule. Each repository defines a rule of how it manages barcodes and then the application knows what to look for a priori. In the absence of this, however, either of the flows above would work to get to the right place.

The most critical thing here is that you're trying to get to a *single* archival object. If you can't arrive at a single unit of archival description then you can't proceed. This is obvious in the context of case 1, but it bears repeating since having a single AO to start with is the precondition for just about everything that comes next.

## The Spreadsheet

Once you've successfully identified the instance in question and tied it to a single archival object then you're ready to start gathering the other data listed in the spreadsheet.

### [Spreadsheet proper](https://yale.box.com/s/zril543yent75z0fvs87e40aepslystu) (to get it into this same doc)

"R" stands for "Resolved" - whether the spreadsheet seems to show consensus on a given data point and its provenance - "SSS Resolved" is just to allow a quick review of what is answered vs. in the questions section of this doc.

### Item Information (moves from lower right to top on ICE Import Screen)

|ID |ICE field|Resolved (SSS) |Resolved |Example |ICE location |Database field (Y/N) |Reason ICE needs |ArchivesSpace field name / location |Voyager field |Rules |Notes |
|---|---------|---------------|---------|--------|-------------|---------------------|-----------------|------------------------------------|--------------|------|------|
|1|Item Barcode|Y|Y|39002123456789|Uses Barcode field|Exists|unique ID, some file names|Indicator2 or Indicator3  / AO --> Instances.where(exists barcode) --> subcontainer --> indicator2|Item Record / Item_barcode|cannot be blank|
|4|Display Title|Y|Y|Pacliacci: E allor perche di?: Ferruccio Corradetti, undated|Uses "Title" field currently under Bibliographic Info|Exists|display value to staff; needed for Preservica ingest|Displaystring / AO|Bib record / datafield tag="245a" and datafield tag="245b"; concatenated|245a cannot be blank|display full text string even if form requires scroll in box to read|
|5|Format Type|Y|N|Betacam SP|Uses "Type" field under Item Information.|Need: Should be an editable database field with controlled vocabulary. |item ID, vendor/shipment assignment|ExtentType / AO --> Extents|007 field --if M (film), then look at Byte 01 (gauge), then at byte 04 (aspect ratio). If 007 is "v" (video recording), then look at byte 04 (type of cassetts). If 007 is "s" (sound), llok at byte 01.(Note: do we need the following? "Bib record / datafield tag="300." If blank, pull datafield tag="245h" or datafield tag="534e")|Display imported choice at top of allowed options, but in red(?) font if not equal to a controlled vocab option. Do not allow it as a selection, unless a perfect match. If perfect match, do not display in red and choose controlled vocab equivalent. Upon Submit, database field will be updated to controlled vocab choice; |save incorrect imported value to report to Content Owner for that Item is needed with barcode, incorrect format, and corrected format type. Generate report based on Import Form submission date and Source.; We will supply the controlled vocab list soon|
|9|Originator (Reference)|Y|Y|CtY-Gilmore Music|Uses "Location" field|Exists?|Identifying Yale/CtY as owner|Master Tables Config based on source|Master Tables Config based on source|Should be populated from Master Tables Config|Master Tables Config need update to add column. We should probably call this field "Originator." "Originator Reference" is a BWF metadata field currently populated with George Blood info. |
|11|Item Creation Date|Y|N|2017-01-01|Uses "Year" field under Item Information|Exists|verification /  .xml metadata |date.expression where related to the appropriate archival_object_id / AO --> Dates.where(typeof = isodate)|Bib record / datafield tag="260c" or "264c"|iso format; if blank, use ?other?|
|16|Preservica Synch Handle|N|||Uses "URL" field, currently under Holding Information. Most likely will be displayed as a url. |Needs|Preservica file ingest|Uri / AO|not applicable|should be blank if a non A-Space record|
|6|TRT / max run time|N|Y|01:10:01|new field|Needs |calculation of run time or maximum run time / possible number of files|runtime / AO --> duration; OK, If blank; ICE will report back after QC how many files, individual file runtimes and physical item runtime aggragate total.|video: if controlfield "007" is m or v, information will be in 008 byte 18 - 20 or datafield="306" / audio: if controlfield  "007" is s, information is in 300 $a following the extent or 505 $a or 500 $g or 306|hour:minute:second format, time time of day|Still need a sample of this field for cases where it does exist. I thought that we were just going to have this come from ICE?|

### Source Location

|ID |ICE field|Resolved (SSS) |Resolved |Example |ICE location |Database field (Y/N) |Reason ICE needs |ArchivesSpace field name / location |Voyager field |Rules |Notes |
|---|---------|---------------|---------|--------|-------------|---------------------|-----------------|------------------------------------|--------------|------|------|
|22|Collection Name|Y|Y||Uses "Author" field currently under Bibliographic Info. |Exists|identification during import; xml metadata|title / resource|Use Author field rules|
|20|Collection Date|N|N||Uses "Year" field currently under Bibliographic Info. |Needs|identification during import|dates / resource|Robert will check||Not sure about Collections Info in Voyager|
|12|Call #|Y|Y||Uses "LC Call #" field currently under Bibliographic Info. |Exists|file name creation|display_call_no / Resource|Bib record / datafield tag="050" or datafield tag="090" or datafield tag="099"||For Voyager items, could be LC Call # or Yale Call #|
8|Source Location|N|Y||Could this be "Holding Library" under Holding information? |Needs to be a database field with a controlled vocabulary. ||DRMS populates this. |||Could be the same as originator and maybe can be deleted|
25|ArchivesSpace ID|Y|?|/resources/10404#tree::archival_object_2442328|Could this use the "Bib ID" field under Bibliographic Information? |Exists|BWF header metadata|aspace.id|Analog Bib ID|||
21|Header Metadata Description|Y|Y||This is a concatenated field, pulling from ArchivesSpace source fields. Place at center-bottom of Item Information. |No|Shipping Form: Vendor instruction view in CSV format|Concatenated field from spreadsheet info (see notes).|see notes||Should concatenate the following fields from our spreadsheet: Originator + Call Number + ArchivesSpace ID + Collection Name + Display Title (description)|
19|Filename|Y|Y||This is a concatenated field, pulling from ArchivesSpace source fields. Place at center-bottom of Item Information. |No|Shipping Form: Vendor instruction view in CSV format|Concatenated field from spreadsheet info (see notes).|see notes||Collection designation/Call number (e.g. “mss_0014hsr_”) + Box/top level container barcode + item barcode (if single AV items are not designated as top level containers)|

### Top Container / Box Information (was Holding Information on ICE Import Screen- stays in same place)

|ID |ICE field|Resolved (SSS) |Resolved |Example |ICE location |Database field (Y/N) |Reason ICE needs |ArchivesSpace field name / location |Voyager field |Rules |Notes |
|---|---------|---------------|---------|--------|-------------|---------------------|-----------------|------------------------------------|--------------|------|------|
2|Box Barcode|Y|Y|39002234567890|Uses "MFHD" field|Exists|get item from LSF; file name |Barcode / TopContainer|Ask about Horror video in storage boxes.||Perhaps not used for file name for MSSA |
14|Box Name / Number|Y|Y||Uses "MFHD call #" field. |Needs|some file names; LSF location|Indicator / TopContainer|n/a||If not using Box Barcode, needed for MSSA file name preceded by "_b"|
24|Item Count|Y|?|1 of 2|Uses "related Items" field, currently under Bibliographic Info. |No|verification during import|(count of child items in top level container)|Confirm this is from MFHD|This field must be calculated by ArchivesSpace. If the count is "0," then "Box Barcode" field is correctly a duplicate of "Item Barcode" field under Item Info.|MSSA uses the top level container as the item, which cannot therefore contain any other items|
13|Series #|Y|Y||Uses space occupied by Preservation Note|Needs|file name creation for MSSA|ComponentID / AO|n/a|MSSA only; may be blank; if blank make 0; if a single Arabic numeral, zero pad to 2 digits; if not only arabic numerals, import as is (eg. 2007-A-056) This is the TOP AO for this particular segment of the tree. If multiple values exist (array), then offer choice of values or 'reject' to user)|Does this apply outside of MSSA? |
15|Folder #|Y|Y||Create new database field for this value. |Needs|file name creation for MSSA|Indicator2 / AO --> Instances.where(exists barcode) --> subcontainer --> indicator2|n/a|MSSA only; This gets tricky because for items with folders the folder number may be placed in indicator2 which is where other records have their instance barcodes (see AO 2088139) - the indicator type provides information about what it is. It may be necessary to inspect the text and then add as folder Id or barcode as relevant.|Does this apply outside of MSSA? |


### Collection Information (was Bibliographic Information on ICE Import Screen - moves from top to lower right)

|ID |ICE field|Resolved (SSS) |Resolved |Example |ICE location |Database field (Y/N) |Reason ICE needs |ArchivesSpace field name / location |Voyager field |Rules |Notes |
|---|---------|---------------|---------|--------|-------------|---------------------|-----------------|------------------------------------|--------------|------|------|
|22|Collection Name|Y|Y||Uses "Author" field currently under Bibliographic Info. |Exists|identification during import; xml metadata|title / resource|Use Author field rules|
|20|Collection Date|N|N||Uses "Year" field currently under Bibliographic Info. |Needs|identification during import|dates / resource|Robert will check||Not sure about Collections Info in Voyager|
|12|Call #|Y|Y||Uses "LC Call #" field currently under Bibliographic Info. |Exists|file name creation|display_call_no / Resource|Bib record / datafield tag="050" or datafield tag="090" or datafield tag="099"||For Voyager items, could be LC Call # or Yale Call #|

### Mapping Spreadsheet Values

Anything marked "N" in the resolved column does not have enough information for me to complete. Otherwise I've done my best to explain how to obtain the various datapoints. For convenience they are referenced by ID. I apologize that the ID numbers are now non-sequential. The order of the sheet was redone in the latest version and it seemed better to keep the references consistent than to reorder.

The devil is in the details here. I'm using "dot" notation to discuss navigating objects for the sake of simplicity, but it will rarely be that simple. Many times values exist in arrays (e.g., instance, date, extent) and you'll need to analyze the values of the array's objects to choose which one you're sourcing from. 

#### Item Information (top on ICE Import Screen)

##### (1) Item Barcode
The barcode rules above dictate that valid barcodes (for the purpose of filling the ICE form) exist only at the level of the media item. Therefore there's no need to pull this from ASpace - just use the value of the scanned barcode.

##### (4) Display title

This is not quite the same as the title property. You have to (again) parse the json property:
`json.display_string` should get you what you need.

##### (5) - Format Type

Per Kevin's comments, this field will need to be a controlled input populated by DRMS staff:

>>>For analog format (https://gist.github.com/SteelsenS/530220a88e2cbfabefae394737c73d2e#5---analog-format), it is my understanding that this data would not be in ArchivesSpace routinely or correctly. This data would come from extent type, but isn't there very much now. There is currently a proposal to improve this a little bit and utlize an existing standard for the choices of AV extent types, but that has not been approved by YAMS and would take significant effort to implement on legacy data. For this project, I still suggest the Preservation department be responsible for filling out this data element after they have the physical item in hand.


##### (9) - Originator

E.g., `CtY-Gilmore` - this should by a controlled input populated by DRMS staff from Master Tables.

##### (11) Origination Date (10 char)

Once again, you'll find this in the json property of the search results.

ASpace understands and differentiates between many different types of date. They are stored in an array hanging off of each relevant archival object. The property `date_type` tells you what you're looking at.

You're going to want to fetch the array that exists in `json.dates` and then check each date object's `date_type` property. When you hit one equal to `isodate` then you've found what you're looking for. Note that it may not exist.

Per Kevin's comment:

>>>I do not think we should inherit origination data (https://gist.github.com/SteelsenS/530220a88e2cbfabefae394737c73d2e#11---origination-date). It is possible such data could be contained in a parent or grandparent, but this data could also not apply. I say stick to the lowest level archival object.

You will take the first valid value.

##### (6) Duration

In theory this could be gathered from the extent information in the archival object. However, there's no standard practice for this (yet) so it will have to be populated by DRMS in the format `HH:MM:SS` per Kevin's comment:

>>>For max run time (https://gist.github.com/SteelsenS/530220a88e2cbfabefae394737c73d2e#6---max-run-time), what Steelsen remembers me saying is that Fortunoff has total runtime that we were hoping to store in the extent field. However, YAMS has not yet completed the work to make an extent type for time. In the meantime, we are storing all this data in notes. This will not fixed in time for this project. I suggest the max runtime be a field that is filled out by the Preservation department after they have the physical item in hand. I further suggest is be formatted as HH: MM: SS.

#### Source Location 

##### (8) Source Location

See (9) - Originator

##### (25) ArchivesSpace ID

Per Kevin's comment:

>>>The ArchivesSpace ID (https://gist.github.com/SteelsenS/530220a88e2cbfabefae394737c73d2e#25---archivesspace-id) referred to is the "id" that shows up in your GET {{BaseUrl}}/repositories/{{ActiveRepository}}/search?type[]=archival_object&page=1&q=39002125006222 instance above as "/repositories/16/archival_objects/2630006". That is what we need to ingest into Preservica.

This is the `uri` property from the archival object or its search result.

##### (21) Header Metadata Description

Should concatenate the following fields from our spreadsheet: Originator + Call Number + ArchivesSpace ID + Collection Name + Display Title (description)

##### (19) Filename

Concatenation of values: Collection designation/Call number (e.g. “mss_0014hsr_”) + Box/top level container barcode + item barcode (if single AV items are not designated as top level containers)

#### Top Container (holdings information, same location)

##### (2) Box Barcode

IF barcode is of type 2 then use the same barcode.
IF barcode is of type 3 then use the top container barcode:
e.g.:
`GET {{BaseUrl}}/repositories/{{ActiveRepository}}/search?type[]=archival_object&page=1&q=39002125006222`
```
{
    "page_size": 100,
    "first_page": 1,
    "last_page": 1,
    "this_page": 1,
    "offset_first": 1,
    "offset_last": 1,
    "total_hits": 1,
    "results": [
        {
            "id": "/repositories/16/archival_objects/2630006",
            "uri": "/repositories/16/archival_objects/2630006",
            "title": "Grand Aria (Semiramide): Chant avec piano: Sung by Roma, 1899 January 29",
            "primary_type": "archival_object",
            "types": [
                "archival_object"
            ],
            "json": "{\"lock_version\":0,\"position\":0,\"publish\":true,\"ref_id\":\"aspace_fc3a5e4e86ef7222d00da601d00fbb0c\",\"title\":\"Grand Aria (Semiramide): Chant avec piano: Sung by Roma\",\"display_string\":\"Grand Aria (Semiramide): Chant avec piano: Sung by Roma, 1899 January 29\",\"restrictions_apply\":false,\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:36Z\",\"system_mtime\":\"2017-10-07T14:50:50Z\",\"user_mtime\":\"2017-09-20T21:23:36Z\",\"suppressed\":false,\"level\":\"item\",\"jsonmodel_type\":\"archival_object\",\"external_ids\":[],\"subjects\":[],\"linked_events\":[],\"extents\":[{\"lock_version\":0,\"number\":\"1\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:36Z\",\"system_mtime\":\"2017-09-20T21:23:36Z\",\"user_mtime\":\"2017-09-20T21:23:36Z\",\"portion\":\"whole\",\"extent_type\":\"phonograph records\",\"jsonmodel_type\":\"extent\"}],\"dates\":[{\"lock_version\":0,\"expression\":\"1899 January 29\",\"begin\":\"1899-01-29\",\"end\":\"1899-01-29\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:36Z\",\"system_mtime\":\"2017-09-20T21:23:36Z\",\"user_mtime\":\"2017-09-20T21:23:36Z\",\"date_type\":\"inclusive\",\"label\":\"creation\",\"jsonmodel_type\":\"date\"}],\"external_documents\":[],\"rights_statements\":[],\"linked_agents\":[],\"ancestors\":[{\"ref\":\"/repositories/16/archival_objects/2630005\",\"level\":\"series\"},{\"ref\":\"/repositories/16/resources/10704\",\"level\":\"collection\"}],\"instances\":[{\"lock_version\":0,\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:36Z\",\"system_mtime\":\"2017-09-20T21:23:36Z\",\"user_mtime\":\"2017-09-20T21:23:36Z\",\"instance_type\":\"mixed materials\",\"jsonmodel_type\":\"instance\",\"is_representative\":false,\"sub_container\":{\"lock_version\":0,\"indicator_2\":\"39002125006222\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:36Z\",\"system_mtime\":\"2017-09-20T21:23:36Z\",\"user_mtime\":\"2017-09-20T21:23:36Z\",\"type_2\":\"item_barcode\",\"jsonmodel_type\":\"sub_container\",\"top_container\":{\"ref\":\"/repositories/16/top_containers/304148\",\"_resolved\":{\"lock_version\":52,\"barcode\":\"39002125667973\",\"indicator\":\"730\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:23:35Z\",\"system_mtime\":\"2017-10-07T14:50:50Z\",\"user_mtime\":\"2017-09-20T21:23:35Z\",\"type\":\"Box\",\"jsonmodel_type\":\"top_container\",\"active_restrictions\":[],\"container_locations\":[],\"series\":[{\"ref\":\"/repositories/16/archival_objects/2630005\",\"identifier\":\"1\",\"display_string\":\"Recordings\",\"level_display_string\":\"Series\",\"publish\":true}],\"collection\":[{\"ref\":\"/repositories/16/resources/10704\",\"identifier\":\"MSS.142\",\"display_string\":\"Berliner Gramophone Disc Collection\"}],\"uri\":\"/repositories/16/top_containers/304148\",\"repository\":{\"ref\":\"/repositories/16\"},\"restricted\":false,\"is_linked_to_published_record\":false,\"display_string\":\"Box 730: Series 1 [39002125667973]\",\"long_display_string\":\"MSS.142, Series 1, Box 730 [39002125667973]\"}}}}],\"notes\":[{\"jsonmodel_type\":\"note_multipart\",\"subnotes\":[{\"publish\":true,\"jsonmodel_type\":\"note_text\",\"content\":\"Disc No. 3072\"}],\"type\":\"scopecontent\",\"persistent_id\":\"aspace_c9906edbfdb141a932d5508867d1e4f9\",\"label\":\"Scope and Contents\",\"publish\":true}],\"uri\":\"/repositories/16/archival_objects/2630006\",\"repository\":{\"ref\":\"/repositories/16\",\"_resolved\":{\"lock_version\":0,\"repo_code\":\"Qdabra-test\",\"name\":\"Qdabra-test\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:18:04Z\",\"system_mtime\":\"2017-09-20T21:18:04Z\",\"user_mtime\":\"2017-09-20T21:18:04Z\",\"publish\":false,\"jsonmodel_type\":\"repository\",\"uri\":\"/repositories/16\",\"display_string\":\"Qdabra-test (Qdabra-test)\",\"agent_representation\":{\"ref\":\"/agents/corporate_entities/7491\",\"_resolved\":{\"lock_version\":0,\"publish\":false,\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:18:04Z\",\"system_mtime\":\"2017-09-20T21:18:04Z\",\"user_mtime\":\"2017-09-20T21:18:04Z\",\"jsonmodel_type\":\"agent_corporate_entity\",\"agent_contacts\":[{\"lock_version\":0,\"name\":\"Qdabra-test\",\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:18:04Z\",\"system_mtime\":\"2017-09-20T21:18:04Z\",\"user_mtime\":\"2017-09-20T21:18:04Z\",\"jsonmodel_type\":\"agent_contact\",\"telephones\":[]}],\"linked_agent_roles\":[],\"external_documents\":[],\"notes\":[],\"used_within_repositories\":[],\"used_within_published_repositories\":[],\"dates_of_existence\":[],\"names\":[{\"lock_version\":0,\"primary_name\":\"Qdabra-test\",\"sort_name\":\"Qdabra-test\",\"sort_name_auto_generate\":true,\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:18:04Z\",\"system_mtime\":\"2017-09-20T21:18:04Z\",\"user_mtime\":\"2017-09-20T21:18:04Z\",\"authorized\":true,\"is_display_name\":true,\"source\":\"local\",\"jsonmodel_type\":\"name_corporate_entity\",\"use_dates\":[]}],\"related_agents\":[],\"uri\":\"/agents/corporate_entities/7491\",\"agent_type\":\"agent_corporate_entity\",\"is_linked_to_published_record\":false,\"display_name\":{\"lock_version\":0,\"primary_name\":\"Qdabra-test\",\"sort_name\":\"Qdabra-test\",\"sort_name_auto_generate\":true,\"created_by\":\"mc2343\",\"last_modified_by\":\"mc2343\",\"create_time\":\"2017-09-20T21:18:04Z\",\"system_mtime\":\"2017-09-20T21:18:04Z\",\"user_mtime\":\"2017-09-20T21:18:04Z\",\"authorized\":true,\"is_display_name\":true,\"source\":\"local\",\"jsonmodel_type\":\"name_corporate_entity\",\"use_dates\":[]},\"title\":\"Qdabra-test\"}}}},\"resource\":{\"ref\":\"/repositories/16/resources/10704\"},\"parent\":{\"ref\":\"/repositories/16/archival_objects/2630005\"},\"has_unpublished_ancestor\":false}",
            "suppressed": false,
            "publish": false,
            "system_generated": false,
            "repository": "/repositories/16",
            "level_enum_s": [
                "item",
                "series",
                "collection"
            ],
            "extent_type_enum_s": [
                "phonograph records"
            ],
            "portion_enum_s": [
                "whole"
            ],
            "date_type_enum_s": [
                "inclusive"
            ],
            "label_enum_s": [
                "creation",
                "Scope and Contents"
            ],
            "instance_type_enum_s": [
                "mixed materials"
            ],
            "type_2_enum_s": [
                "item_barcode"
            ],
            "type_enum_s": [
                "scopecontent"
            ],
            "resource": "/repositories/16/resources/10704",
            "ref_id": "aspace_fc3a5e4e86ef7222d00da601d00fbb0c",
            "created_by": "mc2343",
            "last_modified_by": "mc2343",
            "user_mtime": "2017-09-20T21:23:36Z",
            "system_mtime": "2017-10-07T14:50:50Z",
            "create_time": "2017-09-20T21:23:36Z",
            "notes": "Disc No. 3072 scopecontent aspace_c9906edbfdb141a932d5508867d1e4f9 Scope and Contents",
            "level": "item",
            "summary": "Disc No. 3072",
            "top_container_uri_u_sstr": [
                "/repositories/16/top_containers/304148"
            ],
            "child_container_u_sstr": [
                "item_barcode 39002125006222"
            ],
            "ancestors": [
                "/repositories/16/archival_objects/2630005",
                "/repositories/16/resources/10704"
            ],
            "jsonmodel_type": "archival_object"
        }
    ],
    "facets": {
        "facet_queries": {},
        "facet_fields": {},
        "facet_dates": {},
        "facet_ranges": {},
        "facet_intervals": {}
    }
}
```
At this point you can either run another GET on the URI for top container (results.First().top_container_uri_u_sstr) OR you can use the json property and fetch `json.instances.First().sub_container.top_container._resolved.barcode` I personally avoid using the _resolved property - it was clearly added by developers to avoid making extra API calls when rendering search results and doesn't feel like it should be public. However, it does save quite a bit of time and can be very valuable when trying to maintain responsiveness.

##### (14) Box Number

MSSA ONLY

From the search result's json property get the `indicator` value of top container: `json.instances.sub_container.top_container._resolved.indicator`

##### (24) Item Count

This will apply to repositories *other* than MSSA and represents the number of child containers (sub containers) associated with a particular box (top container).

Somewhat counterintuitively you can't get to this information from the `top_container/[id]` endpoint - you'll need to do a search of the archival objects associated with that top container and then scan each to see if there is subcontainer information. Each different subcontainer barcode is an item to add to the count.

This is a case where looking at the staff interface can give you a hint on approaching an ASpace issue. For example, if you look at:  

[The record for this top container](https://testarchivesspace.library.yale.edu/top_containers/304148#search_embedded) you'll see that the browser polls the AO [search endpoint](https://testarchivesspace.library.yale.edu/search.js?filter_term%5B%5D=%7B%22top_container_uri_u_sstr%22%3A%22%2Frepositories%2F16%2Ftop_containers%2F304148%22%7D&listing_only=true) to opulate the linked records pane. This is basically the list of records you now need to parse for subcontainer information. 

##### (13) Series Number

This value applies to MSSA ONLY. 

Archival objects are arranged hierarchically and can have children. For the purpose of this value you're looking for a property called `component_id` - it exists only at the archival object which is an ancestor of the record in question that has a levl of "series" (there is a property called `level` which will equal "series" when you hit the top of the ancestry tree, see {{BaseUrl}}/repositories/{{ActiveRepository}}/archival_objects/2630005)

From the search results you can use the `ancestors` property to find the parent archival objects. This seems like a new value in this version of ASpace, but it's so much less cumbersome than the alternative (check the github repo I sent for the get series method) that I have to say give it a shot. Fetch the archival objects, grabbing the top parent (where `level` = "series", this will be multiple GET requests, start from the top) and then fetch the `component_id` property.

- If an arabic numeral, zero pad to 2 digits
- If a string, leave as-is

Another approach I haven't validated is to go to the search results' `_resolved` property for top_container (`json.instances.sub_container.top_container`) and grab the `identifier` value from the `series` property (`json.instances.sub_container.top_container._resolved.series.identifier`). This seems like it would be fastest and could minimize query activity.

##### (15) Folder Number

Folder number illustrates why you can't safely use the `indicator` properties without checking their type. MSSA has used `indicator_2`, which we've seen before as a barcode field, to hold folder information. To work around this you have to screen the property named `type_[int]` - if `type_2` = "item_barcode" then `indicator_2` will be a barcode. In this case you want to check each `type_*` for a value of "folder"

This will exist in `json.instances.sub_container` A (non-AV) example of what this might look like:

```json
{
"instances": [
        {
            "lock_version": 0,
            "created_by": "admin",
            "last_modified_by": "admin",
            "create_time": "2015-06-02T05:01:43Z",
            "system_mtime": "2015-06-02T05:01:43Z",
            "user_mtime": "2015-06-02T05:01:43Z",
            "instance_type": "mixed_materials",
            "jsonmodel_type": "instance",
            "is_representative": false,
            "sub_container": {
                "lock_version": 0,
                "indicator_2": "1",
                "created_by": "admin",
                "last_modified_by": "admin",
                "create_time": "2015-06-02T05:01:43Z",
                "system_mtime": "2015-06-02T05:01:43Z",
                "user_mtime": "2015-06-02T05:01:43Z",
                "type_2": "folder",
                "jsonmodel_type": "sub_container",
                "top_container": {
                    "ref": "/repositories/12/top_containers/170091"
                }
            }
        }
    ]
}
```

#### Collection Information (lower right)

##### (22) Collection Name

At last, an easy one. You'll use the same process as (4) but this time you're grabbing the `title` property's value directly.

##### (12) Call Number

You'll also have to use the resource object that you fetched to get (10). The resource will have one or more properties of the form `id_[integer]` - there can be up to 4 (I believe). You will need to concatenate these in order, separating each value with an underscore (_). Any spaces in these values should also be replaced by underscores. E.g., in a record containing an `id_0` and `id_1` with values "Val1" and "Val2" respectively you'll want to concatenate them as `Val1_Val2`.

### Remaining questions

I've provided what I hope is a good starting point for the data points I understand. I'm still not clear on what a few of the lines are and whether they have been finalized or not. I have reproduced my remaining questions below.

#### 16-18 - Preservica

Each of these are marked for followup with Preservica. It would be ideal to finish these off or mark them as unneeded.

#### 20 - Dates

Mark reminds us that there are many possible dates, including at the resource level - it's unclear whether this is needed in ICE.
