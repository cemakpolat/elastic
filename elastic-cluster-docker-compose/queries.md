```
GET _cat/indices

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name":"Maria Perkins"
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "order_date": {
        "gte": "now-7d/d",
        "lt": "now/d"
      }
    }
  }
}
GET kibana_sample_data_ecommerce/_mapping

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "term": {
      "currency": "EUR"
    }
  }
}
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "jeans",
      "fields": ["category.keyword", "manufacturer.keyword"]
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "Men's Clothing",
      "fields": ["category", "manufacturer"]
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "manufacturer": "Low Tide Media"
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "terms": {
      "category.keyword": ["Men's Clothing", "Men's Accessories"]
    }
  }
}




GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "avg_tax": {
      "avg": {
        "field": "taxful_total_price"
      }
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "orders_per_day": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      }
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "top_products": {
      "terms": {
        "field": "products.product_name.keyword",
        "size": 5
      },
      "aggs": {
        "total_quantity_sold": {
          "sum": {
            "field": "products.quantity"
          }
        }
      }
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_customer": {
      "terms": {
        "field": "customer_full_name.keyword",
        "size": 5
      },
      "aggs": {
        "avg_order_value": {
          "avg": {
            "field": "taxful_total_price"
          }
        }
      }
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "runtime_mappings": {
    "discounted_price": {
      "type": "double",
      "script": {
        "source": "emit(doc['products.base_price'].value * (1 - doc['products.discount_percentage'].value / 100))"
      }
    }
  },
  "query": {
    "match_all": {}
  },
  "_source": ["products.product_name", "products.base_price", "discounted_price"]
}

GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "order_date": {
        "gte": "now-30d/d"
      }
    }
  },
  "aggs": {
    "avg_sales": {
      "avg": {
        "field": "taxful_total_price"
      }
    }
  }
}

GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "avg_price_per_category": {
      "terms": {
        "field": "category.keyword",
        "size": 5
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "taxful_total_price"
          }
        }
      }
    }
  }
}

PUT _ilm/policy/ecommerce_policy
{
  "policy": {
    "phases": {
      "hot": {                     
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50GB"
          }
        }
      },
      "warm": {                    
        "min_age": "30d",
        "actions": {
          "allocate": {
            "number_of_replicas": 1
          }
        }
      },
      "cold": {                    
        "min_age": "90d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {                  
        "min_age": "180d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

DELETE kibana_sample_data_ecommerce_2
POST _reindex
{
  "source": {
    "index": "kibana_sample_data_ecommerce"
  },
  "dest": {
    "index": "kibana_sample_data_ecommerce_2"
  }
}

PUT kibana_sample_data_ecommerce_2/_settings
{
  "index": {
    "lifecycle": {
      "name": "ecommerce_policy"
    }
  }
}

PUT _enrich/policy/customer_enrichment
{
  "match": {
    "indices": "kibana_sample_data_ecommerce_2",
    "match_field": "customer_id",
    "enrich_fields": ["customer_loyalty_status"]
  }
}
POST _enrich/policy/customer_enrichment/_execute

PUT _ingest/pipeline/customer_enrichment_pipeline
{
  "processors": [
    {
      "enrich": {
        "policy_name": "customer_enrichment",
        "field": "customer_id",
        "target_field": "customer_info",
        "max_matches": 1
      }
    }
  ]
}


PUT customer_data
{
  "mappings": {
    "properties": {
      "customer_id": {
        "type": "keyword"
      },
      "customer_loyalty_status": {
        "type": "keyword"
      }
    }
  }
}

POST customer_data/_bulk
{ "index": { "_id": "1" } }
{ "customer_id": "123", "customer_loyalty_status": "Gold" }
{ "index": { "_id": "2" } }
{ "customer_id": "456", "customer_loyalty_status": "Silver" }
{ "index": { "_id": "3" } }
{ "customer_id": "789", "customer_loyalty_status": "Bronze" }

PUT _enrich/policy/customer_enrichment_2
{
  "match": {
    "indices": "customer_data",
    "match_field": "customer_id",
    "enrich_fields": ["customer_loyalty_status"]
  }
}
POST _enrich/policy/customer_enrichment_2/_execute


PUT _ingest/pipeline/customer_enrichment_pipeline_2
{
  "processors": [
    {
      "enrich": {
        "policy_name": "customer_enrichment_2",
        "field": "customer_id",
        "target_field": "Temp",
        "max_matches": 1
      }
    },
    {
      "set": {
        "field": "customer_loyalty_status",
        "value": "{{Temp.customer_loyalty_status}}"
      }
    },
    {
      "remove": {
        "field": "Temp"
      }
    }
  ]
}

POST kibana_sample_data_ecommerce_2/_doc?pipeline=customer_enrichment_pipeline_2
{
  "customer_id": "123",
  "category": "Men's Clothing",
  "taxful_total_price": 99.99
}


GET kibana_sample_data_ecommerce_2/_search
{
  "query": {
    "match": {
      "customer_id": "123"
    }
  }
}

```

