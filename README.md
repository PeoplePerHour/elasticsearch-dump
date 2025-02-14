elasticdump
==================

[![Join the chat at https://gitter.im/taskrabbit/elasticsearch-dump](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/taskrabbit/elasticsearch-dump?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Tools for moving and saving indicies.

![picture](https://raw.github.com/taskrabbit/elasticsearch-dump/master/elasticdump.jpg)

[![Nodei stats](https://nodei.co/npm/elasticdump.png?downloads=true)](https://npmjs.org/package/elasticdump)

[![Build Status](https://secure.travis-ci.org/taskrabbit/elasticsearch-dump.png?branch=master)](http://travis-ci.org/taskrabbit/elasticsearch-dump)  [![Code Climate](https://codeclimate.com/github/taskrabbit/elasticsearch-dump/badges/gpa.svg)](https://codeclimate.com/github/taskrabbit/elasticsearch-dump)

## Installing

(local)
```bash
npm install elasticdump
./bin/elasticdump
```

(global)
```bash
npm install elasticdump -g
elasticdump
```

## Use

elasticdump works by sending an `input` to an `output`.  Both can be either an elasticsearch URL or a File.

Elasticsearch:
- format:  `{protocol}://{host}:{port}/{index}`
- example: `http://127.0.0.1:9200/my_index`

File:
- format:  `{FilePath}`
- example: `/Users/evantahler/Desktop/dump.json`

Stdio:
- format: stdin / stdout
- format: `$`

You can then do things like:

```bash
# Copy an index from production to staging with mappings:
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=http://staging.es.com:9200/my_index \
  --type=mapping
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=http://staging.es.com:9200/my_index \
  --type=data

# Backup index data to a file:
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=/data/my_index_mapping.json \
  --type=mapping
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=/data/my_index.json \
  --type=data

# Backup and index to a gzip using stdout:
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=$ \
  | gzip > /data/my_index.json.gz

# Backup ALL indices, then use Bulk API to populate another ES cluster:
elasticdump \
  --all=true \
  --input=http://production-a.es.com:9200/ \
  --output=/data/production.json
elasticdump \
  --bulk=true \
  --input=/data/production.json \
  --output=http://production-b.es.com:9200/

# Backup the results of a query to a file
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=query.json \
  --searchBody '{"query":{"term":{"username": "admin"}}}'
```

## Options

```
Usage: elasticdump --input [SOURCE] --output [DESTINATION] [OPTIONS]

--input                       
                    Source location (required)
--output                      
                    Destination location (required)
--limit                       
                    How many objects to move in bulk per operation 
                    (default: 100)
--debug                       
                    Display the elasticsearch commands being used 
                    (default: false)
--type                        
                    What are we exporting? 
                    (default: data, options: [data, mapping])
--delete                      
                    Delete documents one-by-one from the input as they are 
                    moved.  Will not delete the source index
                    (default: false)
--searchBody                  
                    Preform a partial extract based on search results 
                    (when ES is the input, 
                      default: '{"query": { "match_all": {} } }')
--sourceOnly                  
                    Output only the json contained within the document _source 
                      Normal: {"_index":"","_type":"","_id":"", "_source":{SOURCE}}
                      sourceOnly: {SOURCE}
                      default: false
--jsonLines                  
                    Do not include leading '[', trailing ']' and separating ',' chararacters in output
                      Note: Most useful in conjunction with sourceOnly to create a file of a single JSON entry per line
                      default: false
--all                         
                    Load/store documents from ALL indexes 
                    (default: false)
--bulk                        
                    Leverage elasticsearch Bulk API when writing documents 
                    (default: false)
--ignore-errors               
                    Will continue the read/write loop on write error 
                    (default: false)
--scrollTime                  
                    Time the nodes will hold the requested search in order. 
                    (default: 10m)
--maxSockets                  
                    How many simultaneous HTTP requests can we process make? 
                    (default: 
                      5 [node <= v0.10.x] / 
                      Infinity [node >= v0.11.x] )
--bulk-use-output-index-name  
                    Force use of destination index name (the actual output URL)
                    as destination while bulk writing to ES. Allows 
                    leveraging Bulk API copying data inside the same 
                    elasticsearch instance. 
                    (default: false)
--timeout                     
                    Integer containing the number of milliseconds to wait for 
                    a request to respond before aborting the request. Passed 
                    directly to the request library. If used in bulk writing, 
                    it will result in the entire batch not being written. 
                    Mostly used when you don't care too much if you lose some
                    data when importing but rather have speed.
--skip
                    Integer containing the number of rows you wish to skip
                    ahead from the input transport.  When importing a large
                    index, things can go wrong, be it connectivity, crashes,
                    someone forgetting to `screen`, etc.  This allows you to
                    start the dump again from the last known line written (as
                    logged by the `offset` in the output).  Please be advised
                    that since no sorting is specified when the dump is
                    initially created, there's no real way to guarantee that
                    the skipped rows have already been written/parsed.  This is
                    more of an option for when you want to get most data as
                    possible in the index without concern for losing some rows
                    in the process, similar to the `timeout` option.
--inputTransport    
                    Provide a custom js file to us as the input transport
--outputTransport   
                    Provide a custom js file to us as the output transport
--toLog
                    When using a custom outputTransport, should log lines 
                    be appended to the output stream? 
                    (default: true, except for `$`)
--help
                    This page
```

## Elasticsearch's scan and scroll method
Elasticsearch provides a scan and scroll method to fetch all documents of an index. This method is much safer to use since
it will maintain the result set in cache for the given period of time. This means it will be a lot faster to export the data
and more important it will keep the result set in order. While dumping the result set in batches it won't export duplicate
documents in the export. All documents in the export will unique and therefore no missing documents.

NOTE: only works for output

## Notes

- this tool is likley to require Elasticsearch version 1.0.0 or higher
- elasticdump (and elasticsearch in general) will create indices if they don't exist upon import
- when exporting from elasticsearch, you can have export an entire index (`--input="http://localhost:9200/index"`) or a type of object from that index (`--input="http://localhost:9200/index/type"`).  This requires ElasticSearch 1.2.0 or higher
- we are using the `put` method to write objects.  This means new objects will be created and old objects with the same ID will be updated
- the `file` transport will overwrite any existing files
- If you need basic http auth, you can use it like this: `--input=http://name:password@production.es.com:9200/my_index`
- if you choose a stdio output (`--output=$`), you can also request a more human-readable output with `--format=human`
- if you choose a stdio output (`--output=$`), all logging output will be suppressed
- when using the `--bulk` option, aliases will be ignored and the documents you write will be linked thier original index name.  For example if you have an alias "events" which contains "events-may-2015" and "events-june-2015" and you bulk dump from one ES cluster to another `elasticdump --bulk --import http://localhost:9200/events --output http://other-server:9200`, you will have the source indicies, "events-may-2015" and "events-june-2015", and not "events".
 
Inspired by https://github.com/crate/elasticsearch-inout-plugin and https://github.com/jprante/elasticsearch-knapsack

Built at [TaskRabbit](https://www.taskrabbit.com)
