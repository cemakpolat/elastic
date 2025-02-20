```

# Get Los Angeles Flights only Flight Delay 
GET kibana_sample_data_flights/_search
{
  "query": {
    "match": {
      "OriginCityName": "Los Angeles"
    }
  }
}
# 2. Basic Query – Search for Delayed Flights

GET kibana_sample_data_flights/_search
{
"_source": ["FlightNum"],
  "query": {
    "term": {
      "FlightDelay": true
    }
  }
}
#  Get Average Ticket Price by Carrier
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "avg_ticket_price_per_carrier": {
      "terms": {
        "field": "Carrier.keyword"
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
# Find Flights Longer than 1000 Miles

GET kibana_sample_data_flights/_search
{
  "query": {
    "range": {
      "DistanceMiles": {
        "gte": 1000
      }
    }
  }
}
#Find Flights Near a Specific Location

GET kibana_sample_data_flights/_search
{
  "query": {
    "geo_distance": {
      "distance": "500km",
      "OriginLocation": {
        "lat": 34.0522,
        "lon": -118.2437
      }
    }
  }
}
#Runtime Mapping – Convert Flight Distance to Miles on the Fly
GET kibana_sample_data_flights/_search
{
  "runtime_mappings": {
    "DistanceMilesConverted": {
      "type": "double",
      "script": "emit(doc['DistanceKilometers'].value * 0.621371)"
    }
  },
  "query": {
    "match_all": {}
  },
  "fields": ["DistanceMilesConverted"]
}
# Enrich Policy – Create Carrier Metadata (Requires Setup)
PUT _enrich/policy/carrier_metadata
{
  "match": {
    "indices": "carrier_data",
    "match_field": "Carrier",
    "enrich_fields": ["CarrierType", "CarrierRating"]
  }
}
POST _enrich/policy/carrier_metadata/_execute
PUT _ingest/pipeline/enrich_flight_pipeline
{
  "processors": [
    {
      "enrich": {
        "policy_name": "carrier_metadata",
        "field": "Carrier",
        "target_field": "CarrierDetails"
      }
    }
  ]
}

#Ingestion Pipeline – Add Processing Time Field

PUT _ingest/pipeline/add_processing_time
{
  "description": "Add processing timestamp",
  "processors": [
    {
      "set": {
        "field": "processing_time",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
#Index Template – Define a Custom Mapping for Flights
PUT _index_template/flight_template
{
  "index_patterns": ["flights-*"],
  "template": {
    "mappings": {
      "properties": {
        "Carrier": { "type": "keyword" },
        "AvgTicketPrice": { "type": "float" }
      }
    }
  }
}
#12. ILM – Define a Life Cycle Policy for Flight Data
PUT _ilm/policy/flight_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "30d"
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
#13. Fuzzy Query – Search for Flight Numbers with Typos
GET kibana_sample_data_flights/_search
{
  "query": {
    "fuzzy": {
      "FlightNum": {
        "value": "AA12",
        "fuzziness": "AUTO"
      }
    }
  }
}
# Sorting Query – Get Top 10 Expensive Flights
GET kibana_sample_data_flights/_search
{
  "size": 10,
  "sort": [
    { "AvgTicketPrice": "desc" }
  ]
}
#Terms Query – Get Flights for Multiple Airlines
GET kibana_sample_data_flights/_search
{
  "query": {
    "terms": {
      "Carrier.keyword": ["Kibana Airlines", "JetBeats"]
    }
  }
}

```