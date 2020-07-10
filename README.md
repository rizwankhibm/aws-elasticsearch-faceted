# aws-elasticsearch-faceted

AWS Elasticsearch – Impelementation of Faceted Search 	

Scope: Elastic search as a standalone module(Works as a service) that can be enabled for each module/component of the Assets.
    1. Web Service for feeding data by indexing multi-level tags to AWS elasticsearch. 	
    2. Faceted search will happen on elasticsearch as mentioned in the screenshot. Hierarchy of Data is listed on the left. The hierarchy meta tags (on selection) 	are searched in elasticsearch since data is indexed based on hierarchy (multi-level tags). 	
    3. Highlighting of searched keywords is achieved 	
    4. Page ranking based on the searched key based on “Name of procedure” & “description” of the asset video is taken care
    5. Text search will be a full text search	 	 	
    6. Incremental Indexing - Re-Indexing.
    7. File Indexing
    8. Example - Sample Data 
1. Feeding Data/Content in Elasticsearch
As per we have the following Hierarchy of Data.
Hierarchy 1 – Taxonomy:
- Internal Medicine
- Psychiatry
- Pediatrics
- Surgery
    -- Abdomen
       --- General
       --- Lever
       --- Pancreas
- Medical Education
Hierarchy 2:
- Men
- Women
- Pediatrics
- Geriatrics
Elastic Search Indexing: The approaches mentioned below are open for discussion. These approaches will be finalized only after thorough testing and implementations on ES server.

Approach 1: Multi-level indexes or nested indexes
Entity Diagram:

As per the hierarchy we have relation between each entity as follow below:
    • One speciality has many topics
    • One topic has many sub-topic/sub-category
