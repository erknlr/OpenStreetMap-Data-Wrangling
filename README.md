
# OpenStreetMap - Data Wrangling

In order to brush up my data wrangling skills I would like to work with OpenStreetMap data for **San Francisco**. As a technology enthusiast, this dataset is the perfect opportunity for me to get somewhat familiar with the city. I will first use python to clean up/structure data and then use **NoSQL/MongoDB** to store it and run queries on it.


## Plan of Attack

I will be taking the following steps:

- Getting a general idea about the file.
- Auditing for flaws and coming up with cleaning approaches.
- Reshaping the data into an easy to work with python dictionary format.
- Converting the new dictionary format to JSON.
- Importing the JSON file to MongoDB.
- Running queries and discovering the dataset.
- Coming up with additional ideas about the dataset.


```python
##First thing first, let's import all of the libraries that will be necessary in our wrangling process: 

import os
import pymongo
import xml.etree.cElementTree as ET
import json
from collections import defaultdict
import bson
import pprint
import re
import codecs
import os.path
import sys
import requests
```

## Getting a general idea about the file

The dataset for San Francisco can be obtained here:
 
https://mapzen.com/data/metro-extracts/metro/san-francisco_california/

The zipped bz2 file is **87.1 MB** and the unzipped OSM file is **1.4 GB** big.

Before diving into auditing, it is a good idea to get a general impression about the data that we have. So let's first start by investigating the different tags and their number of occurences in the data:


```python
def count_num_tags(file_name):
        num_tags = {}
        for event, elem in ET.iterparse(file_name):
            if elem.tag in num_tags:
                num_tags[elem.tag] +=1
            else:
                num_tags[elem.tag] = 1
        return sorted(num_tags.items(), key = lambda x: x[1], reverse=True)         
```


```python
pprint.pprint(count_num_tags('san-francisco_california.osm'))
```

    [('nd', 7768878),
     ('node', 6554636),
     ('tag', 2120322),
     ('way', 806770),
     ('member', 54869),
     ('relation', 6211),
     ('bounds', 1),
     ('osm', 1)]


Going one step further, I would like to also see the 5 most frequent occurences of the tag keys:


```python
def audit_tag_keys(file_name):
        num_keys = {}
        for event, elem in ET.iterparse(file_name):
            for attr in elem.iter("tag"):
                key = attr.attrib["k"]

                if key not in num_keys:
                    num_keys[key] = 1
                else:
                    num_keys[key] += 1

        return sorted(num_keys.items(), key = lambda x: x[1], reverse=True)
```


```python
pprint.pprint(audit_tag_keys('san-francisco_california.osm')[0:5])
```

    [('building', 2022663),
     ('height', 430539),
     ('highway', 385911),
     ('name', 270051),
     ('addr:housenumber', 191157)]


## Auditing for flaws and coming up with cleaning approaches

One thing that I would like to audit (and correct if neccesary) are the address indicators like *"Street", "Avenue"*. Initially, I am interested in all non-usual indicators besides:

- Street
- Avenue
- Boulevard
- Drive
- Court
- Place
- Square
- Lane
- Road
- Trail
- Parkway
- Commons
- Way
- Highway
- Circle
- Terrace
- Alley
- Plaza
- Braodway
- Bridgeway
- Mall
- Walk
- Center
- Park
- Route
- View
- Bridge


```python
# I will use the following regex to catch the word endings:
street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)

# These are my standard list of indicators. I am interested in values besides these.
expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road", 
            "Trail", "Parkway", "Commons", "Way", "Highway", "Circle", "Terrace", "Alley", "Plaza", "Broadway", 
            "Bridgeway", "Mall", "Walk", "Center", "Park", "Route", "Pier", "View", "Bridge"]

# Function to catch unusual indicators
def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in expected:
            street_types[street_type].add(street_name)
            
# Function to look only for addr:street keys             
def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")

# Main function to combine the 2 functions above and create a list of unique values that are not in "expected" 
def audit(osmfile):
    osm_file = open(osmfile, "r")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):

        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types
```

Let's use a sample of the map and take a look at which words are not matching our expected group:


