
For a detailled guide you can check the official elastic website: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html

During the testing steps, you may need the command below, at least it was required in my case while testing on ubuntu 2204.

```
sudo sysctl -w vm.max_map_count=262144
```



