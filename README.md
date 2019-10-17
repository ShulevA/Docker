### **There are two easy ways to delete old logs in ELK.**

`1)` If you are using time series index names you can do something like:
```shell script
curl -XDELETE http://localhost:9200/index-yyyy.mm*
```
`2)` If you're not using dates in your index names you should to use Elasticsearch Curator:
```shell script
curator [--config CONFIG.YML] ACTION_FILE.YML
```