```python
pprint.pprint(dict(audit('sample_san_francisco_100.osm')))
```

    {'105': set(['Grand Avenue #105']),
     'Alameda': set(['Alameda']),
     'Ave': set(['Tehama Ave']),
     'D': set(['Avenue D']),
     'Gardens': set(['Wildwood Gardens']),
     'Real': set(['El Camino Real']),
     'St': set(['Kearny St'])}


Looking at the results, we have to address the following cases:

- **Abbreviations**: They need to be replaced with the full version. (eg. "Blvd")
- **Typos**: They need to be corrected. (eg. "Avenie")
- **Capital/Non-Capital Letters**: They need to be standardized. (eg. "way")
- ** Additional Characters**: Some indicators have additional letters & numbers at the end. Those could be either shortened or expanded. (eg. "Grand Avenue #105" => "Grand Avenue")



```python
# Our mapping for corrections:
mapping = { 
            "ave.": "Avenue",
            "Ave.": "Avenue",
            "Ave": "Avenue",
            "AVE": "Avenue",
            "avenue": "Avenue",
            "Avenie": "Avenue",
            "Blvd": "Boulevard",
            "Blvd,": "Boulevard",
            "blvd": "Boulevard",
            "Ctr": "Center",
            "dr": "Drive",
            "Dr": "Drive",    
            "Hwy": "Highway",
            "rd": "Road",
            "rd.": "Road",
            "Rd.": "Road",       
            "ln": "Lane",
            "ln.": "Lane",
            "Ln.": "Lane",
            "st": "Street",
            "St": "Street",
            "St.": "Street",
            "square": "Square",
            "plz": "Plaza",
            "Pl": "Plaza",
            "Plz": "Plaza",
            "way": "Way",
            "Woodside Road, Suite 100": "Woodside Road",
            "Grand Avenue #105": "Grand Avenue",
            "N California Blvd": "N California Boulevard",    
            "Doolittle Dr, Suite 15": "Doolittle Drive",    
            "Brannan St #151": "Brannan Street",    
            "University Ave #155": "University Avenue",    
            "Woodside Road, Suite 155": "Woodside Road",
            "San Francisco Bicycle Route 2": "San Francisco Bicycle Route",
            "Bartlett Street #203": "Bartlett Street",
            "University Dr. Suite 3": "University Drive",
            "Mission Blvd #302": "Mission Boulevard",
            "Sansome St #3500": "Sansome Street",
            "Market Street Suite 3658": "Market Street",
            "Pier 39": "Pier 39 Pier",
            "Menlo Ave  # 4": "Menlo Avenue",
            "16th St #404": "16th Street",
            "Sansome Street Ste 730": "Sansome Street",
            "Sansome Street Suite 730": "Sansome Street",
            "Pier 50 A": "Pier 50 A Pier",
            "San Francisco Internation Airport": "San Francisco International Airport",
            "Avenue B": "Avenue B Avenue",
            "Pier 50 B": "Pier 50 B Pier",
            "Avenue D": "Avenue D Avenue",
            "Marina Boulevard Building D": "Marina Boulevard Building D Boulevard",
            "Avenue E": "Avenue E Avenue",
            "Francisco Boulevard East": "Francisco Boulevard East Boulevard",
            "Sir Francis Drake Boulevard East": "Sir Francis Drake Boulevard East Boulevard",
            "Avenue F": "Avenue F Avenue",
            "Avenue G": "Avenue G Avenue",
            "Ne Quad I-280 / Edgewood Rd Ic": "Ne Quad I-280 / Edgewood Road",
            "Nw Quad Sr 84 / Ardenwood Blvd Ic": "Nw Quad Sr 84 / Ardenwood Blvd Ic Boulevard",
            "Cabrillo Highway North": "Cabrillo Highway",
            "Mission Bay Boulevard North": "Mission Bay Boulevard",
            "Avenue Del Ora": "Avenue Del Ora Avenue",
            "Buena Vista Avenue West": "Buena Vista Avenue West Avenue"
          }

# Function to update the names
def update_name(name, mapping):
     if name in mapping:
        return mapping[name]
     else:  
        name_list = re.findall(r"[\w']+", name)
        end_of_street_name = len(name_list)  
        for i in range(len(name_list)):
            word = name_list[i]
            if word in mapping:
                end_of_street_name = i
                name_list[i] = mapping[word]  
                
        name_list = name_list[:(end_of_street_name+1)]
        corrected_name = ' '.join(name_list)
        return corrected_name
```