Idea is to make use of nested relations in Index & to use Aggregations in elasticsearch to do searching.
Index mapping: {  "mappings": {	
"properties": { 	"id": { "type": "keyword" },  	"assetCode": { "type": "keyword"},
    		                             "name": { "type": "keyword" }, "description": { "type": "keyword"},
             		"speciality": {  "type": "nested",
      		"properties": {  "name": { "type": "keyword"},  
"topic": { "type": "nested", 
      “properties”: {  "name": { "type": "keyword"},
       		 		"sub_topic": { "type": "nested",
         		 		  "properties": {  "name": {"type": "keyword"} }
        				}
      			}
    }  }  }
Reference: https://iridakos.com/programming/2018/10/22/elasticsearch-bucket-aggregations#aggregation-request-format 

Approach 2:  Using the built-in Path tokenizer. Here a single field holds the hierarchy values and while indexing we specify the “path tokenizer” mapping setting for that field.  For example, a single field would hold the comma separated value and path tokenizer would produce the token values as –
   Surgery, Abdomen, General  And produces tokens: Surgery |  Surgery,Abdomen | Surgery,Abdomen,General
In this approach, We are taking the actual user-readable values of categories instead of Ids, it’s only for example sake, as taking Ids gives you the flexibility to update the category names like Surgery, Abdomen etc associated with that Id.
Indexing: PUT _index/
{    "settings": {
	"analysis": {   "analyzer": { "path-analyzer": {   "type": "custom", "tokenizer": "path-tokenizer"  }},
 	"tokenizer": { "path-tokenizer": { "type": "path_hierarchy", "delimiter": "," } }
        }  },
     "mappings": { "my_type": {  "dynamic": "strict",
   	 "properties": { "hierarchy_path": {
         	 "type": "string","analyzer": "path-analyzer",
         	 “search_analyzer": "keyword"
        	}   }
       } }
   }
We have added a document with field hierarchy_path having an example of two set of categories and each set having comma separated values. We make use of terms aggregation on field “hierarchy_path” to get faceted search. Sample is as mentioned below:
GET _search:  { 
 	 "aggs": {
   "category": { "terms": { "field": "hierarchy_path", "size": 0 }
    }   
  }


We get the following buckets
"buckets": [ { "key": "Surgery", "doc_count": 1 }, 
                   { "key": "Surgery,Abdomen","doc_count": 1},
         	     { "key": "Surgery,Abdomen,General", "doc_count": 1}
 	]
Reference: https://sharing.luminis.eu/blog/faceted-search-with-elasticsearch/

3. Highlighting of searched keywords is achieved
It is possible to highlight the text searched in a result. 

Sample: search?q=Knee&highlight.plot={}
  "highlight": {
    "fields": {
      "plot": {}
    },
    "pre_tags": "<strong>",
    "post_tags": "</strong>",
    "fragment_size": 200,
    "boundary_chars": ".,!? "
  }

Reference: https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-searching.html

4.Page ranking/ Result Ranking based on the searched key based on “Name of procedure” & “description” of the asset video is taken care

Boosts the relevance score of documents based on the numeric value of a rank_feature or rank_features field.

Relevance Score => By default, Elasticsearch filters data on the basis of relevance score. which means how well each document matches a query. The relevance score is a number returned in _score meta-field of search API. The higher the _score, the more relevant the document. Each query calculates relevance scores differently. score calculation also depends on whether the query clause is run in a query or filter context.

Query Context => In Query Context, we can get the result or not on the basis of matching query, along with the query clause also calculates a relevance score in the _score meta-field. to use it we need to simply use query parameter in search process.

Filter Context => In Filter context we query the elasticsearch, we can get the result or not on the basis of matching query. no scores are calculated. Filter context is mostly used for filtering structured data.

Example of query and filter context.
GET /_search : 
{
     "query": {
    	"wildcard": {
        	    "name": { "value": "Knee*"  }
              }
       }
}
 
A. The query parameter indicates query context.

B. The book and 2 match clauses are used in query context, which means that they are used to score and how well each document matches.

C. The filter parameter indicates filter context. Its term and range clauses are used in filter context. They will filter out documents which do not match, but they will not affect the score for matching documents.

2. To calculate relevance scores based on rank feature field, the rank_feature query supports this Algorithms or mathematical functions. rank feature field uses saturation as default.

a) Saturation b) Logarithm c) Sigmoid

How to set a Rank Feature Example.

PUT /test
{  
   "mappings": {    
	"properties": {     
     	"id": { "type": "keyword" }, 	 
     	"assetCode": { "type": "keyword"},
   	 "name": { "type": "keyword", "boost": 2 },
     	"description": { "type": "keyword", "boost": 1 },
     	"speciality": {  
         		"type": "nested",
        		"properties": {  
            			"name": { "type": "keyword"},  
            			"topic": { "type": "nested",
            			   "properties": {  
                		   "name": { "type": "keyword"},
                		    "sub_topic": {
                    			"type": "nested",
                    		"properties": {  "name": {"type": "keyword"} }  }
            		}      }
        	}    }
    }   }
}


Boosting Example query
The following query searches for text ‘Knee’ and boosts relevance scores based on name field.

GET /test/_search
{
     "query": {
    	"wildcard": {
        	    "name": { "value": "Knee*"  }
              }
       }
}


Reference: https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-rank-feature-query.html#query-dsl-rank-feature-query 

6. Incremental Indexing: use case scenarios for Re-Indexing:

Re-indexing is very expensive. Better way is to create a new index and drop the old one. To achieve this with zero downtime, use index alias for all your customers. Think of an index called "data-version1". In steps:
    • create your index "data-version1" and give it an alias named "data"
    • only use the alias "data" in all your client applications
    • to update your mapping: create a new index (with the new mapping) called "data-version2" and put all your data in
    • to switch from version1 to version2: drop the alias "data" on version1 and create an alias "data" on version2 (or first create, then drop). the time in between those two steps your clients will have no (or double) data. but the time between dropping and creating an alias should be so short your clients shouldn't recognize it.
It's good practice to always use aliases.

Update/Edit Index Scenario:
Reindexing ES after editing mapping information
Reindexing is performed during the scenario when there is any change in mapping information in indices, for instance consider the scenario were type of a field in index have to be changed from float to string, the index update will not be possible since there is data with float values.
# Get mappings:
curl -XGET 'http://localhost:9200/library/books/_mapping'# Returns:
{   "library": { "mappings": {   	"books": {  "properties": { "price": {  "type": "string"}, "title": {"type": "string"}      }   	}       }    }   }
# Update mapping request:
curl -XPOST 'http://localhost:9200/library/_mapping/books' -d '
{   "properties": { "price": { "type": "float" 	}, "title": { "type": "string"	}  } }
# Returns an error:  
{   "error": {"root_cause": [ { "type": "illegal_argument_exception",
        		"reason": "mapper [price] of different type, current_type [string], merged_type [float]"    		   } ], "type": "illegal_argument_exception",
  	"reason": "mapper [price] of different type, current_type [string], merged_type [float]"}, 
"status": 400
}
During this case the best approach is to reindex the data by creating a new index with update mappings
curl -XPUT 'http://localhost:9200/new_library' -d '
{  	"settings": {"number_of_shards": 2,"number_of_replicas": 1   },
  	"mappings": {
		"books": {"properties": {"price": {"type": "float"},
    		"title": {  "type": "string","index" : "not_analyzed"} }
	}  }
}
after creating new index with updated mapping we can reindex the current data my copying old index content to new one
curl -XPOST 'http://localhost:9200/_reindex' -d '
{  "source": {"index": "library"  },  "dest": {“index": "new_library"  } }'
all the data available in old index will be copied to the new one with new mapping types
Reference: https://medium.com/@eyaldahari/reindex-elasticsearch-documents-is-easier-than-ever-103f63d411c  https://sematext.com/blog//reindexing-data-with-elasticsearch/    	   https://www.elastic.co/guide/en/elasticsearch/reference/7.0/docs-reindex.html 

7.File Indexing approach supported by ElasticSearch:
Most of the file content indexing could be achieved by making most out of apache tika. Basic document file types like .doc,docx,.xls,.ppt can be converted to text using apache tika and could be added to ES indexes as converted text.
We could make use of /tika api supported by apache tika to convert above mentioned document types to text representation and frame text representation as document format supported by ES.
https://ambar.cloud/blog/2017/10/24/ingesting-documents-into-es/
An independent apache tika accessible from ES installed EC2 instance should be set up to follow this approach.
https://hub.docker.com/r/apache/tika
https://tika.apache.org/1.12/gettingstarted.html

8. Sample data: example with ‘Knee’ keyword search query & faceted search result.

    • If we wanted aggregations for the speciality as a category, we will set this property to speciality.name
This query can help in building the faceted left menu with count.

Search Query:
{
     "size": 0,
     "aggs": {
     "speciality": {
     "nested": { "path": "speciality" },
      	"aggs": {
            	"name": {"terms": { "field": "speciality.name", "size": 50  } }
       	}
      }
   }
}

Result:
{
   "took":6,
   "timed_out":false,
   "_shards":{
  	"total":5,
  	"successful":5,
  	"skipped":0,
  	"failed":0
   },
   "hits":{
  	"total":{
     	"value":17,
     	"relation":"eq"
  	},
  	"max_score":null,
  	"Hits":[ ]
   },
   "aggregations":{
  	"speciality":{
     	"doc_count":21,
     	"name":{
        	   "doc_count_error_upper_bound":0,
        	   "sum_other_doc_count":0,
        	   "buckets":[
           		{"key":"Surgery", "doc_count":15},
           		{"key":"Neurology","doc_count":1},
           		{"key":"Pediatrics","doc_count":1},
           		{"key":"Psychiatry","doc_count":1},
           		{"key":"Skin and Soft Tissue","doc_count":1},
           		{"key":"Trauma","doc_count":1},
           		{"key":"Vascular","doc_count":1}
        	   ]
     	}
  	 }
       }
}


    • Query to search all Topics with count under ‘Surgery’ speciality.
Query: _search
   {
 	"size": 0,
  	"query" : {
     "nested" : {
    	"path" : "speciality",
    	"query" :  {
   		 "bool" : {
            	"must" : [ { "match" : { "speciality.name" : "Surgery" }} ]
        	}
    }
 	}
   	},
  	"aggs": {
	"speciality": {
  	"nested": { "path": "speciality" },
  	"aggs": {
    	"topic": {
      		"nested": { "path": "speciality.topic"},
      		"aggs": {
        			"name": {
          			"terms": { "field": "speciality.topic.name" }
        		}
      	}
    }
  }
	}
     }
}

Result:
{
   "took":23,
   "timed_out":false,
   "hits":{
  	"total":{"value":15,"relation":"eq"},
  	"max_score":null,
  	"hits":[ ]
   },
   "aggregations":{
  	"speciality":{
     	"doc_count":19,
     	"topic":{
        	"doc_count":18,
        	"name":{
           	"doc_count_error_upper_bound":0,
           	"sum_other_doc_count":0,
           	"buckets":[
              	{  "key":"Abdomen", "doc_count":15 },
              	{  "key":"Alimentrary Tract", "doc_count":2 },
              	{  "key":"Endoscopy", "doc_count":1 }
           	]
        }
     }
   }
  }
}

Ranking Query Sample:

Mapping
{  
	"mappings": {    
	"properties": {     
     	"id": { "type": "keyword" }, 	 
     	"assetCode": { "type": "keyword"},
   	"name": { "type": "keyword", "boost": 2 },
     	"description": { "type": "keyword", "boost": 1 },
     	"speciality": {  
         		"type": "nested",
        		"properties": {  
            		"name": { "type": "keyword"},  
            	"topic": { "type": "nested",
            	"properties": {  
                	"name": { "type": "keyword"},
                	"sub_topic": {
                    	"type": "nested",
                    	"properties": {  "name": {"type": "keyword"} }
                	}
            	}
            }
        	     }  
      	  }
         }
    }
}


Search Query
{
   "query": {
    	"wildcard": {
        		"name": {  "value": "Knee*" } 
    	}
     }
}

Result
{
	"took": 557,
	"timed_out": false,
	"hits": {
    	"total": {
        	"value": 2,
        	"relation": "eq"
    	},
    	"max_score": 1.0,
    	"hits": [
        	{
            	"_index": "my_custom_rank",
            	"_type": "_doc",
            	"_id": "1",
            	"_score": 1.0,
            	"_source": {
                	"id": "6319",
                	"assetCode": "CP4_VIDEO_5e8fcc690a469",
                	"status": "ACTIVE",
                	"name": "Knee Trauma,Surgery/Abdomen-General,Liver",
                	"description": "Trauma, Vascular /Abdomen/General,Liver",
                	"speciality": [
                    	{
                        	"name": "Surgery",
                        	"topic": [
                            	{
                                	"name": "Abdomen",
                                	"sub_topic": [
                                    	{
                                        	"name": "General"
                                    	},
                                    	{
                                        	"name": "Liver"
                                    	}
                                	]
                            	}
                        	]
                    	},
                    	{
                        	"name": "Trauma",
                        	"topic": []
                    	}
                	]
            	}
        	},
        	{
            	"_index": "my_custom_rank",
            	"_type": "_doc",
            	"_id": "2",
            	"_score": 1.0,
            	"_source": {
                	"id": "6321",
                	"assetCode": "CP4_VIDEO_5e8fcc690a471",
                	"status": "ACTIVE",
                	"name": "Knee,Surgery/Abdomen-General,Liver,Pancreas,Biliary, Hernia, Spleen",
                	"description": "Knee, Vascular /Abdomen/General,Liver, Pancreas,Biliary,Hernia,Spleen",
                	"speciality": [
                    	{
                                "name": "Surgery",
                               "topic": [
                            	{
                                	"name": "Abdomen",
                                	"sub_topic": [
                                    	{ "name": "General"},
                                    	{"name": "Liver"},
                                    	{"name": "Pancreas"},
                                    	{"name": "Biliary"},
                                    	{"name": "Hernia"},
                                    	{"name": "Spleen"}
                                	 ]
                                   }
                               ]
                    	},
                    	{   "name": "Trauma","topic": [] }
                    ]
                }
        	}
       ]
   }
}