Let's now try the function on a small sample once to see if our functions are working properly:


```python
for _, element in ET.iterparse('sample_san_francisco_100.osm'):
    for tag in element.iter("tag"):
        if tag.attrib['k'] == "addr:street":
            if tag.attrib['v'] <> update_name(tag.attrib['v'], mapping):
                print tag.attrib['v'], "=>", update_name(tag.attrib['v'], mapping)

```

    Alvarado-Niles Road => Alvarado Niles Road
    Alvarado-Niles Road => Alvarado Niles Road
    Avenue D => Avenue D Avenue
    Avenue D => Avenue D Avenue
    Grand Avenue #105 => Grand Avenue
    Grand Avenue #105 => Grand Avenue
    Kearny St => Kearny Street
    Kearny St => Kearny Street
    Tehama Ave => Tehama Avenue
    Tehama Ave => Tehama Avenue
    Tehama Ave => Tehama Avenue
    Tehama Ave => Tehama Avenue
    Alvarado-Niles Road => Alvarado Niles Road
    Avenue D => Avenue D Avenue
    Grand Avenue #105 => Grand Avenue
    Kearny St => Kearny Street
    Tehama Ave => Tehama Avenue
    Tehama Ave => Tehama Avenue


It seem to be working as intended. As next, let's use a sample of the map and investigate if all of the zip codes are entered properly and accurately:


```python
def show_zipcodes(file_name):
        num_tags = {}
        for event, elem in ET.iterparse(file_name):
            if elem.tag == "node" or elem.tag == "way":
                for tag in elem.iter("tag"):
                    if tag.attrib['k'] == "addr:postcode":
                        if tag.attrib['v'] in num_tags:
                            num_tags[tag.attrib['v']] += 1
                        else:
                            num_tags[tag.attrib['v']] = 1
        return sorted(num_tags.items(), key = lambda x: x[1], reverse=True) 
                    
    
pprint.pprint(show_zipcodes('sample_san_francisco_100.osm'))
```

    [('94122', 46),
     ('94611', 34),
     ('94116', 20),
     ('94117', 13),
     ('94610', 11),
     ('94133', 9),
     ('94118', 8),
     ('94103', 7),
     ('94127', 6),
     ('94587', 5),
     ('94109', 4),
     ('94061', 4),
     ('94030', 3),
     ('94121', 3),
     ('94102', 3),
     ('94134', 2),
     ('94706', 2),
     ('94704', 2),
     ('94019', 2),
     ('94114', 2),
     ('94115', 2),
     ('94080', 2),
     ('94108', 2),
     ('94063', 2),
     ('94501', 1),
     ('94621', 1),
     ('94131', 1),
     ('94015', 1),
     ('94709', 1),
     ('94703', 1),
     ('94560', 1),
     ('94112', 1),
     ('94110', 1),
     ('94578', 1),
     ('94606', 1),
     ('94002', 1),
     ('94027', 1),
     ('94710', 1),
     ('94577', 1),
     ('94804', 1),
     ('94104', 1),
     ('94608', 1),
     ('94132', 1),
     ('94965', 1)]


In the sample dataset, the zipcodes are okay. However, in the full dataset, there are some wrong ones, that we need to address. 
- **CA abbreviation**: Regular zip codes should consist of 5 digits. There are some with California abbreviation. We need to get rid of these.
- **More than 5 digits**: There are some, with more than 5 digits. 
- **First two digits**: All San Francisco zip codes should start with 94. However, there are some which start with other digits probably due to a typo.


```python
def correct_zip(zipcode):
    if 'CA' in re.findall('[a-zA-Z]*', zipcode) or 'ca' in re.findall('[a-zA-Z]*', zipcode):
        if re.findall('[0-9]+', zipcode):
            if re.findall('[0-9]+', zipcode)[0][0:2] == '94' and len(re.findall('[0-9]+', zipcode)[0]) == 5:
                return re.findall('[0-9]+', zipcode)[0]
        else:
            return None
    else:
        if re.findall('[0-9]+', zipcode):
            if re.findall('[0-9]+', zipcode)[0][0:2] == '94' and len(re.findall('[0-9]+', zipcode)[0]) == 5:
                return re.findall('[0-9]+', zipcode)[0]
        else:
            return None
```

Now, let's try the function over a small sample to see if our function is working properly:


```python
for _, element in ET.iterparse('sample_san_francisco_10.osm'):
    for tag in element.iter("tag"):
        if tag.attrib['k'] == "addr:postcode":
            if tag.attrib['v'] <> correct_zip(tag.attrib['v']):
                print(tag.attrib['v'], "=>", correct_zip(tag.attrib['v']))
```

    ('95430', '=>', None)
    ('95430', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95430', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)
    ('95498', '=>', None)


Ok the code seems to be working properly. We can use the two functions once we are reshaping the data.   

## Reshaping the data into an easy to work with python dictionary format.

So as next, I would like to take my cleaned data and turn it into an easy to work with dictionary format, which I can later convert to JSON format. The data should have the following structure at the end:

```json
{
"id": "2406124091",
"type: "node",
"visible":"true",
"created": {
          "version":"2",
          "changeset":"17206049",
          "timestamp":"2013-08-03T16:43:42Z",
          "user":"linuxUser16",
          "uid":"1219059"
        },
"pos": [41.9757030, -87.6921867],
"address": {
          "housenumber": "5157",
          "postcode": "60625",
          "street": "North Lincoln Ave"
        },
"amenity": "restaurant",
"cuisine": "mexican",
"name": "La Cabana De Don Luis",
"phone": "1 (773)-271-5176"
}
```

To do this, I will use: 


```python
lower = re.compile(r'^([a-z]|_)*$')
lower_colon = re.compile(r'^([a-z]|_)*:([a-z]|_)*$')
problemchars = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')


CREATED = [ "version", "changeset", "timestamp", "user", "uid"]

def shape_element(element):
    node = {}
    node["created"]={}
    node["address"]={}
    node["pos"]=[]
    refs=[]
    
    if element.tag == "node" or element.tag == "way" :
        if "id" in element.attrib:
            node["id"]=element.attrib["id"]
        node["type"]=element.tag

        if "visible" in element.attrib.keys():
            node["visible"]=element.attrib["visible"]
      
        # I will use the "CREATED" list as my keys to create a dict within my dict
        for elem in CREATED:
            if elem in element.attrib:
                node["created"][elem]=element.attrib[elem]
                
        # I will populate the node["pos"] list with longitude and lattitude records       
        if "lat" in element.attrib:
            node["pos"].append(float(element.attrib["lat"])) 
        if "lon" in element.attrib:
            node["pos"].append(float(element.attrib["lon"]))

        # I will iterate through "tag" elements to 
        for tag in element.iter("tag"):
            if not(problemchars.search(tag.attrib['k'])):
                if tag.attrib['k'] == "addr:housenumber":
                    node["address"]["housenumber"]=tag.attrib['v']
                
                # Using the function I wrote before to correct the zipcodes    
                if tag.attrib['k'] == "addr:postcode":
                    node["address"]["postcode"]=correct_zip(tag.attrib['v'])
                
                # Using the function I wrote before to correct the street indicators       
                if tag.attrib['k'] == "addr:street":
                    node["address"]["street"] = update_name(tag.attrib['v'], mapping)

                if tag.attrib['k'].find("addr")==-1:
                    node[tag.attrib['k']]=tag.attrib['v']
                    
        for nd in element.iter("nd"):
             refs.append(nd.attrib["ref"])
                
        if node["address"] =={}:
            node.pop("address", None)

        if refs != []:
           node["node_refs"]=refs
            
        return node
    else:
        return None
```

## Converting the new dictionary format to JSON

After reshaping the dict, converting the file to JSON is fairly easy:


```python
def process_map(file_in, pretty = False):
    '''
    This function process the xml openstreetmap file, 
    write a json out file and return a list of dictionaries.
    '''
    file_out = "{0}.json".format(file_in)
    data = []
    with codecs.open(file_out, "w") as fo:
        for _, element in ET.iterparse(file_in):
            el = shape_element(element)
            if el:
                data.append(el)
                if pretty:
                    fo.write(json.dumps(el, indent=2)+"\n")
                else:
                    fo.write(json.dumps(el) + "\n")
    return data
```


```python
processed_data = process_map('san-francisco_california.osm', True)
```

## Importing the JSON file to MongoDB

As next, I will import the json file that I created to MongoDB:


```python
from pymongo import MongoClient

client = MongoClient()
db = client.osm
collection = db.sf
collection.insert_many(processed_data)
```




    <pymongo.results.InsertManyResult at 0x1048df410>




```python
collection
```




    Collection(Database(MongoClient(host=['localhost:27017'], document_class=dict, tz_aware=False, connect=True), u'osm'), u'sf')



## Running queries and discovering the dataset

So, I successfully imported our database/collection. As next, let's check the size of the database:


```python
print('Statistics about the database:')
db.command("dbstats")
```

    Statistics about the database:





    {u'avgObjSize': 236.82271172110327,
     u'collections': 1,
     u'dataSize': 3486696262.0,
     u'db': u'osm',
     u'indexSize': 148475904.0,
     u'indexes': 1,
     u'numExtents': 0,
     u'objects': 14722812,
     u'ok': 1.0,
     u'storageSize': 1059987456.0,
     u'views': 0}



And, let's see how many documents are there in our collection:


```python
print('Number of documents in the collection: ') 
print(collection.find().count())
```

    Number of documents in the collection: 
    14722812


As next, let's discover the number of unique users as well as number of nodes and ways in our collection:


```python
print('Number of unique users contributed to the map:')
print(len(collection.distinct("created.user")))

print ('Number of nodes in the map: ')
print (collection.find({"type":"node"}).count())

print ('Number of ways in the map: ')
print (collection.find({"type":"way"}).count())
```

    Number of unique users contributed to the map:
    2736
    Number of nodes in the map: 
    13109100
    Number of ways in the map: 
    1613456


Now, let's see the most contributed 20 users to the map:


```python
print("Most contributed 20 users:")
pprint.pprint(list(collection.aggregate(
    [{"$group":{"_id": "$created.user", "count": {"$sum": 1}}},
     {"$sort": {"count": -1}},
     {"$limit": 20}]                 
)))
```

    Most contributed 20 users:
    [{u'_id': u'andygol', u'count': 2995594},
     {u'_id': u'ediyes', u'count': 1776876},
     {u'_id': u'Luis36995', u'count': 1326102},
     {u'_id': u'dannykath', u'count': 1081682},
     {u'_id': u'RichRico', u'count': 808878},
     {u'_id': u'Rub21', u'count': 767318},
     {u'_id': u'calfarome', u'count': 371184},
     {u'_id': u'oldtopos', u'count': 333324},
     {u'_id': u'KindredCoda', u'count': 297044},
     {u'_id': u'karitotp', u'count': 270136},
     {u'_id': u'samely', u'count': 251124},
     {u'_id': u'abel801', u'count': 216510},
     {u'_id': u'oba510', u'count': 192422},
     {u'_id': u'DanHomerick', u'count': 183836},
     {u'_id': u'nikhilprabhakar', u'count': 174340},
     {u'_id': u'Jothirnadh', u'count': 118800},
     {u'_id': u'dchiles', u'count': 113764},
     {u'_id': u'nmixter', u'count': 113750},
     {u'_id': u'saikabhi', u'count': 113328},
     {u'_id': u'hmkandrey', u'count': 110494}]


Let's see the 20 most popular cuisine sorts:


```python
print("Most popular 20 cuisine sorts:")
pprint.pprint(list(collection.aggregate(
    [{"$match": {"amenity":{"$exists": 1}, "amenity": "restaurant", "cuisine": {"$exists":1}}},
     {"$group": {"_id": "$cuisine", "count": {"$sum": 1}}},
     {"$sort": {"count": -1}},
     {"$limit": 20}]
)))
```

    Most popular 20 cuisine sorts:
    [{u'_id': u'mexican', u'count': 464},
     {u'_id': u'chinese', u'count': 362},
     {u'_id': u'pizza', u'count': 356},
     {u'_id': u'japanese', u'count': 308},
     {u'_id': u'italian', u'count': 288},
     {u'_id': u'american', u'count': 236},
     {u'_id': u'thai', u'count': 222},
     {u'_id': u'vietnamese', u'count': 158},
     {u'_id': u'indian', u'count': 134},
     {u'_id': u'sushi', u'count': 132},
     {u'_id': u'burger', u'count': 124},
     {u'_id': u'sandwich', u'count': 110},
     {u'_id': u'asian', u'count': 108},
     {u'_id': u'seafood', u'count': 92},
     {u'_id': u'french', u'count': 64},
     {u'_id': u'mediterranean', u'count': 48},
     {u'_id': u'korean', u'count': 42},
     {u'_id': u'regional', u'count': 40},
     {u'_id': u'greek', u'count': 38},
     {u'_id': u'ice_cream', u'count': 34}]


And, most common 5 religion:


```python
print('Most common 5 religion:')
pprint.pprint(list(collection.aggregate([
    {"$match": {"amenity": {"$exists": 1}, "amenity": "place_of_worship", "religion": {"$exists":1}}},
    {"$group": {"_id": "$religion", "count":{"$sum": 1}}},
    {"$sort": {"count": -1}},
    {"$limit": 5}]
)))
```

    Most common 5 religion:
    [{u'_id': u'christian', u'count': 2008},
     {u'_id': u'buddhist', u'count': 70},
     {u'_id': u'jewish', u'count': 36},
     {u'_id': u'muslim', u'count': 16},
     {u'_id': u'unitarian_universalist', u'count': 8}]


As last, let's check 5 most popular sport types:


```python
print('Most popular 5 sport sorts:')
pprint.pprint(list(collection.aggregate([
    {"$match": {"sport": {"$exists": 1}}},
    {"$group": {"_id": "$sport", "count": {"$sum": 1}}},
    {"$sort": {"count": -1}},
    {"$limit": 5}]
)))
```

    Most popular 5 sport sorts:
    [{u'_id': u'tennis', u'count': 1368},
     {u'_id': u'baseball', u'count': 874},
     {u'_id': u'basketball', u'count': 856},
     {u'_id': u'soccer', u'count': 300},
     {u'_id': u'swimming', u'count': 152}]


## Coming up with additional ideas about the dataset

The open street map dataset for San Francisco is actually pretty well structured. However, there are some flaws, which might be avoided. 

While controlling the `zip codes`, we saw that there were zipcodes with "CA" abbreviation or an additional number attached at the end. As the proper zipcodes should consist of 5 digits, it might be a good idea to make it compulsory to enter only 5 digits and nothing else. 
Similary, we saw many wrong zipcode entries such as '95121'. Knowing that the zipcodes should start with '94', the contributer could be made aware of this mistake while making the entry.
The problem here is, that the Open Street Map is not limited to San Francisco only. That means, users can make their input on any of the cities of their choosing. In that sense, the system has to recognize the city that is being defined, check against a database for legit zipcodes and make this limitation. All in all, such system requires a lot of effort and probably is more work than, correcting the wrong zipcodes afterwards.  

A similar mistake we saw was also with the `street indicators`. Many indicators were written sometimes in full form, sometimes with abbreviations. Standardizing these entries might save a great deal of time for those who would like to analyse this map. However, this might be really difficult to achieve. As an address does not always have the same components even within the same city, the developers are forced to leave this field as free text. Trying to standardize this might be really difficult and would probably not scale once we start moving across different countries with different address systems, languages and etc.    

There are also several address entries that we couldn't correct programmatically(due to wrong entries, etc.) These could be identified and corrected by humans manually in order to make the map complete. 
